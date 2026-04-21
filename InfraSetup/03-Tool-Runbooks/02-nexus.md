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

### 1.1 Official Sonatype guidance

Source: https://help.sonatype.com/repomanager3/product-information/sonatype-nexus-repository-system-requirements

| Size | Sonatype minimum | Target load |
|---|---|---|
| Small | **4 vCPU, 8 GB RAM** | < 20 concurrent users, a few formats, < 200 GB blobs |
| Medium | 8 vCPU, 16 GB RAM | 20–100 users, many formats, 200 GB – 2 TB blobs |
| Large | 16 vCPU, 32 GB+ RAM | > 100 users, HA/cluster, > 2 TB blobs |

The JVM default heap is ~2.7 GB. Nexus also uses direct memory (default 2 GB) for blob I/O. Add OS overhead and you **cannot realistically run Nexus 3.71+ below 4 GB RAM** — that's what the OOM-kill in §21.2 proved.

### 1.2 What we validated on AWS (reference baseline)

The install was proven end-to-end on a **single-user smoke test** with:

| Attribute | Value |
|---|---|
| Instance type | `c7i-flex.large` |
| vCPU | 2 |
| RAM | 4 GB |
| Network | up to 12.5 Gbps |
| Heap tuning | `-Xms1G -Xmx1G -XX:MaxDirectMemorySize=1G` (§8.1) |

Outcome: the UI is responsive for 1 admin, repositories can be created, and maven/npm/pypi proxies work. This is **sub-minimum** vs Sonatype's own spec and is acceptable **only for the install-validation phase**. Day-2 symptoms on this size:

- Heap pressure / GC pauses as soon as 2–3 users hit the UI at once.
- OOM-kill risk if a single large Docker push lands while other traffic is active.
- No headroom for `Admin - Compact blob store` (§20.6) — it runs I/O-heavy and wants direct memory.

Plan to resize up the moment real users/CI land on it. On AWS that's a 2-minute stop → change instance type → start (see §21.2).

### 1.3 On-prem right-sizing (RHEL 9 VM on VMware)

Use the Sonatype minimum as the floor and size by **concurrency + blob footprint**, not by "it's just dev".

| Tier | When | vCPU | RAM | Heap (`-Xmx`) | Direct (`MaxDirectMemorySize`) | Network |
|---|---|---|---|---|---|---|
| **Install validation only** | Proving install works, 1 admin, no CI | 2 | 4 GB | 1 G | 1 G | 1 Gbps |
| **Small (recommended DEV floor)** | ≤ 20 devs + 1–2 CI agents, Maven + npm + pip | 4 | 8 GB | 2.7 G (default) | 2 G (default) | 1 Gbps |
| **Medium (default PROD)** | 20–100 devs, multiple CI agents, Docker hosted+proxy, nightly jobs | 8 | 16 GB | 6 G | 4 G | 10 Gbps |
| **Large** | > 100 devs, heavy Docker/Helm, HA plans, > 2 TB blobs | 16 | 32 GB | 16 G | 8 G | 10 Gbps |

**Rules of thumb for the JVM once you're above 8 GB RAM:**
- `-Xms` = `-Xmx` (avoid heap resizing at runtime).
- Leave roughly **50% of physical RAM** for `-Xmx` + `MaxDirectMemorySize` combined; the rest is for OS page cache (which is what makes blob reads fast) and the JVM's non-heap.
- Example for 16 GB VM: `-Xmx6G -XX:MaxDirectMemorySize=4G` → 10 GB for JVM, 6 GB for page cache.
- Do **not** oversubscribe vCPU on the ESXi host for a production Nexus — Nexus is latency-sensitive under load.

### 1.4 Storage sizing

Sonatype stores DB, config, logs, and **blob stores** (the actual JARs/images/packages) under one `sonatype-work/nexus3` directory. Size `/nexus-data` by forecasted footprint, not by "today's" size — blobs grow monotonically unless cleanup policies run (§20.6).

| Mount | Size | Why |
|---|---|---|
| `/` | 40 GB | OS + logs. Never grows unboundedly if `/var/log` is trimmed by default logrotate. |
| `/opt` | 20 GB | Nexus binary tarballs (each release ~200 MB; keep 1 current + 1 previous for rollback). |
| `/nexus-data` | **dedicated VMDK** | Embedded DB + **blob stores**. See estimate below. |
| swap | 0 (or 2 GB) | Optional. Not a substitute for RAM — if you're swapping, fix RAM instead. |

**Blob footprint estimate (`/nexus-data` sizing):**

| Workload | 6-month estimate | 24-month estimate |
|---|---|---|
| Small dev team (Maven + npm + pip only, no Docker) | 50 GB | 150 GB |
| + Docker hosted for internal apps | 200 GB | 500 GB – 1 TB |
| + Docker proxy for Docker Hub + multiple teams | 500 GB – 1 TB | 2 TB+ |
| Enterprise multi-format (Maven/npm/pip/Docker/Helm/apt/yum) | 1 TB | 3–5 TB |

