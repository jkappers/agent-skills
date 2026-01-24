# Multi-Stage Dockerfile Best Practices

A comprehensive guide for building optimized, secure, and maintainable Docker images using multi-stage builds.

---

## Overview

Multi-stage builds use multiple `FROM` statements in a single Dockerfile, where each `FROM` instruction begins a new build stage. You selectively copy artifacts from one stage to another, leaving behind everything unnecessary in the final image.

### Key Benefits

- **Smaller images**: Reduce image size dramatically (e.g., from 861 MB to 1.83 MB)
- **Enhanced security**: Minimal dependencies reduce attack surface
- **Cleaner separation**: Build tools stay out of production images
- **Better caching**: Efficient layer reuse across builds
- **Parallel execution**: BuildKit can run independent stages concurrently

---

## Core Principles

### 1. Separate Build and Runtime Environments

Use different base images for building and running:

```dockerfile
# Build stage - includes compilers, build tools
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage - minimal image
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

### 2. Name Your Stages

Always use `AS <name>` for readability and maintainability:

```dockerfile
FROM golang:1.21 AS builder
# ... build steps

FROM scratch AS production
COPY --from=builder /app/main /main
```

Named stages prevent breakage when reordering instructions and make `COPY --from=` commands self-documenting.

### 3. Copy Only Necessary Artifacts

Be explicit about what gets copied to the final image:

```dockerfile
# Good: Copy specific artifacts
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# Avoid: Copying entire directories blindly
# COPY --from=builder /app ./
```

### 4. Order Stages for Cache Efficiency

Place stages that change less frequently earlier:

```dockerfile
# Stage 1: Dependencies (changes infrequently)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2: Build (changes with code)
FROM deps AS builder
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:20-alpine AS production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
```

---

## Base Image Selection

### Minimal Base Image Options

| Image Type | Size | Use Case | Trade-offs |
|------------|------|----------|------------|
| `scratch` | 0 MB | Static binaries (Go, Rust) | No shell, no debugging tools, manual CA certs |
| `distroless` | ~2-20 MB | Most compiled languages | No shell, includes CA certs and timezone data |
| `alpine` | ~5-10 MB | When shell access needed | musl libc (compatibility issues possible) |
| `slim` variants | ~50-100 MB | Interpreted languages | Balance of size and compatibility |
| `chiseled` (.NET 10) | ~12-68 MB | .NET apps | Ubuntu-based, no shell, non-root by default |
| `azurelinux-distroless` (.NET) | ~15-70 MB | .NET apps | Microsoft supply chain, no shell, non-root |

### Choosing Between scratch and distroless

**Use `scratch` when:**
- Building fully static binaries with no external dependencies
- Maximum size reduction is critical
- You can handle CA certificates and timezone data manually

**Use `distroless` when:**
- You need CA certificates (HTTPS calls)
- You need timezone data
- You want a non-root user by default
- Enterprise environments with certificate management needs

```dockerfile
# Distroless example for Go
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o main .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/main /main
ENTRYPOINT ["/main"]
```

---

## Language-Specific Patterns

### Go

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Static binary for scratch/distroless
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o main .

FROM scratch
# Copy CA certificates for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/main /main
ENTRYPOINT ["/main"]
```

**Critical**: Set `CGO_ENABLED=0` for static binaries when using distroless or scratch.

### Node.js

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production
USER node
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

### React / Vite

React applications built with Vite have specific requirements that differ from server-side Node.js apps. Vite generates static assets with content hashing, uses ES modules by default, and requires proper web server configuration to serve the SPA correctly.

> **Key insight**: Unlike server-side Node.js apps, React/Vite apps only need Node.js for building—the runtime is just static files served by a web server like Nginx.

#### Standard Multi-Stage Pattern

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:stable-alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

This pattern reduces image size from ~1GB (Node.js with dependencies) to ~50MB (Nginx Alpine with static files).

#### Production Nginx Configuration

