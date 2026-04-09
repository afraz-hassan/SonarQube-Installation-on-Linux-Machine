# SonarQube Enterprise Edition Upgrade Documentation

**Environment:** Linux VM  
**Installation Type:** ZIP-based  
**Edition:** Enterprise Edition  
**Target Version:** Latest Stable LTA (2026.1.x)

---

## 1. Scope

Use this documentation if:

- You are running **SonarQube Enterprise Edition**
- On a **Linux VM**
- Installed from the **ZIP distribution**
- Using an external production database (**PostgreSQL**)

This is the right model for your setup. SonarQube supports Linux for Server editions, and its ZIP installation and update procedure applies to Enterprise Edition.

---

## 2. Important Rule Before You Start

Your exact upgrade path depends on your current version.

### SonarQube upgrade-path rules

- If there is **no LTA** in the path, some non-LTA to non-LTA upgrades can be direct.
- If there is an **LTA** in the path, you must pass through each required intermediate LTA.
- When upgrading to an LTA, always go to the **latest patch** of that LTA.

### Examples

- `2025.1 LTA -> 2026.1 LTA` can be done directly.
- `9.9 LTA -> 2025.1 LTA -> 2026.1 LTA` requires the intermediate LTA.

### Supported paths

- If current version is `9.9.x LTA`:

  `9.9.x LTA -> 2025.1 latest patch -> 2026.1 latest patch`

- If current version is `2025.1.x LTA`:

  `2025.1.x LTA -> 2026.1 latest patch`

If you are on some other version, validate it against SonarQube’s official update-path rules before starting.

---

## 3. Pre-Upgrade Checklist

Complete all of the following before scheduling downtime.

### 3.1 Confirm current state

Record:

- SonarQube edition: **Enterprise**
- Current SonarQube version
- Linux OS version
- Database type and version
- Java version
- Service name and installation path
- Reverse proxy / DNS / TLS configuration
- Installed third-party plugins

This matters because 2026.1 introduces runtime and platform changes, including Java and database support changes.

### 3.2 Validate Java

For SonarQube **2026.1 LTA**, the server requires:

- **JDK**, not just JRE
- **Java 21 minimum**
- **Java 17 is no longer supported**

Check Java with:

```bash
java -version
javac -version
```

### 3.3 Validate database support

For current SonarQube Server releases:

- PostgreSQL **14–18** are supported
- PostgreSQL 13 is no longer supported
- MSSQL **2017 / 2019 / 2022** are supported
- MSSQL 2016 is no longer supported

If your database version is older than the supported versions, upgrade the database first.

### 3.4 Validate Linux host prerequisites

SonarQube on Linux requires Elasticsearch-related kernel and file descriptor settings.

Required minimums:

- `vm.max_map_count >= 524288`
- `fs.file-max >= 131072`
- SonarQube user open files `>= 131072`
- SonarQube user threads `>= 8192`

Check them with:

```bash
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u
```

For 2026.1 and later, also ensure `/tmp` is writable because Elasticsearch 8.x requires it:

```bash
ls -ld /tmp
touch /tmp/sonarqube_test && rm -f /tmp/sonarqube_test
```

### 3.5 Validate plugins

Do **not** blindly copy old plugins into the new installation.

Only install plugins that are confirmed compatible through the plugin compatibility matrix.

### 3.6 Dry run in staging (recommended)

Best practice for production is:

- Restore a recent production DB backup in non-production
- Clone your current SonarQube configuration
- Test the full upgrade path first
- Run sample analyses afterward

This is strongly recommended before touching production.

---

## 4. High-Level Plan

Your upgrade sequence is:

1. Prepare backups and OS/JVM prerequisites
2. Install **OpenJDK 21 JDK** on the VM
3. Download the **Enterprise Edition 2026.1 LTA ZIP**
4. Unzip it into a **new directory**, not over the old one
5. Copy settings from the old `sonar.properties` into the new one manually
6. Stop the old SonarQube
7. Point the service / symlink to the new install
8. Start the new SonarQube
9. Open `http://<sonarqube-vm-ip>:9000/setup` and run the DB migration
10. Validate logs, health, and reindexing, then resume scans

