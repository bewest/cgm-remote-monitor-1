# RFC: Model Context Protocol (MCP) Support for Nightscout

**Status:** Draft  
**Authors:** Nightscout Community  
**Created:** 2026-01-18  
**Last Updated:** 2026-01-18  
**Related:** [Agent Control Plane RFC](./agent-control-plane-rfc.md)

---

## Abstract

This RFC proposes adding **Model Context Protocol (MCP)** support to Nightscout to enable standardized AI agent integration. MCP provides a universal interface for AI models to securely access Nightscout's glucose monitoring data, treatment information, and automation controls, enabling intelligent assistants to help with diabetes management while maintaining safety and auditability.

This proposal aligns with and extends the [Agent Control Plane RFC](./agent-control-plane-rfc.md) by providing a standardized protocol layer for AI agents to interact with Nightscout's control plane.

---

## Table of Contents

1. [Motivation](#motivation)
2. [What is MCP?](#what-is-mcp)
3. [Integration with Agent Control Plane](#integration-with-agent-control-plane)
4. [Architecture Overview](#architecture-overview)
5. [MCP Server Design](#mcp-server-design)
6. [Resources](#resources)
7. [Tools](#tools)
8. [Prompts](#prompts)
9. [Security & Safety Model](#security--safety-model)
10. [Implementation Phases](#implementation-phases)
11. [Use Cases](#use-cases)
12. [Alignment with Agent Control Plane](#alignment-with-agent-control-plane)
13. [Future Extensions](#future-extensions)
14. [References](#references)

---

## Motivation

### Current Challenges

1. **Fragmented AI Integration** — Each AI assistant (ChatGPT, Claude, local LLMs) requires custom integration
2. **No Standard Protocol** — No universal way for AI models to safely access diabetes data
3. **Context Sharing Complexity** — Difficult to provide AI models with relevant historical context
4. **Safety Concerns** — Lack of standardized permissions and audit trails for AI actions
5. **Limited Discoverability** — AI models can't discover available Nightscout capabilities

### Why MCP?

The Model Context Protocol (MCP) provides:

- **Standardized Interface** — One protocol works with all MCP-compatible AI clients
- **Built-in Security** — OAuth-based authentication with granular permissions
- **Context Awareness** — Structured resources that AI models can understand
- **Tool Invocation** — Safe, auditable function calling for AI agents
- **Progressive Disclosure** — AI models discover capabilities dynamically
- **Stateful Sessions** — Maintain context across multiple interactions

### Benefits for Nightscout Users

1. **Natural Language Queries** — "What was my average glucose last week?"
2. **Intelligent Insights** — AI-powered pattern recognition and recommendations
3. **Automated Assistance** — AI agents can help with logging, analysis, and adjustments
4. **Multi-Platform Support** — Works with desktop, mobile, and voice assistants
5. **Safe Delegation** — Controlled AI access with human oversight
6. **Enhanced Caregiving** — AI-assisted remote monitoring and intervention

---

## What is MCP?

The **Model Context Protocol** is an open standard that enables AI models to:

1. **Discover Resources** — Browse available data (glucose readings, treatments, profiles)
2. **Query Context** — Request specific information with structured queries
3. **Invoke Tools** — Execute actions (log treatments, adjust settings, create reports)
4. **Use Prompts** — Leverage pre-defined templates for common tasks
5. **Maintain State** — Track conversation context and user preferences

### MCP Core Concepts

```
┌─────────────────────────────────────────────────────────┐
│                    AI CLIENT (MCP)                       │
│           (Claude Desktop, ChatGPT, etc.)                │
└─────────────────────────────────────────────────────────┘
                            │
                            │ MCP Protocol (JSON-RPC)
                            │
┌─────────────────────────────────────────────────────────┐
│                  MCP SERVER (Nightscout)                 │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  RESOURCES   │  │    TOOLS     │  │   PROMPTS    │  │
│  │              │  │              │  │              │  │
│  │ • Glucose    │  │ • Log Treat- │  │ • Daily      │  │
│  │   Readings   │  │   ment       │  │   Summary    │  │
│  │ • Treatments │  │ • Adjust     │  │ • Pattern    │  │
│  │ • Profiles   │  │   Override   │  │   Analysis   │  │
│  │ • Device     │  │ • Query      │  │ • Bolus Calc │  │
│  │   Status     │  │   Stats      │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│              NIGHTSCOUT CONTROL PLANE                    │
│    (Agent Control Plane RFC Implementation)              │
└─────────────────────────────────────────────────────────┘
```

---

## Integration with Agent Control Plane

The MCP Server acts as a **protocol adapter** for AI agents to interact with Nightscout's control plane.

### Relationship Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        MCP LAYER                             │
│  (Protocol Translation & AI-Specific Logic)                  │
│                                                              │
│  • Natural language understanding                            │
│  • Context formatting for AI models                          │
│  • Tool schema generation                                    │
│  • Session management                                        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   AGENT CONTROL PLANE                        │
│  (Business Logic & Domain Model)                             │
│                                                              │
│  • Authority & permissions                                   │
│  • Event sourcing                                            │
│  • Conflict resolution                                       │
│  • Audit trail                                               │
│  • Delivery tracking                                         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     NIGHTSCOUT API v3                        │
│  (REST API & Data Access)                                    │
└─────────────────────────────────────────────────────────────┘
```

### MCP as an Issuer

In the Agent Control Plane model, MCP clients act as **Agent issuers**:

```yaml
Issuer:
  issuerType: "agent"
  issuerId: "mcp-client:claude-desktop:user-123"
  authority: "delegated"
  
  delegationGrant:
    grantedBy: "user-123"
    scopes: 
      - "entries.read"
      - "override.suggest"
      - "treatments.log"
    constraints:
      requireConfirmation: true
      maxOverrideDuration: 7200  # 2 hours
```

---

## Architecture Overview

### Components

#### 1. MCP Server Module

Location: `/lib/mcp-server/`

- **Server Implementation** — JSON-RPC 2.0 over stdio/HTTP
- **Resource Handlers** — Map Nightscout data to MCP resources
- **Tool Handlers** — Implement safe AI-callable functions
- **Prompt Templates** — Pre-defined AI interaction patterns
- **Session Manager** — Track AI conversation state
- **Security Layer** — OAuth, scoping, rate limiting

#### 2. MCP Transport Adapters

- **stdio Transport** — For local AI clients (Claude Desktop)
- **HTTP/SSE Transport** — For web-based AI clients
- **WebSocket Transport** — For real-time AI interactions

#### 3. Integration Points

- **Event Stream Bridge** — Subscribe to control plane events
- **Authority Mapper** — Translate MCP authentication to agent identity
- **Capability Resolver** — Map AI capabilities to Nightscout features

---

## MCP Server Design

### Server Metadata

```json
{
  "name": "nightscout",
  "version": "1.0.0",
  "description": "Nightscout CGM Remote Monitor - AI-powered diabetes management",
  "capabilities": {
    "resources": true,
    "tools": true,
    "prompts": true,
    "logging": true
  },
  "protocolVersion": "2024-11-05"
}
```

---

## Resources

MCP Resources represent **data that can be read** by AI models.

### Resource Types

#### 1. `nightscout://entries/{timeRange}`

Glucose (CGM) readings.

```json
{
  "uri": "nightscout://entries/today",
  "mimeType": "application/json",
  "name": "Today's Glucose Readings",
  "description": "All CGM entries from today",
  "metadata": {
    "count": 288,
    "avgGlucose": 145,
    "timeInRange": 68.5,
    "unit": "mg/dL"
  }
}
```

**URI Patterns:**
- `nightscout://entries/last24h`
- `nightscout://entries/last7d`
- `nightscout://entries/last30d`
- `nightscout://entries/range/{start}/{end}`
- `nightscout://entries/latest`

#### 2. `nightscout://treatments/{timeRange}`

Treatment events (insulin, carbs, corrections).

```json
{
  "uri": "nightscout://treatments/today",
  "mimeType": "application/json",
  "name": "Today's Treatments",
  "description": "All logged treatments from today",
  "metadata": {
    "count": 12,
    "totalInsulin": 24.5,
    "totalCarbs": 180
  }
}
```

#### 3. `nightscout://profile/current`

Active treatment profile.

```json
{
  "uri": "nightscout://profile/current",
  "mimeType": "application/json",
  "name": "Active Profile",
  "description": "Currently active treatment profile and settings"
}
```

#### 4. `nightscout://devicestatus/latest`

Current device and pump status.

```json
{
  "uri": "nightscout://devicestatus/latest",
  "mimeType": "application/json",
  "name": "Device Status",
  "description": "Current pump, CGM, and controller status"
}
```

#### 5. `nightscout://overrides/active`

Active override instances.

```json
{
  "uri": "nightscout://overrides/active",
  "mimeType": "application/json",
  "name": "Active Overrides",
  "description": "Currently active profile overrides"
}
```

#### 6. `nightscout://reports/{reportType}`

Generated reports and analytics.

**Report Types:**
- `daily-summary`
- `weekly-pattern`
- `time-in-range`
- `insulin-sensitivity`
- `carb-ratios`

---

## Tools

MCP Tools represent **actions that AI can invoke** (with permission).

### Tool Categories

#### Category 1: Read-Only Analytics Tools

##### `query_glucose_stats`

Calculate glucose statistics.

```json
{
  "name": "query_glucose_stats",
  "description": "Calculate glucose statistics for a time period",
  "inputSchema": {
    "type": "object",
    "properties": {
      "timeRange": {
        "type": "string",
        "enum": ["last24h", "last7d", "last14d", "last30d", "last90d"],
        "description": "Time period for analysis"
      },
      "metrics": {
        "type": "array",
        "items": {
          "type": "string",
          "enum": ["average", "median", "stdDev", "timeInRange", "timeAbove", "timeBelow", "hbA1cEstimate"]
        }
      }
    },
    "required": ["timeRange", "metrics"]
  }
}
```

##### `detect_patterns`

Identify glucose patterns.

```json
{
  "name": "detect_patterns",
  "description": "Detect recurring glucose patterns",
  "inputSchema": {
    "type": "object",
    "properties": {
      "patternType": {
        "type": "string",
        "enum": ["dawn-phenomenon", "dusk-phenomenon", "post-meal-spike", "overnight-low", "exercise-drop"]
      },
      "lookbackDays": {
        "type": "integer",
        "minimum": 1,
        "maximum": 90
      }
    },
    "required": ["patternType", "lookbackDays"]
  }
}
```

#### Category 2: Write Tools (Require Elevated Permission)

##### `log_treatment`

Log a treatment event.

```json
{
  "name": "log_treatment",
  "description": "Log a treatment (insulin, carbs, correction)",
  "inputSchema": {
    "type": "object",
    "properties": {
      "eventType": {
        "type": "string",
        "enum": ["Meal Bolus", "Snack Bolus", "Correction Bolus", "Carb Correction", "Temp Basal"]
      },
      "insulin": {
        "type": "number",
        "minimum": 0
      },
      "carbs": {
        "type": "number",
        "minimum": 0
      },
      "notes": {
        "type": "string"
      },
      "timestamp": {
        "type": "string",
        "format": "date-time"
      }
    },
    "required": ["eventType"]
  }
}
```

**Safety:** Requires `treatments.write` permission + human confirmation for insulin >5U.

##### `suggest_override`

Propose a profile override.

```json
{
  "name": "suggest_override",
  "description": "Suggest a profile override (requires human approval)",
  "inputSchema": {
    "type": "object",
    "properties": {
      "overrideType": {
        "type": "string",
        "enum": ["exercise", "preMeal", "illness", "highActivity", "hormones"]
      },
      "duration": {
        "type": "integer",
        "description": "Duration in seconds"
      },
      "targetAdjustment": {
        "type": "number",
        "description": "Target glucose adjustment in mg/dL"
      },
      "basalMultiplier": {
        "type": "number",
        "minimum": 0.1,
        "maximum": 2.0
      },
      "reason": {
        "type": "string",
        "description": "Explanation for the override suggestion"
      }
    },
    "required": ["overrideType", "reason"]
  }
}
```

**Safety:** Creates a `DeliveryRequest` with `authority: "agent"` that requires human approval.

##### `activate_override`

Directly activate a pre-approved override.

```json
{
  "name": "activate_override",
  "description": "Activate a pre-defined override (requires delegation grant)",
  "inputSchema": {
    "type": "object",
    "properties": {
      "definitionId": {
        "type": "string",
        "description": "ID of the override definition"
      },
      "duration": {
        "type": "integer",
        "description": "Duration in seconds (overrides default)"
      }
    },
    "required": ["definitionId"]
  }
}
```

**Safety:** Requires `override.activate` scope in delegation grant.

#### Category 3: Report Generation Tools

##### `generate_daily_summary`

Create a daily summary report.

```json
{
  "name": "generate_daily_summary",
  "description": "Generate a comprehensive daily summary",
  "inputSchema": {
    "type": "object",
    "properties": {
      "date": {
        "type": "string",
        "format": "date"
      },
      "includeInsights": {
        "type": "boolean",
        "description": "Include AI-generated insights"
      }
    },
    "required": ["date"]
  }
}
```

---

## Prompts

MCP Prompts are **pre-defined templates** for common AI interactions.

### Prompt Templates

#### 1. `daily_review`

```json
{
  "name": "daily_review",
  "description": "Analyze today's glucose data and provide insights",
  "arguments": [
    {
      "name": "focus_area",
      "description": "Specific area to focus on",
      "required": false,
      "enum": ["overnight", "post-meal", "exercise", "overall"]
    }
  ]
}
```

**Generated Prompt:**
```
You are a diabetes management assistant reviewing Nightscout data. Analyze today's glucose readings and treatments:

[Glucose Data]
[Treatment Data]
[Device Status]

Focus on: {focus_area}

Provide:
1. Time in range summary
2. Notable patterns or anomalies
3. Suggested adjustments (if any)
4. Positive observations

Keep responses concise and actionable.
```

#### 2. `bolus_advisor`

```json
{
  "name": "bolus_advisor",
  "description": "Help calculate a meal bolus",
  "arguments": [
    {
      "name": "carbs",
      "description": "Estimated carbs in meal",
      "required": true
    },
    {
      "name": "current_glucose",
      "description": "Current glucose reading (mg/dL)",
      "required": true
    }
  ]
}
```

**Note:** This is advisory only; actual bolus delivery requires human confirmation.

#### 3. `pattern_detective`

```json
{
  "name": "pattern_detective",
  "description": "Investigate recurring glucose patterns",
  "arguments": [
    {
      "name": "time_of_day",
      "description": "Time period to investigate",
      "required": false,
      "enum": ["morning", "afternoon", "evening", "overnight", "all"]
    },
    {
      "name": "days",
      "description": "Number of days to analyze",
      "required": false,
      "default": 14
    }
  ]
}
```

#### 4. `caregiver_update`

```json
{
  "name": "caregiver_update",
  "description": "Generate a caregiver-friendly status update",
  "arguments": [
    {
      "name": "recipient",
      "description": "Caregiver relationship",
      "required": false,
      "enum": ["parent", "spouse", "friend", "medical"]
    }
  ]
}
```

---

## Security & Safety Model

### Authentication Flow

```
┌─────────────┐
│  AI Client  │
└──────┬──────┘
       │
       │ 1. Request auth
       ▼
┌─────────────────────┐
│  MCP Server         │
│  (Nightscout)       │
└──────┬──────────────┘
       │
       │ 2. OAuth redirect
       ▼
┌─────────────────────┐
│  User Auth          │
│  (Nightscout Login) │
└──────┬──────────────┘
       │
       │ 3. Grant scopes
       ▼
┌─────────────────────┐
│  Delegation Grant   │
│  Created            │
└──────┬──────────────┘
       │
       │ 4. Return token
       ▼
┌─────────────┐
│  AI Client  │
│  (Authorized)
└─────────────┘
```

### Permission Scopes

```yaml
MCP Scopes:
  # Read-only scopes
  entries.read:
    description: "Read glucose entries"
    risk: "low"
  
  treatments.read:
    description: "Read treatment history"
    risk: "low"
  
  profile.read:
    description: "Read profile settings"
    risk: "low"
  
  # Analytical scopes
  analytics.run:
    description: "Run analytics and generate reports"
    risk: "low"
  
  # Write scopes (higher risk)
  treatments.log:
    description: "Log treatments (requires confirmation for insulin)"
    risk: "medium"
    constraints:
      - maxInsulin: 5.0
      - requireConfirmation: true
  
  override.suggest:
    description: "Suggest profile overrides (requires approval)"
    risk: "medium"
    constraints:
      - requireApproval: true
  
  override.activate:
    description: "Activate pre-defined overrides"
    risk: "high"
    constraints:
      - allowedDefinitionIds: ["exercise-preset", "pre-meal-preset"]
      - maxDuration: 7200
      - requireReason: true
```

### Safety Guardrails

#### 1. Rate Limiting

```yaml
RateLimits:
  perClient:
    requests: 100
    window: 60  # seconds
  
  perTool:
    log_treatment:
      calls: 10
      window: 3600  # 1 hour
    
    suggest_override:
      calls: 5
      window: 3600
    
    activate_override:
      calls: 3
      window: 3600
```

#### 2. Human Confirmation Requirements

AI tools that require human confirmation:

1. **Insulin logging** — Any insulin >5U
2. **Override suggestions** — All override suggestions (until trust established)
3. **Profile changes** — Any profile modifications
4. **Pump commands** — Direct pump control (future)

#### 3. Audit Trail

All MCP tool invocations are logged as events:

```yaml
EventEnvelope:
  eventType: "mcp.tool.invoked"
  issuer: "mcp-client:claude-desktop:user-123"
  payload:
    toolName: "suggest_override"
    arguments:
      overrideType: "exercise"
      duration: 3600
      reason: "User mentioned going for a run"
    outcome: "pending_approval"
    humanApproval: null
```

#### 4. Progressive Trust

```yaml
TrustLevel:
  initial:
    scopes: ["entries.read", "treatments.read", "analytics.run"]
    confirmationRequired: true
  
  after_10_successful_interactions:
    scopes: ["override.suggest"]
    confirmationRequired: true
  
  after_30_successful_interactions:
    scopes: ["treatments.log"]
    confirmationRequired: true  # for insulin >5U
  
  after_explicit_delegation:
    scopes: ["override.activate"]
    confirmationRequired: false  # for pre-approved overrides
```

---

## Implementation Phases

### Phase 1: Read-Only MCP Server (4 weeks)

**Goal:** Basic MCP server with read-only resources and analytics tools.

#### Deliverables:
- [ ] MCP server module (`/lib/mcp-server/`)
- [ ] stdio transport for local clients
- [ ] Resources: `entries`, `treatments`, `profile`, `devicestatus`
- [ ] Tools: `query_glucose_stats`, `detect_patterns`, `generate_daily_summary`
- [ ] Basic OAuth authentication
- [ ] Documentation and setup guide

#### Dependencies:
- None (works with existing Nightscout API)

#### Testing:
- Claude Desktop integration
- Glucose query examples
- Pattern detection validation

---

### Phase 2: Agent Control Plane Integration (4 weeks)

**Goal:** Integrate MCP with agent control plane for write operations.

#### Deliverables:
- [ ] MCP as agent issuer in control plane
- [ ] Delegation grant management UI
- [ ] Tools: `suggest_override`, `log_treatment`
- [ ] Event stream subscription for MCP
- [ ] Human confirmation workflows
- [ ] Audit dashboard for MCP actions

#### Dependencies:
- Agent Control Plane RFC Phase 1 (event model, collections)

#### Testing:
- Override suggestion flow
- Treatment logging with confirmation
- Multi-writer conflict scenarios

---

### Phase 3: Advanced Tools & Prompts (4 weeks)

**Goal:** Enhanced AI capabilities and prompt templates.

#### Deliverables:
- [ ] Tools: `activate_override`, `generate_report`
- [ ] Prompts: `daily_review`, `bolus_advisor`, `pattern_detective`, `caregiver_update`
- [ ] Progressive trust system
- [ ] Rate limiting and safety guardrails
- [ ] WebSocket transport for real-time interaction

#### Dependencies:
- Agent Control Plane RFC Phase 2 (delivery tracking, capabilities)

#### Testing:
- End-to-end AI workflows
- Safety guardrail validation
- Multi-platform client testing

---

### Phase 4: Advanced Features (6 weeks)

**Goal:** Production-ready with advanced capabilities.

#### Deliverables:
- [ ] HTTP/SSE transport for web clients
- [ ] Multi-language prompt support
- [ ] Custom prompt creation UI
- [ ] AI-powered insights and recommendations
- [ ] Integration with voice assistants
- [ ] Mobile app MCP support

#### Dependencies:
- Agent Control Plane RFC Phase 3 (agents & delegation)

#### Testing:
- Production deployment
- Performance benchmarking
- Security audit

---

## Use Cases

### 1. Daily Review Assistant

**User:** "Hey Claude, how did my glucose look yesterday?"

**MCP Flow:**
1. AI client requests `nightscout://entries/yesterday`
2. AI client calls `query_glucose_stats` with timeRange: "last24h"
3. AI analyzes data and responds with insights

**Response:**
```
Yesterday's glucose looked pretty good overall:

• Time in Range: 72% (target: 70-180 mg/dL)
• Average: 142 mg/dL
• Standard Deviation: 38 mg/dL

Notable observations:
- Excellent overnight control (8 hours flat between 95-110)
- Small spike after lunch (+80 mg/dL) - might try pre-bolusing 10min earlier
- Slight low at 4pm (68 mg/dL) after exercise

Overall: Solid day! The overnight control is particularly impressive.
```

---

### 2. Pattern Detective

**User:** "I keep waking up low. Can you help figure out why?"

**MCP Flow:**
1. AI calls `detect_patterns` with `patternType: "overnight-low"`, lookbackDays: 14
2. AI requests `nightscout://entries/range/{14daysago}/{today}` filtered for 2am-8am
3. AI requests `nightscout://treatments/range/{14daysago}/{today}` for evening insulin
4. AI analyzes correlation between evening insulin and morning lows

**Response:**
```
I analyzed your overnight glucose for the past 14 days. Here's what I found:

Pattern detected:
- You went low overnight (below 70) on 6 of the last 14 nights
- Average low time: 3:30am (range: 2:15am - 4:45am)
- Lowest reading: 52 mg/dL

Possible factors:
1. Dinner boluses: On nights you went low, your dinner bolus averaged 8.2U vs 6.5U on stable nights
2. Exercise timing: 4 of the 6 lows occurred after evening exercise (after 6pm)
3. Basal rate: Your 12am-6am basal is 1.1 U/hr

Suggestions:
1. Consider reducing dinner bolus by 10-15% on exercise days
2. Try a temporary -20% basal override when exercising after 6pm
3. Set a low alarm for 3am to catch early

Would you like me to create an "evening exercise" override preset?
```

---

### 3. Meal Bolus Advisor

**User:** "I'm about to eat pasta with 60g carbs. Current glucose is 145. What should I bolus?"

**MCP Flow:**
1. AI uses `bolus_advisor` prompt template
2. AI requests current profile (`nightscout://profile/current`) for ICR
3. AI calls `query_glucose_stats` for recent trends
4. AI calculates suggestion (for reference only)

**Response:**
```
Based on your current profile and glucose:

Carb Coverage:
- 60g carbs ÷ 12 (your IC ratio) = 5.0 U

Correction:
- Current: 145 mg/dL
- Target: 110 mg/dL
- ISF: 40 mg/dL per unit
- Correction: (145-110)/40 = 0.9 U

Total Suggested: 5.9 U (round to 6.0 U)

Notes:
- Pasta is higher glycemic index - consider a dual-wave/extended bolus if your pump supports it
- You might want to pre-bolus 10-15 minutes since you're starting elevated
- Your IOB is currently 0.3U from your last bolus 2 hours ago

⚠️ This is advisory only. Please confirm with your pump/CGM and consult your diabetes care team for personalized advice.

Would you like me to log this meal once you've confirmed the bolus?
```

---

### 4. Caregiver Remote Support

**User:** (Caregiver) "Can you give me an update on Sarah's day?"

**MCP Flow:**
1. AI uses `caregiver_update` prompt
2. AI requests `nightscout://entries/today` and `nightscout://treatments/today`
3. AI formats a parent-friendly summary

**Response:**
```
Sarah's Day Summary (as of 3:30 PM)

Glucose Status: Looking good! ✓
• Current: 128 mg/dL (in range)
• Time in Range today: 78%
• No concerning lows or highs

Meals & Insulin:
• Breakfast (7:45am): 45g carbs, 4U bolus → nice flat curve
• Lunch (12:30pm): 60g carbs, 5U bolus → slight spike to 185, came back down well
• After-school snack (3:00pm): 20g carbs, 1.5U bolus

Activity:
• PE class at 10am - handled it great, stayed 95-110 the whole time

Alerts:
• None today

Overall: Really solid day! The exercise control during PE was especially good.

Anything specific you'd like me to check?
```

---

### 5. Override Assistant

**User:** "I'm going for a 5k run in 30 minutes. What should I do?"

**MCP Flow:**
1. AI analyzes current glucose and IOB
2. AI calls `suggest_override` with `overrideType: "exercise"`
3. Human approves the suggestion
4. AI confirms activation

**Response:**
```
Great! For a 5k run, I recommend:

Exercise Override:
• Target range: 130-160 mg/dL (raised from 80-120)
• Basal reduction: -50% (to 0.55 U/hr from 1.1 U/hr)
• Duration: 2 hours (to cover run + recovery)

Current Status:
• Glucose: 112 mg/dL (a bit low for exercise start)
• IOB: 2.1 U (moderate)

Pre-Run Actions:
1. Consider 15g fast carbs now to raise glucose to ~130
2. Activate the exercise override in 15 minutes (before you start)
3. Keep glucose tabs with you

I've created a suggested override. Would you like me to activate it?

[Approve] [Modify] [Cancel]
```

---

## Alignment with Agent Control Plane

### How MCP Extends the Control Plane

| Control Plane Concept | MCP Implementation |
|-----------------------|--------------------|
| **Event Envelope** | MCP tool calls emit events with issuer = "mcp-client:{id}" |
| **Authority Hierarchy** | MCP agents have `authority: "delegated"` by default |
| **Delegation Grants** | Created via OAuth scope approval flow |
| **Override Instances** | Created by `suggest_override` and `activate_override` tools |
| **Delivery Requests** | Created by treatment logging tools |
| **Policy Composition** | Read by MCP resources for context |
| **Audit Trail** | All MCP actions logged as events |

### MCP-Specific Extensions to Control Plane

#### 1. MCP Session Tracking

```yaml
MCPSession:
  sessionId: string
  clientType: "claude-desktop" | "chatgpt" | "local-llm" | "custom"
  clientVersion: string
  
  delegationGrantId: string
  
  conversationContext:
    messageCount: integer
    toolInvocations: integer
    confirmationsRequired: integer
    confirmationsApproved: integer
  
  trustScore: number  # 0.0 - 1.0
  
  createdAt: datetime
  lastActivityAt: datetime
  expiresAt: datetime
```

#### 2. MCP Tool Invocation Events

```yaml
EventEnvelope:
  eventType: "mcp.tool.invoked"
  issuer: "mcp-client:claude-desktop:user-123"
  
  payload:
    sessionId: string
    toolName: string
    arguments: object
    
    outcome: "success" | "pending_approval" | "rejected" | "error"
    
    humanApproval:
      required: boolean
      approvedBy: string
      approvedAt: datetime
    
    resultingEvents: [string]  # IDs of events created by this tool
```

#### 3. AI-Generated Insights

```yaml
AIInsight:
  insightId: string
  insightType: "pattern" | "recommendation" | "alert" | "summary"
  
  generatedBy:
    mcpSessionId: string
    modelIdentifier: string
  
  content:
    title: string
    description: string
    confidence: number
    
    evidence:
      dataPoints: [string]
      correlations: [object]
  
  actionable: boolean
  suggestedActions: [object]
  
  userFeedback:
    helpful: boolean
    comments: string
  
  createdAt: datetime
```

---

## Future Extensions

### 1. Voice Assistant Integration

Enable voice queries via Alexa, Google Home, Siri:

```
User: "Alexa, ask Nightscout what my glucose is"
Alexa: "Your glucose is 128, in range"

User: "Alexa, ask Nightscout if I need to bolus for 45 grams of carbs"
Alexa: "Based on your carb ratio of 1 to 12, you would need about 4 units. 
        Please confirm the bolus on your pump."
```

### 2. Proactive AI Alerts

AI monitors data and proactively offers insights:

```
AI: "I noticed you've had 3 overnight lows this week after evening exercise. 
     Would you like me to create an exercise override preset that automatically 
     reduces your basal by 50% for 2 hours?"
```

### 3. Multi-Modal Context

Integrate with other data sources:

- **Fitness trackers** — Heart rate, sleep, steps
- **Meal photos** — Visual carb estimation
- **CGM trends** — Predictive low/high alerts
- **Geolocation** — Context-aware suggestions

### 4. Collaborative AI

Multiple AI agents working together:

- **Primary Agent** — General diabetes management
- **Pattern Detection Agent** — Specialized in finding correlations
- **Meal Planning Agent** — Recipe and carb counting help
- **Exercise Coach Agent** — Activity-specific glucose management

### 5. Cross-Platform Sync

MCP sessions synchronized across devices:

```
User on phone: "What was my time in range yesterday?"
[Conversation stored in MCP session]

User on desktop: "And how does that compare to last week?"
[AI has context from previous conversation]
```

---

## References

### MCP Specification
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)

### Nightscout Documentation
- [Agent Control Plane RFC](./agent-control-plane-rfc.md)
- [Integration Questionnaire](./integration-questionnaire.md)
- [OIDC Actor Identity Proposal](./oidc-actor-identity-proposal.md)
- [Nightscout API Documentation](https://nightscout.github.io/)

### Diabetes Management
- [ADA Standards of Care](https://diabetesjournals.org/care/issue/46/Supplement_1)
- [Time in Range Consensus](https://care.diabetesjournals.org/content/42/8/1593)

---

## Changelog

| Date | Change |
|------|--------|
| 2026-01-18 | Initial draft |

---

## Appendix A: Example MCP Server Configuration

```json
{
  "mcpServers": {
    "nightscout": {
      "command": "node",
      "args": ["/path/to/nightscout/lib/mcp-server/index.js"],
      "env": {
        "NIGHTSCOUT_URL": "https://your-nightscout.herokuapp.com",
        "MCP_AUTH_TYPE": "oauth",
        "MCP_ENABLE_TOOLS": "query_glucose_stats,detect_patterns,generate_daily_summary",
        "MCP_REQUIRE_CONFIRMATION": "true",
        "MCP_LOG_LEVEL": "info"
      }
    }
  }
}
```

---

## Appendix B: Security Checklist

- [ ] OAuth 2.0 authentication implemented
- [ ] Scoped permissions enforced
- [ ] Rate limiting active
- [ ] Human confirmation for high-risk actions
- [ ] Audit logging for all tool invocations
- [ ] Input validation on all tool arguments
- [ ] Output sanitization for PII
- [ ] Session expiration and cleanup
- [ ] Progressive trust scoring
- [ ] Anomaly detection for unusual patterns
- [ ] Emergency killswitch for AI access
- [ ] Regular security audits
- [ ] Compliance with healthcare data regulations (HIPAA, GDPR)
