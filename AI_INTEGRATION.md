# 🤖 Amazon Bedrock AI Co-Pilot Blueprint

This blueprint outlines the system design, network routing, backend Java implementation, and frontend UI design for integrating **Amazon Bedrock (Claude 3.5 Sonnet / Claude 3 Haiku)**.

The AI Co-Pilot feature will analyze real-time vehicle mileage, sensor alerts, draft tasks, and maintenance history to provide:
1.  **Breakdown Risk Assessment:** Calculates breakdown likelihood (Low/Medium/High) based on vehicle history.
2.  **Diagnostic Recommendations:** Outlines step-by-step diagnostic and inspection procedures.
3.  **Required Parts & Cost Estimates:** Recommends specific replacement parts and cost forecasts.

---

## 🧭 Logical Architecture

```text
┌─────────────────┐      HTTP POST      ┌─────────────────┐      REST Call      ┌─────────────────┐
│   React Frontend│────────────────────►│   Nginx Gateway │────────────────────►│  Vehicle Service│
│ (AI Chat Panel) │   /api/ai/diagnose  │                 │    (New Endpoints)  │  (Aggregator)   │
└─────────────────┘                     └─────────────────┘                     └────────┬────────┘
                                                                                         │
   ┌─────────────────────────────────────────────────────────────────────────────────────┘
   │
   ▼   1. Fetch Vehicle Data (Self)
   ▼   2. Fetch Task Queue (REST call to Maintenance Service)
   ▼   3. Fetch Request Logs (REST call to Request Service)
   ▼
┌──────────────────────┐   Payload (JSON Prompt)   ┌──────────────────────┐
│  Vehicle Service     │──────────────────────────►│ Amazon Bedrock API   │
│  (AI Engine Module)  │   IAM Role Authorized     │ (Claude 3.5 Sonnet)  │
└──────────────────────┘                           └──────────┬───────────┘
                                                              │
                                                              ▼ JSON Reply
                                                   [Parsed Recommendations]
```

---

## 🔒 AWS Bedrock Permissions & Setup

### 1. Enable Model Access
Before calling the API, you must enable model access in the AWS Bedrock console:
*   Open the **AWS Bedrock Console** (ensure you are in an AI-supported region like `us-east-1` or `us-west-2`).
*   Select **Model Access** in the left navigation panel.
*   Request access for **Anthropic Claude 3 Haiku** (for fast, low-cost diagnostics) or **Claude 3.5 Sonnet** (for complex troubleshooting procedures).

### 2. Configure IAM Policy
Ensure the IAM role running the Backend Task has permission to invoke the Bedrock model:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-20240307-v1:0"
    }
  ]
}
```

---

## ☕ Backend Implementation (Spring Boot)

We will add the Bedrock client and AI routes directly to the **Vehicle Service** since it maintains the core vehicle profiles.

### 1. Add Maven Dependencies (`pom.xml`)
Add the official AWS Software Development Kit (SDK) dependencies to `fleetops-vehicle-service/pom.xml`:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>software.amazon.awssdk</groupId>
            <artifactId>bom</artifactId>
            <version>2.25.15</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- AWS Bedrock Runtime Client -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>bedrockruntime</artifactId>
    </dependency>
</dependencies>
```

### 2. Configure Bedrock Client (`AiConfig.java`)
Create `com.fleetops.vehicle.config.AiConfig.java` to initialize the client:
```java
package com.fleetops.vehicle.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;

@Configuration
public class AiConfig {

    @Value("${aws.region:us-east-1}")
    private String awsRegion;

    @Bean
    public BedrockRuntimeClient bedrockRuntimeClient() {
        return BedrockRuntimeClient.builder()
                .region(Region.of(awsRegion))
                .build();
    }
}
```

