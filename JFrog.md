# JFrog Artifactory OSS Installation Guide (Docker Compose + Manual Master Key)

This guide explains how to install **JFrog Artifactory OSS (7.77.3)** using **Docker Compose**, **PostgreSQL**, and a **manual master key** to avoid initialization failures.

---

## **1. Prerequisites**

Before starting, ensure your system has:

* Ubuntu 20.04 / 22.04 / 24.04
* Docker Engine installed
* Docker Compose (v2+) installed
* Minimum: 4GB RAM, 20GB storage

Verify Docker installation:

```
docker --version
docker compose version
```

---

## **2. Create Required Directories**

Create a workspace for Artifactory:

```
mkdir -p ~/jfrog
cd ~/jfrog

mkdir -p artifactory-data
mkdir -p postgres-data
```

Set correct ownership:

```
sudo chown -R 1030:1030 artifactory-data
sudo chown -R 999:999 postgres-data
```

**Why these IDs?**

* Artifactory runs as user **1030** inside container
* PostgreSQL runs as user **999** inside container

---

## **3. Create docker-compose.yml**

Create a file named **docker-compose.yml** inside `~/jfrog` and add:

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15
    container_name: artifactory-postgres
    restart: always
    environment:
      POSTGRES_DB: artifactory
      POSTGRES_USER: artifactory
      POSTGRES_PASSWORD: artifactory
    volumes:
      - ./postgres-data:/var/lib/postgresql/data

  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-oss:7.77.3
    container_name: artifactory
    restart: always
    depends_on:
      - postgres
    environment:
      DB_TYPE: postgresql
      DB_DRIVER: org.postgresql.Driver
      DB_URL: jdbc:postgresql://postgres:5432/artifactory
      DB_USER: artifactory
      DB_PASSWORD: artifactory

      # Manual Master Key (Fixes initialization delays)
      ARTIFACTORY_MASTER_KEY: thisIsAStrongMasterKey123456789thisIsAStrongKey

    ports:
      - "8081:8081"
      - "8082:8082"
    volumes:
      - ./artifactory-data:/var/opt/jfrog/artifactory
```

---

## **4. Start Artifactory**

Run the following:

```
docker compose up -d
```

Check container status:

```
docker ps
```

Follow logs:

```
docker logs -f artifactory
```

If installation is successful, logs will show:

```
Using master key from environment
All services started successfully
```

---

## **5. Access Artifactory**

Open browser:

```
http://<your-server-ip>:8081
```

Proceed with the UI-based initial setup.

---

## **6. Important File Locations**

Inside host system:

```
./artifactory-data/var/opt/jfrog/artifactory
./postgres-data/
```

Inside Artifactory container:

```
/var/opt/jfrog/artifactory/var/etc/security/master.key
```

---

## **7. Common Commands**

Restart services:

```
docker compose restart
```

Stop services:

```
docker compose down
```

Delete all data (fresh install):

```
sudo rm -rf artifactory-data/* postgres-data/*
```

---

## **8. Troubleshooting Tips**

### **Issue: `Master key is missing`**

Solution: Manual master key solves this 100%.

### **Issue: Permission denied**

Fix ownership:

```
sudo chown -R 1030:1030 artifactory-data
```

### **Issue: DB connection refused**

Check Postgres:

```
docker logs artifactory-postgres
```

---

## **9. Summary**

By using a **manual master key**, you avoid:

* Initialization loops
* 5-minute timeouts
* Permission-related failures
* Missing `master.key` errors

Your JFrog Artifactory will now start cleanly and reliably on every reboot.

---

If you want, I can generate a **PDF version** of this README or add **images placeholders** for documentation.
