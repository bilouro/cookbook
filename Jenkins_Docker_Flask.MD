# Jenkins + Docker + FlaskProject (3 Environments Cookbook)

> This cookbook extends the main Jenkins-on-macOS cookbook:  
> https://github.com/bilouro/cookbook/blob/main/jenkins.MD  
> Target application (Flask):  
> https://github.com/bilouro/FlaskProject  

---

## This document explains how to

- Use **one single Dockerfile** (from the FlaskProject) to build the application image.
- Use **one single Jenkinsfile** with a parameter `ENVIRONMENT` (`dev`, `test`, `prod`) to:
  - Build the Docker image,
  - Run database migrations with Alembic,
  - Run automated tests,
  - Deploy the app to the selected environment.
- Keep **three independent app instances** running at the same time, one per environment, each:
  - Listening on a different host port (`5001`, `5002`, `5003`),
  - Connected to a separate Postgres database (`app_db_dev`, `app_db_test`, `app_db_prod`).

---

## 1. Architecture Overview

### 1.1 Components

#### Flask application

- Repository: https://github.com/bilouro/FlaskProject
- Single Dockerfile (`flask-app:latest` image).
- Flask app in `app.py`, listening on port `5000` inside the container.
- Alembic configured for database migrations.

#### Postgres

- One container: `postgres-db`. (https://github.com/bilouro/cookbook/blob/main/postgresql.MD)
- Three databases inside the same Postgres instance:
  - `app_db_dev`
  - `app_db_test`
  - `app_db_prod`

#### Jenkins

- Running in Docker on macOS, configured as in:  
  https://github.com/bilouro/cookbook/blob/main/jenkins.MD
- Has access to the Docker engine via `/var/run/docker.sock`.

#### Docker network

- Network name: `flask-net`
- Connects:
  - `jenkins` container,
  - `postgres-db` container,
  - All Flask app containers (`flask-app-dev`, `flask-app-test`, `flask-app-prod`).

### 1.2 Resulting Environments

#### DEV

- Container: `flask-app-dev`
- URL: <http://localhost:5001>
- DB: `app_db_dev`

#### TEST

- Container: `flask-app-test`
- URL: <http://localhost:5002>
- DB: `app_db_test`

#### PROD

- Container: `flask-app-prod`
- URL: <http://localhost:5003>
- DB: `app_db_prod`

All 3 can run simultaneously, using the same Docker image but different containers, ports, and databases.

---

## 2. Prerequisites

- macOS with Docker Desktop installed and running.
- Jenkins running in Docker, following the base cookbook:  
  <https://github.com/bilouro/cookbook/blob/main/jenkins.MD>
- Access to GitHub to clone:
  - FlaskProject: <https://github.com/bilouro/FlaskProject>

**Check Docker:**

    docker version
    docker info

**Check Jenkins (if using prefix `/jenkins`):**

    http://localhost:8080/jenkins

---

## 3. FlaskProject Dockerfile (Single Image for All Environments)

Clone the project (if not already on disk):

    git clone https://github.com/bilouro/FlaskProject
    cd FlaskProject

### 3.1 Dockerfile

Project Dockerfile (as of your current setup):

    FROM python:3.9.6-slim
    
    WORKDIR /app
    
    COPY requirements.txt .
    
    RUN pip install --upgrade pip && \
        pip install -r requirements.txt
    
    COPY . .
    
    EXPOSE 5000
    
    CMD ["python", "app.py"]

**Key points:**

`app.py` is configured to run on port `5000`:

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=5000, debug=app.config.get("DEBUG", False))

- `EXPOSE 5000` documents the internal port used by the app.
- We will always treat `5000` as the internal container port.
- Jenkins will map different host ports to `5000`:
  - DEV: `5001:5000`
  - TEST: `5002:5000`
  - PROD: `5003:5000`

**Commit and push (optional, but recommended):**

    git add Dockerfile
    git commit -m "Align Docker EXPOSE with Flask app port 5000"
    git push origin main