Start with a thin-provisioned 200 GB VMDK for a DEV instance and 500 GB for a first PROD. Enable cleanup policies **before** the first 100 GB of blobs — retroactive cleanup on a 1 TB blob store is painful.

**Why a dedicated `/nexus-data` disk:** growth without touching the OS disk, independent snapshot/backup scope, storage-tier placement (SSD for DB, cheaper tier for cold blobs later), and a full disk takes down only Nexus — not SSH/systemd/logging. See §4a for the mount procedure.

### 1.5 Network

- **Bandwidth:** a single Docker image layer push is a multi-GB transfer at CI speed; an npm-install storm from a CI fleet can saturate 1 Gbps. For production on-prem, give the VM a **10 Gbps** vNIC and make sure the uplink from the ESXi host and the storage network match.
- **Latency:** builds feel slow if Nexus-to-client RTT exceeds ~5 ms. Put Nexus in the same site/rack as the bulk of the CI agents.
- **Firewall:** only these ports need external access — `443` (UI + API via NGINX, §11), `5000` (Docker registry via NGINX, §12). Keep `8081`/`8082`/`8083` on localhost once NGINX is live.

### 1.6 Final on-prem recommendation (one concrete spec to copy)

For a typical internal DEV instance serving ~30 users with Maven + npm + pip + Docker:

| Resource | Value |
|---|---|
| vCPU | **4** |
| RAM | **8 GB** |
| `/` | 40 GB (thin) |
| `/opt` | 20 GB (thin) |
| `/nexus-data` | **200 GB dedicated VMDK, thin-provisioned, expandable** |
| vNIC | 1 Gbps (10 Gbps for PROD) |
| OS | RHEL 9.2+ |
| JVM | stock `nexus.vmoptions` (do not tune down) |

For PROD at the same scale, **double RAM to 16 GB, vCPU to 8**, make the data disk 500 GB, and put it on the 10 Gbps network.

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

### 2.1 Java 17 — pick ONE of two options

Nexus Repository 3.71+ **requires Java 17**. Sonatype publishes two tarball variants on the download page:

| Tarball filename | JRE included? | What you do |
|---|---|---|
| `nexus-<ver>-unix.tar.gz` | **No** — binaries only | You must install Java 17 on the host (recommended for enterprise on-prem) |
| `nexus-<ver>-java17-unix.tar.gz` | **Yes** — bundled Temurin 17 | Nothing extra to install |

Pick the model your org wants. Most enterprises choose the **unbundled** tarball so Java patching follows the normal RHEL security update cadence (RHSM / Satellite), not Sonatype's release cycle.

**If you picked the unbundled tarball** (recommended on-prem), install Java 17 now:

```bash
dnf install -y java-17-openjdk-headless

# Find the real JRE path (RHEL versions it like .../java-17-openjdk-17.0.x.y-...)
readlink -f /usr/bin/java
# Example: /usr/lib/jvm/java-17-openjdk-17.0.13.0.11-2.el9.x86_64/bin/java

# Compute JAVA_HOME (strip trailing /bin/java)
JAVA_HOME_PATH="$(dirname "$(dirname "$(readlink -f /usr/bin/java)")")"
echo "JAVA_HOME will be: $JAVA_HOME_PATH"

# Sanity check
"$JAVA_HOME_PATH/bin/java" -version
# Expected: openjdk version "17.0.x" ...
```

Keep `$JAVA_HOME_PATH` handy — §6 uses it.

**If you picked the `java17` bundled tarball**, skip the install above. §6 will not need the `INSTALL4J_JAVA_HOME_OVERRIDE` line.

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

# 1. Get the CURRENT version string from the official download page first:
#    https://help.sonatype.com/repomanager3/product-information/download
#    Look for "Unix Archive (tar.gz)". The filename follows the pattern:
#      nexus-<major>.<minor>.<patch>-<build>-unix.tar.gz
#    Example used below — REPLACE with whatever the page shows today.
NEXUS_VERSION="3.71.0-06"

# 2. Download. -f = fail on HTTP error (so you don't silently save a 404 HTML page)
#              -L = follow redirects
#              -O = save with remote filename
curl -fL -O "https://download.sonatype.com/nexus/3/nexus-${NEXUS_VERSION}-unix.tar.gz"

# 3. Verify the file is a real gzipped tarball, not an error page
file   "nexus-${NEXUS_VERSION}-unix.tar.gz"    # → gzip compressed data, ...
ls -lh "nexus-${NEXUS_VERSION}-unix.tar.gz"    # → ~170–200 MB

# 4. Extract
tar -xzf "nexus-${NEXUS_VERSION}-unix.tar.gz"
ls
# nexus-3.71.0-06/      sonatype-work/
```

> **If `tar` says "not in gzip format":** you didn't get a real tarball. `file` will show `HTML document` or `ASCII text` — that's a 404 or login page saved as `.tar.gz`. Re-check the URL from the download page and re-run with `curl -fL` so the failure is visible next time.

This matches the official procedure: extract into `/opt/sonatype` producing two sibling directories — `nexus-<version>` (binaries) and `sonatype-work` (data).

Move `sonatype-work` onto the dedicated disk and link it back:

```bash
mv /opt/sonatype/sonatype-work /nexus-data/sonatype-work
ln -s /nexus-data/sonatype-work /opt/sonatype/sonatype-work

