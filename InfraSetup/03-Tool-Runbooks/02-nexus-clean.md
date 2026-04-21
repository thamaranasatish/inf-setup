# Sonatype Nexus Repository 3 — Install & Operations Runbook

End-to-end install, configuration, and day-2 operations for Nexus Repository 3 on **RHEL 9** (VMware on-prem is the reference target; AWS EC2 differences are called out inline).

**Run as `root`** unless a step says otherwise.

Official docs referenced throughout:
- Install as a service: https://help.sonatype.com/repomanager3/installation/installation-methods/install-as-a-service-on-linux
- System requirements: https://help.sonatype.com/repomanager3/product-information/sonatype-nexus-repository-system-requirements
- Directories: https://help.sonatype.com/repomanager3/installation/directories
- Download: https://help.sonatype.com/repomanager3/product-information/download
- SSL/reverse proxy: https://help.sonatype.com/repomanager3/planning-your-implementation/security/configuring-ssl
- Docker repos: https://help.sonatype.com/repomanager3/formats/docker-registry
- Backup: https://help.sonatype.com/repomanager3/installation/backing-up-repository-manager
- Upgrade: https://help.sonatype.com/repomanager3/installation/upgrading

---

## Placeholders

Replace consistently everywhere below.

| Placeholder | Example | Meaning |
|---|---|---|
| `<NEXUS_VERSION>` | `3.71.0-06` | Exact version string from the download page |
| `<NEXUS_FQDN>` | `nexus.dev.contoso.internal` | Internal DNS name clients hit |
| `<NEXUS_IP>` | `10.20.30.40` | Static IP of the VM (or EC2 Elastic IP) |
| `<INTERNAL_CA_CRT>` | `/etc/pki/ca-trust/source/anchors/contoso-root-ca.crt` | Internal root CA on the VM |
| `<DOCKER_REGISTRY>` | `<NEXUS_FQDN>:5000` | Registry hostname clients use |
| `<AD_DOMAIN>` / `<AD_DC_HOST>` | `corp.contoso.com` / `dc01.corp.contoso.com` | For LDAPS integration |
| `<SMTP_RELAY>` | `smtp.contoso.internal` | SMTP relay for notifications |

---

## 1. Sizing

### 1.1 Tiers

| Tier | vCPU | RAM | `-Xmx` | `MaxDirectMemorySize` | Network | When |
|---|---|---|---|---|---|---|
| Install-validation only | 2 | 4 GB | 1 G | 1 G | 1 Gbps | Proving the install, 1 admin, no CI |
| **Small (DEV floor)** | **4** | **8 GB** | 2.7 G (default) | 2 G (default) | 1 Gbps | ≤ 20 devs, 1–2 CI agents |
| Medium (PROD default) | 8 | 16 GB | 6 G | 4 G | 10 Gbps | 20–100 devs, Docker + multi-format |
| Large | 16 | 32 GB | 16 G | 8 G | 10 Gbps | > 100 devs, > 2 TB blobs |

JVM rules once above 8 GB RAM: `-Xms` = `-Xmx`; keep JVM (`Xmx` + direct) at roughly 50 % of RAM so the OS page cache can accelerate blob reads.

> The **AWS `c7i-flex.large` (2 vCPU / 4 GB)** baseline we used for first install sits below Sonatype's minimum. It works with heap tuned to 1 G (§6.3), but plan to resize to at least `t3.large`/`m7i.large` (8 GB) before any real users or CI land on it.

### 1.2 Storage

| Mount | Size | Notes |
|---|---|---|
| `/` | 40 GB | OS |
| `/opt` | 20 GB | Nexus binaries (keep current + previous tarball) |
| `/nexus-data` | **dedicated VMDK / EBS**, 200 GB DEV / 500 GB PROD, thin + expandable | DB + blob stores |

Blob footprint forecast:

| Workload | 6 months | 24 months |
|---|---|---|
| Maven + npm + pip only | 50 GB | 150 GB |
| + Docker hosted | 200 GB | 500 GB – 1 TB |
| + Docker proxy + many teams | 500 GB – 1 TB | 2 TB+ |
| Enterprise multi-format | 1 TB | 3–5 TB |

A dedicated `/nexus-data` disk isolates growth, backup scope, and I/O from the OS disk. Enable cleanup policies (§11.5) **before** the first 100 GB accumulates.

### 1.3 Network

