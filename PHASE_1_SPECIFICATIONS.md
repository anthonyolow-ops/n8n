# Phase 1: Foundation - Detailed Specifications (Weeks 1-4)

**Document Version:** 1.0  
**Created:** June 2026  
**Phase:** 1 (Foundation)  
**Timeline:** Weeks 1-4 (4 weeks)  
**Owner:** anthonyolow-ops

---

## Executive Summary

Phase 1 establishes the technical and organizational foundation for the integrated platform. This phase focuses on infrastructure setup, database design, API architecture, and team onboarding.

**Phase 1 Goals:**
- ✅ Deployed n8n instance (self-hosted)
- ✅ Database schema and core data models
- ✅ API gateway and integration patterns
- ✅ Development environment ready
- ✅ Team trained and productive

**Success Criteria:**
- n8n running in development and staging
- Database initialized with core schemas
- 5+ integration templates created
- Team able to build and deploy workflows
- CI/CD pipeline operational

---

## Week 1: Infrastructure & Environment Setup

### Week 1 Goals
- [ ] n8n environment setup (dev, staging, production)
- [ ] Database infrastructure provisioning
- [ ] Version control and CI/CD pipeline
- [ ] Security baseline
- [ ] Team access provisioning

### 1.1 n8n Installation & Configuration

#### Option A: Self-Hosted Docker Setup (Recommended for this project)

**Prerequisites:**
- Docker & Docker Compose installed
- 4+ CPU cores, 8GB+ RAM minimum
- 50GB+ storage for databases and workflows
- Ubuntu 20.04 LTS or similar Linux distribution

**Installation Steps:**

```bash
# 1. Create project directory
mkdir -p /opt/n8n-platform
cd /opt/n8n-platform

# 2. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:14-alpine
    container_name: n8n-postgres
    environment:
      POSTGRES_USER: n8n_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: n8n_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n_user"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # n8n Main Instance
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n-main
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n_db
      - DB_POSTGRESDB_USER=n8n_user
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_LOG_LEVEL=info
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_USER_MANAGEMENT_DISABLED=false
      - N8N_JWT_SECRET=${JWT_SECRET}
      - WEBHOOK_URL=https://n8n.yourdomain.com/
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
      - ./custom-nodes:/home/node/.n8n/custom
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - n8n-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  n8n_data:

networks:
  n8n-network:
    driver: bridge
EOF

# 3. Create .env file
cat > .env << 'EOF'
DB_PASSWORD=secure_postgres_password_123
REDIS_PASSWORD=secure_redis_password_456
N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)
JWT_SECRET=$(openssl rand -hex 32)
EOF

# 4. Create initialization SQL
cat > init-db.sql << 'EOF'
-- Create PostGIS extension for geospatial data
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS uuid-ossp;

-- Create schemas
CREATE SCHEMA IF NOT EXISTS financial_center;
CREATE SCHEMA IF NOT EXISTS banking;
CREATE SCHEMA IF NOT EXISTS satellite;
CREATE SCHEMA IF NOT EXISTS consultancy;
CREATE SCHEMA IF NOT EXISTS audit_logs;

-- Grant permissions
GRANT USAGE ON SCHEMA financial_center, banking, satellite, consultancy, audit_logs TO n8n_user;
GRANT CREATE ON SCHEMA financial_center, banking, satellite, consultancy, audit_logs TO n8n_user;
EOF

# 5. Start services
docker-compose up -d

# 6. Verify services
docker-compose ps
docker-compose logs -f n8n
```

**Expected Output:**
```
n8n ready on port 5678
Database initialized
Redis cache active
All services healthy
```

**Access Points:**
- n8n UI: `http://localhost:5678`
- PostgreSQL: `localhost:5432`
- Redis: `localhost:6379`

#### Initial Setup in n8n UI:

1. Navigate to `http://localhost:5678`
2. Create admin user account
3. Set up basic security:
   - Enable 2FA
   - Set password policy
   - Configure user roles

### 1.2 Database Infrastructure

#### PostgreSQL Schema Design