### 3. Implement the Diagnostic Engine (`AiService.java`)
Create `com.fleetops.vehicle.service.AiService.java` to fetch system metrics, compile the context prompt, invoke Claude, and parse the output:
```java
package com.fleetops.vehicle.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fleetops.vehicle.entity.Vehicle;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.SdkBytes;
import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;
import software.amazon.awssdk.services.bedrockruntime.model.InvokeModelRequest;
import software.amazon.awssdk.services.bedrockruntime.model.InvokeModelResponse;

import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Map;

@Service
public class AiService {

    private final BedrockRuntimeClient bedrockClient;
    private final VehicleService vehicleService;
    private final ObjectMapper objectMapper;

    public AiService(BedrockRuntimeClient bedrockClient, VehicleService vehicleService, ObjectMapper objectMapper) {
        this.bedrockClient = bedrockClient;
        this.vehicleService = vehicleService;
        this.objectMapper = objectMapper;
    }

    public String generateDiagnostics(Long vehicleId, List<Map<String, Object>> requestHistory, List<Map<String, Object>> pendingTasks) {
        Vehicle vehicle = vehicleService.getVehicleById(vehicleId)
                .orElseThrow(() -> new IllegalArgumentException("Vehicle not found."));

        // Compile vehicle profile context
        String contextPrompt = String.format(
            "You are the FleetOps AI Diagnostics Engineer. Analyze this vehicle and output a structured JSON report.\n\n" +
            "VEHICLE PROFILE:\n" +
            "- Brand/Model: %s %s\n" +
            "- Type: %s\n" +
            "- Mileage: %d km\n" +
            "- Last Service Date: %s\n" +
            "- Next Service Mileage Limit: %d km\n" +
            "- Next Service Date Limit: %s\n" +
            "- Current Status: %s\n\n" +
            "PENDING TASKS DRAFTED IN QUEUE:\n%s\n\n" +
            "HISTORICAL MAINTENANCE WORKFLOW RECORD:\n%s\n\n" +
            "CRITICAL INSTRUCTIONS:\n" +
            "Generate a JSON object containing exactly these fields:\n" +
            "1. 'riskLevel': ('LOW', 'MEDIUM', 'HIGH')\n" +
            "2. 'riskAnalysis': 'Short analytical paragraph explaining the risk calculation.'\n" +
            "3. 'inspectionSteps': List of string steps to troubleshoot the current status.\n" +
            "4. 'estimatedParts': List of objects containing 'name', 'estimatedCostUsd', 'urgency' ('IMMEDIATE', 'DEFERRED').\n" +
            "Do not include any chat prefix or markdown wrappers (like ```json). Respond with the raw JSON string only.",
            vehicle.getBrand(), vehicle.getModel(), vehicle.getType(), vehicle.getCurrentMileage(),
            vehicle.getLastServiceDate(), vehicle.getNextServiceMileage(), vehicle.getNextServiceDate(), vehicle.getStatus(),
            pendingTasks.toString(), requestHistory.toString()
        );

        try {
            // Build Bedrock Claude message request payload
            ObjectNode payload = objectMapper.createObjectNode();
            payload.put("anthropic_version", "bedrock-2023-05-31");
            payload.put("max_tokens", 1000);
            payload.put("temperature", 0.2); // Low temperature for consistent JSON reports

            var messageArray = payload.putArray("messages");
            var userMessage = messageArray.addObject();
            userMessage.put("role", "user");
            userMessage.put("content", contextPrompt);

            String requestBody = objectMapper.writeValueAsString(payload);

            InvokeModelRequest request = InvokeModelRequest.builder()
                    .modelId("anthropic.claude-3-haiku-20240307-v1:0")
                    .contentType("application/json")
                    .accept("application/json")
                    .body(SdkBytes.fromUtf8String(requestBody))
                    .build();

            InvokeModelResponse response = bedrockClient.invokeModel(request);
            String responseBody = response.body().asString(StandardCharsets.UTF_8);

            // Parse response content out of Anthropic wrapping
            Map<String, Object> map = objectMapper.readValue(responseBody, Map.class);
            List<Map<String, Object>> contentList = (List<Map<String, Object>>) map.get("content");
            if (contentList != null && !contentList.isEmpty()) {
                return (String) contentList.get(0).get("text");
            }
            return "{ \"error\": \"No response text returned from Claude.\" }";

        } catch (Exception e) {
            return String.format("{ \"error\": \"AI service execution failed: %s\" }", e.getMessage());
        }
    }
}
```

### 4. Create the Controller Endpoint (`AiController.java`)
Create `com.fleetops.vehicle.controller.AiController.java` to expose the diagnostics route:
```java
package com.fleetops.vehicle.controller;

import com.fleetops.vehicle.service.AiService;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/ai")
public class AiController {

    private final AiService aiService;
    private final RestTemplate restTemplate;

    public AiController(AiService aiService, RestTemplate restTemplate) {
        this.aiService = aiService;
        this.restTemplate = restTemplate;
    }

    @GetMapping("/diagnose/{vehicleId}")
    @PreAuthorize("hasAnyRole('DRIVER', 'MANAGER', 'ADMIN')")
    public ResponseEntity<String> getDiagnostics(
            @PathVariable Long vehicleId,
            @RequestHeader("Authorization") String token) {

        // REST calls to extract context from sibling microservices
        List<Map<String, Object>> history = new ArrayList<>();
        try {
            String requestUrl = "http://request-service:8080/api/requests/vehicle/" + vehicleId;
            history = restTemplate.getForObject(requestUrl, List.class);
        } catch (Exception e) {
            // Log fallback - proceed with partial data if sibling is temporarily offline
        }

        List<Map<String, Object>> tasks = new ArrayList<>();
        try {
            String taskUrl = "http://maintenance-service:8080/api/tasks";
            tasks = restTemplate.getForObject(taskUrl, List.class);
        } catch (Exception e) {
            // Log fallback
        }

        String report = aiService.generateDiagnostics(vehicleId, history, tasks);
        return ResponseEntity.ok()
                .header("Content-Type", "application/json")
                .body(report);
    }
}
```

---

## 🎨 Frontend UI Design (React)

Add an **AI Diagnostics Panel** component to the fleet dashboard. It renders loading skeletons, risk indicators, and parts estimates cleanly.

### React Component (`AiDiagnosticsPanel.jsx`)
```jsx
import React, { useState } from 'react';
import axios from 'axios';
import './AiDiagnosticsPanel.css';

