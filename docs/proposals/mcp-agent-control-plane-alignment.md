# MCP and Agent Control Plane Alignment Guide

**Status:** Draft  
**Authors:** Nightscout Community  
**Created:** 2026-01-18  
**Last Updated:** 2026-01-18  
**Related:** [MCP Support Proposal](./mcp-support-proposal.md), [Agent Control Plane RFC](./agent-control-plane-rfc.md)

---

## Abstract

This document provides a detailed mapping between the **Model Context Protocol (MCP)** implementation and the **Agent Control Plane** architecture for Nightscout. It clarifies how MCP acts as a protocol layer that enables AI agents to interact with the control plane's event-driven architecture while maintaining safety, auditability, and compatibility with existing Nightscout systems.

---

## Table of Contents

1. [Architectural Layering](#architectural-layering)
2. [MCP as Protocol Adapter](#mcp-as-protocol-adapter)
3. [Event Mapping](#event-mapping)
4. [Authority and Identity](#authority-and-identity)
5. [Resource-to-Collection Mapping](#resource-to-collection-mapping)
6. [Tool-to-Event Mapping](#tool-to-event-mapping)
7. [Safety and Conflict Resolution](#safety-and-conflict-resolution)
8. [Implementation Coordination](#implementation-coordination)
9. [Testing Strategy](#testing-strategy)

---

## Architectural Layering

The MCP implementation sits **above** the Agent Control Plane, acting as a specialized interface for AI agents.

```
┌─────────────────────────────────────────────────────────────┐
│                   AI CLIENTS (MCP Compatible)                │
│                                                              │
│  Claude Desktop │ ChatGPT │ Local LLMs │ Custom Clients     │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ JSON-RPC 2.0 (MCP Protocol)
                            │
┌─────────────────────────────────────────────────────────────┐
│                       MCP SERVER LAYER                       │
│                  (Protocol Translation)                      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Resources   │  │    Tools     │  │   Prompts    │      │
│  │              │  │              │  │              │      │
│  │ Map Control  │  │ Translate to │  │ Template     │      │
│  │ Plane Data   │  │ Event Calls  │  │ Generation   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │          MCP Session & Auth Manager            │         │
│  │  • OAuth integration                           │         │
│  │  • Delegation grant validation                 │         │
│  │  • Rate limiting                               │         │
│  │  • Confirmation workflows                      │         │
│  └────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Agent Issuer API
                            │
┌─────────────────────────────────────────────────────────────┐
│                   AGENT CONTROL PLANE                        │
│                  (Event-Driven Core)                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  EVENT STREAM                         │   │
│  │  • EventEnvelope                                      │   │
│  │  • Cursor-based ordering                              │   │
│  │  • Issuer tracking                                    │   │
│  │  • Idempotency                                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Config       │  │  Runtime     │  │  Computed    │      │
│  │ Objects      │  │  Events      │  │  State       │      │
│  │              │  │              │  │              │      │
│  │ • Profile    │  │ • Override   │  │ • Policy     │      │
│  │   Definition │  │   Instance   │  │   Composition│      │
│  │ • Override   │  │ • Delivery   │  │ • Capability │      │
│  │   Definition │  │   Request    │  │   Snapshot   │      │
│  │              │  │ • Delivery   │  │              │      │
│  │              │  │   Observation│  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │         Authority & Conflict Resolution        │         │
│  │  • Human > Agent > Controller hierarchy        │         │
│  │  • Multi-writer semantics                      │         │
│  │  • Supersession rules                          │         │
│  └────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ API v3
                            │
┌─────────────────────────────────────────────────────────────┐
│                   NIGHTSCOUT DATA PLANE                      │
│                                                              │
│  Entries │ Treatments │ DeviceStatus │ Profiles             │
└─────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility |
|-------|----------------|
| **AI Clients** | Natural language interaction, context management, user interface |
| **MCP Server Layer** | Protocol translation, AI-specific formatting, session management |
| **Agent Control Plane** | Business logic, event sourcing, conflict resolution, audit trail |
| **Nightscout Data Plane** | Data persistence, legacy API compatibility |

---

## MCP as Protocol Adapter

The MCP Server acts as a **protocol adapter** that translates between:

1. **AI-friendly representations** (natural language, structured resources)
2. **Control plane events** (structured, append-only, versioned)

### Translation Responsibilities

#### Inbound (AI → Control Plane)

```
AI Tool Call
    │
    ▼
┌─────────────────────────┐
│ MCP Tool Handler        │
│ • Validate arguments    │
│ • Check permissions     │
│ • Request confirmation  │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ Event Creation          │
│ • Build EventEnvelope   │
│ • Set issuer metadata   │
│ • Generate idempotency  │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ Control Plane API       │
│ POST /api/v3/events     │
└─────────────────────────┘
```

#### Outbound (Control Plane → AI)

```
Control Plane Query
    │
    ▼
┌─────────────────────────┐
│ MCP Resource Handler    │
│ • Fetch from API v3     │
│ • Apply filters         │
│ • Check read permissions│
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ AI Formatting           │
│ • Summarize if large    │
│ • Add metadata          │
│ • Format for AI context │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ MCP Resource Response   │
│ JSON with annotations   │
└─────────────────────────┘
```

---

## Event Mapping

### How MCP Tools Create Control Plane Events

| MCP Tool | Control Plane Event | Event Type |
|----------|-------------------|------------|
| `suggest_override` | `OverrideInstance` creation | `override.instance.proposed` |
| `activate_override` | `OverrideInstance` activation | `override.instance.activated` |
| `log_treatment` | `DeliveryObservation` | `delivery.observed` |
| `query_glucose_stats` | (Read-only, no event) | N/A |
| `detect_patterns` | `AIInsight` creation | `ai.insight.generated` |
| `generate_daily_summary` | `AIInsight` creation | `ai.insight.generated` |

### Event Envelope for MCP Actions

Every MCP tool invocation that modifies state creates an `EventEnvelope`:

```yaml
EventEnvelope:
  eventId: "uuid-12345"
  eventType: "override.instance.proposed"
  cursor: 98765
  
  issuer: "mcp-client:claude-desktop:user-abc123"
  issuerSeq: 42
  idempotencyKey: "mcp-session-xyz:tool-call-123"
  
  timestamp: "2026-01-18T10:30:00Z"
  
  refs:
    - refType: "delegationGrant"
      refId: "grant-789"
    - refType: "mcpSession"
      refId: "session-xyz"
  
  payload:
    definitionId: null  # Ad-hoc override
    overrideType: "exercise"
    start: "2026-01-18T11:00:00Z"
    duration: 7200
    effectiveEffects:
      targetRange:
        low: 130
        high: 160
      basalMultiplier: 0.5
    requestedBy:
      issuerType: "agent"
      issuerId: "mcp-client:claude-desktop:user-abc123"
      authority: "delegated"
    status: "pending_approval"
    reason: "User mentioned going for a 5k run"
    annotations:
      mcpToolName: "suggest_override"
      mcpSessionId: "session-xyz"
      aiModel: "claude-3.5-sonnet"
      conversationContext: "User: I'm going for a 5k run in 30 minutes"
```

### New Event Types for MCP

#### `mcp.session.started`

```yaml
EventEnvelope:
  eventType: "mcp.session.started"
  issuer: "mcp-client:claude-desktop:user-abc123"
  payload:
    sessionId: "session-xyz"
    clientType: "claude-desktop"
    clientVersion: "1.0.0"
    delegationGrantId: "grant-789"
    scopes: ["entries.read", "override.suggest", "treatments.log"]
```

#### `mcp.tool.invoked`

```yaml
EventEnvelope:
  eventType: "mcp.tool.invoked"
  issuer: "mcp-client:claude-desktop:user-abc123"
  payload:
    sessionId: "session-xyz"
    toolName: "suggest_override"
    arguments:
      overrideType: "exercise"
      duration: 7200
      reason: "User mentioned going for a 5k run"
    outcome: "pending_approval"
    resultingEvents: ["uuid-12345"]  # The override.instance.proposed event
```

#### `mcp.confirmation.requested`

```yaml
EventEnvelope:
  eventType: "mcp.confirmation.requested"
  issuer: "mcp-client:claude-desktop:user-abc123"
  payload:
    confirmationId: "confirm-123"
    toolName: "suggest_override"
    action: "Activate exercise override for 2 hours with -50% basal"
    requiredBy: "2026-01-18T11:00:00Z"
    status: "pending"
```

#### `mcp.confirmation.responded`

```yaml
EventEnvelope:
  eventType: "mcp.confirmation.responded"
  issuer: "user-abc123"  # Human issuer
  payload:
    confirmationId: "confirm-123"
    approved: true
    approvedBy: "user-abc123"
    approvedAt: "2026-01-18T10:32:00Z"
    notes: "Approved via mobile app"
```

#### `ai.insight.generated`

```yaml
EventEnvelope:
  eventType: "ai.insight.generated"
  issuer: "mcp-client:claude-desktop:user-abc123"
  payload:
    insightId: "insight-456"
    insightType: "pattern"
    title: "Recurring overnight lows after evening exercise"
    description: "Detected 6 overnight lows in past 14 days, all following evening exercise"
    confidence: 0.85
    evidence:
      dataPoints: ["entry-1", "entry-2", ...]
      correlations:
        - factor: "evening_exercise"
          strength: 0.82
    actionable: true
    suggestedActions:
      - "Create evening exercise override preset"
      - "Reduce dinner bolus by 10-15% on exercise days"
```

---

## Authority and Identity

### MCP Clients as Agent Issuers

In the Agent Control Plane, MCP clients are treated as **agent issuers** with delegated authority.

#### Issuer Identifier Format

```
mcp-client:{clientType}:{userId}
```

**Examples:**
- `mcp-client:claude-desktop:user-abc123`
- `mcp-client:chatgpt:user-xyz789`
- `mcp-client:local-llm:user-def456`

#### Authority Level

```yaml
Issuer:
  issuerType: "agent"
  issuerId: "mcp-client:claude-desktop:user-abc123"
  authority: "delegated"
  
  delegationGrant:
    grantId: "grant-789"
    grantedBy: "user-abc123"  # The human principal
    grantedAt: "2026-01-18T09:00:00Z"
    expiresAt: "2026-01-18T21:00:00Z"  # 12-hour session
    
    scopes:
      - "entries.read"
      - "treatments.read"
      - "profile.read"
      - "override.suggest"
      - "treatments.log"
    
    constraints:
      maxOverrideDuration: 7200  # 2 hours
      requireConfirmation: true
      maxInsulinPerTreatment: 5.0
```

### Delegation Grant Creation

When a user authorizes an MCP client via OAuth:

```yaml
EventEnvelope:
  eventType: "delegation.grant.created"
  issuer: "user-abc123"  # Human principal
  payload:
    grantId: "grant-789"
    grantedTo: "mcp-client:claude-desktop:user-abc123"
    scopes: [...]
    constraints: {...}
    reason: "AI assistant for diabetes management"
```

### Authority Hierarchy in Practice

```
Human (primary)
    │
    ├─ "user-abc123" (via Nightscout UI)
    │
    ├─ "mcp-client:claude-desktop:user-abc123" (delegated)
    │   │
    │   └─ Must get confirmation for:
    │       • Insulin logging >5U
    │       • Override activation (initially)
    │
    └─ "controller:loop:device-xyz" (automated)
        │
        └─ Can act autonomously within limits
```

**Conflict Resolution:**
- If human activates override → supersedes AI suggestion
- If AI suggests override → controller respects it (after approval)
- If controller suggests override → AI can suggest modification (with approval)

---

## Resource-to-Collection Mapping

### MCP Resources ↔ Control Plane Collections

| MCP Resource URI | Control Plane Collection | Additional Processing |
|------------------|-------------------------|----------------------|
| `nightscout://entries/{range}` | `/api/v3/entries` | Filter by time range, summarize if >1000 entries |
| `nightscout://treatments/{range}` | `/api/v3/treatments` | Group by type, calculate totals |
| `nightscout://profile/current` | `/api/v3/profileSelections` (latest) + `/api/v3/profileDefinitions/{id}` | Resolve reference, materialize current |
| `nightscout://overrides/active` | `/api/v3/overrideInstances` (status=active) | Include computed effects |
| `nightscout://policy/effective` | `/api/v3/policyCompositions` (latest) | Materialized view of effective parameters |
| `nightscout://devicestatus/latest` | `/api/v3/capabilitySnapshots` (latest) | Controller status and limits |
| `nightscout://reports/daily-summary` | Computed on-demand | Query entries + treatments for date range |

### Resource Fetching Example

When AI requests `nightscout://entries/today`:

```javascript
// MCP Resource Handler
async function getEntriesToday(session) {
  // 1. Check read permission
  if (!session.hasScope('entries.read')) {
    throw new PermissionDeniedError();
  }
  
  // 2. Calculate time range
  const today = new Date();
  const start = new Date(today.setHours(0,0,0,0));
  const end = new Date(today.setHours(23,59,59,999));
  
  // 3. Query control plane API
  const entries = await controlPlaneAPI.getEntries({
    start: start.toISOString(),
    end: end.toISOString(),
    limit: 1000
  });
  
  // 4. Calculate summary statistics
  const stats = {
    count: entries.length,
    avgGlucose: calculateAverage(entries),
    timeInRange: calculateTIR(entries),
    unit: 'mg/dL'
  };
  
  // 5. Decide on summarization
  let content;
  if (entries.length > 500) {
    // Summarize for AI context efficiency
    content = {
      summary: stats,
      samples: sampleEntries(entries, 50),  // Representative sample
      fullDataAvailable: true
    };
  } else {
    content = {
      summary: stats,
      entries: entries
    };
  }
  
  // 6. Return MCP resource
  return {
    uri: 'nightscout://entries/today',
    mimeType: 'application/json',
    name: "Today's Glucose Readings",
    metadata: stats,
    content: content
  };
}
```

---

## Tool-to-Event Mapping

### Detailed Tool Implementation Examples

#### Tool: `suggest_override`

```javascript
// MCP Tool Handler
async function suggestOverride(session, args) {
  // 1. Validate arguments
  const { overrideType, duration, targetAdjustment, basalMultiplier, reason } = args;
  
  if (!reason || reason.length < 10) {
    throw new ValidationError('Reason must be at least 10 characters');
  }
  
  // 2. Check permission
  if (!session.hasScope('override.suggest')) {
    throw new PermissionDeniedError();
  }
  
  // 3. Create OverrideInstance (proposed state)
  const overrideInstance = {
    instanceId: generateUUID(),
    definitionId: null,  // Ad-hoc
    overrideType: overrideType,
    start: new Date(Date.now() + 60000).toISOString(),  // Start in 1 min
    duration: duration || 7200,  // Default 2 hours
    effectiveEffects: {
      basalMultiplier: basalMultiplier || 0.8,
      targetRange: {
        low: 100 + (targetAdjustment || 0),
        high: 120 + (targetAdjustment || 0)
      }
    },
    requestedBy: {
      issuerType: 'agent',
      issuerId: session.issuerId,
      authority: 'delegated'
    },
    status: 'pending_approval',
    reason: reason,
    annotations: {
      mcpToolName: 'suggest_override',
      mcpSessionId: session.sessionId,
      aiModel: session.clientMetadata.model,
      conversationContext: session.getRecentContext(3)  // Last 3 messages
    }
  };
  
  // 4. Create EventEnvelope
  const event = {
    eventId: generateUUID(),
    eventType: 'override.instance.proposed',
    issuer: session.issuerId,
    issuerSeq: await session.getNextSeq(),
    idempotencyKey: `${session.sessionId}:suggest_override:${Date.now()}`,
    timestamp: new Date().toISOString(),
    refs: [
      { refType: 'delegationGrant', refId: session.delegationGrantId },
      { refType: 'mcpSession', refId: session.sessionId }
    ],
    payload: overrideInstance
  };
  
  // 5. Submit to control plane
  const result = await controlPlaneAPI.createEvent(event);
  
  // 6. Request human confirmation
  const confirmation = await confirmationService.request({
    confirmationId: generateUUID(),
    eventId: event.eventId,
    action: `Activate ${overrideType} override for ${duration/3600} hours`,
    details: overrideInstance,
    requiredBy: new Date(Date.now() + 3600000).toISOString(),  // 1 hour timeout
    channel: 'mobile_notification'
  });
  
  // 7. Return to AI
  return {
    success: true,
    overrideInstanceId: overrideInstance.instanceId,
    status: 'pending_approval',
    confirmationId: confirmation.confirmationId,
    message: `Override suggested. Waiting for approval via mobile notification.`
  };
}
```

#### Tool: `log_treatment`

```javascript
// MCP Tool Handler
async function logTreatment(session, args) {
  const { eventType, insulin, carbs, notes, timestamp } = args;
  
  // 1. Validate & check permission
  if (!session.hasScope('treatments.log')) {
    throw new PermissionDeniedError();
  }
  
  // 2. Safety check for insulin
  if (insulin > 5.0) {
    if (!session.hasScope('treatments.log.high_insulin')) {
      // Require explicit confirmation
      const confirmation = await confirmationService.request({
        action: `Log ${insulin}U insulin bolus`,
        reason: 'Insulin amount exceeds safety threshold',
        requireExplicitApproval: true
      });
      
      if (!confirmation.approved) {
        return { success: false, reason: 'User declined confirmation' };
      }
    }
  }
  
  // 3. Create DeliveryObservation
  const observation = {
    observationId: generateUUID(),
    observationType: eventType.includes('Bolus') ? 'bolus' : 'basal',
    source: {
      sourceType: 'manual',
      sourceId: 'mcp-logged',
      sourceKind: 'ai-assistant'
    },
    observed: {
      units: insulin || 0,
      startTime: timestamp || new Date().toISOString()
    },
    confidence: 'reported',  // User-reported via AI
    reportedBy: {
      issuerType: 'agent',
      issuerId: session.issuerId
    },
    observedAt: new Date().toISOString(),
    annotations: {
      mcpToolName: 'log_treatment',
      mcpSessionId: session.sessionId,
      carbs: carbs,
      notes: notes
    }
  };
  
  // 4. Create event
  const event = {
    eventId: generateUUID(),
    eventType: 'delivery.observed',
    issuer: session.issuerId,
    payload: observation
  };
  
  // 5. Also create legacy treatment for backward compatibility
  const treatment = {
    eventType: eventType,
    insulin: insulin,
    carbs: carbs,
    notes: notes,
    created_at: timestamp || new Date().toISOString(),
    enteredBy: `AI Assistant (${session.clientMetadata.type})`
  };
  
  // 6. Submit both
  await Promise.all([
    controlPlaneAPI.createEvent(event),
    nightscoutAPI.createTreatment(treatment)
  ]);
  
  return {
    success: true,
    observationId: observation.observationId,
    treatmentId: treatment._id,
    message: `Treatment logged successfully`
  };
}
```

---

## Safety and Conflict Resolution

### MCP-Specific Safety Rules

#### 1. Confirmation Workflows

```yaml
ConfirmationPolicy:
  tool: "suggest_override"
  
  requireConfirmation: true
  confirmationChannels: ["mobile_notification", "web_ui"]
  confirmationTimeout: 3600  # 1 hour
  
  escalation:
    - after: 300  # 5 minutes
      action: "send_reminder"
    - after: 1800  # 30 minutes
      action: "send_urgent_notification"
    - after: 3600  # 1 hour
      action: "cancel_suggestion"
```

#### 2. Progressive Trust

```yaml
TrustPolicy:
  sessionId: "session-xyz"
  
  trustLevel: "initial"  # initial | trusted | verified
  
  confirmationBypass:
    initial:
      enabled: false
    trusted:
      enabled: true
      conditions:
        - "10+ successful interactions"
        - "0 rejected suggestions in last 24h"
        - "User explicitly enabled trust mode"
    verified:
      enabled: true
      conditions:
        - "50+ successful interactions"
        - "User completed AI safety training"
```

#### 3. Rate Limiting

```yaml
RateLimits:
  perSession:
    suggest_override:
      calls: 5
      window: 3600  # per hour
      
    log_treatment:
      calls: 10
      window: 3600
      
    activate_override:
      calls: 3
      window: 3600
      
  perUser:
    ai_tool_invocations:
      calls: 100
      window: 3600
```

### Conflict Resolution with MCP

#### Scenario: AI vs. Human Override

```
Timeline:
1. 10:00 - AI suggests exercise override (pending approval)
2. 10:05 - Human activates pre-meal override via app
3. 10:10 - Human approves AI's exercise override

Resolution:
- Human's pre-meal override takes precedence (human > agent authority)
- AI's exercise override is marked as "superseded"
- Both overrides remain in audit trail
- AI is notified of supersession for learning
```

**Event Sequence:**

```yaml
# Event 1: AI suggestion
EventEnvelope:
  eventType: "override.instance.proposed"
  issuer: "mcp-client:claude-desktop:user-abc123"
  payload:
    instanceId: "ai-override-1"
    overrideType: "exercise"
    status: "pending_approval"

# Event 2: Human activation (supersedes)
EventEnvelope:
  eventType: "override.instance.activated"
  issuer: "user-abc123"
  payload:
    instanceId: "human-override-1"
    overrideType: "preMeal"
    status: "active"
    supersedes: "ai-override-1"

# Event 3: AI override approved but superseded
EventEnvelope:
  eventType: "override.instance.approved_but_superseded"
  issuer: "user-abc123"
  refs:
    - { refType: "overrideInstance", refId: "ai-override-1" }
  payload:
    originalInstanceId: "ai-override-1"
    supersededBy: "human-override-1"
    reason: "Human activated different override before approval"
```

---

## Implementation Coordination

### Development Phases Alignment

| Phase | Agent Control Plane | MCP Implementation |
|-------|-------------------|-------------------|
| **Phase 1** (4-6 weeks) | Event model, Profile/Override collections, Basic bridge | Read-only MCP server, Resources, Analytics tools |
| **Phase 2** (4-6 weeks) | Delivery tracking, Capabilities, WebSocket | Agent integration, Write tools, Confirmations |
| **Phase 3** (6-8 weeks) | Agents & delegation, Conflict resolution | Advanced tools, Progressive trust, Multi-platform |

### Shared Components

Both implementations share:

1. **OAuth/OIDC Authentication** — Unified identity system
2. **Delegation Grant Management** — Same grants for web UI and MCP
3. **Event Store** — Single source of truth for all actions
4. **Audit Dashboard** — Unified view of human, controller, and AI actions

### API Endpoints Used by MCP

```
# Control Plane API v3 (used by MCP server)
GET  /api/v3/events
GET  /api/v3/profileDefinitions
GET  /api/v3/profileSelections
GET  /api/v3/overrideInstances
GET  /api/v3/policyCompositions
POST /api/v3/events
POST /api/v3/confirmations

# Legacy API (for backward compatibility)
GET  /api/v1/entries
GET  /api/v1/treatments
POST /api/v1/treatments
GET  /api/v1/devicestatus

# MCP-specific endpoints (new)
POST /api/mcp/auth/grant
POST /api/mcp/sessions
GET  /api/mcp/sessions/{id}
DELETE /api/mcp/sessions/{id}
```

---

## Testing Strategy

### Integration Tests

#### Test 1: MCP Tool Creates Control Plane Event

```javascript
describe('MCP Tool to Control Plane Integration', () => {
  it('should create override.instance.proposed event when suggest_override is called', async () => {
    // Setup
    const mcpSession = await createMCPSession({
      userId: 'user-123',
      scopes: ['override.suggest']
    });
    
    // Execute
    const result = await mcpClient.callTool('suggest_override', {
      overrideType: 'exercise',
      duration: 7200,
      basalMultiplier: 0.5,
      reason: 'User mentioned 5k run'
    });
    
    // Verify event created
    const events = await controlPlaneAPI.getEvents({
      eventType: 'override.instance.proposed',
      issuer: mcpSession.issuerId
    });
    
    expect(events).toHaveLength(1);
    expect(events[0].payload.overrideType).toBe('exercise');
    expect(events[0].payload.status).toBe('pending_approval');
  });
});
```

#### Test 2: Authority Hierarchy Enforcement

```javascript
describe('Authority Hierarchy', () => {
  it('should allow human to supersede AI suggestion', async () => {
    // AI suggests override
    const aiOverride = await mcpClient.callTool('suggest_override', {
      overrideType: 'exercise',
      duration: 7200
    });
    
    // Human activates different override
    const humanOverride = await nightscoutAPI.activateOverride({
      overrideType: 'preMeal',
      issuerId: 'user-123'
    });
    
    // Verify AI override superseded
    const aiInstance = await controlPlaneAPI.getOverrideInstance(aiOverride.instanceId);
    expect(aiInstance.status).toBe('superseded');
    expect(aiInstance.supersededBy).toBe(humanOverride.instanceId);
  });
});
```

#### Test 3: Confirmation Workflow

```javascript
describe('Confirmation Workflow', () => {
  it('should require and process human confirmation', async () => {
    // AI suggests high-insulin treatment
    const result = await mcpClient.callTool('log_treatment', {
      eventType: 'Meal Bolus',
      insulin: 8.5,
      carbs: 60
    });
    
    expect(result.status).toBe('pending_approval');
    expect(result.confirmationId).toBeDefined();
    
    // Simulate human approval
    await confirmationService.approve({
      confirmationId: result.confirmationId,
      approvedBy: 'user-123'
    });
    
    // Verify treatment created
    const observation = await controlPlaneAPI.getDeliveryObservation(result.observationId);
    expect(observation.observed.units).toBe(8.5);
    expect(observation.reportedBy.issuerId).toMatch(/^mcp-client:/);
  });
});
```

### End-to-End Scenarios

#### Scenario: Daily Review Flow

```javascript
describe('E2E: Daily Review', () => {
  it('should complete full daily review workflow', async () => {
    // 1. AI fetches today's data
    const entries = await mcpClient.getResource('nightscout://entries/today');
    expect(entries.metadata.count).toBeGreaterThan(100);
    
    // 2. AI calculates stats
    const stats = await mcpClient.callTool('query_glucose_stats', {
      timeRange: 'last24h',
      metrics: ['average', 'timeInRange', 'timeAbove', 'timeBelow']
    });
    
    expect(stats.timeInRange).toBeDefined();
    
    // 3. AI detects pattern
    const patterns = await mcpClient.callTool('detect_patterns', {
      patternType: 'overnight-low',
      lookbackDays: 14
    });
    
    // 4. AI generates insight (creates event)
    if (patterns.detected) {
      const events = await controlPlaneAPI.getEvents({
        eventType: 'ai.insight.generated'
      });
      
      expect(events.length).toBeGreaterThan(0);
    }
  });
});
```

---

## Summary

The MCP implementation and Agent Control Plane are **complementary and aligned**:

1. **MCP provides the protocol** — Standardized AI interaction
2. **Control Plane provides the logic** — Event sourcing, conflict resolution, audit
3. **Both maintain safety** — Human-in-the-loop, progressive trust, confirmations
4. **Both enable agents** — AI assistants can participate in therapy management
5. **Both are auditable** — Full event trail of all actions

This architecture enables Nightscout to safely integrate AI capabilities while maintaining the rigorous safety and auditability requirements of diabetes management systems.

---

## Next Steps

1. **Implement Phase 1** of both systems in parallel
2. **Create shared OAuth/delegation infrastructure**
3. **Build unified audit dashboard**
4. **Develop comprehensive test suite**
5. **Document security model and obtain security review**
6. **Create user education materials** for AI delegation
7. **Pilot with limited user group** before general release

---

## References

- [MCP Support Proposal](./mcp-support-proposal.md)
- [Agent Control Plane RFC](./agent-control-plane-rfc.md)
- [OIDC Actor Identity Proposal](./oidc-actor-identity-proposal.md)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
