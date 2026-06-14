# n8n Infrastructure Setup Guide

**Version:** 1.0  
**Date:** June 2026  
**Owner:** DevOps Team

---

## Quick Start - 15 Minutes

### Prerequisites Check
```bash
# Verify Docker
docker --version  # Should be 4.0+
docker-compose --version  # Should be 2.0+

# Verify system resources
free -h  # Should have 8GB+ RAM
df -h /  # Should have 50GB+ storage

# Verify network
curl https://www.docker.com  # Test internet connectivity
```

### One-Command Setup
```bash
# Clone and setup
git clone https://github.com/anthonyolow-ops/n8n.git
cd n8n
chmod +x scripts/setup.sh
./scripts/setup.sh

# Wait for services to start (2-3 minutes)
docker-compose logs -f n8n
```

### Verify Installation
```bash
# Check all services running
docker-compose ps

# Test n8n
curl http://localhost:5678/health

# Test database
docker-compose exec postgres psql -U n8n_user -d n8n_db -c "SELECT version();"

# Test Redis
docker-compose exec redis redis-cli ping
```

---

## Detailed Installation - Docker Compose

### Step 1: Prepare System

```bash
# Create project directory
sudo mkdir -p /opt/n8n-platform
sudo chown $USER:$USER /opt/n8n-platform
cd /opt/n8n-platform

# Create subdirectories
mkdir -p config data logs custom-nodes
```

### Step 2: Download Configuration Files

```bash
# Download docker-compose.yml
wget https://raw.githubusercontent.com/anthonyolow-ops/n8n/main/docker-compose.yml

# Download environment template
wget https://raw.githubusercontent.com/anthonyolow-ops/n8n/main/.env.example
cp .env.example .env
```

### Step 3: Configure Environment

Edit `.env` file:

```env
# Database Configuration
DB_PASSWORD=generate_secure_password_here
DB_HOST=postgres
DB_PORT=5432
DB_NAME=n8n_db
DB_USER=n8n_user

# Redis Configuration
REDIS_PASSWORD=generate_secure_redis_password

# n8n Configuration
N8N_ENCRYPTION_KEY=generate_32_char_hex_key
JWT_SECRET=generate_32_char_hex_key
N8N_LOG_LEVEL=info
NODE_ENV=production

# Domain Configuration
DOMAIN=n8n.yourdomain.com
WEBHOOK_URL=https://n8n.yourdomain.com/

# Security
N8N_USER_MANAGEMENT_DISABLED=false
N8N_DIAGNOSTICS_ENABLED=false
```

**Generate Secure Keys:**
```bash
# Generate encryption key
openssl rand -hex 32

# Generate JWT secret
openssl rand -hex 32

# Generate Redis password
openssl rand -base64 32
```

### Step 4: Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    container_name: n8n-postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n-main
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=${DB_HOST}
      - DB_POSTGRESDB_PORT=${DB_PORT}
      - DB_POSTGRESDB_DATABASE=${DB_NAME}
      - DB_POSTGRESDB_USER=${DB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=336
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_JWT_SECRET=${JWT_SECRET}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL}
      - N8N_USER_MANAGEMENT_DISABLED=${N8N_USER_MANAGEMENT_DISABLED}
      - N8N_DIAGNOSTICS_ENABLED=${N8N_DIAGNOSTICS_ENABLED}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - NODE_ENV=${NODE_ENV}
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
      - ./custom-nodes:/home/node/.n8n/custom
    networks:
      - n8n-network
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  prometheus:
    image: prom/prometheus:latest
    container_name: n8n-prometheus
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - n8n-network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: n8n-grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana-dashboards.yml:/etc/grafana/provisioning/dashboards/dashboards.yml
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - n8n-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  n8n_data:
  prometheus_data:
  grafana_data:

networks:
  n8n-network:
    driver: bridge
```

### Step 5: Initialize Database

```sql
-- init-db.sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS uuid-ossp;

-- Create schemas
CREATE SCHEMA IF NOT EXISTS financial_center;
CREATE SCHEMA IF NOT EXISTS banking;
CREATE SCHEMA IF NOT EXISTS satellite;
CREATE SCHEMA IF NOT EXISTS consultancy;
CREATE SCHEMA IF NOT EXISTS audit_logs;

