# 02 — Sonatype Nexus Repository 3 (DEV) — Production-Ready Runbook

> **Audience:** DevOps / SRE / Platform Engineering
> **Environment:** DEV (VMware vSphere, RHEL 9)
> **Tool Role:** Artifact repository + Docker registry + **single controlled egress** (proxy/offline mirror) for the entire DEV toolchain.
> **Why Nexus first:** Every other tool (Jenkins, SonarQube, Checkmarx, Aqua) pulls its plugins, base images, Maven/npm/PyPI/Helm artifacts, and OS RPMs through Nexus. Installing Nexus before anything else lets us **cut the internet cord** and enforce "Nexus-only egress" from day one.

---

## Dependency Readiness (must be green before you start)

| # | Item | Why it matters | Evidence to proceed |
|---|------|----------------|---------------------|
| 1 | VMware VM provisioned per sizing (see §3) | Undersized VM = OOM on reindex | `lscpu`, `free -h`, `lsblk` output attached |
| 2 | RHEL 9 subscribed or satellite/offline RPM mirror reachable | We cannot `dnf install` otherwise | `dnf repolist` returns ≥1 enabled repo |
| 3 | Internal Root CA file available (`/etc/pki/ca-trust/source/anchors/internal-root-ca.crt`) | Nexus must trust upstream proxies / AD-LDAPS | `trust list \| grep -i <CLIENT_NAME>` |
| 4 | HTTP proxy `proxy.dev.<CLIENT_NAME>.internal:3128` reachable **OR** offline RPM/JDK mirror staged | Required for first-time package install | `curl -x ... https://repo1.maven.org` returns 200 |
| 5 | AD/LDAP service account `svc-nexus-ldap` created with read-only bind | Needed for SSO in §9 | `ldapsearch -x -H ldaps://... -D ... -W -b ...` returns entries |
| 6 | DNS record `nexus.dev.<CLIENT_NAME>.internal` **planned** (not yet cut over) | We start local-first via `/etc/hosts` | DNS change ticket in "scheduled" state |
| 7 | Reverse proxy plan (NGINX on same VM or F5 VIP) decided | TLS terminates there | Architecture diagram signed off |

> **STOP** if any row is red. Do not proceed.

---

## 1. Overview & Intended Architecture

### 1.1 What we are building
A hardened Sonatype **Nexus Repository Manager 3 (OSS)** instance acting as:
- **Proxy repos** → cache upstream (Maven Central, npmjs, PyPI, Docker Hub, Red Hat UBI) so build agents never touch the internet directly.
- **Hosted repos** → store internally-built artifacts (JARs, npm packages, Helm charts, container images).
- **Group repos** → single URL that fronts proxy + hosted for each ecosystem.
- **Docker registry** → TLS-secured, authenticated registry consumed by Jenkins, Aqua (image source), and developer workstations.

### 1.2 Logical diagram

```
            ┌────────────────────────┐
  Devs/CI ──┤ NGINX (TLS 443)        │── F5 VIP (future)
            │ nexus.dev.<client>.    │
            │ internal               │
            └──────────┬─────────────┘
                       │ 127.0.0.1:8081 (UI/API)
                       │ 127.0.0.1:8082 (Docker registry connector)
            ┌──────────┴─────────────┐
            │ Nexus 3 (systemd)      │
            │ /opt/sonatype/nexus    │
            │ data: /var/nexus-data  │
            │ blobs: /data/blobs     │
            └──────────┬─────────────┘
                       │ HTTP proxy
                       ▼
          proxy.dev.<client>.internal:3128  ──► Internet (controlled)
```

### 1.3 Install mode decision — **systemd (RPM tarball), not container**
| Criterion | systemd | Container (Podman) | Decision |
|---|---|---|---|
| Blob store I/O (can reach TB) | Native FS, best perf | Bind-mount volumes, OK but extra layer | systemd |
| Upgrade cadence (quarterly) | `tar -xzf` swap symlink | `podman pull` | Tie |
| Vendor support matrix | Primary | Secondary | systemd |
| Day-2 ops familiarity for 30 devs DEV | `systemctl`, `journalctl` | Needs Podman skills | systemd |

**Chosen:** systemd service running the Nexus unpacked distribution as user `nexus`. (Podman is still our default for app workloads, but Nexus is the one stateful platform we keep on systemd for I/O and vendor alignment.)

---

## 2. Official Documentation Links

1. Install on Linux — https://help.sonatype.com/repomanager3/installation/installation-methods/install-as-a-service-on-linux
2. System requirements — https://help.sonatype.com/repomanager3/product-information/sonatype-nexus-repository-system-requirements
3. Reverse proxy & SSL — https://help.sonatype.com/repomanager3/installation-and-upgrades/configuring-ssl
4. Docker registry connector — https://help.sonatype.com/repomanager3/formats/docker-registry
5. LDAP/AD realm — https://help.sonatype.com/repomanager3/nexus-repository-administration/access-control/ldap
6. Backup & restore — https://help.sonatype.com/repomanager3/installation-and-upgrades/backup-and-restore

---

