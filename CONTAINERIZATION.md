# CipherHorizon Containerization Guide

## Overview

This guide covers the containerization process for the CipherHorizon platform, detailing Dockerfile creation and Docker Compose configuration for multi-container deployment.

## Containerization Strategy

### Design Principles

- Minimal image size
- Multi-stage builds
- Language-specific optimizations
- Secure container configurations
- Consistent development environments

### Technology Stack Containerization

![Technology Stack Containerization](assets/technology-stack-containerization.png)

## Dockerfile Templates

### 1. Base Golang Microservice Dockerfile

```Dockerfile
# Stage 1: Build Stage
FROM golang:1.19-alpine AS builder

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apk add --no-cache \
    git \
    make \
    gcc \
    musl-dev

# Copy go module files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -o service-binary ./cmd/service

# Stage 2: Lightweight Runtime Image
FROM alpine:latest

# Install root certificates
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy compiled binary from builder
COPY --from=builder /app/service-binary .

# Copy configuration files
COPY ./configs ./configs

# Expose service port
EXPOSE 8080

# Set environment to production
ENV GIN_MODE=release

# Run the binary
ENTRYPOINT ["./service-binary"]
```

### 2. Python Machine Learning Service Dockerfile

```Dockerfile
# Use official Python runtime as base image
FROM python:3.9-slim-buster

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    software-properties-common \
    git \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements file
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Install additional ML dependencies
RUN pip install \
    torch \
    tensorflow \
    scikit-learn

# Expose service port
EXPOSE 5000

# Set environment variables
ENV PYTHONUNBUFFERED=1

# Run the application
CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:create_app()"]
```

### 3. Rust Performance-Critical Service Dockerfile

```Dockerfile
# Stage 1: Build Stage
FROM rust:1.65-alpine AS builder

# Install system dependencies
RUN apk add --no-cache \
    musl-dev \
    openssl-dev \
    pkgconfig

# Set working directory
WORKDIR /app

# Copy Cargo configuration files
COPY Cargo.toml Cargo.lock ./

# Create a dummy src to cache dependencies
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs

# Cache dependencies
RUN cargo build --release

# Copy actual source code
COPY . .

# Build the application
RUN cargo build --release

# Stage 2: Lightweight Runtime Image
FROM alpine:latest

# Install necessary runtime dependencies
RUN apk add --no-cache libgcc

# Copy binary from builder
COPY --from=builder /app/target/release/service-binary /usr/local/bin/

# Expose service port
EXPOSE 8080

# Run the binary
ENTRYPOINT ["/usr/local/bin/service-binary"]
```

## Docker Compose Configuration

### docker-compose.yml

```yml
version: "3.8"

services:
  # Infrastructure Services
  kafka:
    image: confluentinc/cp-kafka:6.2.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:6.2.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  redis:
    image: redis:6.2-alpine
    ports:
      - "6379:6379"

  postgresql:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: cipherhorizon
      POSTGRES_PASSWORD: securepassword
    ports:
      - "5432:5432"

  # Microservices
  market-data-service:
    build:
      context: ./services/market-data
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    depends_on:
      - kafka
      - redis
    environment:
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - REDIS_HOST=redis

  trading-signal-service:
    build:
      context: ./services/trading-signal
      dockerfile: Dockerfile.python
    ports:
      - "8082:5000"
    depends_on:
      - market-data-service
      - postgresql
    environment:
      - DATABASE_URL=postgresql://cipherhorizon:securepassword@postgresql:5432/cipherhorizon

  risk-management-service:
    build:
      context: ./services/risk-management
      dockerfile: Dockerfile.rust
    ports:
      - "8083:8080"
    depends_on:
      - trading-signal-service
    environment:
      - RUST_LOG=info

  # Monitoring Services
  prometheus:
    image: prom/prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

# Network Configuration
networks:
  default:
    driver: bridge
    name: cipherhorizon-network

# Persistent Volumes
volumes:
  postgres-data:
  redis-data:
  prometheus-data:
```

## Best Practices

### Containerization Checklist

- [ ] Use multi-stage builds
- [ ] Minimize image size
- [ ] Run as non-root user
- [ ] Include health checks
- [ ] Use specific image tags
- [ ] Implement dependency caching
- [ ] Secure build process

### Performance Optimization

```bash
# Build with caching
docker build --cache-from cipherhorizon/market-data-service:latest .

# Optimize image size
docker image prune
docker system df
```

## Quick Start

```bash
# Build all services
docker-compose build

# Start platform
docker-compose up -d

# View logs
docker-compose logs -f

# Stop platform
docker-compose down
```

### Production Considerations

```bash
# Production build with optimizations
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Prune unused images and containers
docker system prune -af
```

## Optimization Techniques

### 1. Multi-Stage Builds

- Reduce final image size
- Separate build and runtime environments
- Remove unnecessary build dependencies

### 2. Layer Caching

- Order Dockerfile instructions strategically
- Minimize layer changes
- Use .dockerignore to exclude unnecessary files

### 3. Security Scanning

```bash
# Scan Docker images for vulnerabilities
docker scan cipherhorizon/market-data-service
```

## Monitoring Docker Containers

```bash
# Resource usage
docker stats

# Container logs
docker-compose logs market-data-service
```

## Best Practices

- Use specific image tags
- Minimize image layers
- Don't run containers as root
- Use .dockerignore
- Keep images small
- Use multi-stage builds

## Troubleshooting

- Check container logs
- Verify network connectivity
- Inspect container configurations
- Use docker-compose config to validate