chown -R nexus:nexus /opt/sonatype/nexus-3.* /nexus-data/sonatype-work
```

---

## 6. Configure run-as user and Java home

File: `/opt/sonatype/nexus-<version>/bin/nexus.rc`  (replace `<version>` with the actual directory name, e.g. `nexus-3.71.0-06`).

Set **two** lines:

```bash
# Required — run as the non-root service user
run_as_user="nexus"

# Required ONLY if you used the unbundled tarball (§2.1). Point at the RHEL-installed JRE.
# Use the same $JAVA_HOME_PATH you computed in §2.1. Example:
INSTALL4J_JAVA_HOME_OVERRIDE="/usr/lib/jvm/java-17-openjdk-17.0.13.0.11-2.el9.x86_64"
```

> **Why `INSTALL4J_JAVA_HOME_OVERRIDE` and not just `JAVA_HOME`?**
> The Nexus start script is generated by install4j and only honours the `INSTALL4J_*` variable. Setting plain `JAVA_HOME` won't work — you'll get `No suitable Java Virtual Machine could be found on your system. The version of the JVM must be 17.` and the service will exit with status 83.

One-liner to set both lines idempotently:

```bash
NEXUS_BIN=/opt/sonatype/nexus-3.71.0-06/bin   # adjust to your version
JAVA_HOME_PATH="$(dirname "$(dirname "$(readlink -f /usr/bin/java)")")"

# run_as_user
sed -i 's|^#*run_as_user=.*|run_as_user="nexus"|' "$NEXUS_BIN/nexus.rc"
grep -q '^run_as_user=' "$NEXUS_BIN/nexus.rc" || echo 'run_as_user="nexus"' >> "$NEXUS_BIN/nexus.rc"

# INSTALL4J_JAVA_HOME_OVERRIDE (skip if you used the java17 bundled tarball)
sed -i '/^INSTALL4J_JAVA_HOME_OVERRIDE=/d' "$NEXUS_BIN/nexus.rc"
echo "INSTALL4J_JAVA_HOME_OVERRIDE=\"$JAVA_HOME_PATH\"" >> "$NEXUS_BIN/nexus.rc"

# Verify
grep -E '^(run_as_user|INSTALL4J_JAVA_HOME_OVERRIDE)' "$NEXUS_BIN/nexus.rc"
```

Official doc: https://help.sonatype.com/repomanager3/installation/run-as-a-service

---

## 7. Configure listen address (override file)

> **IMPORTANT — never edit `nexus-default.properties`.**
> The file `/opt/sonatype/nexus-<version>/etc/nexus-default.properties` ships with the binary and is the **vendor default**. Its first line says: `## DO NOT EDIT - CUSTOMIZATIONS BELONG IN $data-dir/etc/nexus.properties`. Reasons you must respect this:
>
> 1. **Upgrades overwrite it.** Every new Nexus tarball replaces this file. Any edits are silently lost the next time you upgrade (§16).
> 2. **Override, don't replace.** Nexus reads `nexus-default.properties` first, then `nexus.properties` on top. Your file should contain **only the lines you want to change** — everything else inherits from the vendor default.
> 3. **Clean audit trail.** A 3-line `nexus.properties` makes it obvious what this instance does differently from a stock install. A modified default file hides that.
> 4. **Vendor support.** Sonatype support will ask for `nexus.properties`. They expect only your overrides there. A modified default file is a red flag.
>
> **Rule of thumb:** anything under `/opt/sonatype/nexus-<version>/` is vendor-owned and gets wiped on upgrade. Anything under `/nexus-data/sonatype-work/nexus3/` is yours and survives upgrades.

### Where the override file lives

`/nexus-data/sonatype-work/nexus3/etc/nexus.properties` (i.e. `$data-dir/etc/nexus.properties`).

Create it only if you need to change something. For a first test install where the defaults (`0.0.0.0:8081`, context `/`) are fine, **you can skip this step entirely**.

If you *do* need it (e.g. later to bind to 127.0.0.1 for NGINX in §11):

```bash
# Create the override file with only the lines that differ from defaults
install -o nexus -g nexus -m 640 /dev/null /nexus-data/sonatype-work/nexus3/etc/nexus.properties

cat > /nexus-data/sonatype-work/nexus3/etc/nexus.properties <<'EOF'
application-port=8081
application-host=0.0.0.0
nexus-context-path=/
EOF

chown nexus:nexus /nexus-data/sonatype-work/nexus3/etc/nexus.properties
chmod 640         /nexus-data/sonatype-work/nexus3/etc/nexus.properties
```

Leave `application-host=0.0.0.0` while you're bringing the instance up so you can reach the UI directly. When you add NGINX (§11), change the line to `application-host=127.0.0.1` and restart Nexus.

---

## 8. Register as a systemd service

Official doc: https://help.sonatype.com/repomanager3/installation/run-as-a-service