## 3. Prerequisites (hard requirements + go/no-go checklist)

### 3.1 VM sizing (DEV, ~30 users)
| Resource | Minimum | Recommended | Why |
|---|---|---|---|
| vCPU | 4 | **8** | Concurrent Docker pushes + Maven index rebuild |
| RAM | 8 GB | **16 GB** | JVM heap 4 GB + Direct 8 GB + OS cache |
| `/` (OS) | 40 GB | 60 GB | RHEL + logs |
| `/opt` | 20 GB | 30 GB | Nexus binaries + work dir |
| `/var/nexus-data` | 50 GB | 100 GB | DB (OrientDB/H2), search index, keystores |
| `/data/blobs` | **500 GB** | 1 TB (thin, expandable) | Actual artifacts. Separate LUN so we can grow independently. |
| Network | 1 Gbps | 10 Gbps | Docker push/pull throughput |

### 3.2 OS / platform baseline

| Check | Required state | Command |
|---|---|---|
| OS | RHEL 9.2+ | `cat /etc/redhat-release` |
| Time sync | Chrony OK, offset < 100 ms | `chronyc tracking` |
| SELinux | `Enforcing` (we will add contexts, not disable) | `getenforce` |
| firewalld | Running | `systemctl is-active firewalld` |
| JDK | Eclipse Temurin 17 (bundled with Nexus 3.70+) | handled by installer |
| Users | Local `nexus` user, no shell | created in §4 |
| CA trust | Internal root CA installed | `trust list \| grep -ci <CLIENT_NAME>` ≥ 1 |
| Proxy env | `/etc/environment` has `http_proxy`, `https_proxy`, `no_proxy` | `env \| grep -i proxy` |

### 3.3 Go / No-Go checklist (sign before §4)

- [ ] VM matches §3.1 sizing (`lscpu`, `free -h`, `lsblk`).
- [ ] `/data/blobs` is a **separate mount** on its own LUN (XFS, `noatime`).
- [ ] Internal CA trusted (`update-ca-trust` already run).
- [ ] Proxy reachable: `curl -x http://proxy.dev.<CLIENT_NAME>.internal:3128 -I https://repo1.maven.org/maven2/` → `HTTP/1.1 200`.
- [ ] `nexus.dev.<CLIENT_NAME>.internal` planned in DNS, but **not yet cut over**.
- [ ] AD bind account tested with `ldapsearch`.
- [ ] Backup target (NFS/PowerProtect) mount-tested read/write.

> **STOP** if any box is unchecked.

---

## 4. Manual Setup (step-by-step)

> Run as `root` unless noted. All commands idempotent where possible.

### 4.1 Storage layout
**Why:** Keep binaries, working data, and blob store on separate mounts so we can snapshot/back up/extend each independently.

```bash
# Verify mounts exist (fail fast if not)
mountpoint -q /opt           && echo OK || echo "MISSING /opt mount"
mountpoint -q /var/nexus-data && echo OK || echo "MISSING /var/nexus-data mount"
mountpoint -q /data/blobs    && echo OK || echo "MISSING /data/blobs mount"
```
**Expected:** three `OK` lines. **If not:** request storage team add LUNs before proceeding — **STOP**.

### 4.2 OS hardening & packages

```bash
# Base tooling
dnf install -y java-17-openjdk-headless tar gzip curl policycoreutils-python-utils \
               firewalld chrony nginx openssl bind-utils

# Time sync
systemctl enable --now chronyd
chronyc sources -v
```

### 4.3 Create dedicated service user
**Why:** Nexus must never run as root. A locked-down system account limits blast radius if the JVM is compromised.

```bash
groupadd --system nexus
useradd  --system --gid nexus --home-dir /opt/sonatype/nexus \
         --shell /sbin/nologin --comment "Nexus Repository" nexus
```

### 4.4 Install Nexus (offline tarball pattern)

**Why offline tarball + symlink:** enables zero-copy upgrades. Next quarter we just unpack `nexus-3.NEXT`, flip the symlink, restart — rollback is a single `ln -sfn`.

```bash
# 1. Stage the tarball (fetched via Nexus-of-Nexus or vendor portal → internal share)
VERSION="3.71.0-06"
cd /tmp
curl -o nexus.tar.gz \
  "https://artifacts.internal.<CLIENT_NAME>/third-party/sonatype/nexus-${VERSION}-unix.tar.gz"

# 2. Unpack to versioned directory and symlink
tar -xzf nexus.tar.gz -C /opt/sonatype/
ln -sfn /opt/sonatype/nexus-${VERSION} /opt/sonatype/nexus

# 3. Point sonatype-work to the data mount
rm -rf /opt/sonatype/sonatype-work
ln -sfn /var/nexus-data /opt/sonatype/sonatype-work

# 4. Ownership
chown -R nexus:nexus /opt/sonatype/nexus-${VERSION} /var/nexus-data /data/blobs
```

### 4.5 JVM & memory tuning
Edit `/opt/sonatype/nexus/bin/nexus.vmoptions`:

```
-Xms4G
-Xmx4G
-XX:MaxDirectMemorySize=8G
-XX:+UnlockDiagnosticVMOptions
-Dkaraf.data=/var/nexus-data
-Djava.io.tmpdir=/var/nexus-data/tmp
-Dkaraf.log=/var/nexus-data/log
-Dkaraf.etc=/opt/sonatype/nexus/etc/karaf
-Dhttps.proxyHost=proxy.dev.<CLIENT_NAME>.internal
-Dhttps.proxyPort=3128
-Dhttp.nonProxyHosts=localhost|127.0.0.1|*.dev.<CLIENT_NAME>.internal
-Djavax.net.ssl.trustStore=/etc/pki/java/cacerts
```
**Why** `Xms == Xmx`: avoids heap resize pauses during Docker pushes.
**Why** direct memory 8G: Nexus uses off-heap for Docker blob streaming.

### 4.6 Run-as user and listen address
Edit `/opt/sonatype/nexus/bin/nexus.rc`:
```
run_as_user="nexus"
```
Edit `/opt/sonatype/sonatype-work/nexus3/etc/nexus.properties` (create if missing):
```
application-host=127.0.0.1
application-port=8081
nexus-context-path=/
# Docker connector will be added via UI in §4.9, but declare expected:
# docker.registry.port=8082 (informational)
```
**Why bind to 127.0.0.1:** all public traffic must enter via NGINX. Nothing listens on a public port except 443.

### 4.7 systemd unit

`/etc/systemd/system/nexus.service`:
```ini
[Unit]
Description=Sonatype Nexus Repository Manager
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/sonatype/nexus/bin/nexus start
ExecStop=/opt/sonatype/nexus/bin/nexus stop
User=nexus
Group=nexus
Restart=on-failure
RestartSec=10
TimeoutStartSec=300
# Filesystem hardening
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now nexus
```

### 4.8 firewalld + SELinux

```bash
# Public-facing: only 443 (and 22 for admin)
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# SELinux: allow nginx to proxy to localhost high ports
setsebool -P httpd_can_network_connect 1

# Label blob store so nexus confined domain can read/write
semanage fcontext -a -t var_t "/data/blobs(/.*)?"
restorecon -Rv /data/blobs
```

### 4.9 Local-first DNS + TLS (self-signed, to be rotated in §9)

```bash
# /etc/hosts on this VM and on the admin workstation:
echo "127.0.0.1  nexus.dev.<CLIENT_NAME>.internal" >> /etc/hosts

# Local TLS for bootstrap (replaced by internal CA-signed cert in §9)
mkdir -p /etc/nginx/tls && cd /etc/nginx/tls
openssl req -x509 -nodes -newkey rsa:2048 -days 30 \
  -keyout nexus.key -out nexus.crt \
  -subj "/CN=nexus.dev.<CLIENT_NAME>.internal" \
  -addext "subjectAltName=DNS:nexus.dev.<CLIENT_NAME>.internal"
chmod 600 nexus.key
```

### 4.10 NGINX reverse proxy (UI + Docker registry on one FQDN via ports)

`/etc/nginx/conf.d/nexus.conf`:
```nginx
# ---------- UI / API (HTTPS 443) ----------
server {
    listen 443 ssl http2;
    server_name nexus.dev.<CLIENT_NAME>.internal;

    ssl_certificate     /etc/nginx/tls/nexus.crt;
    ssl_certificate_key /etc/nginx/tls/nexus.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Docker push can be huge
    client_max_body_size 4G;
    proxy_read_timeout   600s;
    proxy_send_timeout   600s;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options    "nosniff" always;
    add_header X-Frame-Options           "SAMEORIGIN" always;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

# ---------- Docker Registry (HTTPS 5000) ----------
server {
    listen 5000 ssl http2;
    server_name nexus.dev.<CLIENT_NAME>.internal;

    ssl_certificate     /etc/nginx/tls/nexus.crt;
    ssl_certificate_key /etc/nginx/tls/nexus.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    client_max_body_size 4G;
    chunked_transfer_encoding on;
    proxy_read_timeout   900s;

    location / {
        proxy_pass http://127.0.0.1:8082;   # docker hosted connector
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```
```bash
firewall-cmd --permanent --add-port=5000/tcp
firewall-cmd --reload
nginx -t && systemctl enable --now nginx
```
**Why a separate port for Docker:** the Docker client requires the registry to be the **root** of the host. Hosting it under `/v2/` of the UI site breaks clients.

### 4.11 First-login and admin password rotation

```bash
cat /var/nexus-data/admin.password    # one-time generated password
# UI: https://nexus.dev.<CLIENT_NAME>.internal/
# Login admin / <above>, then immediately:
#   - Set new admin password (stored in CyberArk, §9)
#   - Disable anonymous access
#   - Re-enable only after RBAC is configured (§9)
```

### 4.12 Create baseline repositories (UI → Repository → Repositories → Create)