---

## 4. Docker Network and Postgres Setup

### 4.1 Create `flask-net` Network

This shared network allows Jenkins, Postgres, and all Flask containers to talk to each other by name.

    docker network create flask-net

If it already exists, Docker will just warn; that is fine.

### 4.2 Run `postgres-db` Container

We will use one Postgres container for all three DBs.

    docker stop postgres-db || true
    docker rm postgres-db || true
    
    docker run -d \
      --name postgres-db \
      --network flask-net \
      -e POSTGRES_USER=app_user \
      -e POSTGRES_PASSWORD=bsd0030 \
      -e POSTGRES_DB=app_db_dev \
      -p 5432:5432 \
      -v postgres_data:/var/lib/postgresql/data \
      postgres:16-alpine

- User: `app_user`
- Password: `bsd0030`
- Initial DB: `app_db_dev` (we will add test and prod databases next).

### 4.3 Create `app_db_test` and `app_db_prod`

    docker exec -it postgres-db psql -U app_user -d postgres -c "CREATE DATABASE app_db_test;"
    docker exec -it postgres-db psql -U app_user -d postgres -c "CREATE DATABASE app_db_prod;"

Now you have:

- `app_db_dev`
- `app_db_test`
- `app_db_prod`

---

## 5. Alembic Integration (Automatic DB Schema Management)

The FlaskProject uses Alembic to manage database migrations.

### 5.1 Requirements

Ensure `requirements.txt` includes:

- `alembic`
- `SQLAlchemy`
- `psycopg2-binary`

The Dockerfile already runs:

    RUN pip install --upgrade pip && \
        pip install -r requirements.txt

So Alembic and SQLAlchemy are available inside the container image.

### 5.2 Alembic Configuration: `alembic.ini`

In the project root, `alembic.ini` should contain:

    [alembic]
    script_location = migrations
    
    # This will be overridden by env.py; leave it blank or as a dummy
    sqlalchemy.url =

We do **not** hard-code `localhost` here; the URL will come from the environment via `env.py`.

### 5.3 Alembic Configuration: `migrations/env.py`

Core parts of `env.py` (simplified to highlight the `DATABASE_URL` logic):

    from __future__ import annotations
    
    import os
    import sys
    from logging.config import fileConfig
    
    from alembic import context
    from sqlalchemy import engine_from_config, pool
    from config import BaseConfig
    
    config = context.config
    
    # Ensure project root is in sys.path (so books.models etc. can be imported)
    project_root = os.path.abspath(os.path.join(os.path.dirname(__file__), os.pardir))
    if project_root not in sys.path:
        sys.path.insert(0, project_root)
    
    from books.models import Base  # noqa: E402
    
    # ---------------------------------------------------------------------------
    # Database URL: prefer DATABASE_URL from environment (for Docker/Jenkins).
    # Fallback to BaseConfig.SQLALCHEMY_DATABASE_URI for local/manual runs.
    # ---------------------------------------------------------------------------
    db_url = os.environ.get("DATABASE_URL")
    if not db_url:
        db_url = BaseConfig.SQLALCHEMY_DATABASE_URI
    
    config.set_main_option("sqlalchemy.url", db_url)
    
    # Logging
    if config.config_file_name is not None:
        fileConfig(config.config_file_name)
    
    target_metadata = Base.metadata
    
    def run_migrations_offline() -> None:
        """Run migrations in 'offline' mode."""
        url = config.get_main_option("sqlalchemy.url")
        context.configure(
            url=url,
            target_metadata=target_metadata,
            literal_binds=True,
            dialect_opts={"paramstyle": "named"},
        )
        with context.begin_transaction():
            context.run_migrations()
    
    def run_migrations_online() -> None:
        """Run migrations in 'online' mode."""
        connectable = engine_from_config(
            config.get_section(config.config_ini_section),
            prefix="sqlalchemy.",
            poolclass=pool.NullPool,
        )
        with connectable.connect() as connection:
            context.configure(
                connection=connection,
                target_metadata=target_metadata,
            )
            with context.begin_transaction():
                context.run_migrations()
    
    if context.is_offline_mode():
        run_migrations_offline()
    else:
        run_migrations_online()