```sql
-- Core Financial Data Models

-- 1. Customers/Accounts
CREATE TABLE financial_center.customers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_type VARCHAR(50) NOT NULL CHECK (customer_type IN ('individual', 'business')),
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    business_name VARCHAR(255),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    kyc_status VARCHAR(50) DEFAULT 'pending',
    kyc_verified_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);

-- 2. Accounts
CREATE TABLE financial_center.accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_id UUID NOT NULL REFERENCES financial_center.customers(id),
    account_number VARCHAR(50) UNIQUE NOT NULL,
    account_type VARCHAR(50) NOT NULL CHECK (account_type IN ('checking', 'savings', 'merchant')),
    currency_code VARCHAR(3) NOT NULL DEFAULT 'USD',
    balance NUMERIC(19,4) DEFAULT 0,
    available_balance NUMERIC(19,4) DEFAULT 0,
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. Transactions
CREATE TABLE financial_center.transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    from_account_id UUID REFERENCES financial_center.accounts(id),
    to_account_id UUID REFERENCES financial_center.accounts(id),
    transaction_type VARCHAR(50) NOT NULL,
    amount NUMERIC(19,4) NOT NULL,
    currency_code VARCHAR(3) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    description TEXT,
    reference_number VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    settled_at TIMESTAMP,
    metadata JSONB
);

-- 4. Audit Logs
CREATE TABLE audit_logs.transaction_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id UUID NOT NULL,
    action VARCHAR(100),
    actor VARCHAR(255),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changes JSONB,
    ip_address VARCHAR(45)
);

-- 5. Geospatial Data (for satellite integration)
CREATE TABLE satellite.locations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    geom GEOMETRY(Point, 4326),
    latitude NUMERIC(10, 8),
    longitude NUMERIC(11, 8),
    region VARCHAR(100),
    country_code VARCHAR(2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 6. Satellite Asset Registry
CREATE TABLE satellite.assets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    asset_name VARCHAR(255) NOT NULL,
    asset_type VARCHAR(100),
    location_id UUID REFERENCES satellite.locations(id),
    last_monitored TIMESTAMP,
    status VARCHAR(50),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 7. Consultancy Projects
CREATE TABLE consultancy.projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_id UUID NOT NULL REFERENCES financial_center.customers(id),
    project_name VARCHAR(255) NOT NULL,
    project_status VARCHAR(50) DEFAULT 'planning',
    start_date DATE,
    end_date DATE,
    budget NUMERIC(19,4),
    currency_code VARCHAR(3) DEFAULT 'USD',
    office_id UUID,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 8. Time Entries
CREATE TABLE consultancy.time_entries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES consultancy.projects(id),
    consultant_id VARCHAR(255) NOT NULL,
    hours_worked NUMERIC(5, 2),
    rate_per_hour NUMERIC(10, 2),
    entry_date DATE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for performance
CREATE INDEX idx_customers_email ON financial_center.customers(email);
CREATE INDEX idx_accounts_customer ON financial_center.accounts(customer_id);
CREATE INDEX idx_transactions_from_account ON financial_center.transactions(from_account_id);
CREATE INDEX idx_transactions_to_account ON financial_center.transactions(to_account_id);
CREATE INDEX idx_transactions_status ON financial_center.transactions(status);
CREATE INDEX idx_satellite_locations_geom ON satellite.locations USING GIST(geom);
CREATE INDEX idx_projects_customer ON consultancy.projects(customer_id);

-- Create views for common queries
CREATE VIEW financial_center.account_summary AS
SELECT 
    a.id,
    a.account_number,
    c.email,
    a.balance,
    a.available_balance,
    a.currency_code,
    a.status,
    COUNT(t.id) as transaction_count
FROM financial_center.accounts a
JOIN financial_center.customers c ON a.customer_id = c.id
LEFT JOIN financial_center.transactions t ON a.id = t.from_account_id OR a.id = t.to_account_id
GROUP BY a.id, c.email;
```

### 1.3 Version Control & CI/CD Setup

#### GitHub Actions Workflow

Create `.github/workflows/phase1-setup.yml`:

