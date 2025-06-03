---
title : "Docker Compose"
date : 2024-06-05
categories : ["Docker"]
tags : ["Docker","Docker_Compose"]
image:
        path: assets/lib/Docker/docker-compose.png
---

Docker Compose is a tool for defining and running multi-container Docker applications. It allows you to define a multi-container setup in a simple YAML file (`docker-compose.yml`) and manage the entire lifecycle of the application, including building, starting, stopping, and scaling the services.

This tutorial will walk you through the basics of Docker Compose, explaining the common commands and YAML structure in detail.

---

## **1. Introduction to Docker Compose**

Docker Compose is used to define and manage multi-container Docker applications. A **service** is typically a container that runs a specific application or part of the application stack. You define services, networks, and volumes in a `docker-compose.yml` file.

### **Key Concepts in Docker Compose:**

- **Service**: A containerized application or part of the application. For example, a web app or database.
- **App**: The application that can consist of multiple services (like a web server, database, cache, etc.).
- **Volume**: Persistent storage for your containers.
- **Network**: A way to enable communication between containers.
- **Docker Compose File**: A YAML file (`docker-compose.yml`) where you define all the services, networks, and volumes.

---

## **2. Installing Docker Compose**

Before you can use Docker Compose, you need to have Docker installed. Docker Compose is typically bundled with Docker Desktop (for Mac and Windows), but for Linux, you might need to install it manually.


## **2. Docker Compose File Structure**

The `docker-compose.yml` file defines the services, networks, and volumes for your application. Here’s a basic structure of a Docker Compose file:

```yaml
version: '3.8'  # Version of Docker Compose file format

services:
  app:  # Service name
    image: nginx:latest  # Image for the container
    ports:
      - "8080:80"  # Expose port 80 in container to port 8080 on the host
    volumes:
      - ./app:/usr/share/nginx/html  # Mount local app folder to container

  db:  # Another service (e.g., database)
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: example  # Set environment variable
    volumes:
      - db_data:/var/lib/mysql  # Named volume for persistent storage

volumes:
  db_data:  # Declare a named volume for MySQL
    driver: local

networks:
  default:
    driver: bridge  # Default network driver
```

### **Explanation of Components:**

- **version**: Specifies the version of the Docker Compose file format. Version 3 is commonly used.
- **services**: Defines the containers that make up your application. Each service runs in its own container.
- **image**: Specifies the Docker image to use for the container.
- **ports**: Maps ports between the container and the host.
- **volumes**: Mounts directories or named volumes to persist data across container restarts.
- **environment**: Sets environment variables inside the container.
- **volumes**: Defines named volumes to store data persistently.
- **networks**: Configures the networks to be used by the containers.

---

## **3. Common Docker Compose Commands**


### **1. `docker-compose up`**

Starts all the services defined in the `docker-compose.yml` file.

```bash
docker-compose up
```

- **-d**: Run in detached mode (in the background).

```bash
docker-compose up -d
```

This command will create and start the containers, networks, and volumes as specified in the YAML file.

### **2. `docker-compose down`**

Stops and removes all the services, networks, and volumes defined in the `docker-compose.yml` file.

```bash
docker-compose down
```

- **--volumes**: Remove named volumes declared in the `volumes` section of the Compose file.

```bash
docker-compose down --volumes
```

### **3. `docker-compose build`**

Builds the images for your services defined in the Compose file (if they need to be built from a Dockerfile).

```bash
docker-compose build
```

### **4. `docker-compose ps`**

Lists the containers currently running as part of your Compose project.

```bash
docker-compose ps
```

### **5. `docker-compose logs`**

Shows logs from the services defined in the `docker-compose.yml` file.

```bash
docker-compose logs
```

- **-f**: Follow logs in real-time.

```bash
docker-compose logs -f
```

### **6. `docker-compose exec`**

Executes a command inside a running container.

```bash
docker-compose exec <service-name> <command>
```

For example, to open a shell inside a running container:

```bash
docker-compose exec app bash
```

### **7. `docker-compose stop`**

Stops the running services without removing the containers.

```bash
docker-compose stop
```

### **8. `docker-compose start`**

Starts the stopped services without recreating the containers.

```bash
docker-compose start
```

### **9. `docker-compose restart`**

Restarts all services.

```bash
docker-compose restart
```

---

## **4. Advanced Docker Compose Features**

### **1. Multi-Stage Builds**

Docker Compose allows you to define multi-stage builds in your Dockerfile. This is useful for optimizing your image sizes.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
```

In your `Dockerfile`, you can define multiple stages:

```Dockerfile
# Stage 1: Build stage
FROM node:14 AS build
WORKDIR /app
COPY . .
RUN npm install

# Stage 2: Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

### **2. Environment Variables**

You can use environment variables in the Compose file, which can be set in an `.env` file or passed directly in the `docker-compose.yml`.

```yaml
services:
  app:
    image: nginx:latest
    environment:
      - APP_ENV=production
    env_file:
      - .env
```

In the `.env` file:

```bash
APP_ENV=production
```

### **3. Dependency Management (`depends_on`)**

You can specify the startup order of services using `depends_on`. For example, if your web service depends on a database:

```yaml
services:
  web:
    image: nginx
    depends_on:
      - db

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: example
```

However, `depends_on` only controls the order of starting services, not the readiness of services. You may need to use health checks to ensure the dependent services are ready.

---

## **5. Docker Compose Networks**

Docker Compose automatically creates a default network for your application. However, you can define custom networks for more complex setups.

```yaml
networks:
  app_network:
    driver: bridge

services:
  web:
    image: nginx
    networks:
      - app_network

  db:
    image: mysql
    networks:
      - app_network
```

### **Network Modes:**

- **bridge**: The default network mode.
- **host**: The container shares the host's network stack.
- **none**: The container has no network access.

---

## **6. Docker Compose Volumes**

Volumes are used to persist data outside the container. Docker Compose makes it easy to define and use volumes.

```yaml
services:
  db:
    image: mysql
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

You can also mount local directories:

```yaml
services:
  app:
    image: nginx
    volumes:
      - ./local-dir:/container-dir
```

---

## **7. Scaling Services**

Docker Compose allows you to scale services to multiple instances. For example, you can scale the web service to run 3 instances:

```bash
docker-compose up --scale web=3
```

---