- 1 Gbps for DEV; 10 Gbps for PROD (a single Docker push or CI npm-install storm saturates 1 Gbps).
- Keep Nexus in the same site as the bulk of CI agents (build latency suffers beyond ~5 ms RTT).
- External ports once NGINX is in front: `443` (UI + API) and `5000` (Docker). Keep `8081/8082/8083` on `127.0.0.1`.

---

## 2. Prerequisites

```bash
cat /etc/redhat-release            # RHEL 9.2+
subscription-manager status || true
dnf repolist                       # baseos + appstream enabled

systemctl enable --now chronyd
getenforce                         # Enforcing

dnf install -y tar gzip curl wget procps-ng firewalld policycoreutils-python-utils
```

### 2.1 Java 17

Nexus 3.71+ requires **Java 17**. Two tarball choices on the download page:

| Tarball | Includes JRE? | Recommended for |
|---|---|---|
| `nexus-<ver>-unix.tar.gz` | No | Enterprise on-prem (OS-managed Java patching) |
| `nexus-<ver>-java17-unix.tar.gz` | Yes (Temurin 17) | Quick/standalone |

**If using the unbundled tarball**, install OS Java now and record `JAVA_HOME`:

```bash
dnf install -y java-17-openjdk-headless
readlink -f /usr/bin/java
JAVA_HOME_PATH="$(dirname "$(dirname "$(readlink -f /usr/bin/java)")")"
"$JAVA_HOME_PATH/bin/java" -version    # openjdk 17.0.x
```

Keep `$JAVA_HOME_PATH` — §6.2 uses it.

### 2.2 Service user

```bash
useradd -M -d /opt/sonatype/nexus -s /bin/bash nexus
id nexus
```

---

## 3. `/nexus-data` mount

### 3.1 On-prem (dedicated VMDK) — recommended

Prerequisite: second VMDK from the VMware team, surfaces as `/dev/sdb`. **Confirm with `lsblk` — never hard-code device names.**

```bash
lsblk                                 # find the empty disk (no MOUNTPOINTS)
mkfs.xfs /dev/sdb
mkdir -p /nexus-data

UUID=$(blkid -s UUID -o value /dev/sdb)
echo "UUID=${UUID}  /nexus-data  xfs  defaults,noatime  0 2" >> /etc/fstab
mount -a
mountpoint /nexus-data                # → /nexus-data is a mountpoint

chown nexus:nexus /nexus-data
chmod 750 /nexus-data
```

`noatime` speeds blob reads. Mode `750` keeps artifacts and the DB non-world-readable.

### 3.2 Cloud / no dedicated disk (test only)

```bash
mkdir -p /nexus-data
chown nexus:nexus /nexus-data
chmod 750 /nexus-data
```

Make sure the root EBS is ≥ 40 GB. Migrate to §3.1 before real use.

---

## 4. Download and extract

```bash
cd /opt && mkdir -p sonatype && cd sonatype

NEXUS_VERSION="3.71.0-06"            # use the current version from the download page
curl -fL -O "https://download.sonatype.com/nexus/3/nexus-${NEXUS_VERSION}-unix.tar.gz"
file   "nexus-${NEXUS_VERSION}-unix.tar.gz"      # must be "gzip compressed data"
tar -xzf "nexus-${NEXUS_VERSION}-unix.tar.gz"
ls                                   # nexus-<ver>/  sonatype-work/
```

Move `sonatype-work` onto the dedicated disk and symlink back:

```bash
mv /opt/sonatype/sonatype-work /nexus-data/sonatype-work
ln -s /nexus-data/sonatype-work /opt/sonatype/sonatype-work
chown -R nexus:nexus /opt/sonatype/nexus-3.* /nexus-data/sonatype-work
```

> If `tar` says "not in gzip format", `curl` saved an HTML error page. Re-check the URL — `-fL` makes the failure visible.

---

## 5. Layout reference