export default function AiDiagnosticsPanel({ vehicleId, vehicleName }) {
  const [loading, setLoading] = useState(false);
  const [report, setReport] = useState(null);
  const [error, setError] = useState('');

  const triggerDiagnosis = async () => {
    setLoading(true);
    setError('');
    setReport(null);
    try {
      const token = localStorage.getItem('token');
      const res = await axios.get(`/api/ai/diagnose/${vehicleId}`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setReport(res.data);
    } catch (err) {
      setError(err.response?.data?.error || 'Failed to complete AI diagnostic check.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="ai-diagnostic-card">
      <div className="ai-card-header">
        <h4>🤖 FleetOps AI Co-Pilot</h4>
        <button onClick={triggerDiagnosis} disabled={loading} className="btn-ai-trigger">
          {loading ? 'Analyzing Data...' : `Diagnose ${vehicleName}`}
        </button>
      </div>

      {loading && (
        <div className="ai-skeleton-loader">
          <div className="skeleton-line risk-meter"></div>
          <div className="skeleton-line paragraph"></div>
          <div className="skeleton-line paragraph"></div>
        </div>
      )}

      {error && <div className="ai-error-banner">⚠️ {error}</div>}

      {report && (
        <div className="ai-report-body">
          <div className={`ai-risk-badge risk-${report.riskLevel.toLowerCase()}`}>
            Breakdown Risk: {report.riskLevel}
          </div>
          
          <p className="ai-analysis-text">{report.riskAnalysis}</p>

          <div className="ai-section">
            <h5>🔍 Recommended Inspection Steps:</h5>
            <ol>
              {report.inspectionSteps.map((step, idx) => (
                <li key={idx}>{step}</li>
              ))}
            </ol>
          </div>

          {report.estimatedParts && report.estimatedParts.length > 0 && (
            <div className="ai-section">
              <h5>🔧 Required Parts Estimate:</h5>
              <table className="ai-parts-table">
                <thead>
                  <tr>
                    <th>Part Name</th>
                    <th>Est. Cost</th>
                    <th>Urgency</th>
                  </tr>
                </thead>
                <tbody>
                  {report.estimatedParts.map((part, idx) => (
                    <tr key={idx}>
                      <td>{part.name}</td>
                      <td>${part.estimatedCostUsd}</td>
                      <td>
                        <span className={`urgency-${part.urgency.toLowerCase()}`}>
                          {part.urgency}
                        </span>
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

### CSS Styles (`AiDiagnosticsPanel.css`)
```css
.ai-diagnostic-card {
  background: rgba(30, 41, 59, 0.7);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  padding: 20px;
  margin: 15px 0;
  box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.3);
}

.ai-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.btn-ai-trigger {
  background: linear-gradient(135deg, #a855f7 0%, #6366f1 100%);
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.2s ease, opacity 0.2s ease;
}

.btn-ai-trigger:hover:not(:disabled) {
  transform: scale(1.03);
  opacity: 0.95;
}

.btn-ai-trigger:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.ai-risk-badge {
  display: inline-block;
  padding: 6px 12px;
  border-radius: 20px;
  font-weight: 700;
  font-size: 0.85rem;
  margin-bottom: 15px;
}

.risk-high { background: rgba(239, 68, 68, 0.25); color: #ef4444; border: 1px solid #ef4444; }
.risk-medium { background: rgba(245, 158, 11 0.25); color: #f59e0b; border: 1px solid #f59e0b; }
.risk-low { background: rgba(16, 185, 129, 0.25); color: #10b981; border: 1px solid #10b981; }

.ai-analysis-text {
  color: #e2e8f0;
  line-height: 1.6;
  font-size: 0.95rem;
  margin-bottom: 20px;
}

.ai-section h5 {
  color: #94a3b8;
  margin-bottom: 8px;
}

.ai-section ol {
  color: #cbd5e1;
  padding-left: 20px;
  font-size: 0.9rem;
  line-height: 1.5;
}

.ai-parts-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 10px;
  font-size: 0.85rem;
}

.ai-parts-table th, .ai-parts-table td {
  padding: 8px 12px;
  text-align: left;
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
}

.ai-parts-table th { color: #94a3b8; }
.ai-parts-table td { color: #cbd5e1; }

.urgency-immediate { color: #f87171; font-weight: 600; }
.urgency-deferred { color: #94a3b8; }

/* Skeleton animation */
.ai-skeleton-loader {
  animation: pulse 1.5s infinite ease-in-out;
}

.skeleton-line {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 4px;
  margin-bottom: 10px;
}

.skeleton-line.risk-meter { width: 150px; height: 28px; }
.skeleton-line.paragraph { width: 100%; height: 16px; }

@keyframes pulse {
  0%, 100% { opacity: 0.6; }
  50% { opacity: 1; }
}
```

---
*End of Blueprint. Systems Ready for AWS Deployment & AI Module Integration.*