| Name | Type | Format | Upstream / Notes |
|---|---|---|---|
| `maven-central-proxy` | proxy | maven2 | https://repo1.maven.org/maven2/ |
| `maven-releases` | hosted | maven2 | Release policy |
| `maven-snapshots` | hosted | maven2 | Snapshot policy |
| `maven-public` | group | maven2 | Group = central + releases + snapshots |
| `npm-proxy` | proxy | npm | https://registry.npmjs.org |
| `npm-hosted` | hosted | npm | |
| `npm-all` | group | npm | |
| `pypi-proxy` | proxy | pypi | https://pypi.org |
| `docker-proxy` | proxy | docker | https://registry-1.docker.io (enable Docker Hub, index=Hub) — **HTTP port blank, HTTPS port blank** (fronted by NGINX) |
| `docker-hosted` | hosted | docker | HTTP connector port **8082** |
| `docker-all` | group | docker | Group = hosted + proxy — used by Aqua/Jenkins |
| `rhel9-proxy` | proxy | yum | https://cdn-ubi.redhat.com/... (offline mirror if no RH sub) |
| `helm-proxy` | proxy | helm | https://charts.bitnami.com/bitnami |

**Why groups:** Clients use **one** URL per ecosystem. We can add/remove upstreams without touching clients.

### 4.13 Configure outbound HTTP proxy inside Nexus
UI → **Administration → System → HTTP**:
- HTTP proxy host: `proxy.dev.<CLIENT_NAME>.internal`, port `3128`
- HTTPS proxy host: same
- Non-proxy hosts: `localhost, 127.0.0.1, *.dev.<CLIENT_NAME>.internal`
- Authentication: from CyberArk (§9) if required by proxy.

**Why:** This is what makes "Nexus-only egress" real. Every proxy repo reaches the internet via the corporate proxy — nothing else on this VM ever does.

---

## 5. Manual Validation

> Every line is **Command → Expected → If not, do this**.

### 5.1 Service & ports
```bash
systemctl is-active nexus       # → active  | else: journalctl -u nexus -n 200
ss -lntp | grep -E '8081|8082'  # → 127.0.0.1:8081 and 127.0.0.1:8082 (nexus)
ss -lntp | grep -E ':443|:5000' # → 0.0.0.0:443 and 0.0.0.0:5000 (nginx)
```

### 5.2 HTTPS UI
```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://nexus.dev.<CLIENT_NAME>.internal/
# Expected: 200
# If 502: nexus not up yet (wait 2 min on first start). journalctl -u nexus
# If 000: NGINX down → systemctl status nginx
```

### 5.3 Maven proxy smoke test
```bash
curl -sk -o /dev/null -w "%{http_code}\n" \
  https://nexus.dev.<CLIENT_NAME>.internal/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom
# Expected: 200 (first call may take a few seconds while proxy caches)
# If 404: repository group missing central → fix in §4.12
# If 502: upstream proxy (3128) blocking → check Nexus HTTP proxy config
```

### 5.4 Docker login / push / pull
```bash
# On a client with /etc/hosts entry and internal CA trusted:
docker login nexus.dev.<CLIENT_NAME>.internal:5000 -u admin
docker pull   nexus.dev.<CLIENT_NAME>.internal:5000/library/alpine:3.19   # via docker-all group
docker tag    alpine:3.19 nexus.dev.<CLIENT_NAME>.internal:5000/dev/alpine:smoke
docker push   nexus.dev.<CLIENT_NAME>.internal:5000/dev/alpine:smoke
# Expected: login Succeeded, pull + push OK
# If "x509: certificate signed by unknown authority":
#   copy internal-root-ca.crt to /etc/docker/certs.d/nexus.dev.<CLIENT_NAME>.internal:5000/ca.crt
```

### 5.5 Blob store sanity
```bash
du -sh /data/blobs/*
# Expected: at least one blob dir with MBs after the docker push above.
```

---

## 6. Automation Setup (Ansible)

**Why Ansible (not Terraform here):** the VM itself is built by Terraform/vSphere module upstream. Inside the VM, configuration is Ansible's job. Everything in §4 is codified so a rebuild is **one play**.

### 6.1 Inventory (`inventories/dev/hosts.yml`)
```yaml
all:
  children:
    nexus:
      hosts:
        nexus-dev-01:
          ansible_host: 10.<TBD>.<TBD>.<TBD>
          ansible_user: ansible
```

### 6.2 Group vars (`group_vars/nexus.yml`)
```yaml
nexus_version: "3.71.0-06"
nexus_fqdn:    "nexus.dev.{{ client_name }}.internal"
nexus_home:    "/opt/sonatype/nexus"
nexus_data:    "/var/nexus-data"
nexus_blobs:   "/data/blobs"
nexus_jvm_xmx: "4G"
nexus_jvm_direct: "8G"

http_proxy_host: "proxy.dev.{{ client_name }}.internal"
http_proxy_port: 3128

# Secrets fetched from CyberArk Conjur (§9), never hardcoded
nexus_admin_password: "{{ lookup('conjur', 'dev/nexus/admin_password') }}"
ldap_bind_password:   "{{ lookup('conjur', 'dev/nexus/ldap_bind') }}"
```