```yaml
name: Phase 1 - Foundation Setup

on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'phase1/**'
      - '.github/workflows/phase1-setup.yml'

jobs:
  setup:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14-alpine
        env:
          POSTGRES_PASSWORD: postgres
        options: >
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/n8n_test
      
      - name: Validate n8n workflows
        run: npm run validate:workflows
      
      - name: Run security checks
        run: npm run security:check
      
      - name: Generate API documentation
        run: npm run docs:generate
```

---

## Week 2: Core Data Models & API Design

### Week 2 Goals
- [ ] Complete database schema implementation
- [ ] Design API gateway architecture
- [ ] Create integration templates
- [ ] Document data models
- [ ] Set up API documentation

### 2.1 API Gateway Architecture

Create `docs/API_ARCHITECTURE.md`:

```markdown
# API Gateway Architecture

## Overview
Central API gateway routing all external integrations through n8n workflows.

## Architecture Diagram

```
External Systems
    ↓
API Gateway (API Key, Rate Limiting, Auth)
    ↓
n8n Webhook Handler
    ↓
Business Logic Workflows
    ↓
Database / External Services
```

## Rate Limiting
- Public endpoints: 100 req/min per IP
- Authenticated endpoints: 1000 req/min per user
- Internal endpoints: Unlimited

## Authentication
- API Key: For service-to-service
- OAuth 2.0: For user authorization
- JWT: For token-based access

## Error Handling
- 4xx: Client errors
- 5xx: Server errors
- Standard error response format

## Versioning
- URL-based: /api/v1/*, /api/v2/*
- Header-based: X-API-Version: 1
```

### 2.2 Integration Templates

Create `integrations/templates/stripe-payment-processor.json`:

```json
{
  "name": "Stripe Payment Processor",
  "version": "1.0.0",
  "trigger": {
    "type": "webhook",
    "event": "payment.created"
  },
  "nodes": [
    {
      "name": "Validate Payment",
      "type": "function",
      "operations": [
        "check_amount",
        "verify_currency",
        "validate_metadata"
      ]
    },
    {
      "name": "Check KYC Status",
      "type": "database",
      "query": "SELECT kyc_status FROM customers WHERE id = $1"
    },
    {
      "name": "Compliance Screen",
      "type": "webhook",
      "url": "https://compliance-api.com/screen",
      "method": "POST"
    },
    {
      "name": "Post to Ledger",
      "type": "database",
      "query": "INSERT INTO transactions (...)"
    },
    {
      "name": "Send Notification",
      "type": "email",
      "template": "payment_received"
    }
  ]
}
```

### 2.3 Data Dictionary

Create `docs/DATA_DICTIONARY.md`:

```markdown
# Data Dictionary

## Financial Center Schema

### customers Table
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | ✓ | Unique customer identifier |
| customer_type | VARCHAR(50) | ✓ | 'individual' or 'business' |
| email | VARCHAR(255) | ✓ | Unique email address |
| kyc_status | VARCHAR(50) | | 'pending', 'approved', 'rejected' |
| created_at | TIMESTAMP | ✓ | Account creation timestamp |

### accounts Table
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | ✓ | Unique account identifier |
| customer_id | UUID | ✓ | Reference to customer |
| account_number | VARCHAR(50) | ✓ | Unique account number |
| balance | NUMERIC(19,4) | ✓ | Current balance |
| currency_code | VARCHAR(3) | ✓ | ISO currency code |
| status | VARCHAR(50) | ✓ | 'active', 'suspended', 'closed' |

### transactions Table
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | ✓ | Unique transaction ID |
| from_account_id | UUID | | Source account |
| to_account_id | UUID | | Destination account |
| amount | NUMERIC(19,4) | ✓ | Transaction amount |
| status | VARCHAR(50) | ✓ | 'pending', 'settled', 'failed' |
| reference_number | VARCHAR(100) | | External reference |

## Banking Schema
[... detailed specs ...]

## Satellite Schema
[... detailed specs ...]

## Consultancy Schema
[... detailed specs ...]
```

---

