# PostgreSQL on macOS Using Docker (Cookbook)

## 1. Overview

This cookbook describes how to run **PostgreSQL** on macOS using **Docker**, following safe and maintainable practices:

- Uses **Docker Desktop for macOS** as the container engine.
- Uses the official `postgres` image (recommended tag: `postgres:16-alpine`).
- Persists database data in a **named Docker volume** (`pgdata_main`).
- Restricts database access to **localhost** (not exposed to the external network by default).
- Makes backup, restore, and version upgrades straightforward.

---

## 2. Rationale (Design & Best Practices)

### 2.1 One Service per Container

- In Docker, best practice is to run **one main service per container**.
- The PostgreSQL container should:
  - Only run the PostgreSQL server.
  - Not bundle other applications or services.
- Benefits:
  - Easier upgrades (you only upgrade PostgreSQL).
  - Easier troubleshooting (dedicated logs).
  - Less blast radius: if the DB misbehaves, it doesn’t directly affect other services.

### 2.2 Using Named Volumes for Persistence

- PostgreSQL stores its data in `/var/lib/postgresql/data` inside the container.
- If you don’t use a **volume**, removing the container will delete your data.
- Using a named volume (e.g. `pgdata_main`):
  - Persists data across container recreation.
  - Avoids performance and permission issues that can occur when mounting macOS directories directly.
  - Makes it easy to attach the data to a temporary container for inspection or backup.

### 2.3 Port Exposure (Basic Security)

- PostgreSQL listens on port **5432** inside the container.
- When mapping the port to the host:
  - Prefer `127.0.0.1:5432:5432` instead of `0.0.0.0:5432:5432`.
  - This ensures **only your Mac** can access the database.
- If you need to expose PostgreSQL to other machines on the network, do so consciously (e.g., `-p 5432:5432`) and protect access via firewall/VPN and proper authentication.

### 2.4 User, Database, and Password

- The `postgres` image uses environment variables to bootstrap:
  - `POSTGRES_DB` → creates an initial database.
  - `POSTGRES_USER` → creates a user (role).
  - `POSTGRES_PASSWORD` → sets that user’s password.
- Best practices:
  - Avoid using `postgres` superuser for everything; use a specific user (e.g., `app_user`).
  - Use strong passwords, even in development.
  - Limit permissions of the user to just the necessary database(s) if external apps will use it.

### 2.5 Image Choice

- Recommended image: `postgres:16-alpine`
  - `16`: recent, stable major version.
  - `alpine`: lightweight base image (smaller download, smaller attack surface).
- For more critical or reproducible environments:
  - Pin to an exact tag (e.g., `16.2-alpine`) instead of a moving tag like `16-alpine`.

### 2.6 Upgrade Strategy

- Separate:
  - **Data** → volume (`pgdata_main`).
  - **Software** → Docker image (`postgres:16-alpine`).
- To upgrade:
  - Pull a new image.
  - Stop and remove the old container.
  - Start a new container reusing the same volume.
- For **major version** upgrades (e.g., 15 → 16):
  - Take a `pg_dump` backup first.
  - Restore into a new container.
  - Test compatibility with your applications.

### 2.7 Backups

- Best practice: use **logical backups** with `pg_dump`.
- Advantages:
  - Independent of the exact internal data format.
  - Portable across environments and versions.
- Common flow:
  - Run `pg_dump` either from the host or inside the container.
  - Save `.sql` or `.dump` files to a backup directory (e.g. `~/pg-backups`).
- For disaster recovery, combine:
  - Regular logical backups.
  - Optionally, storage-level snapshots (if supported by your infrastructure).

---

## 3. Steps and Commands

### 3.1 Prerequisites

- macOS (Big Sur or newer recommended).
- Local administrator privileges.
- Internet access to download Docker and the PostgreSQL image.

#### 3.1.1 Install Docker Desktop (via Homebrew)

If you use Homebrew:

brew install --cask docker

Then:

1. Open the **Docker** app from Launchpad or Spotlight.
2. Wait until the whale icon in the top bar becomes stable (Docker fully started).

Alternatively, download directly from:

https://www.docker.com/products/docker-desktop

Install and start Docker Desktop.

#### 3.1.2 Verify Docker Installation

docker version

docker info

If both commands succeed and show server information, Docker is ready.