### 6.3 Role layout (`roles/nexus/`)
```
roles/nexus/
├── tasks/
│   ├── main.yml          # orchestrates
│   ├── preflight.yml     # §3 checks fail-fast
│   ├── os.yml            # packages, user, SELinux, firewalld
│   ├── install.yml       # tarball + symlink
│   ├── config.yml        # vmoptions, nexus.properties, nexus.rc
│   ├── systemd.yml       # unit + enable
│   ├── nginx.yml         # TLS + reverse proxy
│   └── repos.yml         # REST API: create repos via uri module
├── templates/
│   ├── nexus.service.j2
│   ├── nexus.vmoptions.j2
│   ├── nginx-nexus.conf.j2
│   └── nexus.properties.j2
└── handlers/main.yml
```

### 6.4 Key task snippets

`tasks/preflight.yml`:
```yaml
- name: Fail if /data/blobs is not a separate mount
  ansible.builtin.assert:
    that:
      - ansible_mounts | selectattr('mount','equalto','/data/blobs') | list | length == 1
    fail_msg: "/data/blobs must be a dedicated mount — STOP."

- name: Verify internal CA trusted
  ansible.builtin.command: trust list
  register: ca_trust
  changed_when: false
  failed_when: "'<CLIENT_NAME>' not in ca_trust.stdout"
```

`tasks/repos.yml` (example — Maven group via REST):
```yaml
- name: Ensure maven-public group exists
  ansible.builtin.uri:
    url: "https://{{ nexus_fqdn }}/service/rest/v1/repositories/maven/group"
    method: POST
    user: admin
    password: "{{ nexus_admin_password }}"
    force_basic_auth: true
    validate_certs: true
    body_format: json
    body:
      name: maven-public
      online: true
      storage: { blobStoreName: default, strictContentTypeValidation: true }
      group:  { memberNames: [maven-central-proxy, maven-releases, maven-snapshots] }
    status_code: [201, 400]   # 400 = already exists; treated idempotent
```

### 6.5 Run
```bash
ansible-playbook -i inventories/dev/hosts.yml site.yml --tags nexus --check   # dry run
ansible-playbook -i inventories/dev/hosts.yml site.yml --tags nexus
```

---

## 7. Automation Validation

```bash
# Idempotency: second run must be all "ok", zero "changed"
ansible-playbook -i inventories/dev/hosts.yml site.yml --tags nexus | tee /tmp/run2.log
grep -E 'changed=[0-9]+' /tmp/run2.log
# Expected: changed=0  failed=0
# If changed>0 on 2nd run: a task is not idempotent — fix before sign-off.

# Smoke endpoint
curl -sk https://nexus.dev.<CLIENT_NAME>.internal/service/rest/v1/status | jq .
# Expected: {} HTTP 200 (no body, means healthy)

# Molecule (optional) — run role in a throwaway RHEL9 container
molecule test -s dev
```

---

## 8. Integration Steps

### 8.1 Jenkins → Nexus
**Why:** Jenkins publishes release artifacts to `maven-releases`, pulls build deps through `maven-public`, and pushes images to `docker-hosted`.

1. In Nexus: create roles `nx-jenkins-deploy` (priv: `nx-repository-view-*-*-*` for maven-releases, maven-snapshots, docker-hosted).
2. Create user `svc-jenkins` mapped to that role (or map AD group `CN=grp-ci-jenkins,...`).
3. In Jenkins → Manage Credentials → add "Username+Password" scoped credential `nexus-deploy` using CyberArk binding (§9).
4. Pipeline snippet:
```groovy
withCredentials([usernamePassword(credentialsId:'nexus-deploy',
                                  usernameVariable:'NX_USER',
                                  passwordVariable:'NX_PASS')]) {
  sh '''
    mvn -s $JENKINS_HOME/settings.xml \
        -DaltDeploymentRepository=releases::default::https://nexus.dev.<CLIENT_NAME>.internal/repository/maven-releases/ \
        deploy
    echo "$NX_PASS" | docker login nexus.dev.<CLIENT_NAME>.internal:5000 -u "$NX_USER" --password-stdin
    docker push nexus.dev.<CLIENT_NAME>.internal:5000/dev/myapp:${BUILD_NUMBER}
  '''
}
```
`settings.xml` mirrors everything to `maven-public`:
```xml
<mirror><id>nexus</id><mirrorOf>*</mirrorOf>
  <url>https://nexus.dev.<CLIENT_NAME>.internal/repository/maven-public/</url></mirror>
```

### 8.2 Aqua Enterprise → Nexus (image source)
- In Aqua → Integrations → Registries → add type **"Sonatype Nexus"**, URL `https://nexus.dev.<CLIENT_NAME>.internal:5000`, creds from CyberArk (`svc-aqua-scan`).
- Grant `svc-aqua-scan` the `nx-docker-pull` role on `docker-all`.
- Aqua will periodically list and scan images in `docker-hosted` and `docker-proxy`.

### 8.3 Checkmarx One → Nexus (dependency resolution)
- Checkmarx SCA agent uses the same `settings.xml` / `.npmrc` mirroring to `maven-public` and `npm-all`. No direct Nexus creds required for read (group is anonymous-read to internal, or use `svc-checkmarx` LDAP user).