Create `nginx.conf` with SPA routing, security headers, caching, and compression:

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Hide Nginx version
    server_tokens off;

    # SPA routing - serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache hashed assets forever (Vite generates unique hashes)
    location ~* \.(?:css|js)$ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # Cache static assets
    location ~* \.(?:ico|gif|jpe?g|png|svg|woff2?|ttf|eot)$ {
        expires 6M;
        add_header Cache-Control "public, max-age=15552000";
    }

    # No cache for index.html (entry point must always be fresh)
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript
               application/xml+rss application/atom+xml image/svg+xml;

    # Deny hidden files
    location ~ /\. {
        deny all;
    }
}
```

#### Running as Non-Root User (Recommended)

For production security, run Nginx as a non-root user on a non-privileged port:

```dockerfile
# syntax=docker/dockerfile:1

FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine AS production

# Create nginx directories with correct permissions
RUN mkdir -p /var/run/nginx && \
    chown -R nginx:nginx /var/cache/nginx /var/run/nginx && \
    chmod -R g+w /var/cache/nginx

# Copy custom nginx config for non-root
COPY nginx-main.conf /etc/nginx/nginx.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy static files
COPY --from=builder --chown=nginx:nginx /app/dist /usr/share/nginx/html

USER nginx
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

Create `nginx-main.conf` for non-root operation:

```nginx
worker_processes auto;
pid /var/run/nginx/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

Update `nginx.conf` to listen on port 8080:

```nginx
server {
    listen 8080;
    # ... rest of config
}
```

#### Build-Time Environment Variables

Vite embeds environment variables at build time. Pass them via build arguments:

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app

# Accept build arguments
ARG VITE_API_URL
ARG VITE_APP_TITLE

# Make available to Vite build
ENV VITE_API_URL=$VITE_API_URL
ENV VITE_APP_TITLE=$VITE_APP_TITLE

COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build with:

```bash
docker build \
  --build-arg VITE_API_URL=https://api.example.com \
  --build-arg VITE_APP_TITLE="My App" \
  -t myapp:prod .
```

**Important**: All Vite environment variables must be prefixed with `VITE_`.

#### Runtime Environment Variables (Advanced)

For "build once, deploy anywhere" workflows, inject environment variables at container startup using placeholder substitution:

**Step 1**: Create a config template (`public/config.js`):

```javascript
window.__ENV__ = {
  VITE_API_URL: "__VITE_API_URL__",
  VITE_FEATURE_FLAG: "__VITE_FEATURE_FLAG__"
};
```

**Step 2**: Create an entrypoint script (`docker-entrypoint.sh`):

```bash
#!/bin/sh
set -e

# Replace placeholders with actual environment variables
envsubst < /usr/share/nginx/html/config.js.template > /usr/share/nginx/html/config.js

# Start nginx
exec nginx -g "daemon off;"
```

**Step 3**: Update Dockerfile:

```dockerfile
FROM nginx:stable-alpine AS production
RUN apk add --no-cache gettext

COPY --from=builder /app/dist /usr/share/nginx/html
COPY public/config.js /usr/share/nginx/html/config.js.template
COPY docker-entrypoint.sh /docker-entrypoint.sh
COPY nginx.conf /etc/nginx/conf.d/default.conf

RUN chmod +x /docker-entrypoint.sh

EXPOSE 80
ENTRYPOINT ["/docker-entrypoint.sh"]
```

**Step 4**: Access in your React app:

```javascript
const apiUrl = window.__ENV__?.VITE_API_URL || import.meta.env.VITE_API_URL;
```

#### Development vs Production Stages

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS development
RUN npm install
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host"]

FROM base AS builder
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```bash
# Development with hot reload
docker build --target development -t myapp:dev .
docker run -p 5173:5173 -v $(pwd)/src:/app/src myapp:dev