| Path | Owned by | Survives upgrade? | Purpose |
|---|---|---|---|
| `/opt/sonatype/nexus-<ver>/` | vendor | **No** — replaced on upgrade | Binaries |
| `/opt/sonatype/nexus-<ver>/etc/nexus-default.properties` | vendor | No | **Never edit.** Vendor defaults. |
| `/opt/sonatype/nexus-<ver>/bin/nexus.rc` | you | No (re-apply on upgrade) | `run_as_user`, JAVA override |
| `/opt/sonatype/nexus-<ver>/bin/nexus.vmoptions` | you | No (re-apply on upgrade) | JVM heap/direct-memory |
| `/nexus-data/sonatype-work/nexus3/` | you | **Yes** | DB, config, logs, blobs |
| `/nexus-data/sonatype-work/nexus3/etc/nexus.properties` | you | Yes | Application overrides (bind address, port, context) |
| `/etc/systemd/system/nexus.service` | you | Yes | systemd unit |

**Rule:** anything under `/opt/sonatype/nexus-<ver>/` is vendor-owned and replaced on upgrade. Anything under `/nexus-data/sonatype-work/nexus3/` is yours and persists.

---

## 6. Configure Nexus

### 6.1 Application overrides (optional)

Only create if you need to change defaults (`0.0.0.0:8081`, context `/`). Skip entirely for a first test.

```bash
cat > /nexus-data/sonatype-work/nexus3/etc/nexus.properties <<'EOF'
application-port=8081
application-host=0.0.0.0
nexus-context-path=/
EOF
chown nexus:nexus /nexus-data/sonatype-work/nexus3/etc/nexus.properties
chmod 640         /nexus-data/sonatype-work/nexus3/etc/nexus.properties
```

When NGINX (§9) is in front, change `application-host=127.0.0.1` and restart.

### 6.2 Run-as user and Java home

Edit `/opt/sonatype/nexus-<ver>/bin/nexus.rc`:

```bash
NEXUS_BIN=/opt/sonatype/nexus-3.71.0-06/bin
JAVA_HOME_PATH="$(dirname "$(dirname "$(readlink -f /usr/bin/java)")")"

sed -i 's|^#*run_as_user=.*|run_as_user="nexus"|' "$NEXUS_BIN/nexus.rc"
grep -q '^run_as_user=' "$NEXUS_BIN/nexus.rc" || echo 'run_as_user="nexus"' >> "$NEXUS_BIN/nexus.rc"

# Only if you used the unbundled tarball — otherwise skip:
sed -i '/^INSTALL4J_JAVA_HOME_OVERRIDE=/d' "$NEXUS_BIN/nexus.rc"
echo "INSTALL4J_JAVA_HOME_OVERRIDE=\"$JAVA_HOME_PATH\"" >> "$NEXUS_BIN/nexus.rc"

grep -E '^(run_as_user|INSTALL4J_JAVA_HOME_OVERRIDE)' "$NEXUS_BIN/nexus.rc"
```

> Must be `INSTALL4J_JAVA_HOME_OVERRIDE` — plain `JAVA_HOME` is ignored by the install4j-generated launcher and you'll get *"No suitable Java Virtual Machine could be found ... must be 17."*

### 6.3 JVM heap (only on low-RAM hosts)

Default `bin/nexus.vmoptions` uses ~2.7 GB heap + 2 GB direct memory. On a VM with < 4 GB RAM the JVM can't allocate and the service fails instantly.

```bash
free -h
# If total RAM is 2 GB (test only):
sed -i 's/^-Xms.*/-Xms1G/; s/^-Xmx.*/-Xmx1G/; s/^-XX:MaxDirectMemorySize=.*/-XX:MaxDirectMemorySize=1G/' \
  /opt/sonatype/nexus-3.71.0-06/bin/nexus.vmoptions
```

On 8 GB+ VMs leave the stock values.

---

## 7. systemd service

Unit **must** live at `/etc/systemd/system/nexus.service`. Anywhere else and `systemctl` ignores it.

```bash
NEXUS_VERSION=3.71.0-06

cat > /etc/systemd/system/nexus.service <<EOF
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/sonatype/nexus-${NEXUS_VERSION}/bin/nexus start
ExecStop=/opt/sonatype/nexus-${NEXUS_VERSION}/bin/nexus stop
User=nexus
Group=nexus
Restart=on-abort
TimeoutSec=600

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now nexus
systemctl status nexus
tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
# Wait for: "Started Sonatype Nexus OSS <version>"
```

First start takes 1–3 minutes.

---

## 8. Firewall & first login

```bash
systemctl enable --now firewalld
firewall-cmd --permanent --add-port=8081/tcp
firewall-cmd --reload

# One-time admin password (deleted automatically after the setup wizard)
cat /nexus-data/sonatype-work/nexus3/admin.password
```

