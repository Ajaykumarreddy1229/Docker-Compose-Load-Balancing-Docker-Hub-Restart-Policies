# Docker-Compose-Load-Balancing-Docker-Hub-Restart-Policies


## Table of Contents

1. [Docker Compose](#1-docker-compose)
2. [Docker Compose Commands](#2-docker-compose-commands)
3. [Nginx Load Balancing](#3-nginx-load-balancing)
4. [Python Flask Load Balancer Project](#4-python-flask-load-balancer-project)
5. [Docker Hub](#5-docker-hub)
6. [Docker Restart Policies](#6-docker-restart-policies)
7. [Best Practices](#7-best-practices)
8. [Interview Questions](#8-interview-questions)

---

## 1. Docker Compose

### What Is Docker Compose?

Docker Compose is a tool used to define and run multi-container Docker applications using a YAML configuration file.

Instead of manually running multiple `docker build` and `docker run` commands, we can define application services, ports, networks, volumes, and other settings in a Compose file.

### Why Do We Need Docker Compose?

Before using Docker Compose, we might need to:

* Create separate Dockerfiles for individual applications.
* Build each Docker image.
* Run each container separately.
* Configure port mappings manually.
* Start, stop, and manage containers individually.

Docker Compose simplifies this process by allowing us to manage multiple services together.

### Key Features

* Defines multiple services in a YAML file.
* Creates and manages containers together.
* Supports service networking.
* Supports volumes and persistent storage.
* Provides centralized logs and status information.
* Works well for local development, testing, and single-host deployments.

### HDFC Banking Application Example

Imagine an application with four modules:

* Internet Banking
* Mobile Banking
* Insurance
* Loans

Each module runs in a separate container with an Apache web server.

### Dockerfile

Create a `Dockerfile` inside each application directory:

```dockerfile
FROM ubuntu

RUN apt update -y
RUN apt install apache2 -y

COPY index.html /var/www/html/

CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND"]
```

This Dockerfile installs Apache, copies the HTML page into the web root, and starts Apache in the foreground.

### Build the Images

Run these commands from their respective application directories:

```bash
docker build -t internetbanking:v1 .
docker build -t mobilebanking:v1 .
docker build -t insurance:v1 .
docker build -t loans:v1 .
```

### Create `docker-compose.yml`

Create this file in the project directory:

```yaml
services:
  internetbanking:
    image: internetbanking:v1
    ports:
      - "81:80"

  mobilebanking:
    image: mobilebanking:v1
    ports:
      - "82:80"

  insurance:
    image: insurance:v1
    ports:
      - "83:80"

  loans:
    image: loans:v1
    ports:
      - "84:80"
```

### Start All Services

```bash
docker compose up -d
docker compose ps
```

Open the application using the public IP address of your server:

* `http://SERVER-IP:81` — Internet Banking
* `http://SERVER-IP:82` — Mobile Banking
* `http://SERVER-IP:83` — Insurance
* `http://SERVER-IP:84` — Loans

Ensure the relevant ports are allowed by the server firewall or cloud security group.

> Note: This example assumes the four images have already been built locally. Compose can also build images directly using a `build` configuration.

---

## 2. Docker Compose Commands

| Command                        | Purpose                                         |
| ------------------------------ | ----------------------------------------------- |
| `docker compose up -d`         | Create and start services in the background     |
| `docker compose up -d --build` | Build images and start services                 |
| `docker compose ps`            | Display services and container status           |
| `docker compose logs`          | View logs from services                         |
| `docker compose logs -f`       | Follow logs continuously                        |
| `docker compose stop`          | Stop services without removing containers       |
| `docker compose start`         | Start existing stopped containers               |
| `docker compose restart`       | Restart services                                |
| `docker compose pause`         | Pause running services                          |
| `docker compose unpause`       | Resume paused services                          |
| `docker compose top`           | Display processes running in services           |
| `docker compose images`        | List images associated with the Compose project |
| `docker compose down`          | Stop and remove project containers and networks |
| `docker compose config`        | Validate and display the resolved configuration |

### Manage One Service

```bash
docker compose stop insurance
docker compose start insurance
docker compose restart insurance
docker compose logs insurance
```

### Use a Custom Compose Filename

Compose recognizes conventional filenames such as `compose.yaml` and `docker-compose.yml`.

For a custom filename:

```bash
docker compose -f reyaz.yml up -d
docker compose -f reyaz.yml ps
docker compose -f reyaz.yml down
```

### Docker Compose Installation

Modern Docker installations generally support the Compose plugin:

```bash
docker compose version
```

On Amazon Linux, install the appropriate Docker Engine and Compose plugin packages according to the current distribution instructions. Older standalone Compose installations use the `docker-compose` command, which is different from the modern `docker compose` plugin.

---

## 3. Nginx Load Balancing

### What Is Load Balancing?

Load balancing distributes incoming application requests across multiple backend servers or application instances.

### Architecture

```text
                 Users
                   |
                   v
             Nginx Load Balancer
                   |
          +--------+--------+
          |        |        |
          v        v        v
       Backend1 Backend2 Backend3
```

### Benefits

* Distributes incoming requests across backend instances.
* Helps improve application availability.
* Allows multiple application instances to serve traffic.
* Can simplify the entry point for an application.
* Supports horizontal scaling when the application and infrastructure are designed for it.

### Nginx Configuration

Create `nginx.conf`:

```nginx
upstream backend {
    server backend1:5000;
    server backend2:5000;
    server backend3:5000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Nginx uses Docker Compose service names to locate the backend containers over the project's internal network.

### Nginx Dockerfile

```dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

The `daemon off;` argument keeps Nginx running in the foreground, allowing Docker to track it as the container's main process.

### Important Configuration Rule

All three backends in this example listen on port `5000` inside their respective containers. Therefore, Nginx should use:

```nginx
server backend1:5000;
server backend2:5000;
server backend3:5000;
```

Containers can use the same internal port because they have separate network namespaces.

---

## 4. Python Flask Load Balancer Project

### Project Structure

```text
docker-load-balancer/
├── app.py
├── requirements.txt
├── Dockerfile
├── nginx.conf
└── compose.yaml
```

### Step 1: Create `app.py`

```python
from flask import Flask
import socket

app = Flask(__name__)

@app.route("/")
def hello():
    return f"Hello from {socket.gethostname()}!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

The application returns the hostname of the container, making it easier to observe which backend handles a request.

### Step 2: Create `requirements.txt`

```text
flask
```

### Step 3: Create the Application Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

### Step 4: Create `nginx.conf`

```nginx
upstream backend {
    server backend1:5000;
    server backend2:5000;
    server backend3:5000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Step 5: Create `compose.yaml`

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - backend1
      - backend2
      - backend3

  backend1:
    build: .
    expose:
      - "5000"

  backend2:
    build: .
    expose:
      - "5000"

  backend3:
    build: .
    expose:
      - "5000"
```

### Step 6: Run the Project

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f
```

Open:

```text
http://SERVER-IP
```

Refresh the page multiple times to observe backend hostnames. Nginx uses round-robin distribution by default for this upstream configuration, but observed responses can vary because of connection reuse and client behavior.

### Testing Backend Failure

```bash
docker compose stop backend3
```

Nginx may continue serving requests through the remaining backends, although behavior depends on which backend receives a request and how failures are handled.

`depends_on` controls startup ordering; it does not by itself guarantee that an application is healthy or ready to accept traffic.

---

## 5. Docker Hub

### What Is Docker Hub?

Docker Hub is a container image registry where users can store, share, pull, and distribute Docker images.

Instead of keeping all images only on a local machine, we can push them to Docker Hub and retrieve them on another server.

### Basic Workflow

```text
Dockerfile
    |
    v
Build Image
    |
    v
Tag Image
    |
    v
Docker Hub
    |
    v
Pull Image on Another Server
    |
    v
Run Container
```

### Step 1: Build an Image

```bash
docker build -t internetbanking:v1 .
```

### Step 2: Log In

```bash
docker login
```

Use your Docker Hub credentials or a supported access token.

### Step 3: Tag the Image

Replace `YOUR_DOCKERHUB_USERNAME` with your actual Docker Hub username:

```bash
docker tag internetbanking:v1 YOUR_DOCKERHUB_USERNAME/ib-image:v1
```

### Step 4: Push the Image

```bash
docker push YOUR_DOCKERHUB_USERNAME/ib-image:v1
```

Repeat the process for other images.

### Step 5: Pull an Image

On another Docker host:

```bash
docker pull YOUR_DOCKERHUB_USERNAME/ib-image:v1
```

### Step 6: Run the Container

```bash
docker run -d \
  --name internetbanking \
  -p 80:80 \
  YOUR_DOCKERHUB_USERNAME/ib-image:v1
```

### Useful Commands

```bash
docker images
docker image ls
docker image inspect YOUR_DOCKERHUB_USERNAME/ib-image:v1
docker image rm YOUR_DOCKERHUB_USERNAME/ib-image:v1
```

> Important: Avoid deleting all local images on a working server. Other running or future services may depend on them. Use explicit image names when cleaning up.

---

## 6. Docker Restart Policies

### What Is a Restart Policy?

A Docker restart policy controls whether Docker automatically restarts a container when its main process exits.

Restart policies are useful for long-running services such as Nginx, Apache, monitoring agents, and application containers.

### Types of Restart Policies

| Policy           | Description                                                                 |
| ---------------- | --------------------------------------------------------------------------- |
| `no`             | Do not automatically restart the container                                  |
| `on-failure`     | Restart when the container exits with a non-zero status                     |
| `always`         | Restart automatically after exits, subject to Docker's manual-stop behavior |
| `unless-stopped` | Restart automatically unless the container has been explicitly stopped      |

### 1. `no`

This is the default policy.

```bash
docker run -d --name web --restart=no nginx
```

Docker does not automatically restart the container after it exits.

### 2. `on-failure`

```bash
docker run -d --name failed-job \
  --restart=on-failure:5 \
  ubuntu sh -c "exit 1"
```

Docker retries a container that exits unsuccessfully, subject to the configured retry limit.

Useful for certain retryable batch tasks and scripts.

### 3. `always`

```bash
docker run -d --name nginx-web \
  --restart=always \
  nginx
```

Docker restarts the container after an exit and restarts it when the Docker daemon restarts, subject to manual-stop behavior.

### 4. `unless-stopped`

```bash
docker run -d --name nginx-web \
  --restart=unless-stopped \
  nginx
```

Docker automatically restarts the container unless it has been explicitly stopped. An intentional stop is preserved across daemon restarts.

### Compare Restart Policies

| Behavior                                           | `no` | `on-failure`               | `always`      | `unless-stopped`                  |
| -------------------------------------------------- | ---- | -------------------------- | ------------- | --------------------------------- |
| Retry after a non-zero exit                        | No   | Yes                        | Yes           | Yes                               |
| Restart after daemon restart                       | No   | Not solely for this reason | Yes           | Yes, unless intentionally stopped |
| Preserve an intentional stop across daemon restart | Yes  | Yes                        | Not generally | Yes                               |

### Inspect a Restart Policy

```bash
docker inspect -f '{{ .HostConfig.RestartPolicy.Name }}' nginx-web
```

### Update an Existing Container

```bash
docker update --restart=unless-stopped nginx-web
```

### Best Practices

* Choose a policy based on application behavior and operational requirements.
* Use `unless-stopped` when administrators may intentionally stop a service for maintenance.
* Use `on-failure` for workloads that should retry after unsuccessful exits.
* Combine restart policies with health checks, monitoring, logs, and alerting.
* Remember that restarting a container does not fix an underlying application error.

---

## 7. Best Practices

1. Use Docker Compose to manage related application services.
2. Use clear service names instead of generic names such as `cont1`.
3. Keep application configuration separate from application code where appropriate.
4. Store persistent data in volumes instead of relying on a container's writable layer.
5. Publish only the ports that need to be accessible from outside the Docker network.
6. Use specific image tags instead of relying on `latest` for reproducible deployments.
7. Avoid hardcoding passwords, access tokens, or other secrets in Dockerfiles and Compose files.
8. Use health checks when service readiness matters.
9. Test load balancing by checking backend responses and failure behavior.
10. Use Docker Hub or another container registry to share versioned images.
11. Prefer supported Docker Engine and Compose plugin installation methods.
12. Use Docker Swarm or Kubernetes when orchestration requirements go beyond a single host.

---

## 8. Interview Questions

### Q1. What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications using a YAML configuration file.

### Q2. Why do we use Docker Compose?

It simplifies the configuration and lifecycle management of multiple related containers.

### Q3. What is a Compose service?

A service defines how an application component should run, including its image or build context, ports, environment, volumes, and other settings.

### Q4. What does `docker compose up -d` do?

It creates and starts the services in detached mode.

### Q5. What is the difference between `stop` and `down`?

`stop` stops the containers but retains them. `down` stops and removes the project's containers and networks.

### Q6. What is Nginx load balancing?

It distributes incoming requests across configured backend servers.

### Q7. What is an upstream block in Nginx?

It defines a group of backend servers that Nginx can send requests to.

### Q8. Why use Docker Hub?

It provides a registry for storing, sharing, and distributing container images.

### Q9. What is a Docker restart policy?

It defines how Docker handles automatic container restarts after process exits.

### Q10. What is the difference between `always` and `unless-stopped`?

Both restart containers automatically, but `unless-stopped` preserves an intentional manual stop across daemon restarts.

### Q11. How do you check Compose logs?

```bash
docker compose logs
```

### Q12. How do you stop a single service?

```bash
docker compose stop backend3
```

### Q13. Does `depends_on` guarantee application readiness?

No. It can control startup ordering, but readiness requires appropriate health checks or application-level handling.

### Q14. Can multiple containers listen on the same internal port?

Yes. Each container has its own network namespace. Containers can use the same internal port while exposing different host ports if needed.

### Q15. Does Docker Compose provide multi-host orchestration by itself?

Traditional Docker Compose is primarily used to manage applications on a single Docker host. Multi-host orchestration requires a suitable orchestration platform.

---
## Summary

In this learning session, I covered:

* Docker Compose and multi-container application management
* YAML-based service configuration
* Container lifecycle commands
* Nginx reverse proxy and load balancing
* Python Flask application containerization
* Docker Hub image tagging, pushing, and pulling
* Docker restart policies
* Docker troubleshooting and operational best practices

**Next learning goal:** Continue with Docker networking, health checks, environment variables, and container orchestration.
