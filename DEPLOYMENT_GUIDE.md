# CipherHorizon Deployment Guide

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Local Development Setup](#local-development-setup)
3. [Cloud Deployment](#cloud-deployment)
4. [Kubernetes Configuration](#kubernetes-configuration)
5. [Service Deployment](#service-deployment)
6. [Configuration Management](#configuration-management)
7. [Monitoring and Logging](#monitoring-and-logging)
8. [Scaling and Performance](#scaling-and-performance)
9. [Security Considerations](#security-considerations)
10. [Troubleshooting](#troubleshooting)

## Prerequisites

### Local Development Environment

- Docker (v20.10+)
- Kubernetes Cluster
  - Minikube (v1.25+)
  - Kind (v0.14+)
  - K3s (v1.23+)
- kubectl (v1.21+)
- Helm (v3.9+)
- Git
- Python 3.9+
- Golang 1.18+
- Node.js 16+

### Cloud Provider Requirements

- AWS EKS
- Google GKE
- Azure AKS
- DigitalOcean Kubernetes

### Required Tools

```bash
# Verify installations
docker --version
kubectl version
helm version
python3 --version
go version
node --version
```

## Local Development Setup

### 1. Clone Repository

```bash
git clone https://github.com/cipherhorizon/platform.git
cd platform
```

### 2. Local Kubernetes Cluster Setup

#### Option 1: Minikube

```bash
# Start Minikube
minikube start --cpus 4 --memory 8192 --disk-size 50g

# Enable addons
minikube addons enable metrics-server
minikube addons enable dashboard
```

#### Option 2: Kind

```bash
# Create kind cluster configuration
cat << EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

# Create cluster
kind create cluster --config kind-config.yaml
```

### 3. Environment Configuration

```bash
# Create environment files
cp .env.example .env

# Generate secrets
make generate-secrets
```

## Kubernetes Configuration

### Namespace Creation

```bash
# Create namespaces
kubectl create namespace cipherhorizon-system
kubectl create namespace cipherhorizon-services
kubectl create namespace monitoring
```

### Secrets Management

```bash
# Create Kubernetes secrets
kubectl create secret generic cipherhorizon-secrets \
    --namespace cipherhorizon-system \
    --from-literal=jwt-secret=$(openssl rand -base64 32) \
    --from-literal=encryption-key=$(openssl rand -base64 32)
```

## Service Deployment

### Helm Chart Deployment

```bash
# Add CipherHorizon Helm repository
helm repo add cipherhorizon https://charts.cipherhorizon.com
helm repo update

# Install core infrastructure
helm install cipherhorizon-infra cipherhorizon/infrastructure \
    --namespace cipherhorizon-system

# Deploy microservices
helm install cipherhorizon-services cipherhorizon/microservices \
    --namespace cipherhorizon-services \
    --values values.yaml
```

### Manual Kubernetes Deployment

```bash
# Deploy individual services
kubectl apply -f kubernetes/market-data-service/
kubectl apply -f kubernetes/trading-signal-service/
kubectl apply -f kubernetes/risk-management-service/
```

## Configuration Management

### Environment Variables

```yaml
# Example configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cipherhorizon-config
data:
  KAFKA_BOOTSTRAP_SERVERS: kafka.cipherhorizon-system.svc.cluster.local:9092
  DATABASE_HOST: postgresql.cipherhorizon-system.svc.cluster.local
  REDIS_HOST: redis.cipherhorizon-system.svc.cluster.local
```

### Configuring Service Parameters

```bash
# Update service configurations
kubectl edit configmap cipherhorizon-config -n cipherhorizon-services
```

## Monitoring and Logging

### Prometheus and Grafana Setup

```bash
# Install monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring
```

### Log Collection

```bash
# Deploy ELK Stack
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch \
    --namespace monitoring
helm install kibana elastic/kibana \
    --namespace monitoring
```

## Scaling and Performance

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: market-data-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: market-data-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        targetAverageUtilization: 70
```

### Apply Autoscaling

```bash
kubectl apply -f kubernetes/autoscaling/
```

## Security Considerations

### Network Policies

```bash
# Apply network policies
kubectl apply -f kubernetes/network-policies/
```

### TLS Configuration

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

## Continuous Deployment

### GitHub Actions Workflow

```yaml
name: Deploy to Kubernetes
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Kubernetes
        run: |
          kubectl config set-cluster k8s
          kubectl config set-credentials admin
          helm upgrade cipherhorizon-services ./helm/microservices
```

## Troubleshooting

### Common Debugging Commands

```bash
# Check service status
kubectl get pods -n cipherhorizon-services

# View logs
kubectl logs -f deployment/market-data-service -n cipherhorizon-services

# Describe deployment
kubectl describe deployment market-data-service -n cipherhorizon-services
```

## Post-Deployment Verification

### Health Check Script

```bash
#!/bin/bash
# health-check.sh

# Check microservices
SERVICES=("market-data" "trading-signal" "risk-management")

for service in "${SERVICES[@]}"; do
    status=$(kubectl get pods -l app=$service -n cipherhorizon-services | grep Running | wc -l)
    if [ $status -eq 0 ]; then
        echo "❌ $service service is not running"
        exit 1
    fi
done

echo "✅ All services are running successfully"
```

## Maintenance and Updates

### Rolling Updates

```bash
# Perform rolling update
kubectl set image deployment/market-data-service \
    market-data-service=cipherhorizon/market-data-service:v1.1.0 \
    -n cipherhorizon-services
```