Browse to `http://<NEXUS_IP>:8081/` → Sign in as `admin` → complete the wizard: set a new admin password, **disable anonymous access** unless you explicitly need unauthenticated reads.

**AWS:** also add an inbound TCP 8081 rule in the EC2 Security Group (source = your IP). Without it, `firewalld` alone isn't enough.

---

## 9. HTTPS reverse proxy (NGINX)

```bash
dnf install -y nginx
systemctl enable --now nginx
mkdir -p /etc/nginx/tls
# place /etc/nginx/tls/nexus.crt and /etc/nginx/tls/nexus.key (key: chmod 600)
```

`/etc/nginx/conf.d/nexus.conf`:

```nginx
server {
    listen 443 ssl http2;
    server_name <NEXUS_FQDN>;

    ssl_certificate     /etc/nginx/tls/nexus.crt;
    ssl_certificate_key /etc/nginx/tls/nexus.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    client_max_body_size 1G;
    proxy_read_timeout   600s;
    proxy_send_timeout   600s;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

```bash
setsebool -P httpd_can_network_connect 1
nginx -t && systemctl reload nginx

firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --remove-port=8081/tcp
firewall-cmd --reload
```

In `nexus.properties` set `application-host=127.0.0.1`, then `systemctl restart nexus`.

---

## 10. Docker registry (NGINX on :5000)

Docker clients need the registry at the root of a host, so NGINX on `5000` fronts a Nexus Docker connector.

**In the UI** (⚙️ → Repository → Repositories → Create):

| Repo | Recipe | Connector port | Other |
|---|---|---|---|
| `docker-proxy` | docker (proxy) | — | Remote: `https://registry-1.docker.io`; Docker index: Use Docker Hub |
| `docker-hosted` | docker (hosted) | `8082` | |
| `docker-group` | docker (group) | `8083` | Members: `docker-hosted`, `docker-proxy` |

**Enable realm:** Security → Realms → add **Docker Bearer Token Realm**.

`/etc/nginx/conf.d/nexus-docker.conf`:

```nginx
server {
    listen 5000 ssl http2;
    server_name <NEXUS_FQDN>;

    ssl_certificate     /etc/nginx/tls/nexus.crt;
    ssl_certificate_key /etc/nginx/tls/nexus.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    client_max_body_size 4G;
    chunked_transfer_encoding on;
    proxy_read_timeout   900s;

    location / {
        proxy_pass http://127.0.0.1:8083;
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

```bash
firewall-cmd --permanent --add-port=5000/tcp
firewall-cmd --reload
nginx -t && systemctl reload nginx
```

**Client trust for internal CA:**

```bash
sudo cp <INTERNAL_CA_CRT> /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
sudo mkdir -p /etc/docker/certs.d/<NEXUS_FQDN>:5000
sudo cp <INTERNAL_CA_CRT> /etc/docker/certs.d/<NEXUS_FQDN>:5000/ca.crt
sudo systemctl restart docker
```

**Smoke test:**

```bash
docker login <NEXUS_FQDN>:5000 -u admin
docker pull  <NEXUS_FQDN>:5000/library/alpine:3.19
docker tag   alpine:3.19 <NEXUS_FQDN>:5000/myteam/alpine:smoke
docker push  <NEXUS_FQDN>:5000/myteam/alpine:smoke
```

### AWS quick-test variant (no TLS)

If you can't set up TLS yet, expose the connectors directly and mark the registry insecure on the client. **Test only — never for shared/prod.**

```bash
firewall-cmd --permanent --add-port=8082/tcp
firewall-cmd --permanent --add-port=8083/tcp
firewall-cmd --reload
# Also add inbound TCP 8082 and 8083 in the EC2 Security Group.
```

On the client:

```json
// /etc/docker/daemon.json
{ "insecure-registries": ["<ec2-host>:8082", "<ec2-host>:8083"] }
```

```bash
sudo systemctl restart docker
docker login <ec2-host>:8083
docker pull  <ec2-host>:8083/library/alpine:3.19
docker push  <ec2-host>:8082/myteam/alpine:smoke
```

---

## 11. Repository management

A fresh install only has the default `maven-*` and `nuget*` repos. Everything else you create yourself.

### 11.1 The 3-repo pattern

For every format: **proxy** (caches upstream) + **hosted** (private uploads) + **group** (merges both). **Clients only ever hit the group URL.**

Throughout this section, `<NEXUS_URL>` = `https://<NEXUS_FQDN>` (on-prem with TLS) or `http://<ec2-host>:8081` (test).