### 8.4 SonarQube → Nexus (plugins)
- Point SonarQube `sonar.updatecenter.url` to a proxied plugin URL in Nexus raw repo if airgap required. Typical DEV: skip, SonarQube uses its built-in update center via HTTP proxy.

### 8.5 Developer workstations
- `~/.m2/settings.xml` mirror = `maven-public`.
- `~/.npmrc` → `registry=https://nexus.dev.<CLIENT_NAME>.internal/repository/npm-all/`
- `~/.docker/config.json` → login once to `nexus.dev.<CLIENT_NAME>.internal:5000`.
- Internal CA installed in OS trust store and Docker Desktop.

---

## 9. Security Hardening

### 9.1 Swap self-signed cert for internal-CA-signed (DNS cutover)
**Why now:** §4 used a 30-day self-signed cert only to bootstrap. Before inviting users we replace it and cut DNS.

```bash
# 1. CSR
openssl req -new -newkey rsa:2048 -nodes \
  -keyout /etc/nginx/tls/nexus.key \
  -out    /etc/nginx/tls/nexus.csr \
  -subj  "/CN=nexus.dev.<CLIENT_NAME>.internal/O=<CLIENT_NAME>/OU=DEV" \
  -addext "subjectAltName=DNS:nexus.dev.<CLIENT_NAME>.internal"
# 2. Submit to internal CA → receive nexus.crt + chain
cat nexus.crt chain.crt > /etc/nginx/tls/nexus.crt
chmod 600 /etc/nginx/tls/nexus.key
nginx -t && systemctl reload nginx
# 3. Ask network team to flip DNS record; remove /etc/hosts override.
```

### 9.2 AD/LDAP realm (SSO)
UI → Security → LDAP → Create:
- Protocol: **LDAPS**, host: `<TBD ad-host>`, port `636`.
- Search base: `DC=corp,DC=<CLIENT_NAME>,DC=com`
- Auth: simple, bind DN `CN=svc-nexus-ldap,OU=Service,DC=...`, password from CyberArk.
- User search: `CN=Users` filter `(sAMAccountName=${username})`, attribute `sAMAccountName`.
- Group mapping: dynamic via `memberOf`.

Realms order: **LDAP → Local Authenticating → Docker Bearer Token**.

### 9.3 Role mapping (least privilege)
| AD Group | Nexus Role | Privileges |
|---|---|---|
| `grp-nexus-admin` | `nx-admin` (built-in) | Full |
| `grp-nexus-dev`   | `nx-dev` (custom) | `nx-repository-view-*-*-browse`, `read`, `add`, `edit` on `*-hosted`, `docker-hosted` |
| `grp-nexus-read`  | `nx-readonly` | `browse`, `read` on `*-public`, `*-all` |
| `svc-jenkins` (AD user) | `nx-jenkins-deploy` | deploy to release/snapshot/docker-hosted |
| `svc-aqua-scan` | `nx-docker-pull` | pull from `docker-*` |

**Break-glass:** keep ONE local `admin` account, password in CyberArk safe `dev/nexus/breakglass`, rotated 90 days, audited on every use.

### 9.4 Secrets (CyberArk Conjur)
| Secret path | Consumer | How consumed |
|---|---|---|
| `dev/nexus/admin_password` | Ansible, break-glass | Ansible `lookup('conjur',...)` |
| `dev/nexus/ldap_bind` | Nexus LDAP realm | Summon sidecar writes to Nexus API at deploy time, **not stored in git** |
| `dev/nexus/proxy_user` | Nexus HTTP proxy | Same pattern |
| `dev/nexus/jenkins_deploy` | Jenkins | CyberArk Credential Provider plugin → Jenkins credentials store |
| `dev/nexus/aqua_pull` | Aqua | CyberArk REST pull at Aqua registry config |

**Never store in plaintext:** `nexus.properties`, `vmoptions`, pipeline code, git repos, Slack/Teams, tickets.

**Rotation plan:** admin + svc accounts rotated every **90 days**. Ansible play `nexus-rotate.yml` triggered by scheduler: calls Nexus REST API to set new password, writes to Conjur, notifies consumers via webhook.

### 9.5 Anonymous access & headers
- Disable anonymous access (UI → Security → Anonymous Access → off) after §9.2.
- Enable "Referrer-Policy: no-referrer" in NGINX; HSTS already set.
- Enable Nexus audit log: UI → System → Support → Logging → `audit` at INFO.

### 9.6 Alerting
**Why:** We must know within minutes if Nexus fills its disk or LDAP breaks — otherwise 30 devs and all CI stop.

Capabilities (UI → System → Capabilities → "Outbound HTTP Webhook") + optional Prometheus exporter scraped by central Prom:

| Event | Channel | Pattern |
|---|---|---|
| Disk `/data/blobs` > 80 % | Email `devops-alerts@<CLIENT>` + Teams channel `#dev-platform-alerts` | Prometheus node_exporter → Alertmanager → email + Teams webhook |
| `nexus.service` failed (systemd) | Same | Alertmanager rule on `node_systemd_unit_state{name="nexus.service",state="failed"}` |
| LDAP bind failure > 5 in 5 min | Same | Nexus log regex via Promtail/Loki |
| Cert expiring < 30 days | Email only | `cert-exporter` |
| Blob store quota reached | Teams (high) | Nexus webhook capability → Teams Incoming Webhook URL |