---

# Phase A: Pre-Upgrade Preparation

## Step A1: Announce a maintenance window

Pause any CI/CD pipelines that send analyses to SonarQube during the change.

SonarQube recommends reanalyzing projects after the upgrade, and some pages can remain incomplete until reindexing finishes.

## Step A2: Confirm current state on the VM

Run these commands and save the output to a file:

```bash
hostname
date -u
lsb_release -a || cat /etc/os-release
java -version
javac -version
ps -ef | grep -i sonarqube | grep -v grep
ss -lntp | grep 9000
readlink -f /opt/sonarqube || ls -ld /opt/sonarqube
```

This gives you a quick baseline before touching anything.

## Step A3: Check Linux prerequisites required by SonarQube

Check:

```bash
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u
```

If values are below required minimums, apply:

```bash
sudo tee /etc/sysctl.d/99-sonarqube.conf >/dev/null <<'EOF'
vm.max_map_count=524288
fs.file-max=131072
EOF

sudo sysctl --system
```

Also set user limits:

```bash
sudo tee /etc/security/limits.d/99-sonarqube.conf >/dev/null <<'EOF'
sonarqube   -   nofile   131072
sonarqube   -   nproc    8192
EOF
```

If your service runs under a different user, replace `sonarqube` with that actual service user.

SonarQube must not run as root.

## Step A4: Check `/tmp` access

Verify `/tmp` access:

```bash
ls -ld /tmp
touch /tmp/sonarqube_upgrade_test && rm -f /tmp/sonarqube_upgrade_test
```

If that fails, fix `/tmp` permissions before proceeding.

---

# Phase B: Backups

## Step B1: Back up PostgreSQL

Because your database is local PostgreSQL on `localhost:5432/sonarqube` with user `sonaruser`, create a full dump before the change.

Create backup directory:

```bash
sudo mkdir -p /backup/sonarqube/$(date +%F)
sudo chown -R $USER:$USER /backup/sonarqube/$(date +%F)
```

Compressed dump:

```bash
PGPASSWORD='<DB_PASSWORD>' pg_dump   -h localhost   -U sonaruser   -d sonarqube   -Fc   -f /backup/sonarqube/$(date +%F)/sonarqube-db.dump
```

If you prefer interactive authentication:

```bash
pg_dump -h localhost -U sonaruser -d sonarqube -Fc   -f /backup/sonarqube/$(date +%F)/sonarqube-db.dump
```

Optional plain SQL export:

```bash
pg_dump -h localhost -U sonaruser -d sonarqube   -f /backup/sonarqube/$(date +%F)/sonarqube-db.sql
```

## Step B2: Back up the SonarQube install and service configuration

```bash
sudo cp -a /opt/sonarqube /backup/sonarqube/$(date +%F)/sonarqube-home
sudo cp -a /etc/systemd/system /backup/sonarqube/$(date +%F)/systemd-system-backup 2>/dev/null || true
sudo cp -a /lib/systemd/system /backup/sonarqube/$(date +%F)/lib-systemd-backup 2>/dev/null || true
```

If your VM platform supports snapshots, take one before the change as well.

---

# Phase C: Install Java 21 JDK

SonarQube 2026.1 requires Java 21 minimum and specifically requires a **JDK**, not just a JRE.