---

### 3.2 Create a Data Volume for PostgreSQL

Create a named volume to store PostgreSQL data:

docker volume create pgdata_main

List volumes to confirm:

docker volume ls

You should see `pgdata_main` in the output.

---

### 3.3 Pull the PostgreSQL Image

Download the recommended image:

docker pull postgres:16-alpine

---

### 3.4 Run the PostgreSQL Container (Recommended Configuration)

This command:

- Runs PostgreSQL in the background (`-d`).
- Restarts it automatically unless explicitly stopped (`--restart=unless-stopped`).
- Creates a database `app_db` and user `app_user`.
- Uses `pgdata_main` for persistent storage.
- Exposes the DB only on `127.0.0.1:5432`.

Replace `ChangeThisStrongPassword!` with a strong password of your choice.

docker run -d \
  --name postgres-db \
  --restart=unless-stopped \
  -p 127.0.0.1:5432:5432 \
  -e POSTGRES_DB=app_db \
  -e POSTGRES_USER=app_user \
  -e POSTGRES_PASSWORD=ChangeThisStrongPassword! \
  -v pgdata_main:/var/lib/postgresql/data \
  postgres:16-alpine

Key options (short explanation):

- `--name postgres-db`  
  Container name; used for logs, exec commands, etc.

- `--restart=unless-stopped`  
  Automatically restarts the container with Docker, unless you explicitly stop it.

- `-p 127.0.0.1:5432:5432`  
  Maps container port 5432 to host port 5432, bound to localhost only.

- `POSTGRES_DB=app_db`  
  Creates initial database `app_db` on first startup.

- `POSTGRES_USER=app_user` / `POSTGRES_PASSWORD=...`  
  Creates a user with password for accessing `app_db`.

- `-v pgdata_main:/var/lib/postgresql/data`  
  Mounts the volume for persistent data storage.

---

### 3.5 Check if PostgreSQL Is Running

List running containers:

docker ps

You should see something like:

- NAME: `postgres-db`
- PORTS: `127.0.0.1:5432->5432/tcp`

View logs:

docker logs postgres-db

If everything is okay, you’ll see messages indicating the server is ready to accept connections.

---

### 3.6 Install `psql` Client on macOS (Optional but Recommended)

To connect to PostgreSQL from the host, install the CLI tools:

brew install libpq

brew link --force libpq

Verify:

psql --version

---

### 3.7 Test Connection to PostgreSQL from macOS

Connect using the created user and database:

psql -h 127.0.0.1 -p 5432 -U app_user -d app_db

When prompted for a password, enter:

ChangeThisStrongPassword!

If the connection succeeds, you will see the prompt:

app_db=>

Exit with:

\q

---

### 3.8 Daily Operations for the PostgreSQL Container

#### 3.8.1 List Containers

docker ps

#### 3.8.2 View Logs

docker logs postgres-db

Follow logs in real time:

docker logs -f postgres-db

(Use `Ctrl + C` to stop following.)

#### 3.8.3 Stop, Start, and Restart

Stop the database:

docker stop postgres-db

Start it again:

docker start postgres-db

Restart:

docker restart postgres-db

#### 3.8.4 Open a Shell Inside the Container

docker exec -it postgres-db sh

From inside the container, you can use `psql` if needed:

psql -U app_user -d app_db

---

### 3.9 Backups

#### 3.9.1 Create a Backup Directory on macOS

mkdir -p ~/pg-backups

#### 3.9.2 Logical Backup with `pg_dump` from the Host

Assuming `pg_dump` is available via `libpq`:

PGPASSWORD=ChangeThisStrongPassword! pg_dump \
  -h 127.0.0.1 \
  -p 5432 \
  -U app_user \
  -d app_db \
  > ~/pg-backups/app_db_$(date +%F).sql

This creates a `.sql` file containing schema and data for the `app_db` database.

#### 3.9.3 Logical Backup via `docker exec`

Alternatively, use `pg_dump` inside the container:

docker exec -e PGPASSWORD=ChangeThisStrongPassword! postgres-db \
  pg_dump -U app_user -d app_db > ~/pg-backups/app_db_$(date +%F).sql

- The `>` redirection happens on the host, so the file is stored on macOS.
- Useful when you don’t want to install `libpq` tools on the host.