-- Grant permissions
GRANT USAGE ON SCHEMA financial_center, banking, satellite, consultancy, audit_logs 
TO n8n_user;
GRANT CREATE ON SCHEMA financial_center, banking, satellite, consultancy, audit_logs 
TO n8n_user;
```

### Step 6: Launch Services

```bash
# Start all services
docker-compose up -d

# Monitor startup
docker-compose logs -f n8n

# Wait for services (check logs for "n8n ready")
# Usually takes 30-60 seconds
```

### Step 7: Verify Installation

```bash
# Check all containers running
docker-compose ps

# Expected output:
# NAME                  STATUS
# n8n-postgres         Up (healthy)
# n8n-redis            Up (healthy)
# n8n-main             Up
# n8n-prometheus       Up
# n8n-grafana          Up

# Test connectivity
curl http://localhost:5678/health
# Should return: {"status":"ok"}

# Test database
docker-compose exec postgres psql -U n8n_user -d n8n_db \
  -c "SELECT datname FROM pg_database LIMIT 1;"

# Test Redis
docker-compose exec redis redis-cli -a ${REDIS_PASSWORD} PING
# Should return: PONG
```

---

## Access Points

### After Installation

| Service | URL | Credentials |
|---------|-----|-------------|
| n8n UI | http://localhost:5678 | Set on first login |
| Grafana | http://localhost:3000 | admin / admin123 |
| Prometheus | http://localhost:9090 | N/A |
| PostgreSQL | localhost:5432 | See .env |
| Redis | localhost:6379 | See .env |

### Initial Setup in n8n

1. Access http://localhost:5678
2. Create admin account
3. Set strong password
4. Enable 2FA (recommended)
5. Create API key for integrations

---

## Kubernetes Deployment

For production environments:

```yaml
# kubernetes/n8n-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: n8n
  template:
    metadata:
      labels:
        app: n8n
    spec:
      containers:
      - name: n8n
        image: n8nio/n8n:latest
        ports:
        - containerPort: 5678
        env:
        - name: DB_TYPE
          value: "postgresdb"
        - name: DB_POSTGRESDB_HOST
          value: "postgres-service"
        - name: WEBHOOK_URL
          value: "https://n8n.yourdomain.com/"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5678
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: n8n-service
  namespace: production
spec:
  selector:
    app: n8n
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
  type: LoadBalancer
```

---

## Troubleshooting

### Common Issues

**Issue: Port 5678 already in use**
```bash
# Find process using port
lsof -i :5678

# Kill process
kill -9 <PID>

# Or use different port
PORT=5679 docker-compose up -d
```

**Issue: Database connection failed**
```bash
# Check database logs
docker-compose logs postgres

# Verify credentials
docker-compose exec postgres psql -U n8n_user -d n8n_db

# Reset database
docker-compose down -v
docker-compose up -d postgres
```

**Issue: Out of memory**
```bash
# Increase Docker memory limit
# Docker Desktop: Preferences → Resources → Memory

# Or limit per container
docker-compose down
# Edit docker-compose.yml and add:
# services:
#   n8n:
#     mem_limit: 2g
docker-compose up -d
```

**Issue: Workflows not executing**
```bash
# Check n8n logs
docker-compose logs n8n

# Check Redis connection
docker-compose exec redis redis-cli PING

# Restart n8n service
docker-compose restart n8n
```

---

## Backup & Recovery

### Backup Data

```bash
# Backup database
docker-compose exec postgres pg_dump -U n8n_user n8n_db > backup_$(date +%Y%m%d).sql

# Backup n8n data
tar -czf n8n_backup_$(date +%Y%m%d).tar.gz n8n_data/

# Backup Redis
docker-compose exec redis redis-cli BGSAVE
cp redis_data/dump.rdb backup_redis_$(date +%Y%m%d).rdb
```

### Restore Data

```bash
# Restore database
docker-compose exec -T postgres psql -U n8n_user n8n_db < backup_20260614.sql

# Restore n8n data
tar -xzf n8n_backup_20260614.tar.gz

# Restart services
docker-compose restart n8n
```

---

## Production Checklist

- [ ] SSL/TLS certificates configured
- [ ] Firewall rules set up
- [ ] Backups automated
- [ ] Monitoring active (Prometheus/Grafana)
- [ ] Log aggregation setup
- [ ] Disaster recovery plan
- [ ] Security audit completed
- [ ] Performance baseline established
- [ ] Team trained on operations
- [ ] Runbooks created

---

**End of Infrastructure Setup Guide**