# FlexPrice API Integration Documentation

## Overview

The Vaani backend now integrates with FlexPrice for automated billing, usage metering, and subscription management. This integration enables:

- **Customer Management**: Automatic FlexPrice customer creation when clients are created
- **Plan Management**: Sync pricing plans from FlexPrice to local database
- **Usage Metering**: Track call usage and send events to FlexPrice for billing
- **Agent Pricing**: Assign pricing plans to agents with full history tracking
- **Cost Calculation**: Automatically calculate call costs based on usage metrics

## Architecture

### Components

1. **FlexPrice API Client** (`backend/utils/flexprice_client.py`)
   - HTTP client for FlexPrice API
   - Handles authentication, retries, and error handling
   - Provides methods for customers, plans, usage events, and subscriptions

2. **Pricing Service** (`backend/services/pricing_service.py`)
   - Business logic layer for pricing operations
   - Manages plan syncing from FlexPrice
   - Calculates call costs based on usage
   - Sends usage events to FlexPrice
   - Tracks agent pricing history

3. **API Endpoints** (`backend/server/routers/`)
   - **Clients Router**: Auto-creates FlexPrice customers
   - **Pricing Router**: Plan management and cost calculation endpoints

### Database Schema

```
clients
  ├─ flexprice_customer_id (VARCHAR(255) UNIQUE) - Maps to FlexPrice Customer

pricing (global pricing catalog)
  ├─ flexprice_plan_id (VARCHAR(255) UNIQUE, NOT NULL) - FlexPrice Plan ID
  ├─ pricing_name (VARCHAR(255))
  ├─ description (TEXT)
  ├─ plan_metadata (JSON) - Full plan data from FlexPrice
  ├─ is_active (BOOLEAN)
  └─ timestamps

agents
  └─ flexprice_plan_id (VARCHAR(255)) - Direct reference to pricing plan

calls
  ├─ flexprice_plan_id (VARCHAR(255)) - Pricing snapshot at call start
  └─ call_cost (FLOAT) - Calculated cost

agent_pricing_history
  ├─ agent_id (UUID FK)
  ├─ pricing_id (UUID FK)
  ├─ previous_pricing_id (UUID FK)
  ├─ action (VARCHAR(50))
  ├─ changed_by_user_id (INT FK)
  ├─ comments (TEXT)
  ├─ is_active (BOOLEAN)
  └─ created_at (TIMESTAMP)
```

## Configuration

### Environment Variables

Add to `.env`:

```bash
# FlexPrice Billing API
FLEXPRICE_API_KEY="sk_01KBCH6A8A6VNMYKA5M40P9X7J"
FLEXPRICE_BASE_URL="https://api.cloud.flexprice.io/v1"
```

### Initialization

The FlexPrice client is automatically initialized on first use:

```python
from backend.utils.flexprice_client import get_flexprice_client

# Get global client instance
client = get_flexprice_client()
```

## Usage Guide

### 1. Initial Setup: Sync Plans from FlexPrice

After deploying, sync all published plans from FlexPrice to your local database:

```bash
curl -X POST http://localhost:8000/api/pricing/sync-from-flexprice
```

Response:
```json
{
  "success": true,
  "message": "Synced 5 plans from FlexPrice",
  "synced_count": 5,
  "plan_ids": ["uuid1", "uuid2", ...]
}
```

### 2. Create Clients (Auto-syncs to FlexPrice)

When you create a client, a FlexPrice customer is automatically created:

```bash
curl -X POST http://localhost:8000/api/clients/ \
  -F "client_name=Acme Corp" \
  -F "logo=@logo.png"
```

The client's `flexprice_customer_id` field will be populated with the FlexPrice customer ID.

### 3. Assign Pricing Plans to Agents

Assign a plan to an agent (creates history record):

```bash
curl -X POST http://localhost:8000/api/pricing/agents/{agent_id}/assign-plan \
  -H "Content-Type: application/json" \
  -d '{
    "flexprice_plan_id": "plan_abc123",
    "comments": "Upgraded to premium plan"
  }'
```

Response:
```json
{
  "success": true,
  "message": "Assigned plan plan_abc123 to agent {agent_id}",
  "history_id": "uuid",
  "previous_plan": "plan_old123",
  "new_plan": "plan_abc123"
}
```

### 4. Track Agent Pricing History

View all pricing changes for an agent:

```bash
curl http://localhost:8000/api/pricing/agents/{agent_id}/pricing-history?limit=50
```