# Production
docker build --target production -t myapp:prod .
```

#### Vite Configuration for Docker HMR

When running Vite dev server inside Docker, configure `vite.config.ts` for proper hot module replacement:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    host: '0.0.0.0',  // Listen on all interfaces
    port: 5173,
    watch: {
      usePolling: true,  // Required for Docker file watching
    },
    hmr: {
      host: 'localhost',  // Or your Docker host
      port: 5173,
    },
  },
});
```

**Note**: `usePolling: true` increases CPU usage but is required for reliable file change detection in Docker.

#### Memory Optimization for Large Builds

Node.js defaults to 512MB memory, which may be insufficient for large Vite builds:

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

# Increase Node.js memory limit for large builds
ENV NODE_OPTIONS="--max-old-space-size=4096"
RUN npm run build
```

#### Pre-Compression with Brotli and Gzip

For optimal performance, pre-compress assets during build:

**Install Vite plugin** (`package.json`):

```json
{
  "devDependencies": {
    "vite-plugin-compression": "^0.5.1"
  }
}
```

**Configure Vite** (`vite.config.ts`):

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import viteCompression from 'vite-plugin-compression';

export default defineConfig({
  plugins: [
    react(),
    viteCompression({ algorithm: 'gzip' }),
    viteCompression({ algorithm: 'brotliCompress', ext: '.br' }),
  ],
});
```

**Update Nginx** to serve pre-compressed files:

```nginx
server {
    # ... other config

    # Serve pre-compressed files if available
    gzip_static on;

    # For Brotli (requires ngx_brotli module or fholzer/nginx-brotli image)
    # brotli_static on;
}
```

#### Complete Production Example

```dockerfile
# syntax=docker/dockerfile:1

ARG NODE_VERSION=22

# Stage 1: Dependencies
FROM node:${NODE_VERSION}-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Stage 2: Build
FROM node:${NODE_VERSION}-alpine AS builder
WORKDIR /app

# Build arguments for Vite
ARG VITE_API_URL
ARG VITE_APP_VERSION

ENV VITE_API_URL=$VITE_API_URL
ENV VITE_APP_VERSION=$VITE_APP_VERSION
ENV NODE_OPTIONS="--max-old-space-size=4096"

COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Production
FROM nginx:stable-alpine AS production
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.description="Production React application"

# Security: Create non-root setup
RUN mkdir -p /var/run/nginx && \
    chown -R nginx:nginx /var/cache/nginx /var/run/nginx /usr/share/nginx/html && \
    chmod -R g+w /var/cache/nginx

# Copy nginx configs
COPY nginx-main.conf /etc/nginx/nginx.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy static files
COPY --from=builder --chown=nginx:nginx /app/dist /usr/share/nginx/html

USER nginx
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

#### React/Vite-Specific .dockerignore

```dockerignore
node_modules
npm-debug.log*
dist
build
.git
.gitignore
*.md
.env
.env.*
.vscode
.idea
coverage
*.test.*
*.spec.*
__tests__
Dockerfile*
docker-compose*
.dockerignore
```

### Python

```dockerfile
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim AS production
WORKDIR /app
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . .
CMD ["python", "main.py"]
```

### Java

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS production
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### .NET / ASP.NET Core

.NET multi-stage builds separate the SDK (build tools) from the runtime, reducing image sizes from 2+ GB to under 100 MB.

> **Note**: This section covers .NET 10 (current LTS). The patterns also apply to .NET 8/9 with version adjustments.

#### Breaking Change in .NET 10: Ubuntu is the Default

Starting with .NET 10, **Ubuntu is the default Linux distribution** for all .NET container images. Debian images are no longer provided.

- `docker pull mcr.microsoft.com/dotnet/sdk:10.0` → Ubuntu 24.04 "Noble Numbat"
- Tags like `10.0`, `10.0-noble`, and `10.0-noble-chiseled` all use Ubuntu
- If you require Debian, you must create custom images

#### Standard Multi-Stage Pattern

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS production
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Important**: .NET 8+ uses port 8080 by default (not 80). Configure with `ASPNETCORE_HTTP_PORTS` if needed.

#### .NET 10 Image Variants and Sizes

| Image | Compressed Size | Use Case |
|-------|-----------------|----------|
| `aspnet:10.0` | ~92 MB | Standard ASP.NET Core apps |
| `aspnet:10.0-alpine` | ~52 MB | Smaller footprint, musl libc |
| `aspnet:10.0-noble-chiseled` | ~53 MB | Ubuntu distroless, no shell, non-root |
| `aspnet:10.0-noble-chiseled-extra` | ~68 MB | Chiseled + globalization (icu/tzdata) |
| `runtime-deps:10.0-noble-chiseled` | ~22 MB | Self-contained + trimming |
| `runtime-deps:10.0-noble-chiseled` (AOT) | ~12 MB | Native AOT apps |
| `runtime-deps:10.0-alpine` (AOT) | ~11 MB | Smallest: Native AOT on Alpine |

#### Available Platforms for .NET 10

| Platform | Tags |
|----------|------|
| Ubuntu 24.04 "Noble" | `10.0`, `10.0-noble`, `10.0-noble-chiseled` |
| Alpine Linux 3.22 | `10.0-alpine`, `10.0-alpine3.22` |
| Azure Linux 3.0 | `10.0-azurelinux3.0`, `10.0-azurelinux3.0-distroless` |

#### Chiseled Images (Recommended for Production)

Chiseled images are Ubuntu-based containers stripped to essentials—no shell, no package manager, non-root by default:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS production
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**New in .NET 10**: Chiseled images now include the **Chisel manifest**, which describes package contents and allows installing additional packages on top of existing chiseled base images.

**Limitations**: No shell, so you cannot chain `RUN` commands with `&&` or debug interactively. Use `-extra` variants if you need globalization support.

#### Azure Linux Distroless

Azure Linux provides an alternative distroless option with Microsoft's supply chain security:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0-azurelinux3.0-distroless AS production
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

#### Self-Contained Deployment

Bundles the .NET runtime with your app, allowing use of minimal `runtime-deps` images:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish \
    --self-contained true \
    -r linux-x64

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled AS production
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["./MyApp"]
```

