# FleetOps V2 — Amazon Bedrock Fleet Maintenance Advisor

The Fleet Maintenance Advisor is a live, production feature that analyses all active fleet vehicles and generates AI-powered maintenance recommendations for fleet managers. It runs inside `fleetops-vehicle-service` and calls Amazon Bedrock via the Converse API.

---

## What It Does

When a fleet manager opens the AI Advisor panel in the dashboard, the system:

1. Queries all `ACTIVE` vehicles from the database
2. Filters to vehicles with service overdue, service due soon, or insurance expiring within 30 days
3. Builds a structured prompt with exact numbers (km overdue, days overdue, insurance expiry)
4. Calls Amazon Bedrock (Claude 3 Haiku) via the Converse API
5. Parses the JSON response into structured recommendations
6. Returns a fleet health score (0–100) + per-vehicle recommendations with priority and reasoning
7. Caches the result for 15 minutes (Spring Cache → Redis) to avoid redundant Bedrock calls
8. Logs every invocation to an audit table (`ai_analysis_audit`)

---

## Architecture

```
Fleet Manager (Browser)
        │
        │ GET /api/vehicles/ai/analyse
        ▼
fleetops-vehicle-service (EKS pod)
        │
        │ 1. Query vehicle_db for all ACTIVE vehicles
        │ 2. Filter: service overdue OR insurance expiring
        │ 3. Build structured prompt with real numbers
        │
        ▼
Amazon Bedrock (Converse API)
Model: amazon.nova-lite-v1:0 (or Claude 3 Haiku)
Region: us-east-1
        │
        │ JSON response: fleetHealthScore + recommendations[]
        ▼
Parse → Cache (Redis, 15 min TTL) → Return to frontend
        │
        ▼
Audit log written to ai_analysis_audit table
```

**Authentication:** The vehicle-service pod uses IRSA (`fleetops-prod-app-irsa-role`) which has `bedrock:InvokeModel` permission. No static credentials needed.

---

## Implementation

### Service: FleetAiService.java

Located at: `fleetops-vehicle-service/src/main/java/com/fleetops/vehicle/service/FleetAiService.java`

Key methods:

**`analyseFleet(String requestedBy)`** — public entry point, always writes audit log regardless of cache state

**`invokeBedrockCached()`** — `@Cacheable` inner method; calls Bedrock only on cache miss; cached for 15 minutes under key `fleetAnalysis::current`

**`buildPrompt()`** — constructs the prompt with:
- Fleet overview (total vehicles, service alert count, insurance alert count)
- Per-vehicle detail: current mileage, km until/overdue service, days until/overdue service, insurance status

### Prompt Design

The prompt tells Bedrock to return **only valid JSON** in this shape:

```json
{
  "fleetHealthScore": 72,
  "summary": "2 sentences: overall fleet status and most urgent risk",
  "recommendations": [
    {
      "vehicleId": 3,
      "vehicleNumber": "TN-01-AB-1234",
      "priority": "HIGH",
      "taskType": "ROUTINE_SERVICE",
      "action": "Short imperative title (max 70 chars)",
      "reasoning": "2-3 sentences: specific problem with numbers, operational/financial risk if ignored, recommended urgency",
      "confidence": 85
    }
  ]
}
```

Each vehicle entry in the prompt includes precise data:
```
- TN-01-AB-1234 (id=3): current=52400 km, OVERDUE by 400 km, service date OVERDUE by 12 days, insurance expires in 18 days (2026-07-09)
```

### Bedrock Client Configuration

```java
// BedrockConfig.java
BedrockRuntimeClient.builder()
    .region(Region.of(region))
    .build()  // credentials from IRSA (instance metadata)

// FleetAiService.java
bedrockClient.converse(
    ConverseRequest.builder()
        .modelId(modelId)  // from application.yml: bedrock.model-id
        .messages(Message.builder()
            .role(ConversationRole.USER)
            .content(ContentBlock.fromText(prompt))
            .build())
        .inferenceConfig(InferenceConfiguration.builder()
            .maxTokens(1200)
            .build())
        .build()
)
```

### Caching

```java
@Cacheable(value = "fleetAnalysis", key = "'current'")
public FleetAnalysisResponse invokeBedrockCached() { ... }
```

Cache is backed by Redis (ElastiCache). TTL is configured in `application.yml`:
```yaml
spring:
  cache:
    redis:
      time-to-live: 900000  # 15 minutes in ms
```

The outer `analyseFleet()` method is NOT cached — this ensures the audit log is written on every request even when the response is served from cache.

### Audit Logging

Every call writes to `ai_analysis_audit`:

| Column | Value |
|---|---|
| `requested_by` | Username from JWT |
| `requested_at` | Timestamp |
| `model` | bedrock.model-id |
| `fleet_health_score` | Score from response |
| `recommendation_count` | Number of vehicles with recommendations |
| `execution_time_ms` | End-to-end latency |

---

## IAM Permissions

The app IRSA role (`fleetops-prod-app-irsa-role`) has this Bedrock permission:

```json
{
  "Effect": "Allow",
  "Action": ["bedrock:InvokeModel"],
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/*"
}
```

Defined in `fleetops-terraform/modules/iam/main.tf`.

---

## API Endpoint

```
GET /api/vehicles/ai/analyse
Authorization: Bearer <JWT>
Roles: ROLE_ADMIN, ROLE_MANAGER

Response:
{
  "fleetHealthScore": 68,
  "summary": "3 of 8 active vehicles require immediate attention...",
  "recommendations": [ ... ]
}
```

---

## Frontend Integration

The React dashboard has an **AI Advisor** panel that calls this endpoint and renders:
- A circular fleet health score gauge
- A card per recommendation with priority badge (HIGH/MEDIUM/LOW), action title, and detailed reasoning
- Confidence score per recommendation

The panel is accessible to ADMIN and MANAGER roles only.

---

## Cost Considerations

Amazon Bedrock charges per input/output token. With the 15-minute cache:
- A fleet of 20 vehicles generates ~400 input tokens per unique request
- With 10 managers checking the dashboard, the cache means ~4 Bedrock calls per hour max
- Cost is negligible (fractions of a cent per call for Claude Haiku)