Response:
```json
{
  "agent_id": "uuid",
  "current_plan": "plan_abc123",
  "history": [
    {
      "id": "uuid",
      "action": "changed",
      "pricing_id": "uuid",
      "previous_pricing_id": "uuid",
      "comments": "Upgraded to premium plan",
      "changed_by_user_id": 1,
      "is_active": true,
      "created_at": "2025-12-02T10:00:00Z"
    }
  ],
  "total": 1
}
```

### 5. Finalize Call Costs

When a call completes, calculate its cost and send usage to FlexPrice:

```bash
curl -X POST http://localhost:8000/api/pricing/calls/{call_id}/finalize-cost \
  -H "Content-Type: application/json" \
  -d '{
    "usage_metrics": {
      "voice_minutes": 5.5,
      "api_calls": 10,
      "transcription_minutes": 5.0
    },
    "send_to_flexprice": true
  }'
```

Response:
```json
{
  "success": true,
  "call_id": "uuid",
  "cost": 2.75,
  "currency": "USD",
  "usage_metrics": {
    "voice_minutes": 5.5,
    "api_calls": 10,
    "transcription_minutes": 5.0
  },
  "sent_to_flexprice": true
}
```

**Usage Metrics Sent to FlexPrice:**
- Each metric is sent as a separate usage event
- Events are deduplicated using `call_id + metric_name` as idempotency key
- Events include metadata: `call_id`, `agent_id`

### 6. View Client Usage Summary

Get billing summary from FlexPrice for a client:

```bash
curl "http://localhost:8000/api/pricing/clients/{client_id}/usage-summary?start_time=2025-12-01T00:00:00Z&end_time=2025-12-31T23:59:59Z"
```

Response:
```json
{
  "success": true,
  "client_id": "uuid",
  "flexprice_customer_id": "cust_abc123",
  "usage_summary": {
    "total_usage": 1000,
    "total_cost": 150.00,
    "period": {
      "start": "2025-12-01T00:00:00Z",
      "end": "2025-12-31T23:59:59Z"
    }
  },
  "start_time": "2025-12-01T00:00:00Z",
  "end_time": "2025-12-31T23:59:59Z"
}
```

## API Endpoints Reference

### Pricing Management

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/pricing/sync-from-flexprice` | POST | Sync all plans from FlexPrice |
| `/api/pricing/sync-plan/{plan_id}` | POST | Sync specific plan from FlexPrice |
| `/api/pricing/agents/{agent_id}/assign-plan` | POST | Assign pricing plan to agent |
| `/api/pricing/agents/{agent_id}/pricing-history` | GET | Get agent pricing history |
| `/api/pricing/calls/{call_id}/finalize-cost` | POST | Calculate and finalize call cost |
| `/api/pricing/clients/{client_id}/usage-summary` | GET | Get client usage summary from FlexPrice |

### Client Management

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/clients/` | POST | Create client (auto-syncs to FlexPrice) |

## Programmatic Usage

### In Python Code

```python
from sqlalchemy.orm import Session
from backend.services.pricing_service import get_pricing_service
from backend.database.models import Client, Agent, Call

# Initialize service
db: Session = ...  # Get from dependency injection
pricing_service = get_pricing_service(db)

# Sync a client to FlexPrice
client = db.query(Client).first()
flexprice_customer_id = pricing_service.sync_client_to_flexprice(client)

# Sync plans from FlexPrice
synced_plans = pricing_service.sync_all_plans_from_flexprice()

# Assign plan to agent
agent = db.query(Agent).first()
history = pricing_service.assign_plan_to_agent(
    agent=agent,
    flexprice_plan_id="plan_abc123",
    changed_by_user_id=1,
    comments="Initial assignment"
)

# Calculate call cost
call = db.query(Call).first()
usage_metrics = {
    "voice_minutes": 5.5,
    "api_calls": 10
}
cost = pricing_service.finalize_call_cost(
    call=call,
    usage_metrics=usage_metrics,
    send_to_flexprice=True
)

print(f"Call cost: ${cost}")
```

### Direct FlexPrice API Usage

```python
from backend.utils.flexprice_client import get_flexprice_client

client = get_flexprice_client()

# Create customer
customer = client.create_customer(
    name="Acme Corp",
    email="billing@acme.com",
    external_id="client_123"
)

# List plans
plans = client.list_plans()

# Get plan details
plan = client.get_plan("plan_abc123")

# Send usage event
client.send_usage_event(
    customer_id="cust_abc123",
    event_name="voice_minutes",
    quantity=5.5,
    idempotency_key="call_xyz_voice_minutes",
    metadata={"call_id": "call_xyz"}
)
```

