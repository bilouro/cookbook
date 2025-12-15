# Jenkins on macOS Using Docker (Cookbook)

## 1. Overview

This cookbook describes how to run Jenkins on macOS using Docker, following practical, production-friendly patterns:

- Uses Docker Desktop for macOS as the container engine.
- Uses the official `jenkins/jenkins:lts` image for stability.
- Persists Jenkins data in a named Docker volume (`jenkins_home`).
- Runs Jenkins under a URL prefix (`/jenkins`) to simplify reverse-proxy use later.
- Keeps the setup easy to upgrade, backup, and restore.

---

## 2. Prerequisites

- macOS (Big Sur or newer recommended).
- Local admin privileges.
- Internet access to pull Docker and Jenkins images.

### 2.1 Install Docker Desktop

If you use Homebrew:

brew install --cask docker

Then:

1. Open the **Docker** app from Launchpad or Spotlight.
2. Wait until the whale icon in the top bar is stable (Docker is fully started).

Alternatively, download directly from:

https://www.docker.com/products/docker-desktop

Install and start Docker Desktop.

### 2.2 Verify Docker Installation

docker version

docker info

If both commands succeed and show server information, Docker is ready.

---

## 3. Create Docker Network and Volumes

Creating a dedicated Docker network makes it easier to connect other containers (databases, agents, etc.) to Jenkins later.

### 3.1 Create a Docker Network

docker network create jenkins-net

### 3.2 Create a Data Volume for Jenkins

This volume will persist Jenkins configuration, jobs, plugins, and build history.

docker volume create jenkins_home

List volumes to confirm:

docker volume ls

You should see `jenkins_home` in the output.

---

## 4. Pull Jenkins LTS Image

Always pull the latest LTS image before running:

docker pull jenkins/jenkins:lts

---

## 5. Run Jenkins Container (Recommended Configuration)

This configuration:

- Runs Jenkins in the background (`-d`).
- Automatically restarts it on Docker restart (`--restart=unless-stopped`).
- Exposes Jenkins on `http://localhost:8080/jenkins`.
- Uses a Docker named volume for persistence.
- Uses a dedicated Docker network.

### 5.1 Full Command (with JNLP Agent Port)

Use this if you plan to run remote agents (JNLP) later:

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  --network jenkins-net \
  jenkins/jenkins:lts

### 5.2 Simpler Command (no JNLP port)

Use this if you only need a local Jenkins instance and no JNLP agents:

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  --network jenkins-net \
  jenkins/jenkins:lts

---

## 6. Get Initial Admin Password

The first time Jenkins starts, it generates an initial admin password stored in `JENKINS_HOME`.

### 6.1 Watch Logs Until Jenkins is Ready

docker logs -f jenkins

Wait until you see a line similar to:

Jenkins is fully up and running

In the logs, you should also see a block like:

Please use the following password to proceed to installation:
<long-random-password>

Press `Ctrl + C` to stop following logs.

### 6.2 Directly Read Initial Password (Recommended)

docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword

Copy the printed password. You will need it to unlock Jenkins.

---

## 7. Access Jenkins UI

Open your browser and go to:

http://localhost:8080/jenkins

Follow the wizard:

1. **Unlock Jenkins**  
   - Paste the initial admin password you copied.

2. **Install Plugins**  
   - Select "Install suggested plugins" unless you have a specific plugin set in mind.

3. **Create First Admin User**  
   - Create a strong username and password, even for local usage.

4. **Configure Instance URL**  
   - Set it to something like:
     
     http://localhost:8080/jenkins

Once the wizard is complete, you will land on the Jenkins dashboard.

---

## 8. Optional: Adjust JVM Memory (JAVA_OPTS)

If Jenkins feels slow or Docker indicates memory pressure, you can tune the Java heap.

### 8.1 Stop and Remove Existing Container

docker stop jenkins

docker rm jenkins

### 8.2 Re-run Jenkins with JVM Settings

Example: 512MB min, 2GB max heap (adjust to your Mac’s capacity):

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  -e JAVA_OPTS="-Xms512m -Xmx2048m" \
  --network jenkins-net \
  jenkins/jenkins:lts