> **IMPORTANT — the unit file MUST live at `/etc/systemd/system/nexus.service`.**
> `systemctl` only reads units from three directories (in priority order):
>
> | Path | Owner | Use |
> |---|---|---|
> | `/etc/systemd/system/` | **You (admin)** | This is where `nexus.service` goes. Overrides everything else. |
> | `/run/systemd/system/` | systemd runtime | Reboots wipe it. |
> | `/usr/lib/systemd/system/` | RPM packages | Don't put custom units here — RPM upgrades overwrite. |
>
> A unit placed anywhere else (e.g. `/opt/sonatype/.../etc/systemd/system/nexus.service`) is invisible to systemd and `systemctl enable` will fail with *"Unit file nexus.service does not exist"*.

### 8.1 Low-RAM hosts — tune the JVM heap before first start

Nexus's default `bin/nexus.vmoptions` sets heap to ~2.7 GB. On a VM with less than ~4 GB RAM the JVM will fail to allocate and systemd will report `Main process exited, code=exited, status=1/FAILURE` almost instantly.

```bash
free -h
cat /opt/sonatype/nexus-3.71.0-06/bin/nexus.vmoptions
```

If total RAM is below 4 GB (common on AWS `t3.small` test instances), drop the heap. Example for a 2 GB VM:

```bash
sed -i 's/^-Xms.*/-Xms1G/;  s/^-Xmx.*/-Xmx1G/;  s/^-XX:MaxDirectMemorySize=.*/-XX:MaxDirectMemorySize=1G/' \
  /opt/sonatype/nexus-3.71.0-06/bin/nexus.vmoptions
cat /opt/sonatype/nexus-3.71.0-06/bin/nexus.vmoptions
```

Production on-prem sizing (8 GB RAM VM) keeps the stock values — do not tune heap down unless you have to.

### 8.2 Create the unit file

```bash
NEXUS_VERSION=3.71.0-06    # adjust to your actual extracted version

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
```

First start takes 1–3 minutes while Karaf initializes. Tail the log:

```bash
tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
# Look for:  "Started Sonatype Nexus OSS <version>"
```

### 8.3 If the service fails to start — diagnostic order

The systemd view only shows the wrapper exit. The real error is usually in `nexus.log`. Run these in order:

```bash
# 1. systemd's view of the failure
systemctl status nexus --no-pager -l
journalctl -u nexus -n 200 --no-pager

# 2. Nexus application log — this is where real stack traces go
tail -n 200 /nexus-data/sonatype-work/nexus3/log/nexus.log

# 3. JVM / wrapper log (if log above is empty, JVM never started)
ls /nexus-data/sonatype-work/nexus3/log/
tail -n 200 /nexus-data/sonatype-work/nexus3/log/jvm.log 2>/dev/null
```

| Symptom in the output | Cause | Fix |
|---|---|---|
| `No suitable Java Virtual Machine could be found ... must be 17` (journal) | Missing or wrong Java | Re-do §2.1 + §6 |
| `Main process exited, code=exited, status=1/FAILURE` in < 100 ms, `nexus.log` empty | JVM couldn't allocate heap | §8.1 heap tuning |
| `Permission denied` writing `/nexus-data/...` | Ownership wrong | `chown -R nexus:nexus /nexus-data /opt/sonatype/nexus-*` |
| `admin.password` never appears | Startup didn't finish; check `nexus.log` for stack trace | Fix the error shown there |

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
curl -fL -O "https://download.sonatype.com/nexus/3/nexus-<new-version>-unix.tar.gz"
tar -xzf "nexus-<new-version>-unix.tar.gz"
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

## 20. Nexus management after first login (repository setup for real use)

A fresh Nexus 3 install only ships with the two default format sets (`maven-*` and `nuget*`) visible under **Browse**. npm, Docker, PyPI, Helm, apt, yum, raw, etc. are **not** pre-created — you build them yourself. This section is the minimum admin walk-through to get each format usable.

Open the UI → ⚙️ (gear) → **Repository → Repositories → Create repository**. For every format you create **three** repos in this order:

| Type | Purpose | Clients use it? |
|---|---|---|
| `<fmt> (proxy)` | Caches an upstream public registry (npmjs.org, Docker Hub, PyPI, etc.). | No — hidden behind the group. |
| `<fmt> (hosted)` | Stores your own private uploads / internal releases. | No — hidden behind the group. |
| `<fmt> (group)` | Single URL that merges hosted + proxy and resolves in order. | **Yes — this is the URL every build/client points at.** |

> **Golden rule:** clients always hit the **group** URL (`/repository/<fmt>-group/`). Never point builds at a raw proxy or hosted repo — you lose the ability to add mirrors, move upstreams, or layer private packages later.

Throughout this section, substitute your `<NEXUS_URL>`:
- **On-prem with TLS (from §11):** `https://nexus.dev.contoso.internal` (i.e. `https://<NEXUS_FQDN>`)
- **AWS / no-TLS test install:** `http://<ec2-public-ip-or-dns>:8081`

### 20.1 npm (registry.npmjs.org)