## Week 3: Development Environment & Team Onboarding

### Week 3 Goals
- [ ] Complete development environment setup
- [ ] Team training and documentation
- [ ] Create workflow templates
- [ ] Set up monitoring and logging
- [ ] Create runbooks and troubleshooting guides

### 3.1 Development Environment Setup

Create `docs/SETUP_GUIDE.md`:

```markdown
# Development Environment Setup Guide

## Prerequisites
- Docker Desktop 4.0+
- Node.js 18+
- PostgreSQL Client
- Git

## Installation Steps

### 1. Clone Repository
\`\`\`bash
git clone https://github.com/anthonyolow-ops/n8n.git
cd n8n
\`\`\`

### 2. Create Environment File
\`\`\`bash
cp .env.example .env
# Edit .env with your values
\`\`\`

### 3. Start Services
\`\`\`bash
docker-compose up -d
\`\`\`

### 4. Initialize Database
\`\`\`bash
npm run db:init
npm run db:migrate
\`\`\`

### 5. Access n8n
- Dashboard: http://localhost:5678
- API: http://localhost:5678/api

### Troubleshooting

#### Port Already in Use
\`\`\`bash
lsof -i :5678
kill -9 <PID>
\`\`\`

#### Database Connection Failed
\`\`\`bash
docker-compose logs postgres
# Check credentials in .env
\`\`\`

#### Workflow Not Running
\`\`\`bash
npm run logs:workflow
# Check n8n logs at: http://localhost:5678/execution-list
\`\`\`
```

### 3.2 Team Training Materials

Create `docs/TEAM_TRAINING.md`:

```markdown
# Team Training Program

## Module 1: n8n Fundamentals (2 hours)
- n8n UI overview
- Creating workflows
- Triggers and actions
- Data mapping
- Error handling

## Module 2: Our Architecture (2 hours)
- System overview
- Data flows
- Integration points
- Security model

## Module 3: Database & SQL (2 hours)
- Schema overview
- Query basics
- Writing migrations
- Optimization tips

## Module 4: API Development (2 hours)
- REST principles
- Authentication
- Rate limiting
- Error responses

## Module 5: Workflows in Practice (4 hours)
- Creating first workflow
- Integration with APIs
- Error handling
- Logging and monitoring

## Module 6: Testing & Deployment (2 hours)
- Unit testing
- Integration testing
- Staging deployment
- Production deployment

Total: 14 hours of training
```

### 3.3 Monitoring & Logging Setup

Create `docker/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'n8n'
    static_configs:
      - targets: ['localhost:5678']
    metrics_path: '/metrics'

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']
```

Create `docker/grafana-dashboard.json`:

```json
{
  "dashboard": {
    "title": "n8n Platform Monitoring",
    "panels": [
      {
        "title": "Workflow Executions",
        "targets": [
          {
            "expr": "increase(n8n_workflow_executions_total[5m])"
          }
        ]
      },
      {
        "title": "Database Connections",
        "targets": [
          {
            "expr": "pg_stat_activity_count"
          }
        ]
      },
      {
        "title": "API Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, n8n_api_duration_seconds_bucket)"
          }
        ]
      }
    ]
  }
}
```

---

## Week 4: Integration Templates & Documentation

### Week 4 Goals
- [ ] Create 5+ integration templates
- [ ] Complete API documentation
- [ ] Create deployment procedures
- [ ] Conduct team knowledge transfer
- [ ] Prepare for Phase 2

### 4.1 Integration Templates Repository

Create `integrations/templates/` directory structure:

```
integrations/templates/
├── stripe-payment-processor/
│   ├── workflow.json
│   ├── README.md
│   └── tests/
├── plaid-bank-connector/
│   ├── workflow.json
│   ├── README.md
│   └── tests/
├── planet-labs-satellite/
│   ├── workflow.json
│   ├── README.md
│   └── tests/
├── salesforce-crm/
│   ├── workflow.json
│   ├── README.md
│   └── tests/
├── asana-project-mgmt/
│   ├── workflow.json
│   ├── README.md
│   └── tests/
└── slack-notifications/
    ├── workflow.json
    ├── README.md
    └── tests/
```