**Key behavior:**

- In Jenkins, we pass `DATABASE_URL` explicitly (pointing to `postgres-db` on `flask-net`).
- Alembic reads this from the environment and uses it to connect.

---

## 6. Jenkins Setup (with Docker Access)

Follow the base Jenkins-on-macOS cookbook:  
<https://github.com/bilouro/cookbook/blob/main/jenkins.MD>

### 6.1 Jenkins Container with Docker Access

Example command (assuming you have built `jenkins-with-docker:lts` with the Docker CLI installed; or you can use `jenkins/jenkins:lts` plus Docker CLI in the container):

    docker run -d \
      --name jenkins \
      --user root \
      --restart=unless-stopped \
      -p 8080:8080 -p 50000:50000 \
      -v jenkins_home:/var/jenkins_home \
      -v /var/run/docker.sock:/var/run/docker.sock \
      --network flask-net \
      jenkins-with-docker:lts

- `--network flask-net` puts Jenkins in the same network as `postgres-db` and the app containers.
- `/var/run/docker.sock` gives Jenkins the ability to run `docker` commands.

---

## 7. Parametric Jenkinsfile (DEV / TEST / PROD with One Image)

We use **one Jenkinsfile** with a choice parameter `ENVIRONMENT` that controls:

- Container name,
- Host HTTP port,
- Target database name.

The same Docker image (`flask-app:latest`) is used everywhere.