**Create in UI:**

| Repo | Recipe | Key settings |
|---|---|---|
| `npm-proxy` | `npm (proxy)` | Remote storage: `https://registry.npmjs.org` · Blob store: `default` |
| `npm-hosted` | `npm (hosted)` | Deployment policy: `Allow redeploy` (dev) or `Disable redeploy` (release line) |
| `npm-group` | `npm (group)` | Members (order): `npm-hosted`, `npm-proxy` |

**Enable the npm auth realm:** UI → **Security → Realms** → move **npm Bearer Token Realm** to the active list → Save. Without this, `npm login` fails with `E401`.

**Client use (`.npmrc` in project root or `~/.npmrc`):**
```
registry=<NEXUS_URL>/repository/npm-group/
always-auth=true
```

```bash
# One-time login (stores token in .npmrc)
npm login --registry=<NEXUS_URL>/repository/npm-group/

# Install — first request is fetched from npmjs.org and cached
npm install express

# Publish a private package (goes to npm-hosted through the group)
npm publish --registry=<NEXUS_URL>/repository/npm-hosted/
```

### 20.2 Docker

Docker is the one format where **port matters**: the Docker client demands the registry at the root of a host, and each Nexus docker repo needs its own "HTTP connector port".

**Create in UI:**

| Repo | Recipe | HTTP connector port | Other |
|---|---|---|---|
| `docker-proxy` | `docker (proxy)` | leave blank | Remote: `https://registry-1.docker.io` · Docker index: **Use Docker Hub** |
| `docker-hosted` | `docker (hosted)` | `8082` | Deployment policy as needed |
| `docker-group` | `docker (group)` | `8083` | Members: `docker-hosted`, `docker-proxy` |

**Enable:** UI → **Security → Realms** → add **Docker Bearer Token Realm**.

**On-prem (production path — §12):** do not expose 8082/8083 directly; put NGINX on 5000 in front of the group connector (see §12 for the full `nexus-docker.conf`). Clients then use `<NEXUS_FQDN>:5000` and the internal CA makes TLS work.

**Cloud / quick-test (AWS) path — insecure registry:**

1. Open OS firewall **and** AWS Security Group for 8082 and 8083:
   ```bash
   firewall-cmd --permanent --add-port=8082/tcp
   firewall-cmd --permanent --add-port=8083/tcp
   firewall-cmd --reload
   ```
   AWS Console → EC2 → Security Groups → add inbound TCP 8082 and 8083 from your IP.

2. On the Docker client, mark the registry as insecure (`/etc/docker/daemon.json`):
   ```json
   { "insecure-registries": ["<ec2-public-ip>:8082", "<ec2-public-ip>:8083"] }
   ```
   ```bash
   sudo systemctl restart docker
   ```

3. Use it:
   ```bash
   docker login <ec2-public-ip>:8083
   docker pull  <ec2-public-ip>:8083/library/alpine:3.19          # via group
   docker tag   myapp:1.0 <ec2-public-ip>:8082/myteam/myapp:1.0
   docker push  <ec2-public-ip>:8082/myteam/myapp:1.0             # direct to hosted
   ```

> The insecure-registry shortcut is **test only**. Move to §12 (NGINX + TLS + port 5000) before any real use — even for an internal dev instance.

### 20.3 PyPI (pypi.org)

**Create in UI:**

| Repo | Recipe | Key settings |
|---|---|---|
| `pypi-proxy` | `pypi (proxy)` | Remote: `https://pypi.org/` |
| `pypi-hosted` | `pypi (hosted)` | For internal wheels uploaded via `twine` |
| `pypi-group` | `pypi (group)` | Members: `pypi-hosted`, `pypi-proxy` |

**Client (`~/.pip/pip.conf` on Linux/macOS, `%APPDATA%\pip\pip.ini` on Windows):**
```ini
[global]
index-url = <NEXUS_URL>/repository/pypi-group/simple/
trusted-host = nexus.dev.contoso.internal    # only needed if no TLS, or internal CA not trusted
```

```bash
pip install requests
# upload a private wheel
twine upload --repository-url <NEXUS_URL>/repository/pypi-hosted/ dist/*
```

### 20.4 Other common formats (same 3-repo pattern)

| Format | Recipe | Upstream URL for proxy | Already pre-created? |
|---|---|---|---|
| Maven Central | `maven2 (proxy)` | `https://repo1.maven.org/maven2/` | Yes — `maven-central` (see §13) |
| NuGet v3 | `nuget (proxy)` | `https://api.nuget.org/v3/index.json` | Yes — `nuget.org-proxy` |
| Helm | `helm (proxy)` | e.g. `https://charts.bitnami.com/bitnami` | No |
| apt (Debian/Ubuntu) | `apt (proxy)` | e.g. `http://archive.ubuntu.com/ubuntu/` | No — needs GPG keypair configured |
| yum (RHEL/CentOS) | `yum (proxy)` | e.g. `https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/` | No |
| Raw (anything HTTP) | `raw (proxy/hosted/group)` | Any generic HTTP root | No |
| Go modules | `go (proxy/group)` | `https://proxy.golang.org` | No |

