# API Integration Templates & Patterns

**Version:** 1.0  
**Date:** June 2026  
**Purpose:** Reusable templates for integrating external systems with n8n

---

## Template 1: Stripe Payment Processing

### Use Case
Process payments through Stripe, validate transactions, record in ledger

### Webhook Configuration in Stripe Dashboard
- Endpoint: `https://your-n8n-domain.com/webhook/stripe-payment`
- Events: `payment_intent.succeeded`, `payment_intent.payment_failed`

### Key Features
- Payment validation
- KYC status checking
- Compliance screening
- Ledger recording
- Confirmation notifications
- Team alerts

---

## Template 2: Planet Labs Satellite Data Ingestion

### Use Case
Fetch satellite imagery, process geospatial data, store in database

### Configuration Required
- Planet Labs API Key: Obtain from https://www.planet.com/
- Area of Interest: Define geographic boundaries

### Key Features
- Scheduled imagery fetching
- Geospatial processing
- Image metadata storage
- Cloud cover analysis
- Team notifications
- Historical tracking

---

## Template 3: Salesforce CRM Integration

### Use Case
Sync customer data from CRM to financial center, create accounts

### Setup
1. Create Salesforce Connected App
2. Get OAuth credentials
3. Configure in n8n

### Key Features
- Contact synchronization
- Customer profile creation
- Bidirectional updates
- Field mapping
- Error handling
- Audit logging

---

## Template 4: Asana Project Management Sync

### Use Case
Sync Asana projects to consultancy projects, track time entries

### Configuration
- Asana Personal Access Token
- Project IDs to sync

### Key Features
- Project synchronization
- Status tracking
- Team synchronization
- Budget alignment
- Progress reporting

---

## Template 5: Daily Compliance Monitoring

### Use Case
Daily check for compliance issues, generate reports

### Key Features
- KYC status monitoring
- Failed transaction tracking
- Compliance alerts
- Daily reporting
- Executive dashboards
- Exception handling

---

## Implementation Guide

### Step 1: Import Template into n8n
1. Open n8n dashboard
2. Click "+" → "Create new workflow"
3. Click "..." → "Import from JSON"
4. Paste template JSON
5. Adjust parameters as needed

### Step 2: Configure Credentials
1. Add API keys for each service
2. Set environment variables
3. Test connections

### Step 3: Test Workflow
1. Click "Execute workflow"
2. Monitor logs for errors
3. Verify data in database

### Step 4: Activate Workflow
1. Enable triggering (toggle on)
2. Set appropriate schedule
3. Monitor execution history

---

## Best Practices

### Error Handling
- Always add try-catch blocks
- Log errors to audit table
- Send alerts to team
- Implement retry logic

### Performance
- Batch operations when possible
- Use pagination for large datasets
- Set appropriate rate limits
- Monitor execution time

### Security
- Never log sensitive data
- Encrypt credentials
- Use environment variables
- Rotate API keys regularly

### Monitoring
- Track successful executions
- Monitor failure rates
- Alert on anomalies
- Maintain audit logs

---

**End of API Integration Templates**