---

## 9. Daily Operations

Common commands for working with the Jenkins container:

### 9.1 Check Running Containers

docker ps

### 9.2 View Jenkins Logs

docker logs jenkins

Follow logs in real time:

docker logs -f jenkins

### 9.3 Stop, Start, Restart

Stop Jenkins:

docker stop jenkins

Start Jenkins:

docker start jenkins

Restart Jenkins:

docker restart jenkins

### 9.4 Open a Shell Inside the Container

docker exec -it jenkins bash

---

## 10. Upgrading Jenkins (LTS)

Upgrade strategy:

1. Pull latest LTS image.
2. Stop and remove the old container.
3. Start a new container using the same `jenkins_home` volume.

### 10.1 Pull New LTS Image

docker pull jenkins/jenkins:lts

### 10.2 Stop and Remove Old Container

docker stop jenkins

docker rm jenkins

### 10.3 Start New Container with the Same Volume

Use the same command you used originally (example without JNLP port):

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  --network jenkins-net \
  jenkins/jenkins:lts

Jenkins will detect the existing `JENKINS_HOME` and perform safe migrations.

---

## 11. Backups

Jenkins stores all essential data under `/var/jenkins_home`. Backing up this directory (i.e., the `jenkins_home` volume) is critical.

### 11.1 Create a Backup Directory on the Host

mkdir -p ~/jenkins-backups

### 11.2 Run a Temporary Container to Archive Jenkins Data

docker run --rm \
  -v jenkins_home:/var/jenkins_home \
  -v ~/jenkins-backups:/backup \
  alpine \
  sh -c "cd /var/jenkins_home && tar czf /backup/jenkins-$(date +%F).tar.gz ."

This creates a tar.gz with all Jenkins data under:

~/jenkins-backups

You can schedule this command (e.g., via cron or a manual script) to run regularly.

---

## 12. Common Customizations

### 12.1 Use a Different Host Port

If port 8080 is already in use on your Mac, map a different host port:

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8090:8080 \
  -v jenkins_home:/var/jenkins_home \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  --network jenkins-net \
  jenkins/jenkins:lts

Access Jenkins at:

http://localhost:8090/jenkins

### 12.2 Use a Host Directory Instead of a Named Volume

Only do this if you specifically want to browse Jenkins files directly on the host and understand the potential performance/permission trade-offs on macOS.

mkdir -p ~/jenkins_home

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v ~/jenkins_home:/var/jenkins_home \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  --network jenkins-net \
  jenkins/jenkins:lts

### 12.3 Allow Jenkins to Use Docker (Build Docker Images Inside Jenkins)

This is powerful but not secure by default: it effectively gives Jenkins root-level control on the host via Docker. Use only in trusted/local environments and with strong access controls.

Example (not recommended for production as-is):

docker run -d \
  --name jenkins \
  --restart=unless-stopped \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e JENKINS_OPTS="--prefix=/jenkins" \
  --network jenkins-net \
  jenkins/jenkins:lts

Inside Jenkins, you can then configure build steps that run `docker` commands.

---

## 13. Maintenance and Best Practices Summary

- **Updates**
  - Regularly pull `jenkins/jenkins:lts` and recreate the container with the same volume.
  - Update plugins cautiously, and preferably after taking a backup.

- **Backups**
  - Make regular tar.gz backups of `jenkins_home`.
  - Store backups outside your local machine if the data is important.

- **Security**
  - Use a strong admin password.
  - Disable anonymous write/admin access.
  - If exposing Jenkins beyond `localhost`, front it with a reverse proxy (Nginx/Traefik/Apache) and HTTPS, and consider VPN or firewall rules.

- **Performance**
  - Allocate enough CPU and RAM to Docker via Docker Desktop settings.
  - Tune `JAVA_OPTS` if Jenkins becomes memory constrained or sluggish.

This completes the minimal, Docker-based Jenkins-on-macOS cookbook, ready to be dropped into a GitHub “cookbooks” repository.
