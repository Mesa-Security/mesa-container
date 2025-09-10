# Mesa Security

A comprehensive security platform with frontend and backend services for threat analysis and management.

## Prerequisites

- Docker and Docker Compose
- Node.js (for local development)
- PostgreSQL (running on host or remote server)
- Redis (included in docker-compose)

## Project Structure

- `mesa-frontend/`: React frontend application
- `mesa-backend/`: NestJS backend API
- `docker-compose.yml`: Multi-service deployment configuration

## Quick Start with Docker Compose

1. **Clone the repository** (if not already done)

2. **Configure environment variables**:
   - Update `mesa-backend/.env` with your database and service configurations
   - For local Postgres, set `DB_HOST=host.docker.internal`

3. **Deploy all services**:
   ```bash
   docker-compose up -d
   ```

4. **Access the application**:
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:3001
   - Redis: localhost:6379

## Services

### Frontend
- **Port**: 3000
- **Technology**: React + Nginx
- **Description**: User interface for threat analysis and management

### Backend
- **Port**: 3001
- **Technology**: NestJS + Node.js
- **Description**: REST API for business logic, authentication, and data processing

### PostgreSQL (Host)
- **Port**: 5432
- **Technology**: PostgreSQL (running on host)
- **Description**: Database for data persistence
- **Host**: host.docker.internal

### Redis
- **Port**: 6379
- **Technology**: Redis (Alpine)
- **Description**: Caching and session storage
- **Container Name**: mesa-redis

## Database Configuration

The backend connects to PostgreSQL. Update the following in `mesa-backend/.env`:

```env
DB_HOST=host.docker.internal  # For local Postgres
DB_PORT=5432
DB_NAME=your_database
DB_USER=your_username
DB_PASSWORD=your_password
DB_SSL_ENABLED=false
```

## Development

For local development without Docker:

1. **Backend**:
   ```bash
   cd mesa-backend
   npm install
   npm run start:dev
   ```

2. **Frontend**:
   ```bash
   cd mesa-frontend
   npm install
   npm run dev
   ```

## Environment Variables

Key environment variables in `mesa-backend/.env`:
- Database connection settings
- JWT secret
- AWS credentials
- Redis configuration
- External service URLs

## Docker Commands

### Basic Operations
====================        Start / Stop       ==========================================================
# Start everything (detached)
docker compose up -d

# Start and rebuild changed images
docker compose up -d --build

# Start only a service
docker compose up -d mesa-backend

# Stop containers (keep images/volumes)
docker compose stop

# Stop & remove containers, networks (keep volumes/images)
docker compose down

# Down and remove named volumes too (DB reset!)
docker compose down -v


 ==============   Build / Rebuild. ================
# Build all services
docker compose build

# Build a single service
docker compose build mesa-frontend

# Rebuild without cache
docker compose build --no-cache mesa-backend

# Pull base images (from registry)
docker compose pull



 ==============  Status & Logs   ================

# List running services/containers
docker compose ps

# Follow all logs
docker compose logs -f

# Logs for one service (last 200 lines)
docker compose logs -f --tail=200 mesa-backend

# Show resource usage
docker stats

## Networking

All services run on the `mesa-network` Docker network for secure inter-service communication.

## Configuration Files

### docker-compose.yml
```yaml
# Docker Compose configuration for Mesa Security application
# This file defines the multi-service setup including frontend, backend, and Redis
version: '3.8'

services:
  # Frontend service - React application served by Nginx
  frontend:
    build:
      context: ./mesa-frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"  # Host port 3000 maps to container port 80
    networks:
      - mesa-network
    depends_on:
      - backend  # Ensure backend starts before frontend

  # Backend service - NestJS API server
  backend:
    build:
      context: ./mesa-backend
      dockerfile: Dockerfile
    ports:
      - "3001:3000"  # Host port 3001 maps to container port 3000
    env_file:
      - ./mesa-backend/.env  # Load environment variables from .env file
    networks:
      - mesa-network

  # Redis service - Caching and session storage
  redis:
    image: redis:alpine  # Lightweight Redis image
    container_name: mesa-redis
    ports:
      - "6379:6379"  # Standard Redis port
    networks:
      - mesa-network

# Custom network for inter-service communication
networks:
  mesa-network:
    driver: bridge
```

### mesa-frontend/Dockerfile
```dockerfile
# Multi-stage Dockerfile for Mesa Frontend
# Stage 1: Build stage
# Use Node.js version 22 as the base image for the build stage
FROM --platform=linux/amd64 node:22-alpine as build-stage

# Set the working directory in the container
WORKDIR /app

# Copy package.json and package-lock.json (if available) to the container
COPY /package*.json ./

RUN npm config set strict-ssl false

# Install dependencies
RUN npm install --legacy-peer-deps --force --loglevel verbose

# Copy the rest of the application code to the container
COPY / .

# ENV NODE_OPTIONS="--max-old-space-size=8096"

# RUN apt-get update && apt-get install -y sed
RUN apk update && apk add --no-cache sed tzdata

# Build the project
RUN npm run build

# Stage 2: Production stage
# Use Nginx as the production server
FROM --platform=linux/amd64 nginx:latest

# Copy the built application from the build stage to the production stage
COPY --from=build-stage /app/dist /app
COPY default.conf /etc/nginx/conf.d

# Expose port 80 (for nginx)
EXPOSE 80

ENV NODE_OPTIONS="--max-old-space-size=12288"

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### mesa-backend/Dockerfile
```dockerfile

# syntax=docker/dockerfile:1.7

########################
# Base image
########################
FROM node:22-slim AS base
WORKDIR /app

RUN npm install -g @nestjs/cli

########################
# Install production deps only (for final image)
########################
FROM base AS deps
COPY package*.json ./
# If you use pnpm/yarn, swap this line accordingly
RUN npm ci --omit=dev

########################
# Install all deps (incl. dev) for building
########################
FROM base AS dev-deps
COPY package*.json ./
RUN npm ci

########################
# Build stage
########################
FROM base AS build
# Bring in dev deps to compile/bundle
COPY --from=dev-deps /app/node_modules ./node_modules
# Copy the rest of your source
COPY . .
# Adjust to your build script (e.g., "build" for Nest/TS -> dist)
RUN npm run build

########################
# Runtime (small, prod-only)
########################
FROM node:22-slim AS runner
ENV NODE_ENV=production
ENV PORT=3000
WORKDIR /app

# Only production node_modules
COPY --from=deps /app/node_modules ./node_modules
# App metadata (optional but nice)
COPY package*.json ./
# Built artifacts (change if your build outputs to ./build)
COPY --from=build /app/dist ./dist

# ✅ Copy the .env file (or .env.production if you prefer)
COPY .env .env

# Create logs directory and set permissions before switching to node user
RUN mkdir -p /app/logs && chown -R node:node /app/logs


EXPOSE 3000
USER node

# For Nest/TS builds:
CMD ["node", "dist/main.js"]
# If your app uses a start script, use:
# CMD ["npm", "run", "start"]
```

 