## Cost Calculation Logic

The `calculate_call_cost()` method processes usage metrics against the pricing plan:

```python
def calculate_call_cost(call: Call, usage_metrics: Dict[str, float]) -> float:
    # 1. Retrieve plan from FlexPrice (cached locally)
    plan = get_plan_details(call.flexprice_plan_id)
    
    # 2. Process each charge in the plan
    total_cost = 0.0
    for charge in plan['charges']:
        if charge['type'] == 'usage':
            # Usage-based: rate * quantity
            metric_value = usage_metrics.get(charge['usage_metric'], 0.0)
            total_cost += metric_value * charge['rate']
        
        elif charge['type'] == 'fixed':
            # Fixed charges
            total_cost += charge['amount']
    
    # 3. Store cost in call record
    call.call_cost = total_cost
    
    return total_cost
```

**Example:**

Plan charges:
```json
{
  "charges": [
    {
      "type": "usage",
      "name": "Voice Minutes",
      "usage_metric": "voice_minutes",
      "rate": 0.05
    },
    {
      "type": "usage",
      "name": "API Calls",
      "usage_metric": "api_calls",
      "rate": 0.01
    },
    {
      "type": "fixed",
      "name": "Monthly Fee",
      "amount": 10.00
    }
  ]
}
```

Usage metrics:
```json
{
  "voice_minutes": 5.5,
  "api_calls": 10
}
```

Calculation:
```
voice_minutes: 5.5 × $0.05 = $0.275
api_calls:     10 × $0.01   = $0.10
fixed_fee:                    $10.00
                            --------
Total:                       $10.375 → $10.38
```

## Testing

### Run Integration Test

```bash
docker exec backend_vaani_shailesh bash -c \
  "export FLEXPRICE_API_KEY='sk_01KBCH6A8A6VNMYKA5M40P9X7J' && \
   python -m backend.utils.test_flexprice"
```

Expected output:
```
================================================================================
FlexPrice API Integration Test
================================================================================
✓ FlexPrice client initialized successfully
✓ Found N plans
✓ Retrieved plan: Premium Plan
✓ Found N customers
✓ Created test customer
✓ FlexPrice Integration Test Complete!
```

## Error Handling

### FlexPriceAPIError

All FlexPrice API errors are wrapped in `FlexPriceAPIError`:

```python
from backend.utils.flexprice_client import FlexPriceAPIError

try:
    customer = client.create_customer(...)
except FlexPriceAPIError as e:
    print(f"Status: {e.status_code}")
    print(f"Message: {e.message}")
    print(f"Response: {e.response_data}")
```

### Retry Logic

The client automatically retries failed requests:
- **Retry conditions**: HTTP 429, 500, 502, 503, 504
- **Max retries**: 3
- **Backoff**: Exponential (1s, 2s, 4s)

### Graceful Degradation

If FlexPrice is unavailable:
- Client creation continues (FlexPrice sync happens in background)
- Cost calculation uses local plan cache
- Usage events are logged but don't block operations

## Best Practices

### 1. Always Use the Service Layer

❌ **Don't:**
```python
from backend.utils.flexprice_client import get_flexprice_client
client = get_flexprice_client()
client.create_customer(...)  # Direct API call
```

✅ **Do:**
```python
from backend.services.pricing_service import get_pricing_service
service = get_pricing_service(db)
service.sync_client_to_flexprice(client)  # Handles DB updates
```

### 2. Assign Plans Before Calls

Ensure agents have pricing plans assigned **before** they start taking calls:

```python
# At agent creation/configuration
pricing_service.assign_plan_to_agent(
    agent=agent,
    flexprice_plan_id="plan_default",
    comments="Initial setup"
)
```

### 3. Snapshot Pricing at Call Start

When a call starts, copy the agent's current plan to the call:

```python
call.flexprice_plan_id = agent.flexprice_plan_id
db.commit()
```

This ensures the call is billed at the price in effect when it started, even if the agent's plan changes during the call.

### 4. Finalize Costs After Call Completion

Only finalize costs after the call has ended:

```python
if call.call_status in ["completed", "ended"]:
    usage_metrics = {
        "voice_minutes": call.duration_seconds / 60.0,
        "api_calls": call.api_call_count
    }
    pricing_service.finalize_call_cost(call, usage_metrics)
```

### 5. Use Idempotency Keys

When sending usage events, always use idempotency keys to prevent duplicate billing:

```python
client.send_usage_event(
    customer_id="cust_123",
    event_name="voice_minutes",
    quantity=5.5,
    idempotency_key=f"{call_id}_voice_minutes"  # Unique per call + metric
)
```

### 6. Sync Plans Regularly

Schedule periodic sync of plans from FlexPrice (e.g., daily cron job):

```python
# In a scheduled task
pricing_service.sync_all_plans_from_flexprice()
```

## Monitoring & Logging

### Log Messages

The integration logs key events:

```
INFO: Created FlexPrice customer cust_abc123 for client uuid
INFO: Synced 5 plans from FlexPrice
INFO: Assigned plan plan_abc123 to agent uuid
INFO: Calculated call uuid cost: $2.75
INFO: Sent usage event to FlexPrice: voice_minutes=5.5 for customer cust_abc123
ERROR: Failed to sync client uuid to FlexPrice: API error 500
```

### Metrics to Monitor

- FlexPrice API response times
- API error rates
- Failed usage event submissions
- Clients without FlexPrice customer IDs
- Agents without pricing plans
- Calls without cost calculations

## Troubleshooting

### Issue: "FLEXPRICE_API_KEY not found"

**Solution:** Ensure `.env` has the API key and restart the backend:
```bash
echo 'FLEXPRICE_API_KEY="sk_01KBCH6A8A6VNMYKA5M40P9X7J"' >> .env
docker-compose restart backend
```

### Issue: "FlexPrice API error: 400 - Invalid request format"

**Cause:** Request payload doesn't match FlexPrice API schema.

**Solution:** Check FlexPrice API documentation for correct field types. Common issues:
- `metadata` must be string (JSON string), not object
- Required fields missing
- Invalid field values

### Issue: Client has no flexprice_customer_id

**Solution:** Manually sync the client:
```python
pricing_service.sync_client_to_flexprice(client)
```

### Issue: Call cost is $0.00

**Possible causes:**
1. Call has no pricing plan assigned → Check `call.flexprice_plan_id`
2. Plan has no charges → Verify plan in FlexPrice dashboard
3. Usage metrics don't match plan metrics → Check metric names

**Debug:**
```python
plan = pricing_service.get_plan_details(call.flexprice_plan_id)
print(f"Plan charges: {plan.get('charges')}")
print(f"Usage metrics: {usage_metrics}")
```

## Migration Guide

### For Existing Systems

If you have existing clients/agents/calls:

1. **Sync existing clients to FlexPrice:**
```python
from backend.database.models import Client
from backend.services.pricing_service import get_pricing_service

clients = db.query(Client).all()
for client in clients:
    try:
        pricing_service.sync_client_to_flexprice(client)
    except Exception as e:
        logger.error(f"Failed to sync client {client.id}: {e}")
```

2. **Sync plans from FlexPrice:**
```bash
curl -X POST http://localhost:8000/api/pricing/sync-from-flexprice
```

3. **Assign default plan to all agents:**
```python
from backend.database.models import Agent

default_plan_id = "plan_default"
agents = db.query(Agent).all()
for agent in agents:
    if not agent.flexprice_plan_id:
        pricing_service.assign_plan_to_agent(
            agent=agent,
            flexprice_plan_id=default_plan_id,
            comments="Migration: default plan assignment"
        )
```

4. **Backfill call costs (optional):**
```python
from backend.database.models import Call

calls = db.query(Call).filter(Call.call_cost == None).all()
for call in calls:
    if call.flexprice_plan_id and call.call_status == "completed":
        usage_metrics = extract_usage_from_call(call)  # Your logic
        pricing_service.finalize_call_cost(call, usage_metrics, send_to_flexprice=False)
```

## Future Enhancements

### Planned Features

1. **Webhook Support**: Receive billing events from FlexPrice
2. **Subscription Management**: Auto-create subscriptions for clients
3. **Credit Management**: Track and apply credits/discounts
4. **Invoice Integration**: Sync invoices to accounting systems
5. **Plan Recommendations**: AI-based plan suggestions for clients
6. **Usage Forecasting**: Predict monthly costs based on usage trends

### Contributing

To add new FlexPrice features:

1. Add methods to `FlexPriceClient` (API layer)
2. Add business logic to `PricingService` (service layer)
3. Add endpoints to pricing router (API layer)
4. Update documentation
5. Add tests

## Support

For issues with:
- **FlexPrice API**: Contact FlexPrice support or check their API docs
- **Integration code**: Check logs, review this documentation, or contact dev team

---

**Last Updated**: December 2, 2025  
**Version**: 1.0.0
