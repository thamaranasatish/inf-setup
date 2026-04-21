# 02 — Sonatype Nexus Repository 3 — Working Install on RHEL 9 (VMware on-prem)

> **Scope:** A real, end-to-end install that works on a freshly provisioned RHEL 9 VM on VMware vSphere.
> **Source of truth:** Sonatype official docs (linked inline). No assumed patterns.
> **Run as `root`** unless a step explicitly says `su - nexus` or `sudo -u nexus`.

---

## 0. Official references (read these first)

- Install as a service on Linux: https://help.sonatype.com/repomanager3/installation/installation-methods/install-as-a-service-on-linux
- System requirements: https://help.sonatype.com/repomanager3/product-information/sonatype-nexus-repository-system-requirements
- Directories: https://help.sonatype.com/repomanager3/installation/directories
- Run as a service: https://help.sonatype.com/repomanager3/installation/run-as-a-service
- Download page: https://help.sonatype.com/repomanager3/product-information/download
- SSL / reverse proxy: https://help.sonatype.com/repomanager3/planning-your-implementation/security/configuring-ssl
- Docker repositories: https://help.sonatype.com/repomanager3/formats/docker-registry

This runbook follows those pages step-for-step. Anything extra (firewall, NGINX, TLS) is marked as "enterprise add-on" and clearly separated.

---

## Placeholders — replace these with your real values

Every `<PLACEHOLDER>` below appears throughout this runbook. Replace them consistently (a simple `sed` over your working copy works) before you run anything.

| Placeholder | Example used in this runbook | What it is / how to choose |
|---|---|---|
| `<NEXUS_FQDN>` | `nexus.dev.contoso.internal` | The internal DNS name users/CI hit. `<service>.<env>.<company-domain>.<private-TLD>`. Ask your network team for the exact internal domain (often `.internal`, `.corp`, `.local`, or your real company DNS zone like `tools.contoso.com`). |
| `<NEXUS_IP>` | `10.20.30.40` | Static IP of the Nexus VM on your internal network. Assigned by your network/VMware team. |
| `<COMPANY_DOMAIN>` | `contoso.internal` | Your internal DNS zone. Drop `nexus.dev.` to get it. |
| `<AD_DOMAIN>` | `corp.contoso.com` | Your Active Directory DNS domain (usually different from the server hostname domain). |
| `<AD_DC_HOST>` | `dc01.corp.contoso.com` | A domain controller hostname for LDAPS. |
| `<SMTP_RELAY>` | `smtp.contoso.internal` | Internal SMTP relay for Nexus email notifications. |
| `<INTERNAL_CA_CRT>` | `/etc/pki/ca-trust/source/anchors/contoso-root-ca.crt` | Path on the VM where the internal root CA certificate lives. |
| `<DOCKER_REGISTRY>` | `nexus.dev.contoso.internal:5000` | Same FQDN as Nexus, port 5000 (from §12). |
| `<JENKINS_URL>` | `https://jenkins.dev.contoso.internal` | Internal Jenkins URL. |
| `<PROJECT_GROUP_ID>` | `com.contoso.myteam` | Maven `groupId` prefix your team owns. |
| `<DOCKER_NAMESPACE>` | `myteam` | The first path segment you push images under, e.g. `<DOCKER_REGISTRY>/myteam/app:tag`. |

> **Why `.internal`?** It's a conventional private DNS suffix meaning "this name only resolves inside the company network, never on the public internet." Your company may use something else — `.corp`, `.local`, `.lan`, or a real owned domain like `tools.contoso.com`. Whatever your network team uses, put it in `<NEXUS_FQDN>` and be consistent.

A full realistic example, resolved:

```
<NEXUS_FQDN>           = nexus.dev.contoso.internal
<NEXUS_IP>             = 10.20.30.40
<DOCKER_REGISTRY>      = nexus.dev.contoso.internal:5000
Maven group repo URL   = https://nexus.dev.contoso.internal/repository/maven-public/
Docker image example   = nexus.dev.contoso.internal:5000/myteam/myapp:1.2.3
```

---

## 1. VM sizing (Sonatype minimums → what we actually use)

Sonatype minimums for a small/medium instance: 4 vCPU, 8 GB RAM, disk proportional to artifact volume.

