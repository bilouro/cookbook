# Metabase on macOS Using Docker + PostgreSQL (Cookbook)

## 1. Overview

This cookbook describes how to run **Metabase** on macOS using **Docker**, backed by **PostgreSQL** as its application database — replacing the default embedded H2 database.

- Uses **Docker Desktop for macOS** as the container engine.
- Uses the official `metabase/metabase` image.
- Stores Metabase's own metadata (dashboards, questions, users) in a dedicated **PostgreSQL database**.
- Connects Metabase and PostgreSQL via a **dedicated Docker network** (`metabase-net`).
- Persists Metabase application data in a **named Docker volume** (`metabase_data`).
- Includes step-by-step migration from H2 to PostgreSQL for existing installations.

---

## 2. Rationale (Design & Best Practices)

### 2.1 Why Replace H2 with PostgreSQL?

Metabase ships with an embedded H2 database for quick start. However, H2 is not suitable for production use:

- H2 does not support concurrent access — multiple Metabase instances will corrupt data.
- H2 is stored inside the container by default — data is lost if the container is removed.
- Backups require stopping the container; PostgreSQL supports live backups (pg_dump).
- Migrating H2 → PostgreSQL later is more complex than starting right.

Use PostgreSQL from the start for anything beyond local experimentation.

### 2.2 Dedicated Docker Network

Metabase must reach the PostgreSQL container by name (e.g., `postgres-db`). Docker name resolution only works when containers share the same user-defined network. The default `bridge` network does not support DNS between containers.

Create a dedicated network (`metabase-net`) and connect both containers to it.

### 2.3 Dedicated PostgreSQL Database

Metabase's internal metadata (its own dashboards, users, etc.) should be isolated from your application data. Create a separate database (e.g., `metabase`) and a dedicated user (`metabase_user`) inside your existing PostgreSQL container.

### 2.4 Named Volume for Metabase

Even when using PostgreSQL as the application DB, Metabase still writes some files to disk (plugins, logs, temp files). Using a named volume (`metabase_data`) ensures these persist across container restarts.

---

## 3. Prerequisites

- macOS with Docker Desktop installed and running.
- A running PostgreSQL container (see [PostgreSQL Cookbook](postgresql.MD)).
- The PostgreSQL container is named `postgres-db` in this cookbook; adjust if yours differs.
- Know your PostgreSQL superuser credentials (used only to create the Metabase DB and user).

---

## 4. Create the Docker Network

Create a dedicated network that both Metabase and PostgreSQL will share:

```bash
docker network create metabase-net
```

Connect your existing PostgreSQL container to this network:

```bash
docker network connect metabase-net postgres-db
```

Verify both containers will be on the same network:

```bash
docker network inspect metabase-net
```

---

## 5. Create the Metabase Database in PostgreSQL

Connect to your PostgreSQL container and create a dedicated database and user for Metabase:

```bash
docker exec postgres-db psql -U app_user -d app_db -c "CREATE USER metabase_user WITH PASSWORD 'metabase_pass';"
docker exec postgres-db psql -U app_user -d app_db -c "CREATE DATABASE metabase OWNER metabase_user;"
docker exec postgres-db psql -U app_user -d app_db -c "GRANT ALL PRIVILEGES ON DATABASE metabase TO metabase_user;"
```

> Replace `app_user` and `app_db` with your PostgreSQL superuser and any existing database. Replace `metabase_pass` with a strong password before going to production.

Verify the database was created:

```bash
docker exec postgres-db psql -U app_user -d app_db -c "\l"
```

You should see `metabase` in the list.

---

## 6. Run Metabase with PostgreSQL

### 6.1 Create a Persistent Volume

```bash
docker volume create metabase_data
```

### 6.2 Start Metabase

