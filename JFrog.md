# JFrog Artifactory OSS (stable) — Docker Compose + PostgreSQL

> This document shows a working Docker Compose setup for Artifactory OSS (stable), plus step-by-step installation and the exact fixes we applied (including resolving the **"Master key is missing"** problem).

---

## Overview

* Artifactory image used: `releases-docker.jfrog.io/jfrog/artifactory-oss:7.77.3` (stable recommendation)
* PostgreSQL: `postgres:15`
* Persistence: host volumes (required!)
* Important: **master key** must persist across restarts or migrations. If missing, Artifactory will wait and fail to fully start.

---

## Files & directories

Create a workspace and required folders on the host before running compose:

```bash
mkdir -p ~/jfrog/artifactory-data
mkdir -p ~/jfrog/postgres-data
cd ~/jfrog
```

Set ownership so the container can write files (Artifactory user inside container uses UID/GID 1030 by default):

```bash
sudo chown -R 1030:1030 artifactory-data postgres-data
```

---

## docker-compose.yml (ready-to-run)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: artifactory-postgres
    environment:
      POSTGRES_DB: artifactory
      POSTGRES_USER: artifactory
      POSTGRES_PASSWORD: artifactory
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: always

  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-oss:7.77.3
    container_name: artifactory
    depends_on:
      - postgres
    environment:
      EXTRA_JAVA_OPTIONS: "-Xms512m -Xmx2g"
      DB_TYPE: postgresql
      DB_DRIVER: org.postgresql.Driver
      DB_URL: jdbc:postgresql://postgres:5432/artifactory
      DB_USER: artifactory
      DB_PASSWORD: artifactory
      # Bind to 0.0.0.0 inside container (default) — do NOT change unless you know what you're doing
    ports:
      - "8081:8081"
      - "8082:8082"
    volumes:
      - ./artifactory-data:/var/opt/jfrog/artifactory
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "3"
```

**Notes**:

* We use host-mounted `./artifactory-data` so the `master.key`, certificates, configuration and DB metadata persist.
* The `healthcheck` for Postgres helps ensure database readiness; you can `docker-compose up -d` and watch logs.

---

## Start (clean, first-time run)

```bash
# from ~/jfrog
docker-compose pull
docker-compose up -d

# follow logs
docker logs -f artifactory
```

On first run, Artifactory will initialize DB schemas and generate security keys (including the `master.key`). This file will be written inside the mounted `./artifactory-data` path at:

```
./artifactory-data/var/etc/security/master.key
```

**DO NOT remove this file** — keep it safe and included in backups. If this file is missing or inaccessible on subsequent starts, you will see the `Master key is missing. Pending ...` messages.

---

## Common checks

1. Show container status:

```bash
docker ps --filter name=artifactory --filter name=artifactory-postgres
```

2. Check Artifactory logs for readiness:

```bash
docker logs --tail 200 artifactory
```

3. Check Postgres health:

```bash
docker inspect --format='{{json .State.Health.Status}}' artifactory-postgres
```

---

## Fix: "Master key is missing" (root cause & solutions)

**Root causes**:

* Artifactory's persistent volume doesn't contain `master.key` (new container, lost data, or wrong mount path)
* Wrong file ownership/permissions prevent the container from reading the key
* You migrated data but didn't copy `master.key` from old installation

### Solution A — If this is a fresh install and you have no previous data

1. Ensure `./artifactory-data` is empty before first start (so the container can generate keys):

```bash
sudo rm -rf ./artifactory-data/*
sudo chown -R 1030:1030 ./artifactory-data
```

2. Restart the container:

```bash
docker-compose down
docker-compose up -d
```

Artifactory will generate the master key automatically.

### Solution B — If this is a migration / you have old data

1. Copy `master.key` from the old installation into the host volume path (preserve path):

```bash
# assuming old master.key at /backup/master.key
mkdir -p ./artifactory-data/var/etc/security
cp /path/to/old/master.key ./artifactory-data/var/etc/security/master.key
sudo chown 1030:1030 ./artifactory-data/var/etc/security/master.key
sudo chmod 600 ./artifactory-data/var/etc/security/master.key
```

2. Ensure the rest of the `var` tree (etc, data) is present and owned by UID 1030:

```bash
sudo chown -R 1030:1030 ./artifactory-data
```

3. Restart Artifactory:

```bash
docker-compose restart artifactory
```

Artifactory will find `master.key` and continue startup.

### Solution C — Permission issues (container cannot read file)

```bash
sudo chown -R 1030:1030 ./artifactory-data
sudo chmod -R 700 ./artifactory-data/var/etc/security
sudo chmod 600 ./artifactory-data/var/etc/security/master.key
docker-compose restart
```

**Important**: the UID/GID `1030:1030` is used by the official Artifactory image in many distributions — if you used a different image/tag, check the container's process UID by starting the container and running `docker exec -it artifactory id` to confirm.

---

## Backing up / Restoring master.key

* Backup:

```bash
cp ./artifactory-data/var/etc/security/master.key ~/jfrog-backups/master.key-$(date +%F)
```

* Restore: copy back into `./artifactory-data/var/etc/security/master.key` and set permissions as above.

---

## Extra tips & troubleshooting

* If you see `Connection refused` to `http://localhost:8046` in logs, that is an internal service router issue usually transient during startup — confirm all services eventually register (`All services started successfully` in logs) and the frontend pings succeed.
* If the DB conversion/migration runs for a long time, watch logs under `docker logs artifactory` — large datasets take time.
* If you ever change `DB_*` env values, ensure the DB user has the correct privileges and you restart Artifactory.

---

## Validate success

1. Browser: `http://<your-server-ip>:8081` → Artifactory UI should load
2. Login with default admin (follow UI flows for setting password)
3. Create a local repo and try uploading an artifact
4. Verify logs show no `Master key is missing` messages and show `All services started successfully`.

---

## Useful commands

```bash
# view last 200 lines
docker logs --tail 200 artifactory

# follow logs
docker logs -f artifactory

# restart service
docker-compose restart artifactory

# bring down and up
docker-compose down
docker-compose up -d
```

---

## Appendix: Where files live inside container

* `/var/opt/jfrog/artifactory/var/etc/security/master.key`  (on-host mapped to `./artifactory-data/var/etc/security/master.key`)
* `/var/opt/jfrog/artifactory/var/log/` logs

---

If you want, I can now:

* export this file as a downloadable `.md` in the workspace,
* generate an updated PDF including the exact logs you pasted,
* or create a tarball with this `docker-compose.yml` and a small README for you to download.

Tell me which one you want next.