Example Teams webhook (Nexus Capability → Webhook Global → URL): `https://<CLIENT>.webhook.office.com/webhookb2/<TBD>`.

### 9.7 Port matrix
| Port | Dir | Source | Dest | Proto | Purpose |
|---|---|---|---|---|---|
| 443 | in | DEV VLANs | Nexus VM | TCP/TLS | UI/API |
| 5000 | in | DEV VLANs, Jenkins, Aqua | Nexus VM | TCP/TLS | Docker registry |
| 22 | in | Bastion | Nexus VM | TCP | Admin SSH |
| 3128 | out | Nexus VM | `proxy.dev.<CLIENT>...` | TCP | Upstream egress |
| 636 | out | Nexus VM | AD DCs | TCP/TLS | LDAPS |
| 25/587 | out | Nexus VM | SMTP relay | TCP | Email alerts |
| 443 | out | Nexus VM | Teams webhook endpoint | TCP/TLS | Teams alerts |
| 8081/8082 | loopback only | Nexus | NGINX | TCP | Internal |

**All other outbound traffic is denied by host firewall + network ACL.**

---

## 10. Backup / Restore & Data Paths

### 10.1 What to back up
| Path | Content | Frequency | Method |
|---|---|---|---|
| `/var/nexus-data/db` | Embedded DB (config, users, RBAC) | Every 6 h | Built-in Nexus **Export Databases** task |
| `/var/nexus-data/etc` | karaf config, keystores | Daily | tar → NFS/PowerProtect |
| `/var/nexus-data/blobs.idx` / blob metadata | Pointers | Daily | tar |
| `/data/blobs/*` | Actual artifacts | Daily (incremental rsync) | `rsync --link-dest` snapshots, 14 days retained |
| `/opt/sonatype/nexus-<ver>` | Binaries (re-installable) | On change only | Versioned in artifact share |

**Critical rule:** DB and blobs must be backed up **consistently**. Use the built-in *Admin → Tasks → Export databases for backup* task; it flushes and writes to `/var/nexus-data/backup`. Rsync blobs **only after** the export completes.

### 10.2 Scheduled tasks (UI → System → Tasks)
1. `Export databases for backup` → daily 01:00 → `/var/nexus-data/backup`.
2. `Compact blob store` → weekly Sunday 02:00 (reclaims deletes).
3. `Rebuild repository search index` → monthly.
4. Cron on host: rsync `/data/blobs` and `/var/nexus-data/backup` to NFS at 02:30.

### 10.3 Restore drill (to practice quarterly)
```bash
# On fresh RHEL 9 VM built by Ansible to same version
systemctl stop nexus
rsync -a backup-nfs:/nexus/blobs/  /data/blobs/
rsync -a backup-nfs:/nexus/db-export/  /var/nexus-data/restore-from-backup/
# Nexus auto-imports on start when restore-from-backup/ present
systemctl start nexus
journalctl -u nexus -f | grep -i restore
```
Verify: login, list repos, pull one image, resolve one Maven artifact. RTO target: **< 2 h**, RPO: **≤ 24 h** (DEV).

---

## 11. Troubleshooting (tool-specific, likely issues)

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | `/data/blobs` fills to 100 % | Unpruned snapshots, no cleanup policy | Create Admin → Cleanup Policies (e.g., snapshots > 30 days, prereleases > 15); assign to repos; run `Admin task: Purge unused components`. |
| 2 | `EACCES` / `Permission denied` in `nexus.log` on blob write | `/data/blobs` not owned by `nexus:nexus` or wrong SELinux label | `chown -R nexus:nexus /data/blobs && restorecon -Rv /data/blobs` |
| 3 | Maven proxy hangs, logs show `connect timed out` to repo1.maven.org | HTTP proxy not set in Nexus, or `http.nonProxyHosts` wrong | Set proxy (§4.13); restart; retry; check `curl -x ...` from the VM. |
| 4 | `docker login` → `x509: certificate signed by unknown authority` | Internal CA not installed on client | Copy `internal-root-ca.crt` → `/etc/docker/certs.d/nexus.dev.<CLIENT>...:5000/ca.crt`, restart docker. |
| 5 | `docker push` fails with `413 Request Entity Too Large` | NGINX `client_max_body_size` too low | Set to 4G (already in §4.10), reload nginx. |
| 6 | Port 5000 already in use (registry conflict) | Another registry or dev container on 5000 | `ss -lntp \| grep 5000`; kill/relocate the offender, or change NGINX listen to 5443. |
| 7 | LDAP login fails, `InvalidCredentialsException` | Bind DN or password wrong / LDAPS cert untrusted | `ldapsearch -x -H ldaps://... -D 'CN=svc-nexus-ldap,...' -W`; ensure AD CA chain in `/etc/pki/java/cacerts` via `keytool -importcert`. |
| 8 | Nexus UI 502 for 2–3 min after restart | Karaf still booting OSGi bundles | Wait; `tail -f /var/nexus-data/log/nexus.log`; increase NGINX `proxy_read_timeout` if pattern repeats. |
| 9 | Slow UI, GC logs show long pauses | Heap under-sized for index rebuild | Raise `-Xmx` only if RAM allows; more often raise `MaxDirectMemorySize`. |
| 10 | Docker pull slow, timeouts | Corporate proxy throttling | Add Nexus **Remote storage connection timeout** 120s; consider running a local UBI mirror. |
| 11 | `DB is corrupted` on startup | Unclean shutdown (power loss) | Stop service; run `java -jar nexus/lib/support/nexus-orient-console.jar` → `CONNECT PLOCAL:...; REPAIR DATABASE --fix-graph`; restore from backup if unrecoverable. |
| 12 | Cert expired → all clients broken | No monitoring on cert | Re-issue via internal CA (§9.1); add `cert-exporter` alert (§9.6). |