### 11.2 Format recipes

| Format | Proxy remote URL | Notes |
|---|---|---|
| Maven Central | `https://repo1.maven.org/maven2/` | Pre-created as `maven-central`. Group `maven-public` order: releases, snapshots, central |
| npm | `https://registry.npmjs.org` | Enable **npm Bearer Token Realm** |
| PyPI | `https://pypi.org/` | |
| NuGet v3 | `https://api.nuget.org/v3/index.json` | Pre-created as `nuget.org-proxy` |
| Docker | `https://registry-1.docker.io` | See §10 for connector ports and NGINX |
| Helm | e.g. `https://charts.bitnami.com/bitnami` | |
| apt (Debian/Ubuntu) | e.g. `http://archive.ubuntu.com/ubuntu/` | Requires GPG keypair in the repo config |
| yum (RHEL/Rocky) | e.g. `https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/` | |
| Go modules | `https://proxy.golang.org` | |
| Raw | any HTTP root | For generic artifacts |

Maven release/snapshot split: two **hosted** repos, version policy Release vs Snapshot, with `Disable redeploy` on releases and `Allow redeploy` on snapshots.

### 11.3 Client configs (group URLs)

**Maven (`~/.m2/settings.xml`):**
```xml
<mirrors>
  <mirror>
    <id>nexus</id>
    <mirrorOf>*</mirrorOf>
    <url><NEXUS_URL>/repository/maven-public/</url>
  </mirror>
</mirrors>
```

Project `pom.xml` `<distributionManagement>` uses `maven-releases` / `maven-snapshots` for `mvn deploy`.

**npm (`~/.npmrc`):**
```
registry=<NEXUS_URL>/repository/npm-group/
always-auth=true
```
```bash
npm login --registry=<NEXUS_URL>/repository/npm-group/
npm publish --registry=<NEXUS_URL>/repository/npm-hosted/
```

**pip (`~/.pip/pip.conf`):**
```ini
[global]
index-url = <NEXUS_URL>/repository/pypi-group/simple/
trusted-host = <NEXUS_FQDN>        # only if TLS not trusted yet
```

**Docker:** see §10. Always use the **group** host/port for pulls and the **hosted** host/port for pushes.

### 11.4 Service users and tokens

UI → Security → Users → Create local user. Typical:

| User | Roles | Use |
|---|---|---|
| `svc-ci-deploy` | `nx-repository-view-maven2-*-*`, `nx-repository-view-docker-*-*`, `nx-apikey-all` | CI push |
| `svc-ci-read` | `nx-repository-view-*-*-read` | CI read-only |
| Humans | via AD mapping (§13) | Interactive |

Users generate a **User Token** from their profile for tools that prefer tokens (npm, PyPI, Docker).

### 11.5 Cleanup and scheduled tasks

1. Repository → **Cleanup Policies → Create** (e.g. *npm: delete if not downloaded in 60 days*), attach to each proxy repo.
2. System → **Tasks → Create task**:
   - Admin - Cleanup repositories using their associated policies → **daily**
   - Admin - Compact blob store → **weekly, off-hours**
   - Admin - Export databases for backup → **daily** (see §14)

Set cleanup policies **before** the blob store exceeds 100 GB.

---

## 12. CI integrations

### 12.1 Jenkins pipeline

Add two Jenkins credentials:
- `nexus-deploy` — username/password for `svc-ci-deploy`.
- `nexus-maven-settings` — Config File Provider with a Maven `settings.xml` referencing `${env.NEXUS_DEPLOY_PASSWORD}`.

```groovy
pipeline {
  agent any
  environment {
    REGISTRY = '<NEXUS_FQDN>:5000'
    IMAGE    = "${REGISTRY}/myteam/myapp"
  }
  stages {
    stage('Maven deploy') {
      steps {
        configFileProvider([configFile(fileId: 'nexus-maven-settings', variable: 'MVN_SETTINGS')]) {
          withCredentials([usernamePassword(credentialsId: 'nexus-deploy',
              usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_DEPLOY_PASSWORD')]) {
            sh 'mvn -s "$MVN_SETTINGS" -B clean deploy'
          }
        }
      }
    }
    stage('Docker push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-deploy',
            usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            echo "$NEXUS_PASS" | docker login "$REGISTRY" -u "$NEXUS_USER" --password-stdin
            docker build -t "$IMAGE:${BUILD_NUMBER}" .
            docker push "$IMAGE:${BUILD_NUMBER}"
          '''
        }
      }
    }
  }
}
```

