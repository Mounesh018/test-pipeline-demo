# React JS Application Dockerfile Documentation

## Overview
This Dockerfile is used to:
- Build a React JS application
- Create production build files
- Serve the React application using Nginx
- Create a lightweight production-ready Docker image

---

# Dockerfile for React JS Application

```dockerfile
# Build Stage
FROM node:20.18.1-alpine as build

# Set working directory
WORKDIR /app

# Add node_modules binaries to PATH
ENV PATH /app/node_modules/.bin:$PATH

# Copy application files
COPY . ./

# Install dependencies
RUN npm install --force

# Build React application
RUN npm run build


# Production Stage
FROM nginx:stable-alpine

# Copy build files to nginx html folder
COPY --from=build /app/build /usr/share/nginx/html

# Copy nginx configuration
COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf

# Expose application port
EXPOSE 80

# Start nginx server
CMD ["nginx", "-g", "daemon off;"]
```

---

# React JS Application Docker Build Flow

```text
React Source Code
        ↓
npm install
        ↓
npm run build
        ↓
Create Static Build Files
        ↓
Copy Files into Nginx
        ↓
Run Application on Port 80
```

---

# Step-by-Step Explanation

---

## 1. Node.js Base Image

```dockerfile
FROM node:20.18.1-alpine as build
```

### Purpose
- Uses Node.js image for building React application
- Alpine version is lightweight
- `as build` creates build stage

### Why Node.js?
React applications require:
- npm
- node
- react-scripts

To build frontend files.

---

## 2. Working Directory

```dockerfile
WORKDIR /app
```

### Purpose
Sets `/app` as working directory inside container.

All commands execute from:
```bash
/app
```

---

## 3. Environment PATH

```dockerfile
ENV PATH /app/node_modules/.bin:$PATH
```

### Purpose
Adds local npm binaries into PATH.

Allows commands like:
```bash
react-scripts
vite
webpack
```

Without global installation.

---

## 4. Copy React Application Files

```dockerfile
COPY . ./
```

### Purpose
Copies all React application files into container.

Files copied:
- package.json
- src/
- public/
- nginx/
- package-lock.json

---

## 5. Install React Dependencies

```dockerfile
RUN npm install --force
```

### Purpose
Installs all dependencies required for React app.

Example:
- react
- react-dom
- react-router-dom
- axios

### Note
`--force` bypasses dependency conflicts.

Normally preferred:
```bash
npm install
```

---

## 6. Build React Application

```dockerfile
RUN npm run build
```

### Purpose
Creates optimized production build.

Output generated in:
```bash
/app/build
```

### Build Includes
- Optimized JS
- Minified CSS
- Static assets
- index.html

---

# Production Stage

---

## 7. Nginx Base Image

```dockerfile
FROM nginx:stable-alpine
```

### Purpose
Uses Nginx server to host React static files.

### Benefits
- Fast performance
- Lightweight
- Production ready

---

## 8. Copy Build Files

```dockerfile
COPY --from=build /app/build /usr/share/nginx/html
```

### Purpose
Copies React build files from build container into Nginx web directory.

### Source
```bash
/app/build
```

### Destination
```bash
/usr/share/nginx/html
```

---

## 9. Copy Nginx Configuration

```dockerfile
COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf
```

### Purpose
Adds custom Nginx configuration.

Used for:
- React routing
- Reverse proxy
- API configuration
- Cache handling

---

## 10. Expose Port

```dockerfile
EXPOSE 80
```

### Purpose
Exposes port 80 from Docker container.

Application accessible using:
```bash
http://localhost:80
```

---

## 11. Start Nginx Server

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

### Purpose
Starts Nginx in foreground mode.

Required because:
- Docker container exits if main process stops

---

# Multi-Stage Build Architecture

```text
Stage 1:
Node.js Build Container
        ↓
Build React Files

Stage 2:
Nginx Production Container
        ↓
Serve Static Files
```

---

# Benefits of Multi-Stage Build

| Benefit | Description |
|---|---|
| Smaller Image | Removes unnecessary Node modules |
| Better Security | No Node runtime in production |
| Faster Deployment | Lightweight image |
| Clean Production Setup | Only required files included |

---

# Docker Commands

---

## Build Docker Image

```bash
docker build -t react-app .
```

---

## Run Docker Container

```bash
docker run -d -p 80:80 react-app
```

---

## Check Running Containers

```bash
docker ps
```

---

## Stop Container

```bash
docker stop <container-id>
```

---

# Example Project Structure

```text
react-project/
│
├── Dockerfile
├── package.json
├── package-lock.json
├── src/
├── public/
├── build/
└── nginx/
    └── nginx.conf
```

---

# Example nginx.conf

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri /index.html;
    }
}
```

---

# Why Nginx for React?

| Feature | Benefit |
|---|---|
| High Performance | Fast static file serving |
| Lightweight | Low memory usage |
| Reverse Proxy | API integration |
| Production Ready | Stable deployment |

---

# Important Notes

- Ensure React build succeeds before deployment
- Ensure nginx.conf exists
- Use `.dockerignore` to reduce image size
- Keep Node and Nginx versions updated

---

# Recommended .dockerignore

```text
node_modules
.git
build
coverage
```

---

# Summary

This Dockerfile:
- Builds React JS application
- Creates optimized production build
- Uses Nginx for hosting
- Uses multi-stage Docker build
- Creates lightweight production image
- Suitable for Kubernetes and Docker deployments