#### 3.9.4 Restore from Backup

To restore a backup into an (empty) `app_db`:

PGPASSWORD=ChangeThisStrongPassword! psql \
  -h 127.0.0.1 \
  -p 5432 \
  -U app_user \
  -d app_db \
  < ~/pg-backups/app_db_YYYY-MM-DD.sql

Replace `YYYY-MM-DD` with the actual date.

---

### 3.10 Upgrading PostgreSQL

#### 3.10.1 Upgrade Within the Same Major Version (e.g., 16.x → 16.y)

1. Pull the new image:

   docker pull postgres:16-alpine

2. Stop and remove the old container (data stays in the volume):

   docker stop postgres-db

   docker rm postgres-db

3. Start a new container with the **same** configuration and volume:

   docker run -d \
     --name postgres-db \
     --restart=unless-stopped \
     -p 127.0.0.1:5432:5432 \
     -e POSTGRES_DB=app_db \
     -e POSTGRES_USER=app_user \
     -e POSTGRES_PASSWORD=ChangeThisStrongPassword! \
     -v pgdata_main:/var/lib/postgresql/data \
     postgres:16-alpine

PostgreSQL will reuse the existing data in `pgdata_main`.

#### 3.10.2 Major Version Upgrade (e.g., 15 → 16)

For safety, even in development:

1. Take a backup with `pg_dump`.
2. Start a new container with the new major version and a **different** volume.
3. Restore the backup into the new database.
4. Test your application(s) against the new setup.
5. Once validated, decommission the old container and volume.

---

### 3.11 Common Customizations

#### 3.11.1 Use a Different Host Port

If port 5432 is already in use on your Mac, map to a different host port:

docker run -d \
  --name postgres-db \
  --restart=unless-stopped \
  -p 127.0.0.1:55432:5432 \
  -e POSTGRES_DB=app_db \
  -e POSTGRES_USER=app_user \
  -e POSTGRES_PASSWORD=ChangeThisStrongPassword! \
  -v pgdata_main:/var/lib/postgresql/data \
  postgres:16-alpine

Connect using:

- Host: `127.0.0.1`
- Port: `55432`

Example:

psql -h 127.0.0.1 -p 55432 -U app_user -d app_db

#### 3.11.2 Use a Host Directory Instead of a Named Volume (Not Recommended by Default)

Only do this if you **really need** direct file-level access from macOS:

mkdir -p ~/postgres_data

docker run -d \
  --name postgres-db \
  --restart=unless-stopped \
  -p 127.0.0.1:5432:5432 \
  -e POSTGRES_DB=app_db \
  -e POSTGRES_USER=app_user \
  -e POSTGRES_PASSWORD=ChangeThisStrongPassword! \
  -v ~/postgres_data:/var/lib/postgresql/data \
  postgres:16-alpine

Caveats:
- Potential performance impact due to Docker Desktop’s file sharing layer.
- More complex file permission behavior.
- Named volumes are generally the better choice.

#### 3.11.3 Multiple Databases (Different Projects) on the Same Server

- You can create additional databases in the same PostgreSQL instance:

  PGPASSWORD=ChangeThisStrongPassword! createdb \
    -h 127.0.0.1 \
    -p 5432 \
    -U app_user \
    another_db

- Or create additional users with different privileges.
- Adjust privileges so each application only has the access it needs.

---

## 4. Best Practices Summary

- **Isolation**  
  - One dedicated container for PostgreSQL.
  - Don’t mix unrelated apps in the same container.

- **Persistence**  
  - Use a named volume (`pgdata_main`) for `/var/lib/postgresql/data`.

- **Basic Security**  
  - Expose the port only on `127.0.0.1` by default.
  - Use specific user/database (`app_user` / `app_db`).
  - Use strong passwords, even for dev.

- **Operation & Maintenance**  
  - Monitor with `docker logs`.
  - Take regular logical backups using `pg_dump`.
  - Upgrade the image independently and reuse the volume.

- **Customization**  
  - Adjust the host port if needed.
  - Create multiple DBs/users for different projects.
  - Only mount host directories directly if you understand the trade-offs.

With this cookbook, you get a clean, isolated PostgreSQL instance running in Docker on macOS, ready to be consumed by any application or tool, in a way that is simple, safe, and maintainable.