### 12.2 Kubernetes image pull

```bash
kubectl -n <ns> create secret docker-registry nexus-pull \
  --docker-server=<NEXUS_FQDN>:5000 \
  --docker-username=svc-ci-deploy \
  --docker-password='<password>'
```

```yaml
spec:
  imagePullSecrets: [{ name: nexus-pull }]
  containers:
    - name: app
      image: <NEXUS_FQDN>:5000/myteam/myapp:1.2.3
```

---

## 13. Active Directory (LDAPS) & email

### 13.1 LDAPS

First, trust the AD root CA in Nexus's Java truststore:

```bash
keytool -importcert -noprompt -trustcacerts \
  -alias contoso-root-ca -file <INTERNAL_CA_CRT> \
  -keystore /opt/sonatype/nexus-<ver>/jre/lib/security/cacerts -storepass changeit
systemctl restart nexus
```

UI → Security → LDAP → Create connection:

| Field | Value |
|---|---|
| Protocol / Host / Port | `ldaps` / `<AD_DC_HOST>` / `636` |
| Search base | derived from `<AD_DOMAIN>` (e.g. `DC=corp,DC=contoso,DC=com`) |
| User ID / real name / email | `sAMAccountName` / `displayName` / `mail` |
| Group mapping | Dynamic, `memberOf` |

Map AD groups to Nexus **External** roles:

| AD group | Nexus role includes |
|---|---|
| `grp-nexus-admins` | `nx-admin` |
| `grp-nexus-developers` | `nx-repository-view-*-*-*`, `nx-apikey-all` |
| `grp-nexus-readers` | `nx-repository-view-*-*-read` |

Verify before using the UI:
```bash
dnf install -y openldap-clients
ldapsearch -H ldaps://<AD_DC_HOST>:636 \
  -D "CN=svc-nexus-ldap,OU=ServiceAccounts,DC=corp,DC=contoso,DC=com" \
  -W -b "DC=corp,DC=contoso,DC=com" "(sAMAccountName=yourtestuser)"
```

### 13.2 Email

UI → System → Email Server: Host `<SMTP_RELAY>`, port `25` or `587` (STARTTLS), from `nexus-noreply@contoso.com`, subject prefix `[NEXUS-DEV]`. Use **Verify email server** with a real recipient.

---

## 14. Backup

1. UI → System → Tasks → **Admin - Export databases for backup**, location `/nexus-data/backup`, schedule daily off-hours.
2. Back up these paths together (Veeam / rsync / EBS snapshot):
   - `/nexus-data/sonatype-work/nexus3/blobs`
   - `/nexus-data/sonatype-work/nexus3/etc`
   - `/nexus-data/backup`
3. **Always run the export task before snapshotting blobs** — DB and blobs must come from the same point in time.

---

## 15. Upgrade

```bash
systemctl stop nexus
# 1. Run the backup task + copy data (§14)

# 2. Extract new tarball alongside the old one
cd /opt/sonatype
curl -fL -O "https://download.sonatype.com/nexus/3/nexus-<new>-unix.tar.gz"
tar -xzf "nexus-<new>-unix.tar.gz"
chown -R nexus:nexus nexus-<new>

# 3. Re-apply config on the new install
sed -i 's|^#*run_as_user=.*|run_as_user="nexus"|' /opt/sonatype/nexus-<new>/bin/nexus.rc
# (also re-apply INSTALL4J_JAVA_HOME_OVERRIDE and any vmoptions tuning)

# 4. Point systemd at the new version
sed -i 's|nexus-<old>|nexus-<new>|g' /etc/systemd/system/nexus.service
systemctl daemon-reload
systemctl start nexus
tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
```

Rollback: stop, put `ExecStart`/`ExecStop` back to the old path, start. Restore from pre-upgrade backup only if release notes called out a DB schema migration.

---

## 16. Validation