### 7.1 Jenkinsfile (place in FlaskProject root or in the Jenkins job)

    pipeline {
        agent any
    
        parameters {
            choice(
                name: 'ENVIRONMENT',
                choices: ['dev', 'test', 'prod'],
                description: 'Target environment for deploy (dev, test, prod)'
            )
        }
    
        environment {
            // Docker image for the Flask app
            DOCKER_IMAGE = 'flask-app:latest'
    
            // Shared DB configuration
            DB_HOST     = 'postgres-db'
            DB_PORT     = '5432'
            DB_USER     = 'app_user'
            DB_PASSWORD = 'bsd0030'
    
            // Docker network where Jenkins, app, and Postgres live
            DOCKER_NETWORK = 'flask-net'
    
            // Internal port of the Flask app (as per app.run(..., port=5000))
            CONTAINER_HTTP_PORT = '5000'
    
            // Git repository
            GIT_REPO   = 'https://github.com/bilouro/FlaskProject'
            GIT_BRANCH = 'main'
        }
    
        stages {
    
            stage('Setup Env Config') {
                steps {
                    script {
                        echo "Configuring environment: ${params.ENVIRONMENT}"
    
                        // Per-environment configuration
                        def configs = [
                            dev:  [ containerName: 'flask-app-dev',  hostPort: '5001', dbName: 'app_db_dev'  ],
                            test: [ containerName: 'flask-app-test', hostPort: '5002', dbName: 'app_db_test' ],
                            prod: [ containerName: 'flask-app-prod', hostPort: '5003', dbName: 'app_db_prod' ]
                        ]
    
                        def cfg = configs[params.ENVIRONMENT]
    
                        // Target environment where we will deploy
                        env.TARGET_ENV         = params.ENVIRONMENT
                        env.DOCKER_CONTAINER   = cfg.containerName
                        env.HOST_HTTP_PORT     = cfg.hostPort
                        env.DB_NAME_TARGET_ENV = cfg.dbName
    
                        // DB used by the CI tests (fixed: app_db_test)
                        env.DB_NAME_TEST = 'app_db_test'
    
                        // Build DATABASE_URLs for Alembic (TEST & TARGET)
                        env.DATABASE_URL_TEST = "postgresql+psycopg2://${env.DB_USER}:${env.DB_PASSWORD}@${env.DB_HOST}:${env.DB_PORT}/${env.DB_NAME_TEST}"
                        env.DATABASE_URL_TARGET = "postgresql+psycopg2://${env.DB_USER}:${env.DB_PASSWORD}@${env.DB_HOST}:${env.DB_PORT}/${env.DB_NAME_TARGET_ENV}"
    
                        echo "Target environment: ${env.TARGET_ENV}"
                        echo "Target container: ${env.DOCKER_CONTAINER}"
                        echo "Host HTTP port: ${env.HOST_HTTP_PORT}"
                        echo "Target DB: ${env.DB_NAME_TARGET_ENV}"
                        echo "Test DB (CI): ${env.DB_NAME_TEST}"
                        echo "DATABASE_URL_TEST: ${env.DATABASE_URL_TEST}"
                        echo "DATABASE_URL_TARGET: ${env.DATABASE_URL_TARGET}"
                    }
                }
            }
    
            stage('Checkout') {
                steps {
                    echo "Cleaning workspace..."
                    deleteDir()
    
                    echo "Checking out code from Git..."
                    git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
                }
            }
    
            stage('Build Docker Image') {
                steps {
                    echo "Building Docker image..."
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                }
            }
    
            stage('DB Migrations (TEST)') {
                steps {
                    echo "Running Alembic migrations on TEST DB (${DB_NAME_TEST})..."
                    sh '''
                        docker run --rm \
                          --network ${DOCKER_NETWORK} \
                          -e DATABASE_URL=${DATABASE_URL_TEST} \
                          ${DOCKER_IMAGE} \
                          alembic upgrade head
                    '''
                }
            }
    
            stage('Run Tests (using TEST DB)') {
                steps {
                    echo "Running tests (starting app in test container, using TEST DB)..."
                    sh '''
                        docker run --rm \
                          --network ${DOCKER_NETWORK} \
                          -e APP_DB_HOST=${DB_HOST} \
                          -e APP_DB_PORT=${DB_PORT} \
                          -e APP_DB_NAME=${DB_NAME_TEST} \
                          -e APP_DB_USER=${DB_USER} \
                          -e APP_DB_PASSWORD=${DB_PASSWORD} \
                          ${DOCKER_IMAGE} \
                          sh -c "python app.py & sleep 5 && python -m pytest tests/ -v"
                    '''
                }
            }
    
            stage('Stop Old Container (target env)') {
                steps {
                    echo "Checking if old container exists and stopping/removing it (${DOCKER_CONTAINER})..."
                    sh '''
                        if [ "$(docker ps -aq -f name=${DOCKER_CONTAINER})" ]; then
                          echo "Found container ${DOCKER_CONTAINER}, stopping and removing..."
                          docker stop ${DOCKER_CONTAINER} || true
                          docker rm ${DOCKER_CONTAINER} || true
                        else
                          echo "No container named ${DOCKER_CONTAINER} found. Nothing to stop/remove."
                        fi
                    '''
                }
            }
    
            stage('DB Migrations (TARGET ENV)') {
                steps {
                    echo "Running Alembic migrations on TARGET DB (${DB_NAME_TARGET_ENV}) for environment ${TARGET_ENV}..."
                    sh '''
                        docker run --rm \
                          --network ${DOCKER_NETWORK} \
                          -e DATABASE_URL=${DATABASE_URL_TARGET} \
                          ${DOCKER_IMAGE} \
                          alembic upgrade head
                    '''
                }
            }
    
            stage('Deploy to Target Env') {
                steps {
                    echo "Starting new container for environment ${TARGET_ENV}..."
                    sh '''
                        docker run -d \
                        --name ${DOCKER_CONTAINER} \
                        --network ${DOCKER_NETWORK} \
                        -p ${HOST_HTTP_PORT}:${CONTAINER_HTTP_PORT} \
                        -e APP_DB_HOST=${DB_HOST} \
                        -e APP_DB_PORT=${DB_PORT} \
                        -e APP_DB_NAME=${DB_NAME_TARGET_ENV} \
                        -e APP_DB_USER=${DB_USER} \
                        -e APP_DB_PASSWORD=${DB_PASSWORD} \
                        -e FLASK_ENV=${TARGET_ENV} \
                        ${DOCKER_IMAGE}
                    '''
                }
            }
    
            stage('Health Check') {
                steps {
                    echo "Checking application health (via Docker network, not localhost)..."
                    sh '''
                        echo "Waiting for application to become healthy..."
                        for i in $(seq 1 10); do
                          echo "Attempt $i: checking http://${DOCKER_CONTAINER}:${CONTAINER_HTTP_PORT}/health ..."
                          if curl -fs http://${DOCKER_CONTAINER}:${CONTAINER_HTTP_PORT}/health; then
                            echo "Application is healthy!"
                            exit 0
                          fi
                          echo "Still not healthy. Waiting 2s..."
                          sleep 2
                        done
                        echo "Health check failed after multiple attempts."
                        exit 1
                    '''
                }
            }
        }
    
        post {
            always {
                echo "Pipeline finished for environment ${env.TARGET_ENV ?: params.ENVIRONMENT}."
            }
            success {
                echo "Deploy succeeded!"
                echo "Environment: ${env.TARGET_ENV ?: params.ENVIRONMENT}"
                echo "URL: http://localhost:${env.HOST_HTTP_PORT ?: '???'}"
            }
            failure {
                echo "Pipeline failed! Check logs above."
            }
        }
    }