#### Native AOT (Smallest Images)

Native AOT compiles to native code—fastest startup, smallest size (~12 MB compressed):

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-aot AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled AS production
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["./MyApp"]
```

**New Native AOT SDK images in .NET 10:**
- `mcr.microsoft.com/dotnet/sdk:10.0-aot` (Ubuntu, also tagged `10.0-noble-aot`)
- `mcr.microsoft.com/dotnet/sdk:10.0-alpine-aot`
- `mcr.microsoft.com/dotnet/sdk:10.0-azurelinux3.0-aot`

**Project file settings for AOT:**
```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

**AOT Limitations**: No runtime reflection, not all libraries compatible, larger SDK image.

#### Dockerfile-less Container Publishing

.NET 10 supports publishing containers directly from the SDK—no Dockerfile required:

```bash
# Publish to local container runtime
dotnet publish --os linux --arch x64 /t:PublishContainer

# Publish directly to a registry (no Docker needed)
dotnet publish --os linux --arch x64 /t:PublishContainer \
    -p ContainerRegistry=ghcr.io \
    -p ContainerRepository=myorg/myapp

# Multi-architecture publishing (SDK 8.0.405+, 9.0.102+, 10.0+)
dotnet publish /t:PublishContainer \
    -p ContainerRuntimeIdentifiers="linux-x64;linux-arm64"
```

**New in .NET 10**: Console apps now support `/t:PublishContainer` by default without setting `<EnableSdkContainerSupport>`.

**Project file configuration:**
```xml
<PropertyGroup>
  <!-- Control image format: Docker or OCI -->
  <ContainerImageFormat>OCI</ContainerImageFormat>
  <!-- Customize base image -->
  <ContainerBaseImage>mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled</ContainerBaseImage>
  <!-- Set image name and tag -->
  <ContainerRepository>myapp</ContainerRepository>
  <ContainerImageTag>1.0.0</ContainerImageTag>
</PropertyGroup>
```