```bash
systemctl is-active nexus                                         # active
ss -lntp | grep 8081
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8081/   # 200
curl -s http://localhost:8081/service/rest/v1/status              # HTTP 200, empty body
grep "Started Sonatype Nexus" /nexus-data/sonatype-work/nexus3/log/nexus.log

# After §9 (TLS)
curl -sk -o /dev/null -w "%{http_code}\n" https://<NEXUS_FQDN>/   # 200

# After §11 (Maven)
curl -sk -o /dev/null -w "%{http_code}\n" \
  https://<NEXUS_FQDN>/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom

# After §10 (Docker)
docker login <NEXUS_FQDN>:5000 -u admin
docker pull  <NEXUS_FQDN>:5000/library/alpine:3.19
```

---

## 17. Troubleshooting

### 17.1 Baseline checks (run these first)

```bash
systemctl status nexus --no-pager -l
systemctl is-active nexus
sudo ss -tlnp | grep 8081
curl -I http://localhost:8081
sudo tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
sudo journalctl -u nexus -n 200 --no-pager
```

### 17.2 Error → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| `No suitable Java Virtual Machine ... must be 17` | Wrong/missing Java, or plain `JAVA_HOME` set instead of `INSTALL4J_JAVA_HOME_OVERRIDE` | §2.1 + §6.2 |
| `Active: failed (Result: oom-kill)` | Kernel killed JVM — VM is under-sized | §17.3 |
| `code=exited, status=1/FAILURE` in < 100 ms, `nexus.log` empty | JVM couldn't allocate heap | §6.3 (drop heap) or add RAM |
| `Permission denied` in nexus.log | Ownership wrong | `chown -R nexus:nexus /opt/sonatype /nexus-data` |
| UI 502 for 1–3 min after start | Karaf still booting | Wait; watch `nexus.log` for "Started Sonatype Nexus" |
| `LISTEN 127.0.0.1:8081` only | Bound to loopback | §6.1 set `application-host=0.0.0.0`, restart |
| `tail: .../nexus.log: No such file` | Wrong install path | `sudo find / -name nexus.log 2>/dev/null` |
| `Too many open files` | `LimitNOFILE` not applied | Confirm in unit, `daemon-reload`, restart, `cat /proc/$(pgrep -f nexus)/limits` |
| `OutOfMemoryError: Direct buffer memory` (Docker push) | `MaxDirectMemorySize` too small | Raise in `nexus.vmoptions` (default 2 G → 4/8 G), restart |
| `413 Request Entity Too Large` on docker push | NGINX default 1 MB | `client_max_body_size 4G;` in server block, reload |
| `/nexus-data` full | No cleanup policies | §11.5 |
| After upgrade, service won't start | `ExecStart` still on old path | Re-edit unit, `daemon-reload`, start |

### 17.3 OOM-kill (sizing)

`Result: oom-kill` and `A process of this unit has been killed by the OOM killer` mean the VM ran out of RAM. Not a bug — under-sized.

Confirm:
```bash
free -h
sudo dmesg -T | grep -iE 'oom|killed process' | tail
```

Fix: resize to at least 8 GB (§1.1).
- **VMware:** power off → raise RAM → power on.
- **AWS:** Stop instance → Actions → Change instance type → Start. EBS data is safe; **public IP changes** unless you attach an Elastic IP.

Last-resort heap shrink on ≤ 2 GB hosts: §6.3. Nexus will be slow and may still OOM under load.

### 17.4 External reachability (`Connection refused` vs `timed out`)

| `nc -vz host 8081` result | Meaning |
|---|---|
| `succeeded` | Network fine |
| `Operation timed out` | Firewall dropping (firewalld, EC2 SG, corporate egress) |
| `Connection refused` | Something answered but nothing is listening on that port — often **wrong IP** (a recycled EC2 public IP after stop/start) or service bound to loopback |
| `No route to host` | Routing/DNS; wrong IP, VPN down |

Diagnostic order:
1. Confirm current public IP:
   ```bash
   TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
     -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
   curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
     http://169.254.169.254/latest/meta-data/public-ipv4; echo
   ```
   Empty + no Elastic IP → instance has no public IPv4 (§17.6).
2. From the VM: `curl -I http://localhost:8081` must return 200. If not, §17.1/§17.2.
3. From the VM to its own public address: `curl -I http://<public-ip>:8081`. Fails → host firewall (§17.5). Works → issue is between client and AWS.
4. From your workstation: `nc -vz <host> 8081`. Timed out → §17.5/§17.6. Refused → wrong host or loopback binding.

