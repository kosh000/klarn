---
inclusion: manual
---

# Stage 1: Docker & Containers

## Goal
Understand what containers are, how to build them, run them, and push them to a registry. This is the unit of deployment in Kubernetes.

## Concepts

### What is a Container?
- A container is a process (or group of processes) running in isolation
- It shares the host kernel but has its own filesystem, network, and process namespace
- NOT a VM — no separate OS kernel, much lighter
- Uses Linux kernel features: namespaces (isolation) + cgroups (resource limits)

### Container vs Image
- **Image**: a read-only template (like a class in programming)
- **Container**: a running instance of an image (like an object)
- Images are built in layers — each Dockerfile instruction creates a layer
- Layers are cached and shared between images

### Docker Architecture
- **Docker daemon** (dockerd): manages containers, images, networks, volumes
- **Docker CLI** (docker): client that talks to the daemon
- **containerd**: the actual container runtime (Docker uses this under the hood)
- **Container registry**: stores images (Docker Hub, Amazon ECR)

## Key Commands

```bash
# Images
docker build -t myapp:v1 .          # Build image from Dockerfile
docker images                        # List local images
docker pull nginx:latest             # Pull from registry
docker push myrepo/myapp:v1          # Push to registry
docker tag myapp:v1 myrepo/myapp:v1  # Tag an image

# Containers
docker run -d -p 8080:80 nginx       # Run detached, map port
docker run -it ubuntu bash            # Run interactive
docker ps                             # List running containers
docker ps -a                          # List all (including stopped)
docker logs <container>               # View logs
docker exec -it <container> bash      # Shell into running container
docker stop <container>               # Graceful stop (SIGTERM)
docker kill <container>               # Force stop (SIGKILL)
docker rm <container>                 # Remove stopped container

# Cleanup
docker system prune -a                # Remove all unused data
```

## Dockerfile Best Practices

```dockerfile
# Use specific version tags, never :latest in production
FROM node:20-alpine AS builder

# Set working directory
WORKDIR /app

# Copy dependency files first (layer caching)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Copy source code (changes more often, so this layer rebuilds more)
COPY src/ ./src/

# Multi-stage build: final image is smaller
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app ./

# Run as non-root user (security)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Expose port (documentation, doesn't actually publish)
EXPOSE 3000

# Use exec form (proper signal handling)
CMD ["node", "src/index.js"]
```

### Why Multi-Stage Builds?
- Build tools (compilers, dev dependencies) stay in builder stage
- Final image only has runtime dependencies
- Smaller image = faster pulls, less attack surface

## Docker Compose

For running multiple containers locally (simulates microservices):

```yaml
# docker-compose.yml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://postgres:secret@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```

```bash
docker compose up -d      # Start all services
docker compose logs -f    # Follow logs
docker compose down       # Stop and remove
docker compose down -v    # Also remove volumes
```

## Amazon ECR (Elastic Container Registry)

```bash
# Create repository
aws ecr create-repository --repository-name myapp

# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag myapp:v1 <account-id>.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

## Labs

### Lab 1.1: Build and Run a Simple App
1. Create a simple Node.js/Python/Go app that responds to HTTP requests
2. Write a Dockerfile for it
3. Build the image
4. Run it and verify with curl
5. Check logs, exec into the container, explore the filesystem

### Lab 1.2: Multi-Stage Build
1. Take your app from Lab 1.1
2. Create a multi-stage Dockerfile
3. Compare image sizes between single-stage and multi-stage
4. Verify the app still works

### Lab 1.3: Docker Compose
1. Add a database (PostgreSQL) to your app using Docker Compose
2. Make the app connect to the database
3. Verify data persists across container restarts (volumes)
4. Verify data is lost when you `docker compose down -v`

### Lab 1.4: Push to ECR
1. Create an ECR repository
2. Build, tag, and push your image
3. Pull it on a different machine (or delete local and re-pull)
4. Verify it runs from the registry image

### Lab 1.5: Break Things
1. Run a container with `--memory=50m` and watch it get OOM killed
2. Run without port mapping and try to access it from host
3. Build with a typo in CMD and see what happens
4. Run as root vs non-root and check file permissions

## Under the Hood (Understand It)

### Namespaces (Isolation)
- **PID namespace**: container sees only its own processes (PID 1 is your app)
- **Network namespace**: container has its own network stack, IP, ports
- **Mount namespace**: container has its own filesystem view
- **UTS namespace**: container has its own hostname
- **User namespace**: container can map root to non-root on host (K8s 1.36 GA)

### Cgroups (Resource Limits)
- Control how much CPU, memory, I/O a container can use
- When a container exceeds memory limit → OOM killed
- When a container exceeds CPU limit → throttled (not killed)

### Union Filesystem (Layers)
- Each Dockerfile instruction creates a read-only layer
- Running container adds a writable layer on top
- Layers are shared between images (saves disk space)
- This is why order in Dockerfile matters for caching

## Self-Test Questions

1. What's the difference between a container and a VM? Name 3 specific differences.
2. If you run `docker run nginx` without `-d`, what happens? How do you fix it?
3. Why does the order of instructions in a Dockerfile matter for build speed?
4. You have a Dockerfile with `COPY . .` at the top. Why is this bad for caching?
5. What's the difference between `CMD` and `ENTRYPOINT`? What happens if you specify both?
6. Your image is 1.2GB. Name 3 ways to make it smaller.
7. What happens to data written inside a container when the container is deleted? How do you persist it?
8. You run `docker run -p 8080:80 nginx`. What does `8080:80` mean? Which is the host port?
9. Why should you run containers as non-root? What could go wrong if you don't?
10. What's the difference between `docker stop` and `docker kill`? Which is safer for your app?
11. You push an image tagged `:latest` to production. Why is this dangerous?
12. In a multi-stage build, why doesn't the final image contain the build tools from the first stage?
13. Two containers in a Docker Compose file need to talk to each other. How do they find each other? (Hint: DNS)
14. What's a container layer? If 10 containers run from the same image, how many copies of the image exist on disk?
15. You `exec` into a container and install a package with `apt-get`. Is that package there after the container restarts? Why or why not?

## Checklist Before Moving On

- [ ] Can write a Dockerfile from scratch
- [ ] Understand multi-stage builds and why they matter
- [ ] Can run multi-container apps with Docker Compose
- [ ] Can push/pull images to/from ECR
- [ ] Understand namespaces and cgroups conceptually
- [ ] Know the difference between ENTRYPOINT and CMD
- [ ] Know why you run as non-root
- [ ] Understand layer caching and how to optimize build times
