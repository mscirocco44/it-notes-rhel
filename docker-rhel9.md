# Docker on RHEL 9
> Covers: Docker CE, Rootless Docker, Docker Compose, Networking, SELinux integration, and Persistent Startup

---

## Table of Contents
- [Overview](#overview)
- [How It Works](#how-it-works)
- [Setup & Installation](#setup--installation)
  - [Online Install](#online-install-internet-connected-host)
  - [Offline — RPM Transfer](#method-a--rpm-download--transfer-simplest)
  - [Offline — Local DNF Repo](#method-b--create-a-local-dnf-repository-better-for-teams--repeated-use)
  - [Offline — Image Transfer](#offline-image-transfer-moving-container-images-without-a-registry)
- [Configuration](#configuration)
- [Common Commands & Usage](#common-commands--usage)
- [Rootless Docker](#rootless-docker)
- [Docker Compose](#docker-compose)
- [Networking](#networking)
- [Making It Persistent](#making-it-persistent)
- [Troubleshooting](#troubleshooting)
- [Practice Labs](#practice-labs)
- [Practice Questions](#practice-questions)
- [References](#references)

---

## Overview

### What is Docker?

Docker is an open-source **container runtime** that packages applications and their dependencies into isolated, portable units called **containers**. Unlike VMs, containers share the host OS kernel, making them lightweight and fast to start.

### Why it matters in DevOps / Platform Engineering

- **Consistency**: "Works on my machine" becomes "works everywhere" — same image runs in dev, CI/CD, staging, and prod.
- **CI/CD pipelines**: GitHub Actions, Jenkins, and GitLab CI all use Docker images as build/test environments.
- **Microservices**: Each service runs in its own container with its own dependencies — no conflicts.
- **Platform Engineering**: You'll manage Dockerfiles, image registries, compose stacks, and eventually migrate these skills to Kubernetes (pods = containers under the hood).
- **Rootless Docker**: A security-hardened pattern essential in enterprise/RHEL environments where running daemons as root is discouraged.

### Where it fits in the Linux/infrastructure ecosystem

```
Developer Workstation / CI Server / Production Host
│
├── Host OS: RHEL 9 (kernel, cgroups v2, namespaces)
│   ├── Docker Daemon (dockerd)  ← manages containers
│   │   ├── Container A (nginx)
│   │   ├── Container B (postgres)
│   │   └── Container C (app)
│   └── Docker Compose           ← orchestrates multi-container apps
│
└── Registry (Docker Hub / Quay.io / private ECR)
    └── Images pulled from here
```

> **Note on RHEL 9 & Docker**: Red Hat officially promotes **Podman** (daemonless, rootless by default) over Docker. However, Docker CE is widely used in industry and DevOps tooling. Both are worth knowing. This guide covers Docker CE installation on RHEL 9.

---

## How It Works

### Core Concepts & Architecture

| Concept | Description |
|---|---|
| **Image** | Read-only template built from a `Dockerfile`. Layered filesystem (UnionFS). |
| **Container** | Running instance of an image. Ephemeral by default. |
| **Dockerfile** | Script of instructions to build an image (`FROM`, `RUN`, `COPY`, `CMD`, etc.) |
| **Registry** | Remote store for images. Default: Docker Hub. Enterprise: Quay.io, AWS ECR. |
| **Volume** | Persistent storage mounted into a container. Survives container removal. |
| **Network** | Virtual network connecting containers. Multiple drivers available. |
| **Docker Compose** | YAML-defined multi-container application stack. |

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│              Docker CLI (client)            │
│         docker run / build / push           │
└──────────────────┬──────────────────────────┘
                   │ REST API (Unix socket)
                   ▼
┌─────────────────────────────────────────────┐
│           Docker Daemon (dockerd)           │
│  ┌─────────────┐   ┌──────────────────────┐ │
│  │  containerd │   │    Image Store       │ │
│  │  (runtime)  │   │  /var/lib/docker/    │ │
│  └──────┬──────┘   └──────────────────────┘ │
│         │                                   │
│  ┌──────▼──────────────────────────────┐    │
│  │         runc (OCI runtime)          │    │
│  │  ┌──────────┐  ┌──────────────────┐ │    │
│  │  │Container1│  │   Container 2    │ │    │
│  │  │ (nginx)  │  │   (postgres)     │ │    │
│  │  └──────────┘  └──────────────────┘ │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

### Linux Kernel Features Docker Uses

- **Namespaces**: Isolate PID, network, mount, UTS, IPC, user spaces per container.
- **cgroups v2** (default on RHEL 9): Limit CPU, memory, I/O per container.
- **UnionFS / OverlayFS**: Layers images efficiently — only changed layers are written.
- **seccomp / AppArmor / SELinux**: Restrict system calls and enforce MAC policies.

### Key Terminology

| Term | Meaning |
|---|---|
| `docker0` | Default bridge network interface created by Docker |
| `OCI` | Open Container Initiative — the standard Docker images and runtimes comply with |
| `rootless` | Running Docker daemon and containers as a non-root user |
| `bind mount` | Mounting a host directory directly into a container |
| `named volume` | Docker-managed persistent storage in `/var/lib/docker/volumes/` |
| `overlay network` | Multi-host networking (used in Docker Swarm / Kubernetes) |
| `entrypoint` | The default executable that runs when a container starts |

---

## Setup & Installation

> **Choose your path**: Jump to [Online Install](#online-install-internet-connected-host) if your host has internet access, or [Offline Install](#offline-install-air-gapped--no-internet) if you're working on an air-gapped or restricted server.

### Common Prerequisites (Both Methods)

```bash
# Check RHEL version
cat /etc/redhat-release

# Ensure system is up to date (online) or at current patch level (offline)
sudo dnf update -y

# Remove conflicting packages — podman-docker is a shim that conflicts with Docker CE
sudo dnf remove -y podman-docker containers-common container-selinux \
                   runc podman buildah

# Verify no conflicts remain
rpm -qa | grep -i podman
rpm -qa | grep -i docker
```

---

### Online Install (Internet-Connected Host)

#### Step 1 — Install utilities and add the Docker repo

```bash
# Install repo management tool
sudo dnf install -y dnf-plugins-core curl

# Add the official Docker CE repo for RHEL
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Verify the repo was added
dnf repolist | grep docker
```

#### Step 2 — Install Docker CE

```bash
# Install Docker CE, CLI, containerd runtime, and Compose + Buildx plugins
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
                    docker-buildx-plugin docker-compose-plugin

# If you also need rootless support (covered in Rootless Docker section):
sudo dnf install -y docker-ce-rootless-extras slirp4netns fuse-overlayfs uidmap
```

#### Step 3 — Start, enable, and configure user access

```bash
# Enable and start the Docker daemon
sudo systemctl enable --now docker

# Add your user to the docker group (avoids sudo for every docker command)
sudo usermod -aG docker $USER

# Apply group change without full logout
newgrp docker
```

#### Step 4 — Verify

```bash
sudo systemctl status docker
docker version
docker run --rm hello-world
docker info | grep -E "Cgroup|Storage Driver|Server Version"
```

Expected `docker info` output:
```
 Server Version: 26.x.x
 Storage Driver: overlay2
 Cgroup Driver: systemd
 Cgroup Version: 2
```

---

### Offline Install (Air-Gapped / No Internet)

Air-gapped environments are common in enterprise/government RHEL deployments. The approach is:
1. Download packages + RPMs on an **internet-connected machine** (same RHEL 9 version)
2. Transfer the bundle to the **air-gapped target host**
3. Install from local files

#### Method A — RPM Download & Transfer (Simplest)

**On an internet-connected RHEL 9 machine:**

```bash
# Create a staging directory
mkdir ~/docker-offline && cd ~/docker-offline

# Add the Docker repo (same as online method)
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Download all RPMs and their dependencies WITHOUT installing
# --downloadonly fetches to current dir by default, or use --destdir
sudo dnf download --resolve --destdir=./rpms \
    docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin \
    docker-ce-rootless-extras slirp4netns fuse-overlayfs uidmap

# List what was downloaded
ls -lh ./rpms/

# Bundle them up for transfer
tar -czvf docker-offline-bundle.tar.gz ./rpms/
```

**Transfer to the air-gapped host:**

```bash
# From the internet-connected machine, copy via SCP or USB/shared storage:
scp docker-offline-bundle.tar.gz user@airgapped-host:/tmp/

# Or copy to USB and mount on the target host
```

**On the air-gapped target host:**

```bash
# Extract the bundle
cd /tmp
tar -xzvf docker-offline-bundle.tar.gz

# Install all RPMs at once (dnf handles dependency order)
sudo dnf localinstall ./rpms/*.rpm -y

# If dnf localinstall isn't available, use rpm directly:
sudo rpm -ivh --nodeps ./rpms/*.rpm

# Verify installation
rpm -qa | grep docker
rpm -qa | grep containerd
```

> **Note**: The `--resolve` flag in `dnf download` pulls all transitive dependencies. If you hit missing deps on install, re-run the download with `--alldeps` or manually add the missing packages to the download list.

#### Method B — Create a Local DNF Repository (Better for Teams / Repeated Use)

This is the preferred enterprise approach: host a local mirror that all air-gapped servers pull from.

**On an internet-connected RHEL 9 machine:**

```bash
# Install createrepo tool
sudo dnf install -y createrepo_c

# Create a directory for your local repo
mkdir -p ~/docker-localrepo/rpms

# Download Docker packages + dependencies
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

sudo dnf download --resolve --destdir=~/docker-localrepo/rpms \
    docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin \
    docker-ce-rootless-extras slirp4netns fuse-overlayfs uidmap

# Generate the repository metadata
cd ~/docker-localrepo
createrepo_c ./rpms/

# Bundle for transfer (or serve over HTTP — see below)
tar -czvf docker-localrepo.tar.gz ./rpms/ repodata/
```

**Option: Serve the repo over HTTP on a bastion/jump host:**

```bash
# On the bastion host (which CAN reach both internet and air-gapped network):
# Install a simple HTTP server
sudo dnf install -y nginx

# Copy your repo to the web root
sudo cp -r ~/docker-localrepo /var/www/html/docker-repo
sudo systemctl enable --now nginx
sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --reload

# The repo is now accessible at: http://bastion-ip/docker-repo/rpms/
```

**On each air-gapped target host:**

```bash
# Create a .repo file pointing to the internal HTTP server
sudo tee /etc/yum.repos.d/docker-local.repo <<EOF
[docker-local]
name=Docker CE Local Mirror
baseurl=http://bastion-ip/docker-repo/rpms/
enabled=1
gpgcheck=0
EOF

# Install as normal
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
                    docker-buildx-plugin docker-compose-plugin

# Or from local extracted tarball:
# sudo dnf install --repofrompath=docker-local,/path/to/rpms/ \
#                  --repo=docker-local -y docker-ce docker-ce-cli containerd.io
```

#### Method C — Use `reposync` to Mirror the Full Docker Repo (Most Scalable)

```bash
# On internet-connected machine, sync entire Docker repo locally
sudo dnf install -y dnf-utils createrepo_c

sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Sync the repo (downloads all packages)
sudo reposync --repo=docker-ce-stable --download-path=/opt/docker-mirror/ --download-metadata

# Generate repo metadata
sudo createrepo_c /opt/docker-mirror/docker-ce-stable/

# Serve with nginx or rsync to target hosts
```

#### Offline Image Transfer (Moving Container Images Without a Registry)

In air-gapped environments, you also need to move Docker **images** (not just packages).

**On an internet-connected machine:**

```bash
# Pull the images you need
docker pull nginx:alpine
docker pull postgres:15-alpine
docker pull hello-world

# Save to a tar archive (can bundle multiple images)
docker save nginx:alpine postgres:15-alpine hello-world \
    | gzip > docker-images-bundle.tar.gz

# Check the size
ls -lh docker-images-bundle.tar.gz
```

**Transfer and load on the air-gapped host:**

```bash
# Transfer
scp docker-images-bundle.tar.gz user@airgapped-host:/tmp/

# Load all images from the archive
docker load < /tmp/docker-images-bundle.tar.gz

# Verify images are available
docker images

# Tag and push to an internal registry if one exists
docker tag nginx:alpine internal-registry.company.com/nginx:alpine
docker push internal-registry.company.com/nginx:alpine
```

> **Pro Tip**: In a team/enterprise setup, consider deploying a private registry (**Harbor**, **Quay**, or plain **Docker Registry**) on the air-gapped network, load images into it once, and have all hosts pull from there.

```bash
# Run a simple local registry on the bastion/internal host
docker run -d \
  --name registry \
  --restart always \
  -p 5000:5000 \
  -v /opt/registry-data:/var/lib/registry:Z \
  registry:2

# Push to it from any host that can reach bastion:
docker tag nginx:alpine bastion-ip:5000/nginx:alpine
docker push bastion-ip:5000/nginx:alpine

# Pull from it:
docker pull bastion-ip:5000/nginx:alpine
```

> If using an HTTP (non-TLS) internal registry, add it to `daemon.json`:
> ```json
> { "insecure-registries": ["bastion-ip:5000"] }
> ```

---

### Post-Install Steps (Both Methods)

```bash
# Enable and start Docker
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker version
docker info
docker run --rm hello-world
```

> **SELinux Note**: Docker on RHEL 9 with SELinux enforcing works correctly with the `overlay2` storage driver. Use `:Z` on volume mounts (e.g., `-v /mydata:/data:Z`) to relabel bind mounts automatically. See the [Troubleshooting](#troubleshooting) section for SELinux denial fixes.

---

## Configuration

### Main Config Files & Locations

| File | Purpose |
|---|---|
| `/etc/docker/daemon.json` | Primary Docker daemon configuration |
| `/usr/lib/systemd/system/docker.service` | systemd unit (don't edit directly) |
| `/etc/systemd/system/docker.service.d/` | Drop-in overrides for the service unit |
| `/var/lib/docker/` | Docker data root (images, volumes, networks) |
| `~/.docker/config.json` | Per-user CLI config (registry auth, etc.) |
| `/etc/containers/registries.conf` | (Shared with Podman) registry search order |

### `/etc/docker/daemon.json` — Annotated Example

```json
{
  "log-driver": "journald",
  "log-opts": {
    "tag": "{{.Name}}"
  },
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    }
  ],
  "dns": ["8.8.8.8", "1.1.1.1"],
  "live-restore": true,
  "userns-remap": "",
  "insecure-registries": [],
  "registry-mirrors": [],
  "max-concurrent-downloads": 5,
  "max-concurrent-uploads": 5
}
```

**Key options explained:**

| Option | Purpose |
|---|---|
| `log-driver: journald` | Sends container logs to systemd journal (queryable with `journalctl`) — preferred on RHEL 9 |
| `storage-driver: overlay2` | Best-practice driver for RHEL 9 / XFS filesystems |
| `live-restore: true` | Containers keep running if dockerd restarts (important for production) |
| `default-address-pools` | Customize the subnet range Docker assigns to bridge networks (avoids conflicts with your LAN) |
| `dns` | Override DNS servers used inside containers |
| `userns-remap` | Enable user namespace remapping for security (maps container root to unprivileged host UID) |

After editing `daemon.json`, always reload:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Validate daemon.json syntax before restarting

```bash
# Python one-liner to check JSON is valid
python3 -m json.tool /etc/docker/daemon.json
```

---

## Common Commands & Usage

### Image Management

```bash
# Search for an image
docker search nginx

# Pull an image
docker pull nginx:latest
docker pull nginx:1.25-alpine

# List local images
docker images
docker image ls

# Remove an image
docker rmi nginx:latest
docker image rm nginx:latest

# Build an image from a Dockerfile in current dir
docker build -t myapp:1.0 .

# Tag an image for a registry
docker tag myapp:1.0 quay.io/myorg/myapp:1.0

# Push to a registry
docker login quay.io
docker push quay.io/myorg/myapp:1.0

# Inspect image layers
docker history nginx:latest
docker inspect nginx:latest
```

### Container Lifecycle

```bash
# Run a container (foreground)
docker run nginx:latest

# Run detached (background), with name and port mapping
docker run -d --name webserver -p 8080:80 nginx:latest

# Run with environment variables
docker run -d -e POSTGRES_PASSWORD=secret postgres:15

# Run interactively (e.g., to explore a container)
docker run -it --rm ubuntu:22.04 /bin/bash

# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# Stop / Start / Restart
docker stop webserver
docker start webserver
docker restart webserver

# Remove a container (must be stopped first)
docker rm webserver

# Force remove running container
docker rm -f webserver

# Execute a command inside a running container
docker exec -it webserver /bin/bash
docker exec webserver nginx -t

# Copy files to/from container
docker cp ./index.html webserver:/usr/share/nginx/html/
docker cp webserver:/etc/nginx/nginx.conf ./nginx.conf
```

### Logs & Inspection

```bash
# View container logs
docker logs webserver
docker logs -f webserver          # follow (tail -f style)
docker logs --tail 50 webserver   # last 50 lines
docker logs --since 1h webserver  # last hour

# Inspect container metadata (IP, mounts, env vars, etc.)
docker inspect webserver

# Resource usage stats
docker stats
docker stats webserver --no-stream   # one-shot snapshot

# View processes inside container
docker top webserver
```

### Volume Management

```bash
# Create a named volume
docker volume create pgdata

# Run container with named volume
docker run -d -v pgdata:/var/lib/postgresql/data postgres:15

# Run with bind mount (host path : container path)
docker run -d -v /opt/myapp/config:/app/config:ro myapp:1.0

# List volumes
docker volume ls

# Inspect a volume (shows mountpoint on host)
docker volume inspect pgdata

# Remove a volume
docker volume rm pgdata

# Remove all unused volumes
docker volume prune
```

### Cleanup

```bash
# Remove all stopped containers, unused networks, dangling images, build cache
docker system prune

# Also remove unused images (not just dangling)
docker system prune -a

# Check disk usage
docker system df
```

---

## Rootless Docker

### What is Rootless Docker?

Rootless Docker runs both the Docker daemon **and** containers as a **non-root user**. This is a security hardening technique:
- If a container is compromised, the attacker gets a non-privileged user on the host — not root.
- Required in many enterprise/compliance environments.
- On RHEL 9, this is the recommended approach alongside Podman.

### How it differs from regular Docker

| Aspect | Root Docker | Rootless Docker |
|---|---|---|
| Daemon runs as | `root` | Your user (e.g., `devuser`) |
| Socket location | `/var/run/docker.sock` | `$XDG_RUNTIME_DIR/docker.sock` |
| Network | Full `iptables` access | Uses `slirp4netns` or `pasta` (userspace networking) |
| Port binding <1024 | Allowed | Requires `net.ipv4.ip_unprivileged_port_start` tweak |
| systemd unit | `docker.service` (system) | `docker.service` (user) |
| Data root | `/var/lib/docker` | `~/.local/share/docker` |

### Prerequisites for Rootless Docker

#### Online (Internet-Connected Host)

```bash
# Install rootless prerequisites
sudo dnf install -y slirp4netns fuse-overlayfs uidmap docker-ce-rootless-extras

# Verify subuid/subgid mappings exist for your user
grep $USER /etc/subuid   # should show: username:100000:65536
grep $USER /etc/subgid

# If missing, add them:
sudo usermod --add-subuids 100000-165535 $USER
sudo usermod --add-subgids 100000-165535 $USER
```

#### Offline (Air-Gapped Host)

The rootless extras (`slirp4netns`, `fuse-overlayfs`, `uidmap`, `docker-ce-rootless-extras`) must be downloaded on a connected machine and transferred — same pattern as the main offline install.

**On internet-connected machine:**

```bash
mkdir ~/rootless-offline && cd ~/rootless-offline

sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

sudo dnf download --resolve --destdir=./rpms \
    docker-ce-rootless-extras slirp4netns fuse-overlayfs uidmap

tar -czvf rootless-extras.tar.gz ./rpms/
scp rootless-extras.tar.gz user@airgapped-host:/tmp/
```

**On air-gapped host:**

```bash
cd /tmp && tar -xzvf rootless-extras.tar.gz
sudo dnf localinstall ./rpms/*.rpm -y

# Verify subuid/subgid
grep $USER /etc/subuid
grep $USER /etc/subgid

# If missing:
sudo usermod --add-subuids 100000-165535 $USER
sudo usermod --add-subgids 100000-165535 $USER
```

### Install Rootless Docker

```bash
# Run the rootless setup script (as your regular user, NOT root)
dockerd-rootless-setuptool.sh install

# If the script isn't found after offline install, check the path:
find / -name "dockerd-rootless-setuptool.sh" 2>/dev/null
# Typically at: /usr/bin/dockerd-rootless-setuptool.sh

# Then run setup again as your user:
dockerd-rootless-setuptool.sh install
```

### Configure the environment

Add to your `~/.bashrc` or `~/.bash_profile`:

```bash
# Rootless Docker environment
export PATH=/usr/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

```bash
source ~/.bashrc
```

### Verify rootless is working

```bash
# Check the daemon is running as your user
systemctl --user status docker

# Confirm socket path
echo $DOCKER_HOST

# Run a test container
docker run --rm hello-world

# Confirm the daemon PID is owned by your user, not root
ps aux | grep dockerd
```

### Allowing low-numbered ports (< 1024) in rootless mode

```bash
# Check current setting
sysctl net.ipv4.ip_unprivileged_port_start

# Temporarily allow ports >= 80
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80

# Make it persistent
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-rootless-docker.conf
sudo sysctl --system
```

---

## Docker Compose

### What is Docker Compose?

Docker Compose lets you define and run **multi-container applications** using a single `docker-compose.yml` (or `compose.yaml`) file. On RHEL 9 with Docker CE, the modern **Compose v2** plugin is used (`docker compose` — no hyphen).

### Compose File Structure

```yaml
# compose.yaml (modern name) or docker-compose.yml

services:
  web:
    image: nginx:1.25-alpine
    container_name: web
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro,Z   # :Z = SELinux relabel
    networks:
      - frontend
    depends_on:
      - app
    restart: unless-stopped

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=mydb
      - DB_USER=appuser
      - DB_PASS=${DB_PASSWORD}    # from .env file
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    container_name: db
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data:Z
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pgdata:
    driver: local

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # no external access — only internal container-to-container
```

### `.env` file (secrets/config, never commit to git)

```bash
# .env
DB_PASSWORD=supersecretpassword
APP_ENV=production
```

### Docker Compose Commands

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# View running services
docker compose ps

# View logs (all services)
docker compose logs -f

# View logs for specific service
docker compose logs -f app

# Execute command in a running service container
docker compose exec app /bin/sh

# Stop all services (containers remain)
docker compose stop

# Stop and remove containers, networks (volumes preserved)
docker compose down

# Stop and remove containers, networks, AND volumes
docker compose down -v

# Scale a service (run multiple instances)
docker compose up -d --scale app=3

# Validate compose file syntax
docker compose config

# Pull latest images for all services
docker compose pull
```

### Dockerfile Example (used by `build:` in compose)

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy and install dependencies first (layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user to run the app
RUN useradd -m appuser
USER appuser

EXPOSE 8000

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Networking

### Docker Network Drivers

| Driver | Use Case |
|---|---|
| `bridge` | Default. Isolated network per compose project / manual network. Containers on same bridge can talk to each other. |
| `host` | Container shares the host's network namespace. No isolation. High performance. |
| `none` | No networking. Complete network isolation. |
| `overlay` | Multi-host (Docker Swarm). Not needed for single-host setups. |
| `macvlan` | Assigns container a MAC address on the physical network — appears as separate device. |

### Common Network Commands

```bash
# List networks
docker network ls

# Inspect a network (shows containers attached, subnet, etc.)
docker network inspect bridge

# Create a custom bridge network
docker network create --driver bridge \
  --subnet 172.25.0.0/24 \
  --gateway 172.25.0.1 \
  mynet

# Connect/disconnect a running container to a network
docker network connect mynet webserver
docker network disconnect mynet webserver

# Remove a network
docker network rm mynet

# Remove all unused networks
docker network prune
```

### Common Networking Issues on RHEL 9

#### 1. Containers can't reach the internet

The most common issue on RHEL 9. Caused by **firewalld** not allowing forwarded traffic through the Docker bridge.

```bash
# Check if IP forwarding is enabled
sysctl net.ipv4.ip_forward   # should be 1

# Enable if not (Docker sets this, but verify)
sudo sysctl -w net.ipv4.ip_forward=1

# Permanent
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-docker-forward.conf
sudo sysctl --system

# Check firewalld zone for docker0
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=trusted --list-all

# Add docker0 interface to trusted zone (allows container traffic)
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --reload

# If using custom bridge networks (their interfaces are veth/br- prefixed):
# Add the entire Docker subnet to trusted
sudo firewall-cmd --permanent --zone=trusted --add-source=172.17.0.0/16
sudo firewall-cmd --reload
```

#### 2. Port published containers unreachable from outside the host

```bash
# Check firewalld is allowing the published port
sudo firewall-cmd --list-ports

# Add the port (e.g., for a container published on 8080)
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Verify the iptables rule Docker added
sudo iptables -t nat -L DOCKER --line-numbers
sudo iptables -L DOCKER-USER --line-numbers
```

#### 3. firewalld and Docker iptables conflicts

Docker writes its own `iptables` rules. firewalld can flush these on reload. Solution:

```bash
# Check if DOCKER-USER chain exists (it should — Docker creates it)
sudo iptables -L DOCKER-USER

# Add custom rules to DOCKER-USER (persists across docker restarts)
# Block all traffic to container except from specific source:
sudo iptables -I DOCKER-USER -i ens3 -j DROP
sudo iptables -I DOCKER-USER -i ens3 -s 192.168.1.0/24 -j RETURN

# To make iptables rules survive reboot:
sudo dnf install -y iptables-services
sudo iptables-save | sudo tee /etc/sysconfig/iptables
sudo systemctl enable iptables
```

#### 4. DNS resolution failing inside containers

```bash
# Test DNS from inside a container
docker run --rm busybox nslookup google.com

# If failing, check /etc/resolv.conf on the host
cat /etc/resolv.conf

# Override DNS in daemon.json
# Edit /etc/docker/daemon.json:
# { "dns": ["8.8.8.8", "8.8.4.4"] }
sudo systemctl restart docker
```

#### 5. SELinux blocking container volume mounts

```bash
# Symptom: container starts but can't read/write bind-mounted host directory
# Fix 1: Use :Z (private relabel) or :z (shared relabel) in volume mount

docker run -v /opt/mydata:/data:Z myimage   # :Z = SELinux private label
docker run -v /opt/mydata:/data:z myimage   # :z = SELinux shared label

# In compose.yaml:
volumes:
  - ./data:/app/data:Z

# Fix 2: Check and set context manually
ls -Z /opt/mydata
sudo chcon -Rt container_file_t /opt/mydata

# Fix 3: Check SELinux denials
sudo ausearch -m avc -ts recent
sudo audit2allow -a    # see what rules would allow it
```

#### 6. Subnet conflicts with existing network

```bash
# If Docker's default 172.17.0.0/16 conflicts with your VPN or LAN:
# Edit /etc/docker/daemon.json to change the default address pool:
{
  "default-address-pools": [
    { "base": "192.168.100.0/16", "size": 24 }
  ],
  "bip": "192.168.100.1/24"
}

sudo systemctl restart docker
```

---

## Making It Persistent

### Root Docker — systemd (System-wide)

The standard Docker CE installation on RHEL 9 runs as a **system service**.

```bash
# Enable Docker to start at boot
sudo systemctl enable docker

# Start immediately
sudo systemctl start docker

# Check status
sudo systemctl status docker

# View logs
sudo journalctl -u docker -f
sudo journalctl -u docker --since "1 hour ago"
```

#### Custom systemd drop-in override (don't edit the unit file directly)

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d/

sudo tee /etc/systemd/system/docker.service.d/override.conf <<EOF
[Service]
# Add environment variables or override ExecStart flags
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.internal.domain"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### Auto-restart containers on boot with Docker run

```bash
# --restart policies:
# no          = never restart (default)
# always      = always restart (even after docker restart / reboot)
# unless-stopped = restart unless manually stopped
# on-failure  = restart on non-zero exit code

docker run -d --restart unless-stopped --name webserver -p 80:80 nginx:latest
```

#### Auto-restart with Docker Compose (persistent stack)

In `compose.yaml`, add `restart: unless-stopped` to each service (as shown in the Compose example above). Then run:

```bash
docker compose up -d
# Containers will auto-restart after reboot as long as dockerd starts at boot
```

### Rootless Docker — systemd User Service

Rootless Docker uses **user-level systemd** (`--user`). The critical difference: user systemd services only start when the user **logs in** by default. For headless servers, you need `loginctl enable-linger`.

```bash
# Enable rootless docker service for your user
systemctl --user enable docker

# Start it now
systemctl --user start docker

# Check status
systemctl --user status docker

# View logs
journalctl --user -u docker -f

# CRITICAL: Enable linger so the user service starts at boot WITHOUT login
sudo loginctl enable-linger $USER

# Verify linger is enabled
loginctl show-user $USER | grep Linger
# Should show: Linger=yes
```

#### Why `loginctl enable-linger` matters

Without linger enabled:
- Rootless Docker only starts when you SSH in
- All rootless containers stop when you log out
- On server reboots, no containers start until login

With linger enabled:
- User systemd session starts at boot
- Rootless Docker daemon starts automatically
- Containers with `restart: unless-stopped` come back up

```bash
# Complete rootless persistent startup checklist:
systemctl --user enable docker               # enable user service
sudo loginctl enable-linger $USER            # start at boot without login
# + Set restart policy on containers/compose services
```

#### Auto-start a rootless compose stack at boot

```bash
# Create a systemd user service to manage the compose stack
mkdir -p ~/.config/systemd/user/

cat > ~/.config/systemd/user/mystack.service <<EOF
[Unit]
Description=My Docker Compose Stack
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/%i/myapp
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now mystack.service
systemctl --user status mystack.service
```

### firewalld rules to persist after reboot

Any `--permanent` firewall-cmd rules persist across reboots automatically:

```bash
# These already persist (--permanent flag):
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Verify persistent rules
sudo firewall-cmd --permanent --zone=trusted --list-all
```

---

## Troubleshooting

### Common Issues & Fixes

#### Docker daemon won't start

```bash
# Check status and recent errors
sudo systemctl status docker
sudo journalctl -u docker -n 50

# Common causes:
# 1. Invalid daemon.json syntax
python3 -m json.tool /etc/docker/daemon.json

# 2. Port conflict on dockerd
sudo ss -tlnp | grep dockerd

# 3. SELinux blocking dockerd
sudo ausearch -m avc -ts recent | grep docker
```

#### "permission denied" on docker socket

```bash
# Check socket permissions
ls -la /var/run/docker.sock

# Ensure your user is in the docker group
groups $USER

# Apply group change without logout
newgrp docker

# If still failing, verify group membership:
id $USER
```

#### Container exits immediately

```bash
# Check exit code and logs
docker ps -a   # look at STATUS column
docker logs <container_name>

# Common fix: run interactively to debug
docker run -it --rm <image> /bin/sh
```

#### "No space left on device"

```bash
# Check Docker disk usage
docker system df

# Clean up aggressively
docker system prune -a --volumes

# Check actual disk space
df -h /var/lib/docker
```

#### SELinux AVC denials

```bash
# View recent denials
sudo ausearch -m avc -ts recent

# One-liner to see docker-related denials
sudo ausearch -m avc -ts recent | grep docker

# Generate a permissive policy module (for testing ONLY, not production)
sudo ausearch -m avc -ts recent | sudo audit2allow -M mydockerfix
sudo semodule -i mydockerfix.pp

# Preferred fix: use :Z on volume mounts, or chcon
sudo chcon -Rt container_file_t /path/to/host/dir
```

#### Rootless Docker: containers can't access internet

```bash
# Rootless uses slirp4netns for networking — check it's installed
which slirp4netns

# Test connectivity
docker run --rm busybox ping -c 3 8.8.8.8

# Check DNS inside container
docker run --rm busybox nslookup google.com

# If DNS fails in rootless, set nameserver explicitly:
# Add to daemon.json: { "dns": ["8.8.8.8"] }
# (Located at: ~/.config/docker/daemon.json for rootless)
```

### Useful Log Locations

| Location | Contents |
|---|---|
| `journalctl -u docker` | Docker daemon logs (system) |
| `journalctl --user -u docker` | Rootless Docker daemon logs |
| `docker logs <name>` | Container stdout/stderr |
| `/var/lib/docker/containers/<id>/<id>-json.log` | Raw container log file (if using json-file driver) |
| `sudo ausearch -m avc` | SELinux AVC denial logs |
| `/var/log/audit/audit.log` | Full audit log (SELinux, etc.) |

### Diagnostic One-Liners

```bash
# Full system overview
docker system info

# All containers with resource usage
docker stats --no-stream

# Inspect container networking
docker inspect <name> | python3 -m json.tool | grep -A 20 "Networks"

# Check what ports are published
docker port <name>

# Trace a container's filesystem changes
docker diff <name>

# Test if Docker daemon API is responding
curl --unix-socket /var/run/docker.sock http://localhost/version

# Rootless: test API
curl --unix-socket $XDG_RUNTIME_DIR/docker.sock http://localhost/version

# Check iptables rules Docker created
sudo iptables -t nat -L -n --line-numbers | grep -A5 DOCKER
```

---

## Practice Labs

### Lab 1 — Deploy a Static Website with Nginx

**Objective**: Run a containerised nginx web server serving a custom HTML file, accessible from the host browser.

**Steps**:

```bash
# 1. Create a working directory
mkdir ~/lab1-nginx && cd ~/lab1-nginx

# 2. Create a simple HTML page
cat > index.html <<EOF
<!DOCTYPE html>
<html>
  <body><h1>Hello from Docker on RHEL 9!</h1></body>
</html>
EOF

# 3. Run nginx with the HTML bind-mounted
#    :Z = relabels the mount for SELinux
docker run -d \
  --name lab1-web \
  --restart unless-stopped \
  -p 8080:80 \
  -v $(pwd)/index.html:/usr/share/nginx/html/index.html:ro,Z \
  nginx:alpine

# 4. Test
curl http://localhost:8080

# 5. View logs
docker logs lab1-web

# 6. Open the port in firewalld if testing from another machine
sudo firewall-cmd --add-port=8080/tcp --permanent && sudo firewall-cmd --reload

# 7. Cleanup
docker rm -f lab1-web
```

**Expected outcome**: `curl http://localhost:8080` returns your custom HTML.

---

### Lab 2 — Multi-Container App with Docker Compose

**Objective**: Deploy a WordPress + MariaDB stack using Docker Compose with persistent volumes and a custom network.

**Steps**:

```bash
mkdir ~/lab2-wordpress && cd ~/lab2-wordpress

# 1. Create .env file
cat > .env <<EOF
MYSQL_ROOT_PASSWORD=rootpass123
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser
MYSQL_PASSWORD=wppass123
EOF

# 2. Create compose.yaml
cat > compose.yaml <<EOF
services:
  db:
    image: mariadb:10.11
    container_name: wp-db
    environment:
      MYSQL_ROOT_PASSWORD: \${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: \${MYSQL_DATABASE}
      MYSQL_USER: \${MYSQL_USER}
      MYSQL_PASSWORD: \${MYSQL_PASSWORD}
    volumes:
      - dbdata:/var/lib/mysql:Z
    networks:
      - wpnet
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      retries: 5

  wordpress:
    image: wordpress:latest
    container_name: wp-app
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: \${MYSQL_DATABASE}
      WORDPRESS_DB_USER: \${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: \${MYSQL_PASSWORD}
    volumes:
      - wpdata:/var/www/html:Z
    networks:
      - wpnet
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  dbdata:
  wpdata:

networks:
  wpnet:
    driver: bridge
EOF

# 3. Start the stack
docker compose up -d

# 4. Watch logs until wordpress is ready
docker compose logs -f

# 5. Access: http://localhost:8080

# 6. Stop and clean up (keeps volumes)
docker compose down

# 7. Full cleanup including volumes
docker compose down -v
```

**Expected outcome**: WordPress setup wizard accessible at `http://localhost:8080`.

---

### Lab 3 — Rootless Docker Setup & Verification

**Objective**: Configure and verify rootless Docker for a non-root user.

**Steps**:

```bash
# 1. Install prerequisites (as root/sudo)
sudo dnf install -y slirp4netns fuse-overlayfs uidmap docker-ce-rootless-extras

# 2. Verify subuid/subgid (as root/sudo)
grep $USER /etc/subuid || sudo usermod --add-subuids 100000-165535 $USER
grep $USER /etc/subgid || sudo usermod --add-subgids 100000-165535 $USER

# 3. Switch to your regular user (if running as root)
# su - youruser

# 4. Install rootless Docker (as your user)
dockerd-rootless-setuptool.sh install

# 5. Set environment variables
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc

# 6. Start and enable user service
systemctl --user enable --now docker

# 7. Enable linger for boot persistence
sudo loginctl enable-linger $USER

# 8. Verify
systemctl --user status docker
docker run --rm hello-world
docker info | grep -i rootless

# 9. Test internet access from container
docker run --rm busybox ping -c 3 8.8.8.8
```

**Expected outcome**: `docker info` shows rootless mode; hello-world runs successfully; containers persist after logout/reboot.

---

### Lab 4 — Diagnose and Fix a Networking Problem

**Objective**: Deliberately break Docker networking, diagnose it, and fix it.

**Steps**:

```bash
# 1. Start a simple web container
docker run -d --name nettest -p 8081:80 nginx:alpine

# 2. Verify it works
curl http://localhost:8081   # should return nginx welcome page

# 3. "Break" networking by removing docker0 from firewalld trusted zone
sudo firewall-cmd --zone=trusted --remove-interface=docker0

# 4. Try to access the internet FROM inside the container
docker exec nettest wget -qO- http://google.com --timeout=3   # should fail

# 5. Diagnose
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=trusted --list-all
sudo iptables -L FORWARD | head -20

# 6. Fix: re-add docker0 to trusted zone (permanent)
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --reload

# 7. Verify fix
docker exec nettest wget -qO- http://google.com --timeout=5 | head -5

# 8. Cleanup
docker rm -f nettest
```

**Expected outcome**: After fix, container can reach the internet; curl still works from host.

---

### Lab 5 — Build a Custom Image and Push to a Registry

**Objective**: Write a Dockerfile, build an image, and push it to Docker Hub (or a local registry).

**Steps**:

```bash
mkdir ~/lab5-custom && cd ~/lab5-custom

# 1. Write a simple Python app
cat > app.py <<EOF
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from my custom RHEL 9 Docker image!")

HTTPServer(("", 8000), Handler).serve_forever()
EOF

# 2. Write the Dockerfile
cat > Dockerfile <<EOF
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
RUN useradd -m appuser
USER appuser
EXPOSE 8000
CMD ["python", "app.py"]
EOF

# 3. Build the image
docker build -t myapp:1.0 .

# 4. Run it locally
docker run -d --name myapp-test -p 8082:8000 myapp:1.0
curl http://localhost:8082

# 5. Tag for Docker Hub (replace 'yourusername')
docker tag myapp:1.0 yourusername/myapp:1.0

# 6. Login and push
docker login
docker push yourusername/myapp:1.0

# 7. Pull it back down to verify
docker rmi myapp:1.0 yourusername/myapp:1.0
docker pull yourusername/myapp:1.0
docker run --rm -p 8083:8000 yourusername/myapp:1.0 &
curl http://localhost:8083

# 8. Cleanup
docker rm -f myapp-test
```

**Expected outcome**: Custom image built, pushed, pulled, and running successfully.

---

## Practice Questions

**Q1** — What is the default storage driver used by Docker on RHEL 9, and why is it preferred?

<details>
<summary>Answer</summary>

**`overlay2`** is the default and preferred storage driver on RHEL 9.

It is preferred because:
- It works natively with the **XFS** filesystem (default on RHEL 9) using `d_type` support.
- It is efficient — only changed layers are written (copy-on-write semantics).
- It has lower overhead than the older `devicemapper` driver.
- It is fully supported by the OCI spec and Docker CE.

Verify with: `docker info | grep "Storage Driver"`

</details>

---

**Q2** — What command enables a container to restart automatically after a host reboot?

<details>
<summary>Answer</summary>

Use the `--restart` flag when running the container:

```bash
docker run -d --restart unless-stopped --name mycontainer nginx:alpine
```

For an existing container, update it:
```bash
docker update --restart unless-stopped mycontainer
```

In Docker Compose:
```yaml
services:
  web:
    restart: unless-stopped
```

`unless-stopped` is generally preferred over `always` because it respects a manual `docker stop` — the container won't restart again until you explicitly start it or reboot.

> Note: This only works if the **Docker daemon itself** starts at boot (`systemctl enable docker`).

</details>

---

**Q3** — You run `docker run -v /opt/data:/app/data myimage` on RHEL 9 with SELinux enforcing, and the container can't read the files. What is the fix?

<details>
<summary>Answer</summary>

SELinux is blocking the bind mount because the host directory doesn't have the correct SELinux context (`container_file_t`).

**Fix 1** — Add `:Z` (or `:z`) to the volume mount:
```bash
docker run -v /opt/data:/app/data:Z myimage   # private label (recommended)
docker run -v /opt/data:/app/data:z myimage   # shared label
```

**Fix 2** — Manually relabel the host directory:
```bash
sudo chcon -Rt container_file_t /opt/data
```

**Fix 3** — Check what SELinux is actually denying:
```bash
sudo ausearch -m avc -ts recent | grep docker
```

`:Z` is the idiomatic Docker-on-RHEL approach. Use `:z` only when the directory is shared between multiple containers simultaneously.

</details>

---

**Q4** — What is the key difference between rootless Docker and running Docker with a non-root user inside a container?

<details>
<summary>Answer</summary>

These are two different concepts:

| | Rootless Docker | Non-root user in container |
|---|---|---|
| **What runs as non-root** | The Docker **daemon** and all container processes | Only the process inside the container |
| **Daemon privilege** | Runs under your user UID | Runs as `root` on the host |
| **Security benefit** | If daemon is compromised, attacker only gets your user | If container is compromised but daemon is root, privilege escalation to host root is possible |
| **Configuration** | `dockerd-rootless-setuptool.sh install` | `USER appuser` in Dockerfile or `--user` flag |

Both are good security practices and **complement each other** — you can run rootless Docker AND have a non-root user inside the container.

</details>

---

**Q5** — A container running in Docker Compose can't connect to another container by hostname. What should you check first?

<details>
<summary>Answer</summary>

In Docker Compose, containers can communicate by **service name** (not container name) when they are on the **same network**.

Check:
1. Both services are on the **same named network** in `compose.yaml`.
2. You're using the **service name** (e.g., `db`) as the hostname, not the container name or IP.
3. The target service is actually running: `docker compose ps`

Example — if `app` can't reach `db`:
```yaml
services:
  app:
    networks:
      - backend
  db:
    networks:
      - backend
networks:
  backend:
```

Inside `app`, connect to `db:5432` — Docker's embedded DNS resolves service names within the same network.

If they're on **different networks**, you need to either connect both to the same network or expose a port.

</details>

---

**Q6** — Explain `loginctl enable-linger` and why it's required for rootless Docker on a headless RHEL 9 server.

<details>
<summary>Answer</summary>

`loginctl enable-linger <user>` tells systemd to **start and maintain a user session at boot**, even when the user is not logged in.

Without linger:
- User systemd services (like rootless `docker.service`) only start when the user logs in via SSH or console.
- All rootless containers stop when the user logs out.
- After a server reboot, no rootless containers start until you log in.

With linger enabled:
```bash
sudo loginctl enable-linger $USER
```
- A `user@UID.service` systemd session starts at boot.
- `systemctl --user enable docker` causes rootless dockerd to start automatically.
- Containers with restart policies come back up without any login required.

Verify:
```bash
loginctl show-user $USER | grep Linger
# Linger=yes
```

</details>

---

**Q7** — What does the `DOCKER-USER` iptables chain do, and why should you use it instead of modifying Docker's other chains?

<details>
<summary>Answer</summary>

`DOCKER-USER` is a **custom iptables chain** that Docker creates specifically for administrators to add their own rules. It is processed **before Docker's own rules** in the `FORWARD` chain.

Why use `DOCKER-USER` instead of `DOCKER` or `DOCKER-ISOLATION-*` chains:
- Docker **overwrites** its own chains (`DOCKER`, `DOCKER-ISOLATION-STAGE-1/2`) every time the daemon starts or `docker network` commands run.
- `DOCKER-USER` is **never modified by Docker** — your rules persist across daemon restarts.

Example — restrict container access to only a specific source IP:
```bash
# Block all forwarded traffic to containers
sudo iptables -I DOCKER-USER -i eth0 -j DROP
# But allow from a management subnet
sudo iptables -I DOCKER-USER -i eth0 -s 10.0.1.0/24 -j RETURN
```

To make rules survive reboot:
```bash
sudo iptables-save | sudo tee /etc/sysconfig/iptables
sudo systemctl enable iptables
```

</details>

---

**Q8** — You have a `compose.yaml` with a `db` service, and you've deleted the postgres container. After running `docker compose up -d` again, your data is gone. What went wrong and how do you prevent it?

<details>
<summary>Answer</summary>

**Root cause**: The database data was stored in the container's writable layer (no volume defined), so when the container was removed, all data was lost.

**Fix**: Always use a **named volume** for database data:

```yaml
services:
  db:
    image: postgres:15-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data:Z   # named volume
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  pgdata:   # Docker manages this at /var/lib/docker/volumes/pgdata/
```

Named volumes:
- Live in `/var/lib/docker/volumes/` (or `~/.local/share/docker/volumes/` for rootless)
- Survive `docker compose down` and `docker rm`
- Are **only** removed with `docker compose down -v` or `docker volume rm`

Never use a bind mount for database data in production (permissions, SELinux context issues). Use named volumes.

</details>

---

**Q9** — Walk through all the steps to set up rootless Docker so that containers start automatically on RHEL 9 boot, even when no user is logged in.

<details>
<summary>Answer</summary>

Complete end-to-end checklist:

```bash
# 1. Install prerequisites (as sudo)
sudo dnf install -y slirp4netns fuse-overlayfs uidmap docker-ce-rootless-extras

# 2. Ensure subuid/subgid mappings exist
grep $USER /etc/subuid || sudo usermod --add-subuids 100000-165535 $USER
grep $USER /etc/subgid || sudo usermod --add-subgids 100000-165535 $USER

# 3. Run rootless setup script (as your user)
dockerd-rootless-setuptool.sh install

# 4. Set DOCKER_HOST (add to .bashrc)
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc

# 5. Enable rootless docker user service
systemctl --user enable --now docker

# 6. Enable linger (CRITICAL for headless boot)
sudo loginctl enable-linger $USER

# 7. Verify linger
loginctl show-user $USER | grep Linger  # should be yes

# 8. Start containers with restart policy
docker run -d --restart unless-stopped --name myapp -p 8080:80 nginx:alpine

# 9. Or use compose with restart: unless-stopped and systemctl --user enable
```

**Test**: Reboot the server and verify `docker ps` shows your containers running without logging in first (SSH in after boot and check).

</details>

---

**Q10** — You deploy a Docker Compose stack on RHEL 9 and containers on the `backend` network (marked `internal: true`) can communicate with each other, but they cannot reach external internet — which is intended. However, the `frontend` containers also can't reach the internet. Describe your debugging process.

<details>
<summary>Answer</summary>

**Systematic debugging process**:

**Step 1** — Confirm the symptom is real and narrow the scope:
```bash
docker compose exec frontend_service wget -qO- http://1.1.1.1 --timeout=3  # test by IP
docker compose exec frontend_service wget -qO- http://google.com --timeout=3  # test DNS
```
If IP works but DNS fails → DNS issue. If both fail → routing/firewall issue.

**Step 2** — Check IP forwarding:
```bash
sysctl net.ipv4.ip_forward   # must be 1
```

**Step 3** — Check firewalld zones:
```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=trusted --list-all
# Is docker0 / the frontend bridge interface in the trusted zone?
```

**Step 4** — Identify the bridge interface for the frontend network:
```bash
docker network ls
docker network inspect <project>_frontend | grep -i interface
ip addr show | grep br-   # find the veth/bridge interface
```

**Step 5** — Add the frontend network interface to the trusted zone:
```bash
sudo firewall-cmd --permanent --zone=trusted --add-interface=br-<id>
sudo firewall-cmd --reload
```

**Step 6** — Or add the subnet:
```bash
docker network inspect <project>_frontend | grep Subnet
sudo firewall-cmd --permanent --zone=trusted --add-source=172.20.0.0/24
sudo firewall-cmd --reload
```

**Step 7** — Check iptables FORWARD chain:
```bash
sudo iptables -L FORWARD -n | grep DROP
```

**Step 8** — Verify `internal: true` is only on the backend network (not accidentally on frontend):
```bash
docker network inspect <project>_backend | grep '"Internal"'   # should be true
docker network inspect <project>_frontend | grep '"Internal"'  # should be false
```

The likely root cause is the frontend bridge interface not being in the `trusted` firewalld zone, which prevents the kernel from forwarding packets from containers to the default gateway.

</details>

---

## References

### Man Pages

```bash
man docker
man docker-run
man docker-build
man docker-compose
man dockerd
man Dockerfile   # (may need: docker help Dockerfile)
```

### Official Documentation

- [Docker CE on RHEL — Official Install Guide](https://docs.docker.com/engine/install/rhel/)
- [Rootless Docker — Official Docs](https://docs.docker.com/engine/security/rootless/)
- [Docker Compose File Reference](https://docs.docker.com/compose/compose-file/)
- [Docker Networking Overview](https://docs.docker.com/network/)
- [Red Hat — Container Security Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/index)
- [Red Hat — Using Docker on RHEL](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9)
- [SELinux and Containers — Red Hat Blog](https://www.redhat.com/en/blog/container-permission-denied-errors)
- [firewalld and Docker — Known Issues](https://docs.docker.com/network/iptables/)
- [systemd User Services & Linger](https://www.freedesktop.org/software/systemd/man/loginctl.html)
- [Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)

### Useful `journalctl` Commands for Docker

```bash
# System dockerd logs
sudo journalctl -u docker -f

# Rootless dockerd logs
journalctl --user -u docker -f

# Container logs (if log-driver = journald)
journalctl CONTAINER_NAME=webserver -f

# Logs since last boot
sudo journalctl -u docker -b

# Filter by time
sudo journalctl -u docker --since "2024-01-01 10:00" --until "2024-01-01 11:00"
```