### 4.2 Deployment Procedures

Create `docs/DEPLOYMENT_GUIDE.md`:

```markdown
# Deployment Guide

## Staging Deployment

### 1. Pre-deployment Checks
\`\`\`bash
npm run test:all
npm run lint
npm run security:check
npm run db:validate
\`\`\`

### 2. Deploy to Staging
\`\`\`bash
git push origin develop
# CI/CD triggers automatically
\`\`\`

### 3. Verify Deployment
\`\`\`bash
curl -H "Authorization: Bearer $STAGING_TOKEN" \
  https://staging-n8n.yourdomain.com/api/health
\`\`\`

## Production Deployment

### 1. Create Release
\`\`\`bash
git checkout main
git pull origin main
npm version patch
git push origin main --tags
\`\`\`

### 2. Monitor Deployment
\`\`\`bash
# Check logs
kubectl logs -f deployment/n8n-prod

# Check metrics
# Access Grafana dashboard
\`\`\`

### 3. Rollback if Needed
\`\`\`bash
kubectl rollout undo deployment/n8n-prod
\`\`\`
```

### 4.3 Knowledge Transfer Checklist

Create `docs/KNOWLEDGE_TRANSFER_CHECKLIST.md`:

```markdown
# Knowledge Transfer Checklist

## Week 4 - Team Enablement

### Documentation
- [ ] Architecture documentation completed
- [ ] API documentation live
- [ ] Database schema documented
- [ ] Troubleshooting guides created
- [ ] Runbooks written

### Training
- [ ] All team members trained (6 modules)
- [ ] Hands-on labs completed
- [ ] Certification quiz passed
- [ ] Q&A sessions held

### Infrastructure
- [ ] Dev environment working for all devs
- [ ] Staging environment tested
- [ ] Monitoring dashboards accessible
- [ ] Backup procedures verified

### Workflows
- [ ] 5+ integration templates deployed
- [ ] Example workflows executable
- [ ] Error handling tested
- [ ] Logging verified

### Processes
- [ ] Git workflow established
- [ ] Code review process defined
- [ ] Deployment procedures documented
- [ ] On-call rotation setup

## Sign-off
- [ ] Project Manager: ___________
- [ ] Tech Lead: ___________
- [ ] Team Lead: ___________
```

---

## Phase 1 Deliverables Summary

### Infrastructure
✅ n8n deployed (dev, staging)
✅ PostgreSQL with PostGIS
✅ Redis cache operational
✅ CI/CD pipeline active

### Development
✅ Database schema initialized
✅ 5+ integration templates ready
✅ API gateway designed
✅ Development environment documented

### Team
✅ All developers trained
✅ Documentation complete
✅ Runbooks and troubleshooting guides
✅ Knowledge transfer completed

### Monitoring
✅ Prometheus + Grafana running
✅ Health checks operational
✅ Logging centralized
✅ Alerts configured

---

## Success Metrics - Phase 1

| Metric | Target | Status |
|--------|--------|--------|
| n8n Uptime | 99%+ | |
| DB Query Latency | <100ms (p95) | |
| API Response Time | <200ms (p95) | |
| Workflow Success Rate | >99% | |
| Team Productivity | >80% allocation | |
| Knowledge Gaps | <5% | |

---

## Risk Mitigation - Phase 1

| Risk | Mitigation |
|------|------------|
| Infrastructure delays | Pre-provision all resources, parallel setup |
| Database issues | Thorough testing, backup strategies |
| Team learning curve | Comprehensive training, pair programming |
| Integration issues | POC before full implementation |
| Security gaps | Regular security reviews, penetration testing |

---

## Transition to Phase 2

Upon completion of Phase 1:
1. Code review of all implementations
2. Security audit of infrastructure
3. Performance baseline testing
4. Team readiness assessment
5. Stakeholder sign-off
6. Phase 2 kickoff (Week 5)

---

**End of Phase 1 Specifications**

Questions? See `docs/FAQ.md` or contact the technical team.