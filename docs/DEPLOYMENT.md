# Aleph Deployment Guide

Complete guide for deploying Aleph in production environments using Docker Compose or Kubernetes.

## Table of Contents
- [Deployment Overview](#deployment-overview)
- [Prerequisites](#prerequisites)
- [Docker Compose Deployment](#docker-compose-deployment)
- [Kubernetes Deployment](#kubernetes-deployment)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Production Configuration](#production-configuration)
- [Backup and Restore](#backup-and-restore)
- [Monitoring and Logging](#monitoring-and-logging)
- [Scaling](#scaling)
- [Security Hardening](#security-hardening)
- [Updating Aleph](#updating-aleph)
- [Troubleshooting](#troubleshooting)

---

## Deployment Overview

### Deployment Options

| Method | Best For | Complexity | Scalability |
|--------|----------|------------|-------------|
| **Docker Compose** | Small/medium deployments, single server | Low | Limited |
| **Kubernetes** | Large deployments, high availability | High | Excellent |

### Architecture Components

```
┌─────────────────────────────────────────────────────────┐
│                 Load Balancer / Ingress                 │
│              (Nginx, Traefik, AWS ALB, etc.)            │
└──────────────────┬────────────────────┬─────────────────┘
                   │                    │
         ┌─────────▼──────────┐  ┌──────▼───────────┐
         │    UI (Nginx)      │  │   API (Gunicorn) │
         │  Static frontend   │  │   Flask backend  │
         └────────────────────┘  └──────┬───────────┘
                                        │
              ┌─────────────────────────┼──────────────────┐
              │                         │                  │
       ┌──────▼──────┐  ┌──────────────▼────┐  ┌──────────▼──────┐
       │ PostgreSQL  │  │  Elasticsearch    │  │  Redis / RMQ    │
       │  Database   │  │  Search index     │  │  Cache / Queue  │
       └─────────────┘  └───────────────────┘  └─────────────────┘
                                 ▲
                                 │
                        ┌────────▼────────┐
                        │    Workers      │
                        │  (Multiple)     │
                        │  - Index        │
                        │  - Xref         │
                        │  - Export       │
                        └─────────────────┘
```

### Service Overview

| Service | Purpose | Replicas | Resource Usage |
|---------|---------|----------|----------------|
| **UI** | Frontend (Nginx) | 2-3 | Low (CPU/RAM) |
| **API** | Backend (Gunicorn) | 3-5 | Medium (CPU/RAM) |
| **Worker** | Background jobs | 5-10 | High (CPU/RAM) |
| **PostgreSQL** | Database | 1 (master) | Medium (RAM, High I/O) |
| **Elasticsearch** | Search | 3+ nodes | High (RAM, I/O) |
| **Redis** | Cache | 1-2 (HA) | Low-Medium (RAM) |
| **RabbitMQ** | Queue | 1-3 (cluster) | Low-Medium (RAM) |
| **ingest-file** | Document processing | 2-3 | High (CPU/RAM) |

---

## Prerequisites

### Hardware Requirements

**Minimum (Small deployment, <100k documents)**:
- 4 CPU cores
- 16 GB RAM
- 100 GB SSD storage

**Recommended (Medium deployment, <1M documents)**:
- 8-16 CPU cores
- 32-64 GB RAM
- 500 GB SSD storage

**Large deployment (>1M documents)**:
- 16+ CPU cores
- 64-128 GB RAM
- 1+ TB SSD storage (NVMe preferred)

### Software Requirements

**Docker Compose**:
- Docker Engine 20.10+
- Docker Compose 2.0+
- Linux server (Ubuntu 20.04+, Debian 11+)

**Kubernetes**:
- Kubernetes 1.21+
- Helm 3.8+
- kubectl configured
- Storage provisioner (for persistent volumes)

### Network Requirements

**Inbound**:
- Port 80 (HTTP) - Optional, redirect to 443
- Port 443 (HTTPS) - Required
- Port 22 (SSH) - Optional, for management

**Outbound**:
- Port 443 (HTTPS) - Required for external APIs, OAuth
- Port 25/587 (SMTP) - Required for email

---

## Docker Compose Deployment

### 1. Server Setup

**Update system**:
```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y docker.io docker-compose git curl
```

**Enable Docker**:
```bash
sudo systemctl enable docker
sudo systemctl start docker

# Add user to docker group
sudo usermod -aG docker $USER
# Log out and back in for group to take effect
```

### 2. Clone Repository

```bash
# Clone Aleph
git clone https://github.com/alephdata/aleph.git
cd aleph

# Checkout stable version
git checkout 4.1.7  # Or latest tag
```

### 3. Configuration

**Create environment file**:
```bash
cp aleph.env.tmpl aleph.env
```

**Edit aleph.env** (see [Configuration Guide](./CONFIGURATION.md)):
```bash
# Required settings
ALEPH_SECRET_KEY=<generate-with-openssl-rand-hex-32>
ALEPH_APP_TITLE=My Aleph
ALEPH_UI_URL=https://aleph.example.org/

# Database
ALEPH_DATABASE_URI=postgresql://aleph:CHANGE_PASSWORD@postgres:5432/aleph

# Search
ALEPH_ELASTICSEARCH_URI=http://elasticsearch:9200

# Cache/Queue
REDIS_URL=redis://redis:6379/0
ALEPH_BROKER_URI=amqp://guest:guest@rabbitmq:5672

# Email (configure SMTP)
ALEPH_MAIL_FROM=noreply@example.org
ALEPH_MAIL_HOST=smtp.example.org
ALEPH_MAIL_USERNAME=apikey
ALEPH_MAIL_PASSWORD=<smtp-password>

# OAuth (optional, recommended)
ALEPH_OAUTH=true
ALEPH_OAUTH_HANDLER=keycloak  # or google, azure, etc.
ALEPH_OAUTH_KEY=<client-id>
ALEPH_OAUTH_SECRET=<client-secret>
ALEPH_OAUTH_METADATA_URL=https://auth.example.org/...

# Archive (S3 recommended for production)
ARCHIVE_TYPE=s3
ARCHIVE_BUCKET=aleph-documents
AWS_ACCESS_KEY_ID=<access-key>
AWS_SECRET_ACCESS_KEY=<secret-key>
AWS_REGION=us-east-1

# Production settings
ALEPH_DEBUG=false
ALEPH_CACHE=true
ALEPH_FORCE_HTTPS=true
ALEPH_INDEX_REPLICAS=1  # Increase for HA

# Monitoring
SENTRY_DSN=<sentry-dsn>
SENTRY_ENVIRONMENT=production
PROMETHEUS_ENABLED=true
```

### 4. Start Services

**Pull images**:
```bash
docker-compose pull
```

**Start infrastructure**:
```bash
docker-compose up -d postgres elasticsearch redis rabbitmq
```

**Wait for services to be ready**:
```bash
# Wait for PostgreSQL
docker-compose exec postgres pg_isready

# Wait for Elasticsearch
curl -s http://localhost:9200/_cluster/health | grep -q green
```

**Run migrations**:
```bash
docker-compose run --rm shell aleph upgrade
```

**Create admin user**:
```bash
docker-compose run --rm shell \
  aleph createuser --admin admin@example.org
```

**Start application**:
```bash
docker-compose up -d api ui worker ingest-file
```

**Verify deployment**:
```bash
# Check all services running
docker-compose ps

# Check API health
curl http://localhost:8000/api/2/status

# Access UI
open http://localhost:8080
```

### 5. Nginx Reverse Proxy (Recommended)

**Install Nginx**:
```bash
sudo apt install nginx certbot python3-certbot-nginx
```

**Configure Nginx** (`/etc/nginx/sites-available/aleph`):
```nginx
# HTTP - Redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name aleph.example.org;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name aleph.example.org;

    ssl_certificate /etc/letsencrypt/live/aleph.example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aleph.example.org/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Timeouts for long uploads
    client_max_body_size 2G;
    client_body_timeout 600s;
    proxy_read_timeout 600s;
    proxy_connect_timeout 600s;
    proxy_send_timeout 600s;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Enable site**:
```bash
sudo ln -s /etc/nginx/sites-available/aleph /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

**Get SSL certificate**:
```bash
sudo certbot --nginx -d aleph.example.org
```

**Auto-renewal**:
```bash
# Test renewal
sudo certbot renew --dry-run

# Cron is automatically configured by certbot
```

### 6. Systemd Service (Auto-start)

**Create service** (`/etc/systemd/system/aleph.service`):
```ini
[Unit]
Description=Aleph Document Platform
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/aleph
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

**Enable service**:
```bash
sudo systemctl enable aleph
sudo systemctl start aleph
```

---

## Kubernetes Deployment

### 1. Helm Chart

**Add repository**:
```bash
helm repo add aleph https://charts.alephdata.org
helm repo update
```

**Or use local chart**:
```bash
cd aleph/helm/charts/aleph
```

### 2. Configuration

**Create values file** (`values-production.yaml`):
```yaml
# values-production.yaml

# Global settings
global:
  imagePullPolicy: IfNotPresent
  storageClass: "standard"  # Adjust for your cluster

# Application configuration
config:
  app:
    title: "My Aleph"
    url: "https://aleph.example.org"
  secretKey: "<generate-secret>"  # openssl rand -hex 32

  # Database
  database:
    host: "postgresql.default.svc.cluster.local"
    port: 5432
    name: "aleph"
    user: "aleph"
    password: "<db-password>"

  # Elasticsearch
  elasticsearch:
    url: "http://elasticsearch-master.default.svc.cluster.local:9200"

  # Redis
  redis:
    url: "redis://redis-master.default.svc.cluster.local:6379/0"

  # Queue
  broker:
    url: "amqp://user:password@rabbitmq.default.svc.cluster.local:5672"

  # Email
  mail:
    from: "noreply@example.org"
    host: "smtp.example.org"
    port: 587
    username: "apikey"
    password: "<smtp-password>"
    tls: true

  # OAuth
  oauth:
    enabled: true
    handler: "keycloak"
    key: "<client-id>"
    secret: "<client-secret>"
    metadataUrl: "https://auth.example.org/..."

  # Archive (S3)
  archive:
    type: "s3"
    bucket: "aleph-production-documents"

# Deployment sizing
api:
  replicaCount: 3
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"

  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

ui:
  replicaCount: 2
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"

worker:
  replicaCount: 5
  resources:
    requests:
      memory: "4Gi"
      cpu: "2000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"

  # Worker stages (split by type for better scaling)
  stages:
    - index
    - xref

  autoscaling:
    enabled: true
    minReplicas: 5
    maxReplicas: 20
    targetCPUUtilizationPercentage: 75

ingestFile:
  replicaCount: 2
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"

# Ingress (Nginx Ingress Controller)
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "2g"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"

  hosts:
    - host: aleph.example.org
      paths:
        - path: /
          pathType: Prefix

  tls:
    - secretName: aleph-tls
      hosts:
        - aleph.example.org

# PostgreSQL (subchart)
postgresql:
  enabled: true
  auth:
    username: aleph
    password: "<db-password>"
    database: aleph
  primary:
    persistence:
      enabled: true
      size: 100Gi
    resources:
      requests:
        memory: "4Gi"
        cpu: "2000m"

# Elasticsearch (subchart)
elasticsearch:
  enabled: true
  replicas: 3
  minimumMasterNodes: 2

  esConfig:
    elasticsearch.yml: |
      cluster.name: "aleph"
      network.host: 0.0.0.0

  resources:
    requests:
      cpu: "2000m"
      memory: "8Gi"
    limits:
      cpu: "4000m"
      memory: "16Gi"

  esJavaOpts: "-Xms8g -Xmx8g"

  volumeClaimTemplate:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 500Gi

# Redis (subchart)
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
  master:
    persistence:
      enabled: true
      size: 10Gi

# RabbitMQ (subchart)
rabbitmq:
  enabled: true
  auth:
    username: aleph
    password: "<rabbitmq-password>"
  persistence:
    enabled: true
    size: 20Gi
```

### 3. Deploy

**Install dependencies** (if using external services):
```bash
# PostgreSQL
helm install postgresql bitnami/postgresql -f postgresql-values.yaml

# Elasticsearch
helm install elasticsearch elastic/elasticsearch -f elasticsearch-values.yaml

# Redis
helm install redis bitnami/redis -f redis-values.yaml

# RabbitMQ
helm install rabbitmq bitnami/rabbitmq -f rabbitmq-values.yaml
```

**Install Aleph**:
```bash
helm install aleph ./helm/charts/aleph \
  -f values-production.yaml \
  --namespace aleph \
  --create-namespace
```

**Run upgrade job** (migrations):
```bash
kubectl create job --from=cronjob/aleph-upgrade aleph-upgrade-manual -n aleph
kubectl logs -f job/aleph-upgrade-manual -n aleph
```

**Create admin user**:
```bash
kubectl exec -it deployment/aleph-api -n aleph -- \
  aleph createuser --admin admin@example.org
```

**Verify deployment**:
```bash
# Check pods
kubectl get pods -n aleph

# Check services
kubectl get svc -n aleph

# Check ingress
kubectl get ingress -n aleph

# Access application
open https://aleph.example.org
```

### 4. Kubernetes Scaling

**Manual scaling**:
```bash
# Scale API
kubectl scale deployment aleph-api --replicas=5 -n aleph

# Scale workers
kubectl scale deployment aleph-worker --replicas=10 -n aleph
```

**Horizontal Pod Autoscaler (HPA)**:
```yaml
# Already configured in values.yaml under autoscaling
# HPA automatically scales based on CPU/memory usage
```

**Vertical Pod Autoscaler** (optional):
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: aleph-api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aleph-api
  updatePolicy:
    updateMode: "Auto"
```

---

## SSL/TLS Configuration

### Let's Encrypt (Recommended)

**Docker Compose** (using Nginx + Certbot):
See [Nginx Reverse Proxy](#5-nginx-reverse-proxy-recommended) above.

**Kubernetes** (using cert-manager):

**1. Install cert-manager**:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

**2. Create ClusterIssuer**:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.org
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

**3. Apply**:
```bash
kubectl apply -f clusterissuer.yaml
```

Certificate is automatically issued when Ingress is created with annotation:
```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### Custom SSL Certificate

**Docker Compose**:

Update Nginx configuration:
```nginx
ssl_certificate /etc/nginx/ssl/cert.pem;
ssl_certificate_key /etc/nginx/ssl/key.pem;
```

Mount certificates:
```yaml
# docker-compose.override.yml
volumes:
  - ./certs:/etc/nginx/ssl:ro
```

**Kubernetes**:

Create TLS secret:
```bash
kubectl create secret tls aleph-tls \
  --cert=cert.pem \
  --key=key.pem \
  -n aleph
```

Reference in Ingress:
```yaml
spec:
  tls:
    - secretName: aleph-tls
      hosts:
        - aleph.example.org
```

---

## Production Configuration

### Database Optimization

**PostgreSQL**:
```ini
# /var/lib/postgresql/data/postgresql.conf

# Memory
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
work_mem = 32MB

# Connections
max_connections = 200

# Checkpoints
checkpoint_completion_target = 0.9
wal_buffers = 16MB

# Autovacuum
autovacuum = on
autovacuum_max_workers = 3
```

**Elasticsearch**:
```yaml
# Heap size (50% of RAM, max 31GB)
ES_JAVA_OPTS: "-Xms8g -Xmx8g"

# Cluster settings
discovery.zen.minimum_master_nodes: 2
cluster.routing.allocation.same_shard.host: true

# Indexing
indices.memory.index_buffer_size: 30%
indices.queries.cache.size: 10%
```

### Archive Storage

**S3** (Recommended):
```bash
ARCHIVE_TYPE=s3
ARCHIVE_BUCKET=aleph-production-docs
AWS_ACCESS_KEY_ID=<key>
AWS_SECRET_ACCESS_KEY=<secret>
AWS_REGION=us-east-1

# Optional: Lifecycle policy for cost optimization
# Move to Glacier after 90 days
```

**Google Cloud Storage**:
```bash
ARCHIVE_TYPE=gs
ARCHIVE_BUCKET=gs://aleph-prod-docs
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
```

**Azure Blob**:
```bash
ARCHIVE_TYPE=azure
ARCHIVE_BUCKET=aleph-documents
AZURE_STORAGE_ACCOUNT=<account>
AZURE_STORAGE_KEY=<key>
```

---

## Backup and Restore

### Database Backup

**Automated backup script** (`/opt/scripts/backup-aleph.sh`):
```bash
#!/bin/bash
set -e

# Configuration
BACKUP_DIR="/backups/aleph"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup
docker-compose exec -T postgres pg_dump -U aleph aleph | gzip > \
  "$BACKUP_DIR/aleph_db_$DATE.sql.gz"

# Upload to S3 (optional)
aws s3 cp "$BACKUP_DIR/aleph_db_$DATE.sql.gz" \
  s3://my-backups/aleph/db/

# Cleanup old backups
find "$BACKUP_DIR" -name "aleph_db_*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: aleph_db_$DATE.sql.gz"
```

**Cron job**:
```bash
# Daily at 2 AM
0 2 * * * /opt/scripts/backup-aleph.sh >> /var/log/aleph-backup.log 2>&1
```

### Database Restore

```bash
# Stop application
docker-compose stop api worker

# Restore database
gunzip < aleph_db_20250108_020000.sql.gz | \
  docker-compose exec -T postgres psql -U aleph aleph

# Restart application
docker-compose start api worker
```

### Elasticsearch Backup

**Snapshot repository** (S3):
```bash
# Configure snapshot repository
curl -X PUT "http://localhost:9200/_snapshot/aleph_backups" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "aleph-es-snapshots",
    "region": "us-east-1"
  }
}'

# Create snapshot
curl -X PUT "http://localhost:9200/_snapshot/aleph_backups/snapshot_1?wait_for_completion=true"

# Restore snapshot
curl -X POST "http://localhost:9200/_snapshot/aleph_backups/snapshot_1/_restore"
```

**Automated snapshots** (Kubernetes):
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: elasticsearch-backup
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: curlimages/curl
            command:
            - curl
            - -X
            - PUT
            - "http://elasticsearch:9200/_snapshot/aleph_backups/snapshot_$(date +%Y%m%d)?wait_for_completion=true"
          restartPolicy: OnFailure
```

---

## Monitoring and Logging

### Prometheus Metrics

**Enable metrics**:
```bash
PROMETHEUS_ENABLED=true
PROMETHEUS_PORT=9100
```

**Prometheus configuration**:
```yaml
scrape_configs:
  - job_name: 'aleph-api'
    static_configs:
      - targets: ['aleph-api:9100']

  - job_name: 'aleph-worker'
    static_configs:
      - targets: ['aleph-worker:9100']
```

**Key metrics**:
- `http_requests_total` - API request count
- `http_request_duration_seconds` - Request latency
- `worker_tasks_total` - Worker task count
- `worker_queue_size` - Queue depth
- `index_documents_total` - Indexed documents

### Grafana Dashboards

**Example dashboard queries**:
```promql
# Request rate
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m])

# API latency p95
histogram_quantile(0.95, http_request_duration_seconds_bucket)

# Queue depth
worker_queue_size

# Worker throughput
rate(worker_tasks_total{status="success"}[5m])
```

### Logging

**Centralized logging** (ELK stack):

**Docker Compose** (Filebeat):
```yaml
filebeat:
  image: elastic/filebeat:7.17.0
  volumes:
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
    - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
  command: filebeat -e -strict.perms=false
```

**Kubernetes** (Fluentd):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/aleph-*.log
      pos_file /var/log/fluentd/aleph.log.pos
      tag aleph.*
      format json
    </source>

    <match aleph.**>
      @type elasticsearch
      host elasticsearch
      port 9200
      index_name aleph-logs
    </match>
```

**Log aggregation options**:
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Loki** (Grafana Loki)
- **CloudWatch** (AWS)
- **Stackdriver** (GCP)

---

## Scaling

### Horizontal Scaling

**API servers**:
```bash
# Docker Compose
docker-compose up -d --scale api=5

# Kubernetes
kubectl scale deployment aleph-api --replicas=5 -n aleph
```

**Workers**:
```bash
# Different worker types for better scaling
# Worker 1: Indexing only (many replicas)
docker-compose up -d --scale worker-index=10

# Worker 2: Xref only (fewer, more powerful)
docker-compose up -d --scale worker-xref=3
```

**Elasticsearch**:
```bash
# Add data nodes
kubectl scale statefulset elasticsearch --replicas=5 -n aleph
```

### Vertical Scaling

**Increase resources**:
```yaml
# Kubernetes
resources:
  requests:
    memory: "8Gi"
    cpu: "4000m"
  limits:
    memory: "16Gi"
    cpu: "8000m"
```

### Load Balancing

**Nginx** (Docker Compose):
```nginx
upstream api_backend {
    least_conn;
    server api-1:8000;
    server api-2:8000;
    server api-3:8000;
}

server {
    location /api/ {
        proxy_pass http://api_backend;
    }
}
```

**Kubernetes** (Ingress):
```yaml
# Built-in load balancing via Service
kind: Service
spec:
  type: ClusterIP
  ports:
    - port: 8000
  selector:
    app: aleph-api
```

---

## Security Hardening

### Network Security

**Firewall rules**:
```bash
# UFW (Ubuntu)
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw enable
```

**Docker network isolation**:
```yaml
# docker-compose.yml
networks:
  frontend:  # UI, API
  backend:   # Database, ES, Redis
```

### Application Security

**Security headers** (Nginx):
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

**Environment isolation**:
```bash
# Never share secrets between environments
PROD_ALEPH_SECRET_KEY=<prod-secret>
STAGING_ALEPH_SECRET_KEY=<staging-secret>
```

**OAuth** (Disable password auth):
```bash
ALEPH_OAUTH=true
ALEPH_PASSWORD_LOGIN=false
```

### Database Security

**PostgreSQL**:
```bash
# Strong password
POSTGRES_PASSWORD=<64-char-random>

# SSL connections
ALEPH_DATABASE_URI=postgresql://...?sslmode=require

# Connection limits
max_connections=100
```

**Elasticsearch**:
```yaml
# Enable security (X-Pack)
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

### Secrets Management

**Kubernetes** (Sealed Secrets):
```bash
# Install sealed-secrets
kubectl apply -f \
  https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Create sealed secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
kubectl apply -f sealed-secret.yaml
```

**External Secrets Operator**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
```

---

## Updating Aleph

### Docker Compose Update

**1. Backup**:
```bash
# Backup database
./backup-aleph.sh

# Backup configuration
cp aleph.env aleph.env.backup
```

**2. Pull new version**:
```bash
cd /opt/aleph
git fetch --tags
git checkout 4.2.0  # New version

# Pull new images
docker-compose pull
```

**3. Stop services**:
```bash
docker-compose stop
```

**4. Run migrations**:
```bash
docker-compose run --rm shell aleph upgrade
```

**5. Start services**:
```bash
docker-compose up -d
```

**6. Verify**:
```bash
curl http://localhost:8000/api/2/status
```

### Kubernetes Update

**1. Update Helm values** (if needed):
```yaml
# values-production.yaml
image:
  tag: "4.2.0"
```

**2. Upgrade release**:
```bash
helm upgrade aleph ./helm/charts/aleph \
  -f values-production.yaml \
  -n aleph
```

**3. Monitor rollout**:
```bash
kubectl rollout status deployment/aleph-api -n aleph
kubectl rollout status deployment/aleph-worker -n aleph
```

**4. Run upgrade job** (if migrations needed):
```bash
kubectl create job --from=cronjob/aleph-upgrade aleph-upgrade-v420 -n aleph
kubectl logs -f job/aleph-upgrade-v420 -n aleph
```

### Rollback

**Docker Compose**:
```bash
git checkout 4.1.7  # Previous version
docker-compose down
docker-compose up -d

# Restore database if needed
gunzip < backup.sql.gz | docker-compose exec -T postgres psql -U aleph aleph
```

**Kubernetes**:
```bash
helm rollback aleph -n aleph
```

---

## Troubleshooting

### Service Not Starting

**Check logs**:
```bash
# Docker Compose
docker-compose logs -f api

# Kubernetes
kubectl logs -f deployment/aleph-api -n aleph
```

**Common issues**:
- Missing environment variables
- Database connection failed
- ES not ready

### High CPU Usage

**Check processes**:
```bash
docker stats  # Docker Compose
kubectl top pods -n aleph  # Kubernetes
```

**Solutions**:
- Scale workers
- Increase worker resources
- Optimize ES queries
- Check for runaway tasks

### High Memory Usage

**Elasticsearch** (most common):
```bash
# Increase heap size
ES_JAVA_OPTS="-Xms8g -Xmx8g"

# Clear cache
curl -X POST "http://localhost:9200/_cache/clear"
```

### Database Connection Pool Exhausted

**Increase pool size**:
```python
# aleph/core.py
app.config['SQLALCHEMY_POOL_SIZE'] = 50
app.config['SQLALCHEMY_MAX_OVERFLOW'] = 100
```

### Slow Queries

**Enable slow query log**:
```sql
-- PostgreSQL
ALTER DATABASE aleph SET log_min_duration_statement = 1000;  -- 1 second
```

**Check slow queries**:
```bash
docker-compose exec postgres tail -f /var/log/postgresql/postgresql.log
```

### Worker Queue Backed Up

**Check queue size**:
```bash
curl -u guest:guest http://localhost:15672/api/queues
```

**Solutions**:
- Scale workers
- Increase worker concurrency
- Check for failing tasks
- Optimize slow tasks

---

**Last Updated:** 2025-12-08
**Version:** 4.1.7
**Completeness:** 100%