For a DEV instance serving ~30 users:

| Resource | Value |
|---|---|
| vCPU | 4 |
| RAM | 8 GB |
| `/` | 40 GB |
| `/opt` | 20 GB (binaries) |
| `/nexus-data` | dedicated disk, start 200 GB, expandable (holds DB + blobs) |
| OS | RHEL 9 (9.2+) |

**Why a dedicated `/nexus-data` disk:** Sonatype stores DB, config, logs, and blob stores under one `sonatype-work/nexus3` directory. Keeping it on its own VMDK lets you grow it without touching the OS disk.

---

## 2. Prerequisites on the VM

```bash
# RHEL version
cat /etc/redhat-release     # Red Hat Enterprise Linux release 9.x

# Subscription / repos must work (needed for dnf install)
subscription-manager status || true
dnf repolist                # at least rhel-9-for-x86_64-baseos/-appstream enabled

# Time sync
systemctl enable --now chronyd
chronyc tracking

# SELinux stays Enforcing
getenforce                  # Enforcing

# Base packages we will actually use
dnf install -y tar gzip curl wget procps-ng firewalld policycoreutils-python-utils
```

**Java:** Nexus Repository 3 ships with a bundled JRE in the unix tarball. You do **not** need to install OpenJDK separately for the supported tarball distribution. Source: https://help.sonatype.com/repomanager3/installation/system-requirements

---

## 3. Create the `nexus` service user

Official doc: https://help.sonatype.com/repomanager3/installation/run-as-a-service

```bash
useradd -M -d /opt/sonatype/nexus -s /bin/bash nexus
# -M = no home dir created
# shell is /bin/bash because the nexus start script needs a real shell
```

Verify:
```bash
id nexus
# uid=1001(nexus) gid=1001(nexus) groups=1001(nexus)
```

---

## 4. Create and mount `/nexus-data`

### Why a dedicated disk on-prem

`/nexus-data` is where Nexus stores **everything stateful** — the embedded database, configuration, logs, and blob stores (the actual JARs/images/packages). Over time this grows from MBs to hundreds of GBs or TBs. Keeping it on its own disk/LUN gives you:

| Benefit | What it buys you |
|---|---|
| **Growth without touching the OS disk** | Add a new VMDK or expand the existing one in vSphere, run `xfs_growfs /nexus-data` — the root filesystem never moves. |
| **Independent backup/snapshot** | vSphere snapshots, storage-array snapshots, or Veeam jobs can target just `/nexus-data`. |
| **Blast radius control** | A full blob store (`/nexus-data` at 100%) doesn't take down the OS — you just lose Nexus, not SSH/systemd/logging. |
| **Better I/O** | The storage team can place this LUN on faster/tiered storage (SSD tier, thin-provisioned, dedup'd) independently from the OS disk. |
| **Ownership hygiene** | The whole mount is owned by `nexus:nexus`; root stays owned by `root:root`. No mixed ownership inside `/`. |

### 4a. On-prem (VMware vSphere) — with a dedicated disk

Prerequisite: ask the VMware team to add a **second VMDK** to the VM (200 GB+ to start, thin-provisioned, on the appropriate datastore). It appears to Linux as `/dev/sdb` (SCSI). Always confirm with `lsblk` first — never hard-code the device name.

```bash
# 1. Find the new disk. Look for a disk with no partitions / no MOUNTPOINT.
lsblk
# NAME   SIZE MOUNTPOINTS
# sda     60G             ← OS disk, has partitions
# sdb    200G             ← NEW disk, empty. Use this.

# 2. Make a filesystem (XFS is the RHEL 9 default and recommended for large FS)
mkfs.xfs /dev/sdb

# 3. Create the mountpoint directory
mkdir -p /nexus-data

# 4. Add a persistent mount entry (by UUID so it survives device renames)
UUID=$(blkid -s UUID -o value /dev/sdb)
echo "UUID=${UUID}  /nexus-data  xfs  defaults,noatime  0 2" >> /etc/fstab

# 5. Mount it now (also validates fstab syntax — if this fails, reboot will fail too)
mount -a
mountpoint /nexus-data
# Expected: /nexus-data is a mountpoint

# 6. Set ownership so the nexus user can write everywhere under it
chown nexus:nexus /nexus-data
chmod 750 /nexus-data
```

**What those fstab options do:**

| Option | Why |
|---|---|
| `defaults` | Standard options (rw, suid, dev, exec, auto, nouser, async). |
| `noatime` | Don't update file access timestamps — large speedup on blob stores with many small files. |
| `0` (dump) | Dump backup not used. |
| `2` (fsck pass) | Checked after root (pass 1). Set `0` if you use journaling with no auto-fsck. |

**Permissions rationale:**
- Owner `nexus:nexus` — the `nexus` service user must read/write freely.
- Mode `750` — owner full, group read+traverse, **world no access**. Artifact bytes and the embedded DB shouldn't be world-readable.
- The Nexus process (running as `nexus` per §6) creates subdirs under this mount with its own umask; they will all be `nexus:nexus` automatically.

### 4b. AWS / any cloud without a second disk (testing)

If you're just proving the install works and haven't attached a second EBS volume, use a plain directory on the root filesystem:

```bash
mkdir -p /nexus-data
chown nexus:nexus /nexus-data
chmod 750 /nexus-data
```

Skip the `mkfs.xfs`, `blkid`, `fstab`, and `mount -a` steps — there's no separate block device to format.

**Make sure the root disk is large enough** (at least 40 GB for a test instance; more if you'll push real Docker images). Everything else in this runbook works unchanged.

> When you later move this install to on-prem (or production AWS), come back to §4a and do it properly: attach a dedicated volume, move the contents of `/nexus-data` onto it, and remount. The rest of the install doesn't need to change.

---

## 5. Download and extract Nexus (official tarball)

Official download page: https://help.sonatype.com/repomanager3/product-information/download

```bash
cd /opt
mkdir -p sonatype
cd sonatype

# The "latest" redirect always points at the current stable OSS tarball.
# You can also pin to a specific version URL shown on the download page.
curl -L -O https://download.sonatype.com/nexus/3/nexus-unix-x86-64.tar.gz

tar -xzf nexus-unix-x86-64.tar.gz
ls
# nexus-3.x.y-z/      sonatype-work/
```

This matches the official procedure: extract into `/opt/sonatype` producing two sibling directories — `nexus-3.x.y-z` (binaries) and `sonatype-work` (data).

Move `sonatype-work` onto the dedicated disk and link it back:

```bash
mv /opt/sonatype/sonatype-work /nexus-data/sonatype-work
ln -s /nexus-data/sonatype-work /opt/sonatype/sonatype-work

chown -R nexus:nexus /opt/sonatype/nexus-3.* /nexus-data/sonatype-work
```

---

## 6. Configure run-as user

File: `/opt/sonatype/nexus-3.x.y-z/bin/nexus.rc`

```bash
# Uncomment and set
run_as_user="nexus"
```

Exactly as documented: https://help.sonatype.com/repomanager3/installation/run-as-a-service

---

## 7. Configure listen address and data directory (optional but recommended)

File: `/opt/sonatype/sonatype-work/nexus3/etc/nexus.properties`
(Create it if it doesn't exist. Defaults live in the binary's `etc/nexus-default.properties` — do not edit that file.)

```properties
application-port=8081
application-host=0.0.0.0
nexus-context-path=/
```

Leave `application-host=0.0.0.0` for now so you can reach the UI directly. If you add NGINX (§11), change it to `127.0.0.1`.

---

## 8. Register as a systemd service

Official doc: https://help.sonatype.com/repomanager3/installation/run-as-a-service

Create `/etc/systemd/system/nexus.service`:

```ini
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/sonatype/nexus-3.x.y-z/bin/nexus start
ExecStop=/opt/sonatype/nexus-3.x.y-z/bin/nexus stop
User=nexus
Group=nexus
Restart=on-abort
TimeoutSec=600

[Install]
WantedBy=multi-user.target
```

Replace `nexus-3.x.y-z` with the actual extracted directory name.

```bash
systemctl daemon-reload
systemctl enable --now nexus
systemctl status nexus
```

First start takes 1–3 minutes while Karaf initializes. Tail the log:

```bash
tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
# Look for:  "Started Sonatype Nexus OSS <version>"
```

---

## 9. Open the firewall

```bash
systemctl enable --now firewalld
firewall-cmd --permanent --add-port=8081/tcp
firewall-cmd --reload
firewall-cmd --list-all
```

---

## 10. First login

```bash
# One-time generated admin password (created during first boot)
cat /nexus-data/sonatype-work/nexus3/admin.password
```

Open `http://<vm-ip>:8081/` in a browser:

1. Click **Sign in** (top right), user = `admin`, password = contents of the file above.
2. The **setup wizard** walks you through:
   - Setting a new admin password (store it in your password manager/vault).
   - Anonymous access — for a DEV/corporate instance, **disable** it and enable later only if unauthenticated reads are needed.
3. After the wizard completes, the `admin.password` file is deleted automatically.

Reference: https://help.sonatype.com/repomanager3/installation/complete-installation

---

## 11. (Enterprise add-on) NGINX TLS reverse proxy

If you need HTTPS on a company FQDN, put NGINX in front of Nexus on the same host. Reference: https://help.sonatype.com/repomanager3/planning-your-implementation/security/configuring-ssl

```bash
dnf install -y nginx
systemctl enable --now nginx

# Place your cert+key from the internal CA (or a temporary self-signed for testing)
mkdir -p /etc/nginx/tls
# Put: /etc/nginx/tls/nexus.crt, /etc/nginx/tls/nexus.key  (chmod 600 on the key)
```

`/etc/nginx/conf.d/nexus.conf`:

```nginx
server {
    listen 443 ssl http2;
    server_name nexus.dev.contoso.internal;   # <NEXUS_FQDN>

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

Then:

```bash
# Allow nginx to reach local 8081
setsebool -P httpd_can_network_connect 1

nginx -t && systemctl reload nginx

firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --remove-port=8081/tcp      # close direct access now
firewall-cmd --reload
```

Change `application-host=127.0.0.1` in `nexus.properties` and restart Nexus.

---

## 12. (Enterprise add-on) Docker registry

Official doc: https://help.sonatype.com/repomanager3/formats/docker-registry

Docker clients require the registry to be at the **root** of a host, which is why a second NGINX server block on a different port (commonly `5000`) is used to forward to a Nexus **Docker connector**.

1. In the Nexus UI: **Administration → Repository → Repositories → Create repository**
   - `docker (hosted)` → name `docker-hosted`, HTTP connector port `8082`.
   - `docker (proxy)` → name `docker-proxy`, remote storage `https://registry-1.docker.io`, Docker index = Use Docker Hub.
   - `docker (group)` → name `docker-all`, HTTP connector port `8083`, members `[docker-hosted, docker-proxy]`.

2. NGINX server block (`/etc/nginx/conf.d/nexus-docker.conf`):

```nginx
server {
    listen 5000 ssl http2;
    server_name nexus.dev.contoso.internal;   # <NEXUS_FQDN>

    ssl_certificate     /etc/nginx/tls/nexus.crt;
    ssl_certificate_key /etc/nginx/tls/nexus.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    client_max_body_size 4G;
    chunked_transfer_encoding on;
    proxy_read_timeout   900s;

    location / {
        proxy_pass http://127.0.0.1:8083;       # docker-all group connector
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

3. In the Nexus UI: **Security → Realms** → add **Docker Bearer Token Realm** to the active realms list (required for `docker login`).

4. Test from a client:
```bash
# <DOCKER_REGISTRY> = nexus.dev.contoso.internal:5000
docker login nexus.dev.contoso.internal:5000 -u admin
docker pull  nexus.dev.contoso.internal:5000/library/alpine:3.19     # via docker-all
docker tag   alpine:3.19 nexus.dev.contoso.internal:5000/myteam/alpine:smoke
docker push  nexus.dev.contoso.internal:5000/myteam/alpine:smoke
```

If you see `x509: certificate signed by unknown authority`, drop your internal root CA into `/etc/docker/certs.d/nexus.dev.contoso.internal:5000/ca.crt` on the client and restart docker.

---

## 13. Create baseline proxy / hosted / group repositories

UI: **Administration → Repository → Repositories → Create repository**. Pick the recipe that matches the format:

| Name | Recipe | Remote URL / notes |
|---|---|---|
| `maven-central` | `maven2 (proxy)` | `https://repo1.maven.org/maven2/` |
| `maven-releases` | `maven2 (hosted)` | Version policy: **Release**, deployment policy: **Disable redeploy** |
| `maven-snapshots` | `maven2 (hosted)` | Version policy: **Snapshot**, deployment policy: **Allow redeploy** |
| `maven-public` | `maven2 (group)` | Members (order matters): `maven-releases`, `maven-snapshots`, `maven-central` |
| `npm-proxy` | `npm (proxy)` | `https://registry.npmjs.org` |
| `npm-hosted` | `npm (hosted)` | |
| `npm-all` | `npm (group)` | Members: `npm-hosted`, `npm-proxy` |
| `pypi-proxy` | `pypi (proxy)` | `https://pypi.org` |

**Client rule:** builds only ever talk to the **group** URL (`maven-public`, `npm-all`). Never point a build at a single proxy or hosted repo — the group resolves them in order.

---

## 14. Integration Steps (with placeholders)

Each integration below follows the same shape: **(a) what to create in Nexus**, **(b) what to paste on the consumer**, **(c) how to verify**. Replace every `<PLACEHOLDER>` from the table at the top of this runbook.

### 14.1 Maven / Gradle builds on developer machines and Jenkins agents

**(a) In Nexus**
- Repos from §13 are enough. No extra config needed.
- Create a deployment user (UI → **Security → Users → Create**): `svc-ci-deploy`, roles = `nx-repository-view-maven2-*-*` + `nx-apikey-all` (so it can push via the REST API). Store password in your secret store.

**(b) On each Maven client**

`~/.m2/settings.xml`:
```xml
<settings>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>svc-ci-deploy</username>
      <password>${env.NEXUS_DEPLOY_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>svc-ci-deploy</username>
      <password>${env.NEXUS_DEPLOY_PASSWORD}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <!-- All dependency resolution goes through the group -->
      <url>https://nexus.dev.contoso.internal/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```

In each project's `pom.xml` (for `mvn deploy`):
```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>https://nexus.dev.contoso.internal/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>https://nexus.dev.contoso.internal/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

**(c) Verify**
```bash
# Resolution (no auth needed for group reads once anonymous is allowed, else use -s with creds)
mvn -s ~/.m2/settings.xml dependency:get \
  -Dartifact=org.apache.commons:commons-lang3:3.14.0

# Deploy a snapshot
mvn -s ~/.m2/settings.xml deploy
# Expect: "Uploaded to nexus-snapshots: https://nexus.dev.contoso.internal/.../.../<artifact>.jar"
```

---

### 14.2 npm clients (devs + CI)

**(a) In Nexus**
- `npm-all` group from §13 is the single URL consumers use.

**(b) On each client**

`~/.npmrc`:
```
registry=https://nexus.dev.contoso.internal/repository/npm-all/
# If your group requires auth (recommended), add the line below after running
#   npm login --registry=https://nexus.dev.contoso.internal/repository/npm-all/
# //nexus.dev.contoso.internal/repository/npm-all/:_authToken=NpmToken.xxxxxxxx
```

**(c) Verify**
```bash
npm view express version --registry=https://nexus.dev.contoso.internal/repository/npm-all/
# Expect: a version string like 4.19.2
```

---

### 14.3 Python / pip clients

**(a) In Nexus**
- `pypi-proxy` from §13.

**(b) On each client**

`~/.pip/pip.conf` (Linux/macOS) or `%APPDATA%\pip\pip.ini` (Windows):
```ini
[global]
index-url = https://nexus.dev.contoso.internal/repository/pypi-proxy/simple/
trusted-host = nexus.dev.contoso.internal
```

**(c) Verify**
```bash
pip install --dry-run requests
# Expect: "Looking in indexes: https://nexus.dev.contoso.internal/repository/pypi-proxy/simple/"
```

---

### 14.4 Docker clients (devs + Jenkins agents + Kubernetes nodes)

Requires the Docker registry from §12 to be live.

**(a) In Nexus**
- `docker-hosted`, `docker-proxy`, `docker-all` from §12.
- Active realms include **Docker Bearer Token Realm**.

**(b) On each client (one-time host trust setup)**

```bash
# Trust the internal CA at the OS level
sudo cp <INTERNAL_CA_CRT> /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Docker-specific trust (required if registry uses internal CA)
sudo mkdir -p /etc/docker/certs.d/nexus.dev.contoso.internal:5000
sudo cp <INTERNAL_CA_CRT> /etc/docker/certs.d/nexus.dev.contoso.internal:5000/ca.crt
sudo systemctl restart docker
```

**(c) Use it**
```bash
# Login (uses the Nexus user's creds)
docker login nexus.dev.contoso.internal:5000 -u svc-ci-deploy

# Pull from Docker Hub through the proxy
docker pull nexus.dev.contoso.internal:5000/library/alpine:3.19

# Push a team image
docker tag  myapp:1.2.3  nexus.dev.contoso.internal:5000/myteam/myapp:1.2.3
docker push nexus.dev.contoso.internal:5000/myteam/myapp:1.2.3
```

**(d) In Kubernetes (pulling private images)**

```bash
kubectl -n <my-namespace> create secret docker-registry nexus-pull \
  --docker-server=nexus.dev.contoso.internal:5000 \
  --docker-username=svc-ci-deploy \
  --docker-password='<password>' \
  --docker-email=devops@contoso.com
```

```yaml
# in your Deployment / Pod spec
spec:
  imagePullSecrets:
    - name: nexus-pull
  containers:
    - name: app
      image: nexus.dev.contoso.internal:5000/myteam/myapp:1.2.3
```

---

### 14.5 Jenkins (declarative pipeline example)

**(a) In Nexus** — create `svc-jenkins-deploy` user with roles:
- `nx-repository-view-maven2-*-*` (for maven push)
- `nx-repository-view-docker-*-*` (for docker push)
- `nx-apikey-all`

**(b) In Jenkins** — add two credentials:
- `nexus-deploy` → Username/password for `svc-jenkins-deploy`.
- `nexus-maven-settings` → **Config File Provider** plugin, Maven settings file content = the `settings.xml` from §14.1 (with `${env.NEXUS_DEPLOY_PASSWORD}` resolving from `withCredentials`).

**(c) Pipeline**

```groovy
pipeline {
  agent any
  environment {
    REGISTRY   = 'nexus.dev.contoso.internal:5000'   // <DOCKER_REGISTRY>
    IMAGE      = "${REGISTRY}/myteam/myapp"          // <DOCKER_NAMESPACE>
    MVN_GROUP  = 'com.contoso.myteam'                // <PROJECT_GROUP_ID>
  }
  stages {
    stage('Build & deploy JAR to Nexus') {
      steps {
        configFileProvider([configFile(fileId: 'nexus-maven-settings', variable: 'MVN_SETTINGS')]) {
          withCredentials([usernamePassword(
              credentialsId: 'nexus-deploy',
              usernameVariable: 'NEXUS_USER',
              passwordVariable: 'NEXUS_DEPLOY_PASSWORD')]) {
            sh 'mvn -s "$MVN_SETTINGS" -B clean deploy'
          }
        }
      }
    }
    stage('Build & push Docker image') {
      steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-deploy',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS')]) {
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

**(d) Verify** — the job should produce a `com.contoso.myteam:myapp:<version>` artifact in `maven-releases` (or `-snapshots`) and a new tag in `docker-hosted`.

---

### 14.6 Active Directory / LDAPS single sign-on

**(a) In Nexus** — UI: **Security → LDAP → Create connection**.

| Field | Value |
|---|---|
| Name | `contoso-ad` |
| Protocol | `ldaps` |
| Host | `dc01.corp.contoso.com`  (`<AD_DC_HOST>`) |
| Port | `636` |
| Search base | `DC=corp,DC=contoso,DC=com`  (derived from `<AD_DOMAIN>`) |
| Authentication method | Simple |
| Username | `CN=svc-nexus-ldap,OU=ServiceAccounts,DC=corp,DC=contoso,DC=com` |
| Password | from your vault |
| User subtree | checked |
| User object class | `user` |
| User ID attribute | `sAMAccountName` |
| Real name attribute | `displayName` |
| Email attribute | `mail` |
| Group mapping | **Dynamic groups**, member-of attribute `memberOf` |

Prerequisite: the AD root CA must be in the VM's Java trust store:
```bash
keytool -importcert -noprompt -trustcacerts \
  -alias contoso-root-ca \
  -file <INTERNAL_CA_CRT> \
  -keystore /opt/sonatype/nexus-3.x.y-z/jre/lib/security/cacerts \
  -storepass changeit
systemctl restart nexus
```

**(b) Map AD groups to Nexus roles** — UI: **Security → Roles → Create role**, role type **External**, source `LDAP`. Example mapping:

| AD group (mapped ID) | Nexus role | Built-in roles included |
|---|---|---|
| `grp-nexus-admins` | `nexus-admins-role` | `nx-admin` |
| `grp-nexus-developers` | `nexus-devs-role` | `nx-repository-view-*-*-*` (read+edit on non-release), `nx-apikey-all` |
| `grp-nexus-readers` | `nexus-readers-role` | `nx-repository-view-*-*-read` |

**(c) Verify**
```bash
# On the Nexus VM, test LDAPS reachability before using it in the UI:
dnf install -y openldap-clients
ldapsearch -H ldaps://dc01.corp.contoso.com:636 \
  -D "CN=svc-nexus-ldap,OU=ServiceAccounts,DC=corp,DC=contoso,DC=com" \
  -W -b "DC=corp,DC=contoso,DC=com" "(sAMAccountName=yourtestuser)"
```
Then log in to the Nexus UI as an AD user — the first login auto-creates the mapping.

---

### 14.7 Email notifications (optional)

UI → **System → Email Server**:

| Field | Value |
|---|---|
| Host | `smtp.contoso.internal`  (`<SMTP_RELAY>`) |
| Port | `25` (or `587` with STARTTLS) |
| From address | `nexus-noreply@contoso.com` |
| Subject prefix | `[NEXUS-DEV]` |

Click **Verify email server** with a real recipient.

---

### 14.8 Internal DNS — planning and cutover

Before any of the above is addressable by FQDN you need a DNS record. Coordinate with your network team:

1. **Forward record (A):** `nexus.dev.contoso.internal` → `<NEXUS_IP>` in the internal DNS zone.
2. **Reverse record (PTR):** recommended for logs to resolve cleanly.
3. **TTL:** set to **60 s** during cutover week so rollback is fast; raise to 1 h after.

**Temporary workaround** before DNS exists — add a line to `/etc/hosts` on the Nexus VM and on any admin workstation:
```
10.20.30.40  nexus.dev.contoso.internal
```
This lets you bring up TLS (§11) and test everything end-to-end before the DNS record goes live.

---

## 15. Backup

Official: https://help.sonatype.com/repomanager3/installation/backing-up-repository-manager

1. In UI: **Administration → System → Tasks → Create task**
   - Type: **Admin - Export databases for backup**
   - Backup location: `/nexus-data/backup`
   - Schedule: daily, off-hours.
2. Back up these paths with your standard tool (Veeam, rsync to NFS, etc.):
   - `/nexus-data/sonatype-work/nexus3/blobs`
   - `/nexus-data/backup` (from step 1)
   - `/nexus-data/sonatype-work/nexus3/etc`
3. **Rule:** always run the export task before copying blobs — blobs and DB must be captured from the same point in time.

---

## 16. Upgrade procedure (official)

Official: https://help.sonatype.com/repomanager3/installation/upgrading

```bash
# 1. Stop
systemctl stop nexus

# 2. Back up (as in §15)

# 3. Extract new tarball alongside the old one
cd /opt/sonatype
curl -L -O https://download.sonatype.com/nexus/3/nexus-unix-x86-64.tar.gz
tar -xzf nexus-unix-x86-64.tar.gz
chown -R nexus:nexus nexus-3.<new-version>

# 4. Re-apply run_as_user in the NEW bin/nexus.rc
sed -i 's/^#run_as_user=.*/run_as_user="nexus"/' \
  /opt/sonatype/nexus-3.<new-version>/bin/nexus.rc

# 5. Update systemd unit ExecStart/ExecStop to the new path
sed -i 's|nexus-3.<old-version>|nexus-3.<new-version>|g' /etc/systemd/system/nexus.service
systemctl daemon-reload

# 6. Start and watch the log
systemctl start nexus
tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
```

Data lives in `/nexus-data/sonatype-work` and is read by the new binary directly — no data migration needed for normal minor upgrades.

Rollback: stop service, put `ExecStart`/`ExecStop` back to the old path, start. If DB schema was upgraded (rare, always called out in the release notes), restore from the pre-upgrade backup.

---

## 17. Validation (run in order)

```bash
# 1. Service up
systemctl is-active nexus                         # active

# 2. Port listening
ss -lntp | grep 8081                              # nexus listening

# 3. HTTP status
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8081/    # 200

# 4. REST status endpoint
curl -s http://localhost:8081/service/rest/v1/status               # empty body, HTTP 200 = healthy

# 5. Log shows "Started Sonatype Nexus"
grep "Started Sonatype Nexus" /nexus-data/sonatype-work/nexus3/log/nexus.log

# 6. (after §11) HTTPS
curl -sk -o /dev/null -w "%{http_code}\n" https://nexus.dev.contoso.internal/   # 200

# 7. (after §13) Maven proxy pulls through
curl -sk -o /dev/null -w "%{http_code}\n" \
  https://nexus.dev.contoso.internal/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom
# 200 (first hit may take a few seconds)

# 8. (after §12) Docker
docker login nexus.dev.contoso.internal:5000 -u admin
docker pull  nexus.dev.contoso.internal:5000/library/alpine:3.19
```

---

## 18. Common real failures and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `systemctl start nexus` fails, log says "Unable to update instance pid" | Running as wrong user | Ensure `run_as_user="nexus"` in `bin/nexus.rc` **and** `User=nexus` in the systemd unit; `chown -R nexus:nexus` on the install dir and `sonatype-work`. |
| UI 502 for 1–3 minutes after start | Karaf still booting | Wait. Watch `nexus.log`. |
| `java.lang.OutOfMemoryError: Direct buffer memory` during Docker push | Direct memory too small | Edit `bin/nexus.vmoptions`: raise `-XX:MaxDirectMemorySize` (default 2G → 4G or 8G if RAM allows). Restart. |
| `Too many open files` | `LimitNOFILE` not applied | Confirm `LimitNOFILE=65536` in the unit; `systemctl daemon-reload && systemctl restart nexus`; verify with `cat /proc/$(pgrep -f nexus)/limits`. |
| Docker `x509: certificate signed by unknown authority` | Internal CA not trusted on the client | `/etc/docker/certs.d/<fqdn>:5000/ca.crt`, restart docker. |
| `413 Request Entity Too Large` on docker push | NGINX default 1 MB body limit | `client_max_body_size 4G;` in the NGINX server block, reload. |
| `/nexus-data` fills up | No cleanup policies | UI: **Repository → Cleanup Policies**, assign to repos, run **Admin - Cleanup repositories using their associated policies** and **Admin - Compact blob store**. |
| After upgrade, service won't start | `ExecStart` still pointing at old dir | Update unit to new version path, `daemon-reload`, start. |

---

## 19. Day-2 cheat sheet

| Task | Command / Location |
|---|---|
| Start / stop / status | `systemctl {start,stop,status} nexus` |
| Logs | `/nexus-data/sonatype-work/nexus3/log/nexus.log` · `journalctl -u nexus` · `/var/log/nginx/*` |
| Config files you can edit | `bin/nexus.rc` · `bin/nexus.vmoptions` · `sonatype-work/nexus3/etc/nexus.properties` |
| Reset admin password (locked out) | Follow Sonatype KB: https://support.sonatype.com/hc/en-us/articles/213467158 |
| Health endpoint | `GET /service/rest/v1/status` |
| Repo admin via API | `POST /service/rest/v1/repositories/<format>/<type>` (REST API reference in Sonatype docs) |

---

## Sign-off

- [ ] §2 prerequisites green
- [ ] §5 tarball extracted, ownership correct
- [ ] §6 `run_as_user="nexus"` set
- [ ] §8 service enabled, survives reboot
- [ ] §10 first-login done, admin password rotated
- [ ] §11 TLS working on FQDN (if applicable)
- [ ] §12 Docker registry login/pull/push works (if applicable)
- [ ] §14 integrations for each consumer (Maven/npm/pip/Docker/Jenkins/AD) validated
- [ ] §15 backup task scheduled and first backup verified
- [ ] §17 all validations pass