---

## 8. Configuring the Jenkins Job

### 8.1 Create the Pipeline Job

1. Open Jenkins: <http://localhost:8080/jenkins>
2. Click **New Item**.
3. Choose:
   - Name: `flask-multi-env-pipeline` (for example).
   - Type: **Pipeline**.

In the **Pipeline** section:

#### Option A – Jenkinsfile in the repo

- Definition: `Pipeline script from SCM`
- SCM: `Git`
- Repository URL: `https://github.com/bilouro/FlaskProject`
- Script Path: `Jenkinsfile`

#### Option B – Direct script

- Definition: `Pipeline script`
- Paste the Jenkinsfile content directly.
- Save the job.

---

## 9. Running Builds for Each Environment

### 9.1 DEV Environment

1. Click the job `flask-multi-env-pipeline`.
2. Click **Build with Parameters**.
3. Select: `ENVIRONMENT = dev`.
4. Start the build and watch the console output.

**Expected result:**

- Container: `flask-app-dev`
- Host port: `5001`
- URL: <http://localhost:5001>
- DB: `app_db_dev`
- Health check passes (`/health` returns HTTP 200).

### 9.2 TEST Environment

Repeat, but:

- `ENVIRONMENT = test`

**Expected:**

- Container: `flask-app-test`
- Host port: `5002`
- URL: <http://localhost:5002>
- DB: `app_db_test`

### 9.3 PROD Environment

Repeat, but:

- `ENVIRONMENT = prod`

**Expected:**

- Container: `flask-app-prod`
- Host port: `5003`
- URL: <http://localhost:5003>
- DB: `app_db_prod`

All three containers can coexist:

    docker ps

Should show something similar to:

- `flask-app-dev` → `0.0.0.0:5001->5000/tcp`
- `flask-app-test` → `0.0.0.0:5002->5000/tcp`
- `flask-app-prod` → `0.0.0.0:5003->5000/tcp`
- `postgres-db`
- `jenkins`

---

## 10. Manual Verification

### 10.1 Health and Books Endpoints

**DEV:**

    curl -f http://127.0.0.1:5001/health
    curl -f http://127.0.0.1:5001/books/