```bash
docker run -d \
  --name metabase \
  --restart=unless-stopped \
  -p 3000:3000 \
  -v metabase_data:/metabase-data \
  -e MB_DB_TYPE=postgres \
  -e MB_DB_HOST=postgres-db \
  -e MB_DB_PORT=5432 \
  -e MB_DB_DBNAME=metabase \
  -e MB_DB_USER=metabase_user \
  -e MB_DB_PASS=metabase_pass \
  --network metabase-net \
  metabase/metabase
```

### 6.3 Verify Startup

```bash
docker logs -f metabase
```

Wait for:

```
Metabase Initialization COMPLETE
```

Access Metabase at:

```
http://localhost:3000
```

---

## 7. Migrate Existing Metabase from H2 to PostgreSQL

Skip this section if this is a fresh installation.

### 7.1 Locate the H2 Database File

If your existing Metabase container stores H2 data inside the container (no named volume):

```bash
docker exec metabase find / -name "metabase.db.mv.db" 2>/dev/null
```

Typical location: `/metabase.db/metabase.db.mv.db`

### 7.2 Backup the H2 File to the Host

```bash
mkdir -p ~/metabase-h2-backup
docker cp metabase:/metabase.db/metabase.db.mv.db ~/metabase-h2-backup/
```

### 7.3 Copy H2 File into the Running Container for Migration

```bash
docker cp ~/metabase-h2-backup/metabase.db.mv.db metabase:/tmp/metabase.db.mv.db
```

### 7.4 Connect the Metabase Container to the Network

```bash
docker network connect metabase-net metabase
```

### 7.5 Run the Migration

This command reads from the H2 file and writes all data into PostgreSQL:

```bash
docker exec \
  -e MB_DB_TYPE=postgres \
  -e MB_DB_HOST=postgres-db \
  -e MB_DB_PORT=5432 \
  -e MB_DB_DBNAME=metabase \
  -e MB_DB_USER=metabase_user \
  -e MB_DB_PASS=metabase_pass \
  metabase \
  java -jar /app/metabase.jar load-from-h2 /tmp/metabase.db
```

Wait for the final line:

```
Setting Postgres sequence ids to proper values...
[OK]
```

### 7.6 Stop and Remove the Old Container

```bash
docker stop metabase
docker rm metabase
```

### 7.7 Start Metabase with PostgreSQL (Section 6.2)

Follow **Section 6** above. All your dashboards, questions, and users will be present.

---

## 8. Daily Operations

### 8.1 Check Running Containers

```bash
docker ps
```

### 8.2 View Metabase Logs

```bash
docker logs metabase
docker logs -f metabase   # follow in real time
```

### 8.3 Stop, Start, Restart

```bash
docker stop metabase
docker start metabase
docker restart metabase
```

### 8.4 Open a Shell Inside the Container

```bash
docker exec -it metabase bash
```

---

## 9. Upgrading Metabase

1. Pull the latest image:

```bash
docker pull metabase/metabase
```

2. Stop and remove the old container:

```bash
docker stop metabase
docker rm metabase
```

3. Start a new container using the same volume and env vars (Section 6.2). Metabase will auto-migrate its schema on startup.

---

## 10. Backups

Metabase data lives in PostgreSQL. Back up the `metabase` database using `pg_dump`:

```bash
docker exec postgres-db pg_dump -U metabase_user metabase > ~/metabase-backup-$(date +%F).sql
```

Restore with:

```bash
docker exec -i postgres-db psql -U metabase_user -d metabase < ~/metabase-backup-<date>.sql
```

---

## 11. Maintenance and Best Practices Summary

- **Network isolation**: Always use a user-defined network. The default `bridge` network does not support container DNS.
- **Dedicated database**: Never share the Metabase internal database with your application data.
- **Named volume**: Even with PostgreSQL, always use a named volume (`metabase_data`) so plugin files and logs persist.
- **Backups**: Back up the PostgreSQL `metabase` database regularly with `pg_dump`, not the container filesystem.
- **Credentials**: Change `metabase_pass` to a strong password before using in any shared or production environment.
- **H2 backup**: Before migrating, always copy the H2 file to the host first (`docker cp`) as a safety net.