### 17.5 firewalld + AWS Security Group

```bash
sudo systemctl status firewalld
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload
```

Also open `443` (+ `5000` if using Docker registry). Keep `8082/8083` closed at the perimeter once NGINX (§10) is in front.

On AWS, **additionally** add a Security Group inbound rule: Console → EC2 → Instance → Security tab → edit SG → Add rule Custom TCP / port / **My IP**. Common misses: editing the wrong SG (instance can have multiple), or your own public IP changed (https://checkip.amazonaws.com).

### 17.6 EC2 public IP disappeared

After stop/start, IMDSv2 returns empty `public-ipv4` or Console shows a blank Public IPv4. Subnet isn't auto-assigning, and no Elastic IP is attached.

Fix permanently: EC2 → Elastic IPs → **Allocate Elastic IP address** → select it → **Actions → Associate Elastic IP address** → choose your instance. Then update DNS / `/etc/hosts` / SG source rules to the EIP.

### 17.7 Docker client errors

| Error | Fix |
|---|---|
| `x509: certificate signed by unknown authority` | Drop `<INTERNAL_CA_CRT>` into `/etc/docker/certs.d/<host>:5000/ca.crt`, restart docker |
| `http: server gave HTTP response to HTTPS client` | Hitting a bare connector over HTTP. Either front with TLS (§10) or add `"insecure-registries"` in `/etc/docker/daemon.json` (test only) |
| `denied: access to the requested resource is not authorized` | Missing Docker Bearer Token Realm, or user lacks role on that repo |
| `413 Request Entity Too Large` on push | NGINX `client_max_body_size 4G;`, reload |
| Push hangs then times out | NGINX `proxy_read_timeout 900s;` |

### 17.8 One-shot diagnostic

Paste into a ticket when escalating:

```bash
echo "=== systemd ===";            systemctl status nexus --no-pager -l | head -30
echo "=== listening ===";          sudo ss -tlnp | grep -E ':(8081|8082|8083|5000|443)\b' || echo none
echo "=== local HTTP ===";         curl -sS -I http://localhost:8081 | head -5 || true
echo "=== firewall ===";           sudo firewall-cmd --list-all 2>/dev/null | grep -E 'services|ports' || echo "firewalld inactive"
echo "=== memory ===";             free -h
echo "=== nexus.log tail ===";     sudo tail -n 50 /nexus-data/sonatype-work/nexus3/log/nexus.log 2>/dev/null \
                                    || sudo tail -n 50 /opt/sonatype/sonatype-work/nexus3/log/nexus.log 2>/dev/null \
                                    || echo "nexus.log not found"
echo "=== journalctl ===";         sudo journalctl -u nexus -n 50 --no-pager
```

---

## 18. Day-2 cheat sheet

| Task | Command / Location |
|---|---|
| Start / stop / status | `systemctl {start,stop,status} nexus` |
| Logs | `/nexus-data/sonatype-work/nexus3/log/nexus.log` · `journalctl -u nexus` · `/var/log/nginx/*` |
| Editable config | `bin/nexus.rc` · `bin/nexus.vmoptions` · `sonatype-work/nexus3/etc/nexus.properties` |
| Health endpoint | `GET /service/rest/v1/status` |
| Repo admin via API | `POST /service/rest/v1/repositories/<format>/<type>` |
| Reset admin password | https://support.sonatype.com/hc/en-us/articles/213467158 |

---

## Sign-off checklist

- [ ] §2 prerequisites (Java 17, nexus user) done
- [ ] §3 `/nexus-data` mounted, owned by `nexus`
- [ ] §4 tarball extracted, `sonatype-work` on dedicated disk
- [ ] §6 `run_as_user`, JAVA override, vmoptions correct for RAM
- [ ] §7 service enabled, survives reboot, log shows "Started Sonatype Nexus"
- [ ] §8 first login done, admin password rotated, anonymous disabled
- [ ] §9 TLS on `<NEXUS_FQDN>` (prod)
- [ ] §10 Docker registry login/pull/push verified (if used)
- [ ] §11 group repos created for every format in use; cleanup policies attached
- [ ] §12 CI pushes/pulls via service user
- [ ] §13 AD login + role mapping verified (prod)
- [ ] §14 backup task scheduled, first backup restored successfully in test
- [ ] §16 validations all pass
