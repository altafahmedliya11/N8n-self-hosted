# Self-Hosted n8n – Production-Ready Docker Deployment

A production-oriented, self-hosted deployment of [n8n](https://n8n.io/) using Docker Compose, PostgreSQL, Redis, Queue Mode, multiple workers, and external task runners.

This project provides a scalable foundation for running automation workflows with persistent storage, distributed execution, and isolated code execution.

## 🚀 Features

- **n8n Automation Platform** – Self-hosted workflow automation.
- **PostgreSQL** – Persistent storage for workflows, credentials, and execution data.
- **Redis** – Queue management and worker coordination.
- **Queue Mode** – Distributed workflow execution.
- **Multiple Workers** – Parallel workflow processing.
- **External Task Runners** – Isolated execution of JavaScript and Python Code nodes.
- **Docker Compose** – Simple deployment and service orchestration.
- **Persistent Volumes** – Data persistence across container restarts.
- **Health Checks** – PostgreSQL and Redis service health monitoring.
- **Environment-Based Configuration** – Centralized configuration and secrets.
- **Timezone Support** – Configured for Asia/Kolkata by default.

## 🏗️ Architecture

The deployment consists of the following services:

| Service | Description |
|---|---|
| `n8n` | Main n8n instance handling UI, API, webhooks, and scheduling |
| `postgres` | PostgreSQL database |
| `redis` | Redis queue backend |
| `n8n-worker-1` | First workflow execution worker |
| `n8n-worker-2` | Second workflow execution worker |
| `n8n-runner` | External task runner for the main instance |
| `n8n-worker-1-runner` | External task runner for worker 1 |
| `n8n-worker-2-runner` | External task runner for worker 2 |

### Architecture Overview

```text
                    Internet
                       |
                       |
                 Reverse Proxy
                (Nginx / Caddy)
                       |
                       |
                  n8n Main
                  Port 5678
                       |
             +---------+---------+
             |                   |
         PostgreSQL             Redis
         Database             Job Queue
                                 |
                    +------------+------------+
                    |                         |
                Worker 1                   Worker 2
              Concurrency 10             Concurrency 10
                    |                         |
                 Runner                    Runner
                    |                         |
                    +------------+------------+
                                 |
                          Workflow Execution
```

**Execution flow:**

1. Users create and manage workflows through the n8n interface.
2. The main instance receives workflow execution requests.
3. Execution jobs are added to Redis.
4. Available workers retrieve jobs from the queue.
5. Workers execute workflows.
6. External task runners handle supported Code node execution.
7. Execution data and workflow information are persisted in PostgreSQL.

## 📋 Prerequisites

Make sure the following software is installed:

- Docker Engine
- Docker Compose V2
- Git
- OpenSSL (for generating secure secrets)

Check your installations:

```bash
docker --version
docker compose version
git --version
```

## 📁 Project Structure

```text
self-host-n8n/
│
├── docker-compose.yml
├── .env
├── init-data.sh
├── .gitignore
└── README.md
```

| File | Purpose |
|---|---|
| `docker-compose.yml` | Defines all application services |
| `.env` | Stores environment variables and secrets |
| `init-data.sh` | Initializes the PostgreSQL application user |
| `.gitignore` | Prevents sensitive files from being committed |
| `README.md` | Project documentation |

## ⚙️ Configuration

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/self-host-n8n.git

cd self-host-n8n
```

### 2. Create the environment file

Create a `.env` file in the project root.

```env
# --------------------------------
# n8n Version
# --------------------------------

N8N_VERSION=2.47.4


# --------------------------------
# Domain Configuration
# --------------------------------

N8N_HOST=n8n.example.com

WEBHOOK_URL=https://n8n.example.com/

N8N_EDITOR_BASE_URL=https://n8n.example.com/


# --------------------------------
# PostgreSQL Configuration
# --------------------------------

POSTGRES_USER=postgres

POSTGRES_PASSWORD=CHANGE_ME_POSTGRES_PASSWORD

POSTGRES_DB=n8n


# PostgreSQL Application User

POSTGRES_NON_ROOT_USER=n8n

POSTGRES_NON_ROOT_PASSWORD=CHANGE_ME_N8N_DB_PASSWORD


# --------------------------------
# Redis Configuration
# --------------------------------

REDIS_PASSWORD=CHANGE_ME_REDIS_PASSWORD


# --------------------------------
# n8n Encryption
# --------------------------------

N8N_ENCRYPTION_KEY=CHANGE_ME_ENCRYPTION_KEY


# --------------------------------
# External Task Runners
# --------------------------------

RUNNERS_AUTH_TOKEN=CHANGE_ME_RUNNERS_TOKEN
```

**Important:** Replace all placeholder passwords and tokens with secure values.

Generate secure random values using:

```bash
openssl rand -hex 32
```

Generate a separate value for each secret.

Never commit your `.env` file to GitHub.

### 3. PostgreSQL Initialization

The `init-data.sh` script creates a dedicated PostgreSQL application user for n8n.

Make the script executable:

```bash
chmod +x init-data.sh
```

The initialization script runs automatically when PostgreSQL initializes a new database volume.

**Note:** PostgreSQL initialization scripts execute only when the database is first initialized. Existing database volumes are not automatically modified.

## 🐳 Deployment

### 1. Validate Docker Compose configuration

```bash
docker compose config
```

### 2. Pull Docker images

```bash
docker compose pull
```

### 3. Start all services

```bash
docker compose up -d
```

### 4. Verify running containers

```bash
docker compose ps
```

### 5. Access n8n

Open your configured domain:

```text
https://n8n.example.com
```

For local testing, the main instance is bound to:

```text
http://127.0.0.1:5678
```

For production, configure an HTTPS reverse proxy such as Nginx, Caddy, or Traefik.

## 🔄 Queue Mode and Worker Configuration

This deployment uses Redis-backed Queue Mode.

Current configuration:

| Configuration | Value |
|---|---|
| Execution Mode | Queue |
| Redis | Redis 7 |
| Database | PostgreSQL |
| Number of Workers | 2 |
| Worker 1 Concurrency | 10 |
| Worker 2 Concurrency | 10 |
| Total Configured Worker Concurrency | 20 |
| Task Runner Mode | External |

### Scaling Workers

To increase workflow execution capacity, add more worker services.

For example:

```yaml
n8n-worker-3:
  image: docker.n8n.io/n8nio/n8n:${N8N_VERSION}
  restart: unless-stopped
  command: worker --concurrency=10
  environment: *n8n_environment
  volumes:
    - n8n_data:/home/node/.n8n
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
  networks:
    - n8n-network
```

Add a corresponding external task-runner service for the new worker.

The total execution capacity depends on available CPU, memory, database connections, and workload characteristics.

## 🏃 External Task Runners

External task runners provide isolated execution for supported Code node operations.

This deployment uses:

```env
N8N_RUNNERS_ENABLED=true

N8N_RUNNERS_MODE=external

N8N_RUNNERS_AUTH_TOKEN=YOUR_RUNNERS_AUTH_TOKEN

N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
```

Each n8n main or worker instance requiring external task execution has its own runner sidecar.

The runner authentication token must be identical across the corresponding n8n instances and runner containers.

For production, use the official runner configuration and security recommendations.

Documentation:

https://docs.n8n.io/hosting/configuration/task-runners/

## 📊 Monitoring and Logs

### View all container logs

```bash
docker compose logs -f
```

### Main n8n logs

```bash
docker compose logs -f n8n
```

### Worker logs

```bash
docker compose logs -f n8n-worker-1 n8n-worker-2
```

### Task runner logs

```bash
docker compose logs -f n8n-runner
```

### Redis logs

```bash
docker compose logs -f redis
```

### PostgreSQL logs

```bash
docker compose logs -f postgres
```

### Check running services

```bash
docker compose ps
```

## 🔐 Production Security Recommendations

For production environments, consider the following:

- Use strong, randomly generated passwords.
- Never expose PostgreSQL or Redis directly to the public internet.
- Use HTTPS through a reverse proxy.
- Keep Docker images pinned to specific versions.
- Protect `.env` using restrictive file permissions.
- Back up PostgreSQL regularly.
- Securely back up the n8n encryption key.
- Monitor CPU, memory, disk usage, and container health.
- Restrict access to the n8n editor.
- Configure appropriate execution data retention.
- Use external object storage for binary data when required.
- Keep task runners isolated and follow n8n's runner security recommendations.

## 💾 Data Persistence

Docker volumes are used to persist application data.

| Volume | Purpose |
|---|---|
| `postgres_data` | PostgreSQL database files |
| `redis_data` | Redis persistent data |
| `n8n_data` | n8n application data |

List available volumes:

```bash
docker volume ls
```

### PostgreSQL Backup

Example database backup:

```bash
docker compose exec -T postgres \
  pg_dump -U postgres -d n8n > n8n_backup.sql
```

Restore:

```bash
cat n8n_backup.sql | docker compose exec -T postgres \
  psql -U postgres -d n8n
```

For production, implement scheduled backups, retention policies, and restore testing.

Also back up the n8n encryption key separately.

## 🛠️ Useful Docker Commands

| Command | Description |
|---|---|
| `docker compose up -d` | Start all services |
| `docker compose down` | Stop and remove containers |
| `docker compose restart` | Restart services |
| `docker compose pull` | Pull configured images |
| `docker compose ps` | List running services |
| `docker compose logs -f` | Follow container logs |
| `docker compose config` | Validate configuration |
| `docker compose top` | View running processes |

**Warning:** Avoid `docker compose down -v` unless you intentionally want to delete persistent volumes and their stored data.

## 🚀 Updating n8n

Update the pinned version in your `.env` file:

```env
N8N_VERSION=YOUR_NEW_STABLE_VERSION
```

Then execute:

```bash
docker compose pull

docker compose up -d
```

Always:

1. Read the release notes.
2. Back up PostgreSQL.
3. Back up the encryption key.
4. Verify compatibility with the runner image.
5. Test the deployment after upgrading.

Use matching n8n and runner versions.

## ⚠️ Important Production Notes

This project provides a production-oriented Docker Compose architecture, but a single-host deployment is not a fully highly available architecture.

For larger deployments, consider:

- Managed PostgreSQL with automated backups and failover.
- Highly available Redis.
- Multiple n8n main instances behind a load balancer, where licensing permits.
- Distributed workers across multiple hosts.
- S3-compatible binary storage.
- Centralized logging and monitoring.
- Automated deployment and recovery procedures.

Queue Mode does not support filesystem binary-data storage. Configure supported external binary storage when workflows require it.

## 📚 Official Documentation

- [n8n Official Website](https://n8n.io/)
- [n8n Documentation](https://docs.n8n.io/)
- [n8n Docker Installation](https://docs.n8n.io/hosting/installation/docker/)
- [Queue Mode](https://docs.n8n.io/hosting/scaling/queue-mode/)
- [External Task Runners](https://docs.n8n.io/hosting/configuration/task-runners/)
- [n8n Environment Variables](https://docs.n8n.io/hosting/configuration/environment-variables/)
- [n8n Docker Hosting Examples](https://github.com/n8n-io/n8n-hosting)

## 🤝 Contributing

Contributions, suggestions, and improvements are welcome.

If you find a bug or want to improve the deployment:

1. Fork this repository.
2. Create a feature branch.
3. Commit your changes.
4. Open a Pull Request.

## 📄 License

This repository contains deployment configuration and documentation.

n8n is a separate software project and is distributed under its own license. Refer to the [n8n licensing documentation](https://docs.n8n.io/sustainable-use-license/) for details.

---

**Built with ❤️ using Docker, n8n, PostgreSQL, and Redis.**