Repeat the **proxy + hosted + group** recipe for each format you need. The group URL is always the one clients consume.

### 20.5 Users, roles, and tokens for clients

Before you point CI at Nexus, create non-admin service users:

UI → **Security → Users → Create local user**:

| User | Typical roles | Use |
|---|---|---|
| `svc-ci-deploy` | `nx-repository-view-maven2-*-*` + `nx-repository-view-docker-*-*` + `nx-apikey-all` | Jenkins / GitHub Actions pushing artifacts + images |
| `svc-ci-read` | `nx-repository-view-*-*-read` | Resolve-only user for builds that only pull |
| Individual devs | `nx-anonymous` off, use AD mapping (§14.6) | Human identities |

For integrations that only accept tokens (npm, PyPI, Docker), the user generates a **User Token** from their profile (top-right avatar → **User token**) once anonymous access is disabled. In CI, store these as secrets — never commit them.

### 20.6 Cleanup, tasks, and capacity management

After a few weeks the blob store balloons. Set this up early:

1. UI → **Repository → Cleanup Policies → Create**: one per format, e.g. *"npm: delete if not downloaded in 60 days"*.
2. Attach the policy to every proxy repo (**Repositories → repo → Cleanup**).
3. UI → **System → Tasks → Create task**:
   - **Admin - Cleanup repositories using their associated policies** — daily.
   - **Admin - Compact blob store** — weekly (off-hours; I/O heavy).
   - **Admin - Export databases for backup** — daily (see §15).

### 20.7 Works the same on-prem and in cloud

Everything in §20 is UI/REST only — it does not care whether the VM is on VMware, RHEL on bare metal, or EC2. The only differences between on-prem and cloud are:

| Concern | On-prem (RHEL VM) | AWS / cloud |
|---|---|---|
| URL exposed to clients | `<NEXUS_URL> = https://<NEXUS_FQDN>` via §11 NGINX TLS | `http://<ec2-public-ip>:8081` (test) or NGINX+ACM behind an ALB (real) |
| Docker registry host | `<NEXUS_FQDN>:5000` | `<ec2-public-ip>:8083` (insecure) or an ALB/NLB fronted FQDN |
| Firewall to open | `firewalld` on the VM (`firewall-cmd --add-port=...`) | `firewalld` **plus** the AWS Security Group inbound rules |
| CA trust for clients | Internal root CA → `update-ca-trust` and `/etc/docker/certs.d/…/ca.crt` | Public ACM cert (ALB) works out of the box; otherwise same as on-prem |
| Stable address | Static IP assigned by network team (doesn't move) | **Attach an Elastic IP** — public IPv4 changes on every stop/start otherwise |

---

## 21. Troubleshooting walkthroughs (problems we have actually seen)

§18 is the quick-reference matrix. This section is the diagnostic order for the failures that repeatedly bite people on first install, with the exact commands to run.

### 21.1 "Can Nexus see me? Am I running?" — baseline service checks

Always start here before chasing network issues:

```bash
# Unit-file view
systemctl status nexus --no-pager -l
systemctl is-active nexus        # active
systemctl is-enabled nexus       # enabled

# Is something listening on 8081, and as whom?
sudo ss -tlnp | grep 8081
# Expect:  LISTEN 0 50  *:8081 *:*  users:(("java",pid=...,fd=...))

# Local HTTP — proves the JVM answered
curl -I http://localhost:8081
# Expect: HTTP/1.1 200 OK  Server: Nexus/<ver> (OSS)

# Live application log
sudo tail -f /nexus-data/sonatype-work/nexus3/log/nexus.log
# (for tarballs installed into /opt/sonatype/nexus-<ver>/ with no dedicated disk:
#  /opt/sonatype/sonatype-work/nexus3/log/nexus.log)
```

Interpret:
- `LISTEN` binding is `127.0.0.1:8081` (not `*:8081`) → Nexus only listens on localhost. Fix in `nexus.properties`: `application-host=0.0.0.0`, restart (§7).
- No LISTEN at all + active service → JVM still starting (wait up to ~2 min on first boot), or silently failed (check `nexus.log`).
- `systemctl status` shows `Active: failed` → jump to §21.2 based on the reason line.

### 21.2 `Active: failed (Result: oom-kill)` — the Linux OOM killer took Nexus out

Symptom (from `systemctl status nexus`):
```
Active: failed (Result: oom-kill) ...
Main PID: 1327 (code=killed, signal=KILL)
nexus.service: A process of this unit has been killed by the OOM killer.
```

Meaning: the kernel ran out of RAM and force-killed the Nexus JVM to keep the system alive. This is **not** a Nexus bug — the VM is undersized. Nexus 3.71+ wants ~4 GB RAM minimum.

**Confirm:**
```bash
free -h
sudo dmesg -T | grep -iE 'oom|killed process' | tail
# You'll see "Out of memory: Killed process ... java"
```

**Fix — on-prem (VMware):** ask the VMware team to increase the VM to the sizing in §1 (8 GB RAM for a small/medium instance). Power off, raise RAM, power on. No data loss.

**Fix — AWS:** stop the instance, change instance type to at least `t3.medium` (4 GB) or `t3.large` (8 GB), start again. Full procedure:

1. EC2 Console → Instances → select → **Instance state → Stop instance**. Wait until `stopped`.
2. **Actions → Instance settings → Change instance type** → pick e.g. `t3.large` → Save.
3. **Instance state → Start instance**.
4. SSH back in, `free -h` to confirm, `systemctl status nexus` (auto-starts because it's `enabled`).

**Caveats on stop/start in AWS:**
- **Public IP changes** unless you've allocated an **Elastic IP**. Your security group, DNS, and client-side configs that use the old IP all break.
- **Private IP stays the same** (within the same subnet).
- **EBS-backed data (`/nexus-data`, `/opt/sonatype`) persists.** Instance-store data would be lost — Nexus wouldn't be installed on that anyway.

**Last-resort shrink (not recommended):** on a ≤2 GB VM you can try lowering the heap, but Nexus will be slow and flaky:
```bash
sed -i 's/^-Xms.*/-Xms512M/; s/^-Xmx.*/-Xmx512M/; s/^-XX:MaxDirectMemorySize=.*/-XX:MaxDirectMemorySize=512M/' \
  /opt/sonatype/nexus-3.71.0-06/bin/nexus.vmoptions
systemctl restart nexus
```
Better: resize.

### 21.3 `curl -I http://localhost:8081` — `Connection refused`

On the Nexus VM itself. This means nothing is listening on 8081 locally.

```bash
# 1. Is the service even running?
systemctl status nexus --no-pager -l

# 2. If running, where are the logs? On some installs the path is different.
sudo find / -name "nexus.log"      2>/dev/null
sudo find / -type d -name "sonatype-work" 2>/dev/null
# Common real locations:
#   /nexus-data/sonatype-work/nexus3/log/nexus.log   (dedicated disk, this runbook)
#   /opt/sonatype/sonatype-work/nexus3/log/nexus.log (no dedicated disk)

# 3. Service logs (always available, even if nexus.log doesn't exist yet)
sudo journalctl -u nexus -n 200 --no-pager
```

Common outcomes and fixes:

| What you see | Cause | Fix |
|---|---|---|
| `tail: /opt/sonatype/sonatype-work/nexus3/log/nexus.log: No such file or directory` | You're looking at the wrong install path. | Use `find` above to locate the real `sonatype-work/`. Typically `/opt/sonatype/nexus-<ver>/` is installed, `sonatype-work/` is a sibling or under `/nexus-data/`. |
| `No suitable Java Virtual Machine could be found ... must be 17` | Missing / wrong Java, or `INSTALL4J_JAVA_HOME_OVERRIDE` not set | Re-do §2.1 and §6. |
| `Permission denied` writing sonatype-work | Ownership wrong | `chown -R nexus:nexus /opt/sonatype /nexus-data` |
| `Active: failed (Result: oom-kill)` | RAM exhausted | §21.2 |
| Service running, LISTEN shows only `127.0.0.1:8081` | Bound to loopback | §7 — set `application-host=0.0.0.0` |

### 21.4 From outside the VM: `Connection refused` / `Operation timed out`

Interpret the error literally — they mean different things:

| Error from `nc -vz <host> 8081` (or browser) | What it means |
|---|---|
| `succeeded!` / HTTP 200 | Network is fine; problem (if any) is application-level. |
| `Operation timed out` | A firewall silently dropped the packet. Something between you and the VM. |
| `Connection refused` | Something **answered** but nothing is listening on that port there — often you're reaching the wrong host (e.g. a recycled public IP). |
| `Host is unreachable` / `No route to host` | Routing/DNS problem; wrong IP, VPN not connected, etc. |

**Diagnostic order:**

1. **Confirm you have the right address right now.**
   - On-prem: `ip -br a` on the VM matches what your DNS resolves to? `dig +short nexus.dev.contoso.internal` returns the right IP?
   - AWS: the public IP **changes on every stop/start** unless an Elastic IP is attached. Re-check:
     ```bash
     # From inside the instance (IMDSv2)
     TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
       -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
     curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
       http://169.254.169.254/latest/meta-data/public-ipv4; echo
     ```
     Or Console → EC2 → Instances → your instance → **Public IPv4 address**.

2. **Prove the VM itself is healthy — from the VM:**
   ```bash
   curl -I http://localhost:8081          # should be 200
   curl -I http://<vm-own-public-or-fqdn>:8081   # should also be 200
   ```
   If `localhost:8081` works but `<public-ip>:8081` from the VM itself doesn't, it's a **host firewall** issue (§21.5), not a cloud/SG issue.

3. **Test the port from your workstation:**
   ```bash
   nc -vz <host> 8081
   ```
   - Timed out → firewall in the path (host firewalld §21.5, AWS Security Group §21.6, corporate egress filter).
   - Refused → wrong IP / recycled address / service bound to loopback only.

### 21.5 `firewalld` on RHEL is blocking 8081 (or 5000, 8082, 8083)

Symptom: from the VM, `curl localhost:8081` works. From outside, `nc -vz` times out. `systemctl status firewalld` shows it active.

```bash
sudo systemctl status firewalld
sudo firewall-cmd --list-all
```

If the `ports:` line doesn't include `8081/tcp`, open it:
```bash
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all | grep ports
```

Remember to open every port you use:
- `8081/tcp` — Nexus UI + HTTP API
- `443/tcp` (add `--add-service=https`) — if NGINX TLS from §11 is in front
- `5000/tcp` — Docker registry via NGINX (§12)
- `8082, 8083/tcp` — only if exposing Docker connectors directly (§20.2 cloud path)

### 21.6 AWS Security Group is blocking the port

Symptom: `firewalld` allows 8081 and `curl localhost:8081` works on the VM, but from your Mac `nc -vz <public-ip> 8081` times out.

Fix — AWS Console → **EC2 → Instances → your instance → Security tab → click the SG** → **Edit inbound rules → Add rule**:

| Field | Value |
|---|---|
| Type | Custom TCP |
| Port range | `8081` (repeat for 5000 / 8082 / 8083 / 443 as needed) |
| Source | **My IP** (locks to your current public egress IP) or a tighter CIDR; avoid `0.0.0.0/0` |
| Description | `Nexus UI` |

Common misses:
- Edited the wrong SG — an instance can have multiple; check the Security tab on the instance page to see which ones are attached.
- Your own public IP changed (home/office). Re-check with https://checkip.amazonaws.com, update the rule.
- NACLs on the subnet explicitly deny the port (rare in default VPCs; check VPC → Network ACLs).

### 21.7 EC2 public IP disappeared after stop/start

Symptom: after stopping+starting the instance, IMDSv2 returns an empty `public-ipv4`, or the Console shows a blank **Public IPv4 address**.

Cause: the instance is in a subnet/launch configuration without **auto-assign public IPv4**, and no Elastic IP is attached. Stopping releases the ephemeral public IP; on start, the subnet default decides whether a new one is given.

**Fix — attach an Elastic IP (do this once, permanently solves it):**

1. AWS Console → **EC2 → Elastic IPs → Allocate Elastic IP address** → Allocate.
2. Select the new EIP → **Actions → Associate Elastic IP address** → Resource type: Instance → pick your Nexus instance → **Associate**.
3. Update:
   - Your DNS (`nexus.dev.contoso.internal` A-record if you use one) to the EIP.
   - Any `/etc/hosts` pins on client machines.
   - Security Group rule sources if they restricted by your IP (those don't change, but double-check).
4. `nc -vz <eip> 8081` from your workstation to confirm.

The EIP survives stop/start and reboot. It's released only when you explicitly dissociate and release it.

### 21.8 UI works but Docker client fails

Most common on first use of §12 / §20.2:

| Error from `docker login/pull/push` | Fix |
|---|---|
| `x509: certificate signed by unknown authority` | Internal CA not trusted on the client. Copy `<INTERNAL_CA_CRT>` to `/etc/docker/certs.d/<fqdn>:5000/ca.crt` and `sudo systemctl restart docker`. |
| `Get "https://<host>:5000/v2/": http: server gave HTTP response to HTTPS client` | You're hitting the raw Nexus connector (HTTP) but Docker expects TLS. Either put NGINX in front with TLS (§12) or add `"insecure-registries": ["<host>:<port>"]` to `/etc/docker/daemon.json`. |
| `denied: access to the requested resource is not authorized` | Missing **Docker Bearer Token Realm** (Security → Realms), or user lacks the role for that repo. |
| `413 Request Entity Too Large` on push | NGINX default 1 MB body limit — set `client_max_body_size 4G;` in the server block (§12) and reload. |
| Push hangs then times out | NGINX proxy timeouts too low — raise `proxy_read_timeout` to at least 900s (§12). |

### 21.9 Diagnostic cheat sheet (copy-paste)

Run the whole block when you don't know where to start:

```bash
echo "=== systemd ==="
systemctl status nexus --no-pager -l | head -30
echo "=== listening sockets ==="
sudo ss -tlnp | grep -E ':(8081|8082|8083|5000|443)\b' || echo "nothing listening"
echo "=== local HTTP ==="
curl -sS -I http://localhost:8081 | head -5 || true
echo "=== firewall ==="
sudo firewall-cmd --list-all 2>/dev/null | grep -E 'services|ports' || echo "firewalld not active"
echo "=== memory ==="
free -h
echo "=== last nexus.log lines ==="
sudo tail -n 50 /nexus-data/sonatype-work/nexus3/log/nexus.log 2>/dev/null \
  || sudo tail -n 50 /opt/sonatype/sonatype-work/nexus3/log/nexus.log 2>/dev/null \
  || echo "nexus.log not found — run: sudo find / -name nexus.log 2>/dev/null"
echo "=== last journal lines ==="
sudo journalctl -u nexus -n 50 --no-pager
```

Paste the output into whatever ticket/chat you're escalating to — it covers every layer from systemd down to the application log in one go.

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