**STOP criteria during troubleshooting:**
- If DB is corrupted and backup < 24 h: restore. Do not attempt repair in DEV for > 30 min.
- If blob store FS is read-only: do NOT restart Nexus (risk of partial writes) — fix FS first.

---

## 12. Rollback Plan

### 12.1 Rollback a failed upgrade (binary swap)
**Pre-condition:** We always keep the previous version tarball unpacked.

```bash
systemctl stop nexus
ln -sfn /opt/sonatype/nexus-<PREV_VERSION> /opt/sonatype/nexus
systemctl start nexus
# Data in /var/nexus-data is backward-compatible for patch versions.
# For minor-version downgrade: restore DB export from pre-upgrade snapshot.
```

### 12.2 Rollback a bad config change
- All files under `/opt/sonatype/nexus/etc` and `/var/nexus-data/etc` are versioned in git (Ansible).
- `git revert <sha> && ansible-playbook ... --tags nexus` → service reload via handler.

### 12.3 Rollback DNS cutover
- Re-add `/etc/hosts` override on clients; ask DNS team to restore prior record (or shorten TTL pre-cutover to 60 s so rollback is fast).

### 12.4 Full destructive rollback (decommission)
1. Quiesce: disable anonymous + all non-admin roles.
2. Final backup: export DB + rsync blobs to cold storage labeled `nexus-dev-final-<date>`.
3. `systemctl disable --now nexus nginx`.
4. Detach `/data/blobs` LUN (keep for 90 days before destroy).
5. Remove DNS record; remove AD service accounts; rotate break-glass secrets.
6. Update CMDB + architecture docs.

**Data preservation rule:** never delete `/data/blobs` or backup exports in the same change window as decommission. 90-day cold retention minimum.

---

## Day-2 Ops (quick reference)

| Task | Command / Location |
|---|---|
| Start / stop / status | `systemctl {start,stop,status} nexus` |
| Logs | `/var/nexus-data/log/nexus.log`, `journalctl -u nexus`, `/var/log/nginx/*` |
| Patching | Stage new tarball → swap symlink → `systemctl restart nexus` → smoke (§5) |
| Cert renewal | Re-run §9.1 CSR → import → `systemctl reload nginx` |
| JVM tuning change | Edit `bin/nexus.vmoptions` → `systemctl restart nexus` |
| Add a repo | UI or Ansible `tasks/repos.yml` (preferred) |
| Rotate admin pwd | Ansible `nexus-rotate.yml` (reads/writes CyberArk) |
| Free disk | Run cleanup policy + `Compact blob store` task |
| Health endpoint | `GET /service/rest/v1/status` (200) and `/service/metrics/healthcheck` (admin) |

---

## Acceptance Checklist (sign-off)

- [ ] §3 Go/No-Go fully checked.
- [ ] `systemctl is-active nexus` = active; service survives `reboot`.
- [ ] `https://nexus.dev.<CLIENT_NAME>.internal/` returns 200 with internal-CA cert (no browser warnings).
- [ ] `docker login / push / pull` works end-to-end against `:5000`.
- [ ] Maven, npm, PyPI, Docker, Helm, yum proxy repos all return artifacts from their upstream.
- [ ] Outbound: only 3128 (proxy), 636 (LDAPS), 25/587 (SMTP), 443 to Teams endpoint. All other outbound denied (verified by `iptables -L` / flow logs).
- [ ] LDAP login works for an AD user in each of the 3 mapped groups; RBAC verified.
- [ ] Anonymous access disabled.
- [ ] Break-glass account documented; password in CyberArk; last-used audit enabled.
- [ ] Alerts: forced disk fill test fires an email **and** a Teams notification within 5 min.
- [ ] Backup job ran, restore drill executed on a sandbox VM within RTO 2 h.
- [ ] Ansible run is fully idempotent (second run: `changed=0`).
- [ ] Runbook reviewed and signed by: DevOps lead, Security, Platform Ops.

> **Sign-off below:**
> DevOps Lead: ________________  Date: ________
> Security:    ________________  Date: ________
> Platform:    ________________  Date: ________