#### Trimmed Self-Contained

Reduces binary size by removing unused code:

```xml
<PropertyGroup>
  <SelfContained>true</SelfContained>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>full</TrimMode>
  <PublishSingleFile>true</PublishSingleFile>
</PropertyGroup>
```

#### Cache Optimization for .NET

Restore NuGet packages before copying source code:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy only project files first (cache layer)
COPY *.sln ./
COPY src/MyApp/*.csproj ./src/MyApp/
COPY src/MyApp.Core/*.csproj ./src/MyApp.Core/
RUN dotnet restore

# Copy everything else and build
COPY . .
RUN dotnet publish src/MyApp -c Release -o /app/publish
```

#### NuGet Cache Mount

```dockerfile
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore
```

#### Development vs Production Stages

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS base
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore

FROM base AS development
COPY . .
ENTRYPOINT ["dotnet", "watch", "run"]

FROM base AS build
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS production
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

#### .NET-Specific .dockerignore

```dockerignore
**/bin/
**/obj/
**/.vs/
**/.vscode/
**/node_modules/
**/*.user
**/*.userosscache
**/*.suo
*.md
.git
.gitignore
Dockerfile*
docker-compose*
```

#### Including SHA for Security

Pin exact image digests for reproducibility:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0@sha256:abc123... AS build
```

---

## BuildKit Features

### Enable BuildKit

```bash
export DOCKER_BUILDKIT=1
# Or in Docker daemon config
```

BuildKit is the default builder since Docker Engine 23.0.

### Cache Mounts

Persist package manager caches across builds without including them in the image:

```dockerfile
# npm
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# pip
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# NuGet (.NET)
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore

# apt (requires sharing=locked for parallel builds)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y gcc
```

### Parallel Stage Execution

BuildKit automatically parallelizes independent stages:

```dockerfile
FROM node:20 AS frontend-builder
# ... build frontend

FROM golang:1.21 AS backend-builder
# ... build backend

FROM alpine AS production
COPY --from=frontend-builder /app/dist /static
COPY --from=backend-builder /app/main /main
```

Frontend and backend stages build concurrently.

---

## The `--target` Flag

Build specific stages for different environments:

```dockerfile
FROM node:20 AS base
WORKDIR /app
COPY package*.json ./

FROM base AS development
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS builder
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

```bash
# Development build
docker build --target development -t myapp:dev .

# Production build (default - last stage)
docker build -t myapp:prod .
```

### Docker Compose Integration

```yaml
services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/app
```

---

## ARG Variables Across Stages

### Scope Rules

ARG variables are scoped to the stage where they're defined. To share across stages:

```dockerfile
# Global scope - before first FROM
ARG NODE_VERSION=20
ARG APP_VERSION=1.0.0

FROM node:${NODE_VERSION} AS builder
# Must redeclare to use
ARG APP_VERSION
RUN echo "Building version ${APP_VERSION}"

FROM node:${NODE_VERSION}-alpine AS production
ARG APP_VERSION
LABEL version="${APP_VERSION}"
```

### Parent-Child Inheritance

Child stages inherit ARGs from parent stages:

```dockerfile
FROM alpine AS base
ARG ENVIRONMENT=production

FROM base AS child
# ENVIRONMENT is available here
RUN echo "Environment: ${ENVIRONMENT}"
```

---

## Copying from External Images

Copy artifacts from any image, not just previous stages:

```dockerfile
# Copy from official images
COPY --from=nginx:alpine /etc/nginx/nginx.conf /nginx.conf

# Copy binaries
COPY --from=busybox:uclibc /bin/sh /bin/sh
```

Useful for adding specific tools without installing them via package manager.

---

## Security Best Practices

### 1. Run as Non-Root User

```dockerfile
FROM node:20-alpine AS production
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser
USER appuser
WORKDIR /home/appuser/app
```

Or use distroless with `:nonroot` tag.

### 2. Use COPY Instead of ADD

```dockerfile
# Good
COPY requirements.txt .

# Avoid (can fetch remote URLs, auto-extracts archives)
ADD requirements.txt .
```

### 3. Pin Image Versions

```dockerfile
# Good - specific version
FROM node:20.10.0-alpine3.18

# Avoid - unpredictable
FROM node:latest
```

### 4. Scan Images Regularly

Integrate vulnerability scanning into CI/CD:

```bash
docker scout cves myimage:latest
# or
trivy image myimage:latest
```

### 5. Minimize Installed Packages

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    package-name && \
    rm -rf /var/lib/apt/lists/*
```

### 6. Use .dockerignore

```dockerignore
.git
.gitignore
node_modules
*.md
.env
.env.*
Dockerfile*
docker-compose*
.vscode
.idea
coverage
tests
__pycache__
*.pyc
```

**Important**: `.dockerignore` only affects `COPY` from build context, not `COPY --from=` between stages.

---

## Health Checks

Add health checks for production readiness:

```dockerfile
FROM node:20-alpine AS production
# ... app setup

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node healthcheck.js || exit 1
```

### For Minimal Images (scratch/distroless)

Compile a healthcheck binary:

```dockerfile
FROM golang:1.21 AS healthcheck-builder
COPY healthcheck.go .
RUN CGO_ENABLED=0 go build -o healthcheck healthcheck.go

FROM scratch
COPY --from=builder /app/main /main
COPY --from=healthcheck-builder /healthcheck /healthcheck
HEALTHCHECK --interval=30s CMD ["/healthcheck"]
```

---

## Caching Strategies

### Layer Cache Optimization

1. Put rarely-changing instructions first
2. Copy dependency files before source code
3. Use cache mounts for package managers

```dockerfile
# Dependencies change less often than source code
COPY package*.json ./
RUN npm ci

# Source code changes frequently - put last
COPY . .
RUN npm run build
```

### CI/CD Cache Export

```bash
# Export cache to registry
docker build \
  --cache-to type=registry,ref=myregistry/myimage:cache \
  --cache-from type=registry,ref=myregistry/myimage:cache \
  -t myimage:latest .
```

### Cache Types

| Type | Description | Multi-stage Support |
|------|-------------|---------------------|
| `inline` | Embedded in image | Final stage only |
| `registry` | Separate cache image | All stages |
| `local` | Local directory | All stages |

---

## Anti-Patterns to Avoid

### 1. Not Using Multi-Stage Builds

```dockerfile
# Bad: Build tools in production image
FROM node:20
RUN npm install
RUN npm run build
CMD ["node", "dist/index.js"]
```

### 2. Leaving Build Tools in Final Image

```dockerfile
# Bad: gcc remains in production
FROM python:3.11
RUN apt-get install -y gcc
RUN pip install -r requirements.txt
```

### 3. Poor Cache Utilization

```dockerfile
# Bad: Copying everything invalidates cache
COPY . .
RUN npm install
```

### 4. Using git clone in Dockerfile

```dockerfile
# Bad: Non-reproducible builds
RUN git clone https://github.com/user/repo.git
```

Clone externally and copy into build context instead.

### 5. Single Role Violation

Each image should serve one purpose. Don't ship test frameworks to production:

```dockerfile
# Bad: Test dependencies in production
FROM python:3.11
RUN pip install pytest coverage myapp
```

### 6. Hardcoding Secrets

```dockerfile
# Never do this
ENV API_KEY=secret123
```

Use build secrets or runtime environment variables instead.

---

## Complete Example: Production-Ready Node.js

```dockerfile
# syntax=docker/dockerfile:1

ARG NODE_VERSION=20

# Stage 1: Dependencies
FROM node:${NODE_VERSION}-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# Stage 2: Build
FROM node:${NODE_VERSION}-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build && npm run test

# Stage 3: Production
FROM node:${NODE_VERSION}-alpine AS production
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.description="Production Node.js application"

ENV NODE_ENV=production
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 nodejs && \
    adduser -u 1001 -G nodejs -s /bin/sh -D nodejs

# Copy artifacts
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## Quick Reference

### Commands

```bash
# Build with target
docker build --target <stage> -t image:tag .

# Build with cache export
docker build --cache-to type=local,dest=./cache .

# Build with cache import
docker build --cache-from type=local,src=./cache .

# Build with build args
docker build --build-arg NODE_VERSION=20 .
```

### Dockerfile Directives

| Directive | Purpose |
|-----------|---------|
| `FROM ... AS name` | Start named stage |
| `COPY --from=name` | Copy from previous stage |
| `COPY --from=image:tag` | Copy from external image |
| `RUN --mount=type=cache` | Use persistent cache |
| `ARG` | Build-time variable |
| `HEALTHCHECK` | Container health monitoring |

---

## Sources

### Docker Best Practices
- [Docker Multi-stage Builds Documentation](https://docs.docker.com/build/building/multi-stage/)
- [Docker Build Best Practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Build Variables](https://docs.docker.com/build/building/variables/)
- [Docker Cache Optimization](https://docs.docker.com/build/cache/optimize/)
- [Sysdig Dockerfile Best Practices](https://www.sysdig.com/learn-cloud-native/dockerfile-best-practices)
- [Snyk Docker Security Best Practices](https://snyk.io/blog/10-docker-image-security-best-practices/)
- [iximiuz Labs Multi-Stage Builds Tutorial](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds)
- [Advanced BuildKit Patterns](https://medium.com/@tonistiigi/advanced-multi-stage-build-patterns-6f741b852fae)
- [Docker Anti-Patterns](https://codefresh.io/blog/docker-anti-patterns/)
- [Distroless vs Scratch for Go](https://baeke.info/2021/03/28/distroless-or-scratch-for-go-apps/)

### .NET Container Documentation
- [Microsoft: Containerize a .NET App](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container)
- [Microsoft: .NET Container Images](https://learn.microsoft.com/en-us/dotnet/core/docker/container-images)
- [Microsoft: SDK Container Publishing](https://learn.microsoft.com/en-us/dotnet/core/containers/sdk-publish)
- [Microsoft: Native AOT Deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)
- [Announcing .NET Chiseled Containers](https://devblogs.microsoft.com/dotnet/announcing-dotnet-chiseled-containers/)
- [Secure Your .NET Cloud Apps with Rootless Containers](https://devblogs.microsoft.com/dotnet/securing-containers-with-rootless/)
- [.NET Docker Image Variants](https://github.com/dotnet/dotnet-docker/blob/main/documentation/image-variants.md)
- [.NET Docker Sample Image Size Report](https://github.com/dotnet/dotnet-docker/blob/main/documentation/sample-image-size-report.md)
- [9 Tips for Containerizing .NET Applications](https://www.docker.com/blog/9-tips-for-containerizing-your-net-application/)

### .NET 10 Specific
- [.NET 10 Container Images Now Available](https://github.com/dotnet/dotnet-docker/discussions/6801)
- [.NET 10 Breaking Change: Default Images Use Ubuntu](https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/10.0/default-images-use-ubuntu)
- [.NET 10 SDK AOT Container Images](https://github.com/dotnet/dotnet-docker/discussions/6312)
- [.NET 10 Container Updates Preview](https://github.com/dotnet/dotnet-docker/discussions/6324)
- [Ubuntu Chiseled Images Documentation](https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md)
- [Azure Linux Distroless Images](https://github.com/dotnet/dotnet-docker/blob/main/documentation/azurelinux.md)

### React / Vite Container Documentation
- [Docker: How to Dockerize a React App](https://www.docker.com/blog/how-to-dockerize-react-app/)
- [Docker: Containerize React Apps Guide](https://docs.docker.com/guides/reactjs/containerize/)
- [Docker: Develop React Apps Guide](https://docs.docker.com/guides/reactjs/develop/)
- [React Vite + Docker + Nginx Production Guide](https://www.buildwithmatija.com/blog/production-react-vite-docker-deployment)
- [Building a Production Docker Container for Vite + React Apps](https://alvincrespo.hashnode.dev/react-vite-production-ready-docker)
- [Guide to Containerizing a Modern JavaScript SPA with Multi-Stage Nginx Build](https://dev.to/it-wibrc/guide-to-containerizing-a-modern-javascript-spa-vuevitereact-with-a-multi-stage-nginx-build-1lma)
- [Production Multistage Dockerfile For React Application](https://dev.to/sre_panchanan/production-multistage-dockerfile-for-react-application-2p6i)
- [Optimizing Docker Builds: Multi-Stage Builds with React and Vite](https://medium.com/@ryanmambou/optimizing-docker-builds-a-practical-guide-to-multi-stage-builds-with-react-and-vite-c9692414961c)
- [Dockerizing the Frontend with React.js + Vite](https://www.innokrea.com/dockerizing-the-frontend-do-it-right-with-react-js-vite/)

### Vite Environment Variables
- [Vite: Env Variables and Modes](https://vite.dev/guide/env-and-mode)
- [Managing Runtime Environment Variables in Vite, Docker, and React](https://www.mykolaaleksandrov.dev/posts/2025/10/vite-docker-react-environment-variables/)
- [Runtime ENV Config for Vite: Build Once, Deploy Anywhere](https://engineering.simpl.de/post/runtime-env-config-part1/)
- [Setting Up Dynamic Environment Variables with Vite and Docker](https://dev.to/dutchskull/setting-up-dynamic-environment-variables-with-vite-and-docker-5cmj)
- [vite-envs: Env Variables at Container Startup](https://github.com/garronej/vite-envs)

### Vite Docker Development (HMR)
- [ViteJS Hot Reload in Docker Container](https://patrickdesjardins.com/blog/docker-vitejs-hot-reload)
- [Vite Docker HMR Guide](https://www.restack.io/p/vite-answer-docker-hmr-guide)
- [Enabling HMR in Vite with NGINX Reverse Proxy](https://aronschueler.de/blog/2024/07/29/enabling-hot-module-replacement-hmr-in-vite-with-nginx-reverse-proxy/)
- [Vite Performance Guide](https://vite.dev/guide/performance)

### Nginx Caching and Compression
- [NGINX Browser Caching Guide](https://www.getpagespeed.com/server-setup/nginx/nginx-browser-caching)
- [Vite PWA NGINX Deployment](https://vite-pwa-org.netlify.app/deployment/nginx)
- [Pre-compress Static Assets with Brotli and Gzip](https://erikpoehler.com/2021/05/24/pre-compress-static-assets-with-brotli-or-gzip/)
- [Enable GZIP and Brotli Compression for Nginx](https://computingforgeeks.com/how-to-enable-gzip-brotli-compression-for-nginx-on-linux/)
- [Brotli Compression in React Apps](https://engineering.brevo.com/brotli-compression-in-react-apps/)

### Security Headers
- [Content-Security-Policy Headers on Nginx](https://content-security-policy.com/examples/nginx/)
- [React Content Security Policy Guide](https://www.stackhawk.com/blog/react-content-security-policy-guide-what-it-is-and-how-to-enable-it/)
- [MDN: Content-Security-Policy Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy)
- [React CSP with Nonce](https://www.hectane.com/blog/react-content-security-policy-webpack-node-nginx)

### Build Optimization
- [Avoiding Memory Issues During Docker-Vite-React Builds](https://joetatusko.com/2024/10/18/avoiding-memory-issues-during-docker-vite-react-builds/)
- [Optimized Docker Setup for Vite-Powered React Apps](https://medium.com/@pierre.fourny/optimized-docker-setup-for-vite-powered-react-apps-e7b7f5a82bb4)
- [Optimize Vite Build Time Guide](https://dev.to/perisicnikola37/optimize-vite-build-time-a-comprehensive-guide-4c99)