On Ubuntu 24.04:

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
```

Verify:

```bash
java -version
javac -version
update-alternatives --list java
update-alternatives --list javac
```

Do **not** switch the system default yet if other applications on the VM depend on Java 17.

Instead, point SonarQube specifically to Java 21 in the service definition unless you know it is safe to make Java 21 global.

---

# Phase D: Download the New Enterprise ZIP

SonarSource’s ZIP update procedure explicitly says to download and unzip the new version into a **fresh directory**.

Do not overwrite the old install.

Create a releases directory:

```bash
sudo mkdir -p /opt/releases
sudo chown -R $USER:$USER /opt/releases
cd /opt/releases
```

Download the **SonarQube Server Enterprise Edition 2026.1 LTA latest patch ZIP** from your SonarSource account or portal, then unzip it:

```bash
unzip sonarqube-enterprise-2026.1.*.zip
```

Move it to a versioned directory:

```bash
sudo mv sonarqube-2026.1.* /opt/sonarqube-2026.1
sudo chown -R sonarqube:sonarqube /opt/sonarqube-2026.1
```

If your runtime user is not `sonarqube`, replace it with the actual service user.

---

# Phase E: Prepare the New Configuration

SonarSource’s guidance is to copy the **settings**, not blindly replace the new configuration files with the old ones.

The important step is to merge the contents of the old `conf/sonar.properties` into the new file.

## Step E1: Compare old and new configuration

```bash
sudo diff -u /opt/sonarqube/conf/sonar.properties /opt/sonarqube-2026.1/conf/sonar.properties | less
```

## Step E2: Edit the new configuration

```bash
sudo cp /opt/sonarqube-2026.1/conf/sonar.properties /opt/sonarqube-2026.1/conf/sonar.properties.bak
sudo vi /opt/sonarqube-2026.1/conf/sonar.properties
```

Copy over your production settings from the current instance, especially:

- `sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube`
- `sonar.jdbc.username=sonaruser`
- `sonar.web.host=0.0.0.0`
- `sonar.web.port=9000`
- `sonar.path.home=/opt/sonarqube`
- `sonar.path.data=/opt/sonarqube/data`
- `sonar.path.logs=/opt/sonarqube/logs`
- `sonar.path.temp=/opt/sonarqube/temp`
- Your JVM options for Web / CE / Search if you want to keep them

### Important path decision

Because the current install uses paths inside `/opt/sonarqube` for data, temp, logs, and home, the cleanest production approach is:

- Keep the new binaries in `/opt/sonarqube-2026.1`
- Switch `/opt/sonarqube` to a symlink pointing to the new directory
- Keep the same configured paths and service references

This avoids rewriting many paths and matches a clean in-place operational model.

## Step E3: No plugin migration needed

If there are no third-party plugins, that simplifies the upgrade because SonarSource only requires manual plugin compatibility handling when plugins exist.

---

# Phase F: Check the Service Definition

Before stopping anything, identify how SonarQube is started:

```bash
systemctl list-unit-files | grep -i sonar
systemctl status sonarqube --no-pager
systemctl cat sonarqube
```

If the service name is different, adjust commands accordingly.

Look for these patterns in the unit:

- `ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start`
- `Environment=JAVA_HOME=...`
- `User=sonarqube`

If there is no explicit `JAVA_HOME`, add one so SonarQube uses Java 21 even if the OS default stays on Java 17.

Example systemd override:

```bash
sudo systemctl edit sonarqube
```

Add:

```ini
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
Environment="PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

This keeps the change scoped to SonarQube and is safer on shared VMs.

---

# Phase G: Cutover

## Step G1: Stop SonarQube

```bash
sudo systemctl stop sonarqube
sudo systemctl status sonarqube --no-pager
```

If systemd is not controlling it, SonarSource supports stopping via the ZIP startup script:

```bash
sudo -u sonarqube /opt/sonarqube/bin/linux-x86-64/sonar.sh stop
```

If it hangs:

```bash
sudo -u sonarqube /opt/sonarqube/bin/linux-x86-64/sonar.sh force-stop
```

## Step G2: Switch `/opt/sonarqube` to the new version

First rename the current directory so rollback remains simple:

```bash
sudo mv /opt/sonarqube /opt/sonarqube-2025.5-backup
```

Now create the symlink:

```bash
sudo ln -s /opt/sonarqube-2026.1 /opt/sonarqube
sudo chown -h sonarqube:sonarqube /opt/sonarqube
```

Verify:

```bash
readlink -f /opt/sonarqube
ls -ld /opt/sonarqube /opt/sonarqube-2026.1 /opt/sonarqube-2025.5-backup
```

## Step G3: Recreate writable runtime directories if needed

If the old install kept live `data`, `temp`, and `logs` inside `/opt/sonarqube`, ensure these now exist and have correct ownership in the new target:

```bash
sudo mkdir -p /opt/sonarqube/data /opt/sonarqube/temp /opt/sonarqube/logs
sudo chown -R sonarqube:sonarqube /opt/sonarqube/data /opt/sonarqube/temp /opt/sonarqube/logs
```

Because the prior install already used these directories, you may instead want to move them from the old tree if they contain persistent runtime content you want to preserve:

```bash
sudo rsync -a /opt/sonarqube-2025.5-backup/data/ /opt/sonarqube/data/
sudo rsync -a /opt/sonarqube-2025.5-backup/temp/ /opt/sonarqube/temp/
sudo rsync -a /opt/sonarqube-2025.5-backup/logs/ /opt/sonarqube/logs/
sudo chown -R sonarqube:sonarqube /opt/sonarqube/data /opt/sonarqube/temp /opt/sonarqube/logs
```

Given this is a single-node VM and PostgreSQL is the real source of truth, logs and temp are optional to preserve. Data is worth preserving if present.

## Step G4: Start the new version

```bash
sudo systemctl start sonarqube
sudo systemctl status sonarqube --no-pager
```

If using the script:

```bash
sudo -u sonarqube /opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

After startup, the next step is to open `/setup` and follow the setup instructions.

---

# Phase H: Run the DB Migration

Open:

```text
http://<sonarqube-vm-ip>:9000/setup
```

This is where the SonarQube setup flow runs the database upgrade.

Do not resume user traffic or pipelines until this completes successfully.

If the page does not load, check logs first.

---

# Phase I: Validate Logs and Health

Tail the logs in separate terminals:

```bash
sudo tail -f /opt/sonarqube/logs/sonar.log
sudo tail -f /opt/sonarqube/logs/web.log
sudo tail -f /opt/sonarqube/logs/ce.log
sudo tail -f /opt/sonarqube/logs/es.log
```

What you want to see:

- No Java runtime/version errors
- No DB migration failures
- No Elasticsearch startup failures
- No filesystem permission failures
- Normal startup completion

Java 21 and `/tmp` access are the two most important upgrade-sensitive items for 2026.1.

### Quick API checks

Once the server is up:

```bash
curl -s http://localhost:9000/api/server/version
curl -s http://localhost:9000/api/system/health
```

You should see the new 2026.1 version and healthy status.

---

# Phase J: Reindexing and Post-Upgrade Checks

SonarSource recommends reanalyzing your projects after the upgrade for a better experience.

Reindexing may continue in the background, and some pages can be incomplete until it finishes.

Check in the UI:

- **Administration -> Projects -> Background Tasks**
- Confirm no failed tasks
- Verify users can log in
- Confirm a few key projects open correctly
- Run one .NET analysis and one non-.NET analysis if you use both

## Scanner consideration

Server-side runtime must be Java 21, which you are fixing on the VM.

For scanners, if you rely on JRE auto-provisioning there may be no impact. Otherwise scanners may also need Java 21 or newer depending on scanner/version. That becomes relevant only if older pipelines start failing after the server upgrade.

---

# Rollback Plan

If the upgrade fails and you need to revert:

1. Stop SonarQube
2. Restore the pre-upgrade PostgreSQL backup
3. Remove the `/opt/sonarqube` symlink
4. Point `/opt/sonarqube` back to `/opt/sonarqube-2025.5-backup`
5. Start the old SonarQube again

Commands:

```bash
sudo systemctl stop sonarqube
sudo rm -f /opt/sonarqube
sudo ln -s /opt/sonarqube-2025.5-backup /opt/sonarqube
sudo chown -h sonarqube:sonarqube /opt/sonarqube
sudo systemctl start sonarqube
```

### DB restore example

```bash
dropdb -h localhost -U sonaruser sonarqube
createdb -h localhost -U sonaruser sonarqube
pg_restore -h localhost -U sonaruser -d sonarqube /backup/sonarqube/<DATE>/sonarqube-db.dump
```

Do not try to run old binaries against a partially upgraded database. If you roll back, restore the database backup as part of the rollback.
