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

### n8n Workflow Template

```json
{
  "name": "Stripe Payment Processor",
  "nodes": [
    {
      "parameters": {
        "path": "stripe-payment",
        "method": "POST"
      },
      "name": "Webhook - Stripe",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "functionCode": "const payment = $input.first().json;\n\n// Validate payment amount\nif (!payment.amount || payment.amount <= 0) {\n  throw new Error('Invalid payment amount');\n}\n\n// Validate currency\nif (!['usd', 'eur', 'gbp'].includes(payment.currency)) {\n  throw new Error('Unsupported currency');\n}\n\nreturn { payment, validated: true };"
      },
      "name": "Validate Payment",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [450, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "SELECT kyc_status, id FROM financial_center.customers WHERE email = $1",
        "queryParams": ["{{ $node[\"Webhook - Stripe\"].json.customer.email }}"]
      },
      "name": "Check KYC Status",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "url": "https://api.compliance-service.com/screen",
        "method": "POST",
        "headers": {
          "Authorization": "Bearer {{ $env.COMPLIANCE_API_KEY }}"
        },
        "body": {
          "amount": "{{ $node[\"Webhook - Stripe\"].json.amount }}",
          "currency": "{{ $node[\"Webhook - Stripe\"].json.currency }}",
          "customer_id": "{{ $node[\"Check KYC Status\"].json.id }}"
        }
      },
      "name": "Compliance Screen",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [850, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "INSERT INTO financial_center.transactions (from_account_id, amount, currency_code, transaction_type, status, metadata) VALUES ($1, $2, $3, 'payment', 'settled', $4) RETURNING id",
        "queryParams": [
          "{{ $node[\"Check KYC Status\"].json.id }}",
          "{{ $node[\"Webhook - Stripe\"].json.amount }}",
          "{{ $node[\"Webhook - Stripe\"].json.currency }}",
          "{{ JSON.stringify($node[\"Webhook - Stripe\"].json) }}"
        ]
      },
      "name": "Record in Ledger",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [1050, 300]
    },
    {
      "parameters": {
        "toEmail": "{{ $node[\"Webhook - Stripe\"].json.customer.email }}",
        "subject": "Payment Confirmation - {{ $node[\"Record in Ledger\"].json.id }}",
        "emailType": "text",
        "textContent": "Your payment of {{ $node[\"Webhook - Stripe\"].json.amount }} {{ $node[\"Webhook - Stripe\"].json.currency }} has been processed."
      },
      "name": "Send Confirmation Email",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 1,
      "position": [1250, 300]
    },
    {
      "parameters": {
        "text": "Payment processed successfully: {{ $node[\"Record in Ledger\"].json.id }}",
        "channel": "#payments"
      },
      "name": "Notify Slack",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [1450, 300]
    }
  ],
  "connections": {
    "Webhook - Stripe": {
      "main": [
        [
          {
            "node": "Validate Payment",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Validate Payment": {
      "main": [
        [
          {
            "node": "Check KYC Status",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Check KYC Status": {
      "main": [
        [
          {
            "node": "Compliance Screen",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Compliance Screen": {
      "main": [
        [
          {
            "node": "Record in Ledger",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Record in Ledger": {
      "main": [
        [
          {
            "node": "Send Confirmation Email",
            "type": "main",
            "index": 0
          },
          {
            "node": "Notify Slack",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

## Template 2: Planet Labs Satellite Data Ingestion

### Use Case
Fetch satellite imagery, process geospatial data, store in database

### Configuration Required
- Planet Labs API Key: Obtain from https://www.planet.com/
- Area of Interest: Define geographic boundaries

### n8n Workflow Template

```json
{
  "name": "Satellite Data Ingestion - Planet Labs",
  "nodes": [
    {
      "parameters": {
        "trigger": "every",
        "unit": "hours",
        "value": 6
      },
      "name": "Schedule - Every 6 Hours",
      "type": "n8n-nodes-base.trigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "url": "https://api.planet.com/data/v1/searches/",
        "method": "POST",
        "authentication": "genericCredentialType",
        "genericCredentialType": "httpBasicAuth",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": {
          "name": "recent_imagery_search",
          "search_type": "AndSearchFilter",
          "filter": {
            "type": "AndFilter",
            "config": [
              {
                "type": "GeometryFilter",
                "field_name": "geometry",
                "config": {
                  "type": "Polygon",
                  "coordinates": "{{ $env.SEARCH_AREA_COORDINATES }}"
                }
              },
              {
                "type": "RangeFilter",
                "field_name": "acquired",
                "config": {
                  "gte": "{{ $now.minus(6, 'hours').toISO() }}"
                }
              },
              {
                "type": "StringFilter",
                "field_name": "ground_control",
                "config": ["true"]
              }
            ]
          }
        }
      },
      "name": "Search Recent Imagery",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [450, 300]
    },
    {
      "parameters": {
        "functionCode": "const results = $input.all();\nconst imageItems = results[0].json.features || [];\n\nif (imageItems.length === 0) {\n  return { message: 'No new imagery found' };\n}\n\nconst processedItems = imageItems.map(item => ({\n  item_id: item.id,\n  acquired_time: item.properties.acquired,\n  cloud_cover: item.properties.cloud_cover,\n  geometry: item.geometry,\n  asset_type: item.properties.item_type,\n  confidence: item.properties.ground_control ? 'high' : 'medium'\n}));\n\nreturn { items: processedItems, count: imageItems.length };"
      },
      "name": "Parse Imagery Results",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "operation": "loopOver",
        "overParam": "items"
      },
      "name": "For Each Image",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 1,
      "position": [850, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "INSERT INTO satellite.satellite_images (item_id, acquired_at, cloud_cover, geometry, asset_type, confidence, metadata) VALUES ($1, $2, $3, ST_GeomFromGeoJSON($4), $5, $6, $7) ON CONFLICT (item_id) DO UPDATE SET updated_at = NOW() RETURNING id",
        "queryParams": [
          "{{ $node[\"For Each Image\"].json.item_id }}",
          "{{ $node[\"For Each Image\"].json.acquired_time }}",
          "{{ $node[\"For Each Image\"].json.cloud_cover }}",
          "{{ JSON.stringify($node[\"For Each Image\"].json.geometry) }}",
          "{{ $node[\"For Each Image\"].json.asset_type }}",
          "{{ $node[\"For Each Image\"].json.confidence }}",
          "{{ JSON.stringify($node[\"For Each Image\"].json) }}"
        ]
      },
      "name": "Store Image Metadata",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [1050, 300]
    },
    {
      "parameters": {
        "text": "New satellite image ingested: {{ $node[\"For Each Image\"].json.item_id }} (Cloud cover: {{ $node[\"For Each Image\"].json.cloud_cover }}%)",
        "channel": "#satellite-operations"
      },
      "name": "Notify Team",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [1250, 300]
    }
  ],
  "connections": {
    "Schedule - Every 6 Hours": {
      "main": [
        [
          {
            "node": "Search Recent Imagery",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Search Recent Imagery": {
      "main": [
        [
          {
            "node": "Parse Imagery Results",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Imagery Results": {
      "main": [
        [
          {
            "node": "For Each Image",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "For Each Image": {
      "main": [
        [
          {
            "node": "Store Image Metadata",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Store Image Metadata": {
      "main": [
        [
          {
            "node": "Notify Team",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

## Template 3: Salesforce CRM Integration

### Use Case
Sync customer data from CRM to financial center, create accounts

### Setup
1. Create Salesforce Connected App
2. Get OAuth credentials
3. Configure in n8n

### n8n Workflow Template

```json
{
  "name": "Salesforce Customer Sync",
  "nodes": [
    {
      "parameters": {
        "trigger": "every",
        "unit": "hours",
        "value": 1
      },
      "name": "Schedule - Hourly Sync",
      "type": "n8n-nodes-base.trigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "resource": "contact",
        "operation": "getAll",
        "limit": 100
      },
      "name": "Get Updated Contacts",
      "type": "n8n-nodes-base.salesforce",
      "typeVersion": 1,
      "position": [450, 300]
    },
    {
      "parameters": {
        "operation": "loopOver",
        "overParam": "data"
      },
      "name": "For Each Contact",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "INSERT INTO financial_center.customers (email, first_name, last_name, phone, created_at) VALUES ($1, $2, $3, $4, NOW()) ON CONFLICT (email) DO UPDATE SET updated_at = NOW() RETURNING id",
        "queryParams": [
          "{{ $node[\"For Each Contact\"].json.Email }}",
          "{{ $node[\"For Each Contact\"].json.FirstName }}",
          "{{ $node[\"For Each Contact\"].json.LastName }}",
          "{{ $node[\"For Each Contact\"].json.Phone }}"
        ]
      },
      "name": "Sync to Financial Center",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [850, 300]
    },
    {
      "parameters": {
        "operation": "update",
        "resource": "contact",
        "id": "{{ $node[\"For Each Contact\"].json.Id }}",
        "updateData": {
          "FC_Customer_ID__c": "{{ $node[\"Sync to Financial Center\"].json.id }}"
        }
      },
      "name": "Update Salesforce with FC ID",
      "type": "n8n-nodes-base.salesforce",
      "typeVersion": 1,
      "position": [1050, 300]
    }
  ],
  "connections": {
    "Schedule - Hourly Sync": {
      "main": [
        [
          {
            "node": "Get Updated Contacts",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Updated Contacts": {
      "main": [
        [
          {
            "node": "For Each Contact",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "For Each Contact": {
      "main": [
        [
          {
            "node": "Sync to Financial Center",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Sync to Financial Center": {
      "main": [
        [
          {
            "node": "Update Salesforce with FC ID",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

## Template 4: Asana Project Management Sync

### Use Case
Sync Asana projects to consultancy projects, track time entries

### Configuration
- Asana Personal Access Token
- Project IDs to sync

### n8n Workflow Template

```json
{
  "name": "Asana Project Sync",
  "nodes": [
    {
      "parameters": {
        "trigger": "every",
        "unit": "minutes",
        "value": 15
      },
      "name": "Schedule - Every 15 Minutes",
      "type": "n8n-nodes-base.trigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "operation": "getAll",
        "resource": "project",
        "limit": 50
      },
      "name": "Get Asana Projects",
      "type": "n8n-nodes-base.asana",
      "typeVersion": 1,
      "position": [450, 300]
    },
    {
      "parameters": {
        "operation": "loopOver",
        "overParam": "data"
      },
      "name": "For Each Project",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "INSERT INTO consultancy.projects (customer_id, project_name, project_status, start_date, budget) VALUES ((SELECT id FROM financial_center.customers LIMIT 1), $1, $2, $3, $4) ON CONFLICT (project_name) DO UPDATE SET project_status = $2 RETURNING id",
        "queryParams": [
          "{{ $node[\"For Each Project\"].json.name }}",
          "{{ $node[\"For Each Project\"].json.completed ? 'completed' : 'in_progress' }}",
          "{{ new Date($node[\"For Each Project\"].json.created_at).toISOString().split('T')[0] }}",
          "0"
        ]
      },
      "name": "Sync to Database",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [850, 300]
    }
  ]
}
```

---

## Template 5: Daily Compliance Monitoring

### Use Case
Daily check for compliance issues, generate reports

### n8n Workflow Template

```json
{
  "name": "Daily Compliance Monitoring",
  "nodes": [
    {
      "parameters": {
        "trigger": "every",
        "unit": "days",
        "value": 1,
        "hour": 1,
        "minute": 0
      },
      "name": "Schedule - Daily at 1 AM",
      "type": "n8n-nodes-base.trigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "SELECT COUNT(*) as failed_kyc, COUNT(CASE WHEN kyc_status='pending' THEN 1 END) as pending FROM financial_center.customers WHERE kyc_status != 'approved'"
      },
      "name": "Check KYC Status",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [450, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "SELECT COUNT(*) as failed_transactions FROM financial_center.transactions WHERE status = 'failed' AND created_at > NOW() - INTERVAL '1 day'"
      },
      "name": "Check Failed Transactions",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "toEmail": "compliance@company.com",
        "subject": "Daily Compliance Report - {{ $now.toFormat('yyyy-MM-dd') }}",
        "htmlContent": "<h2>Daily Compliance Report</h2><p>Failed KYC: {{ $node[\"Check KYC Status\"].json.failed_kyc }}</p><p>Pending KYC: {{ $node[\"Check KYC Status\"].json.pending }}</p><p>Failed Transactions: {{ $node[\"Check Failed Transactions\"].json.failed_transactions }}</p>"
      },
      "name": "Send Report Email",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 1,
      "position": [850, 300]
    }
  ]
}
```

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