**TEST:**

    curl -f http://127.0.0.1:5002/health
    curl -f http://127.0.0.1:5002/books/

**PROD:**

    curl -f http://127.0.0.1:5003/health
    curl -f http://127.0.0.1:5003/books/

### 10.2 Swagger UI

Per environment:

- DEV: <http://127.0.0.1:5001/docs>
- TEST: <http://127.0.0.1:5002/docs>
- PROD: <http://127.0.0.1:5003/docs>

Your `app.py` defines:

    @app.get("/swagger.json")
    def swagger_spec():
        spec = {
            "openapi": "3.0.0",
            "info": {
                "title": "Books API",
                "version": "1.0.0",
                "description": "Simple Books API example with Flask",
            },
            "servers": [
                {"url": "/"}
            ],
            "paths": {
                ...
            },
            "components": {
                ...
            },
        }
        return jsonify(spec)
    
    @app.get("/docs")
    def docs():
        html = """
        <!DOCTYPE html>
        <html>
          <head>
            <title>Books API - Swagger UI</title>
            <link rel="stylesheet" type="text/css"
                  href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css" />
          </head>
          <body>
            <div id="swagger-ui"></div>
            <script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
            <script>
              window.onload = function() {
                SwaggerUIBundle({
                  url: '/swagger.json',
                  dom_id: '#swagger-ui'
                });
              };
            </script>
          </body>
        </html>
        """
        return html

Because `servers: [{"url": "/"}]` is relative, the Swagger UI works correctly for all 3 environments (it uses the same host/port where `/docs` is served).

---

## 11. Best Practices and Next Steps

### 11.1 Migrations (Alembic)

For every schema change, create a new Alembic revision:

    alembic revision -m "add field X to books"

Then:

- Edit the `upgrade()` and `downgrade()` functions in the new migration file.
- Optionally test locally against one DB:

      alembic upgrade head

- Commit and push migrations and `alembic.ini` changes to Git.

The Jenkins pipeline will automatically apply migrations to:

- `app_db_test` (before running tests),
- `app_db_dev` / `app_db_test` / `app_db_prod` (depending on selected `ENVIRONMENT`) before deployment.

### 11.2 Data and Isolation

- `app_db_test` can be freely modified by tests; keep test data ephemeral.
- Treat `app_db_prod` as production-like data — be careful with destructive tests.
- `app_db_dev` is a good place for developers to manually test new behavior without affecting prod-like data.

### 11.3 Security

DB credentials are hard-coded for simplicity in this lab. In real setups:

- Store DB credentials in Jenkins Credentials.
- Inject them as environment variables or binding steps in the pipeline.
- Restrict Jenkins access if it is exposed beyond `localhost` (VPN, firewall, reverse proxy with HTTPS, etc.).

### 11.4 Performance

- Be mindful of Docker Desktop resource limits (CPU/RAM) on macOS.
- If Jenkins feels slow or memory-constrained, tune `JAVA_OPTS` as shown in the main Jenkins cookbook (example: `-Xms512m -Xmx2048m`).

### 11.5 Possible Extensions

- Add linting stages:
  - `flake8` validation,
  - Code formatting checks with `black`.
- Push images to a registry:
  - Docker Hub,
  - GitHub Container Registry (GHCR).
- Use GitHub webhooks to trigger Jenkins builds on each push to `main`.
- Introduce a separate CI-only database (e.g., `app_db_ci`) if you want stricter isolation between automated tests and the `test` environment.

---

## 12. Summary

With this cookbook, together with:

- Jenkins-on-macOS Cookbook: <https://github.com/bilouro/cookbook/blob/main/jenkins.MD>
- FlaskProject: <https://github.com/bilouro/FlaskProject>

you have a full CI/CD pipeline with 3 environments (`dev`, `test`, `prod`) driven by:

- A single Dockerfile,
- A single parametric Jenkinsfile,
- And environment-specific configuration controlled by a Jenkins parameter.
