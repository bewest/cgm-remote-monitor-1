# Nightscout Future Architecture: MCP & Agent Control Plane

**Document Version:** 1.0 (Future State)  
**Last Updated:** 2026-01-18  
**Purpose:** Architectural vision for AI agent integration via MCP and control plane separation

---

## Overview

This document describes the future architecture of Nightscout with MCP (Model Context Protocol) support and the Agent Control Plane, as proposed in:
- [MCP Support Proposal](./proposals/mcp-support-proposal.md)
- [Agent Control Plane RFC](./proposals/agent-control-plane-rfc.md)
- [MCP-Agent Control Plane Alignment](./proposals/mcp-agent-control-plane-alignment.md)

---

## Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   CLIENT LAYER                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                  │
│  │  Web Dashboard   │  │  Mobile Apps     │  │  Voice Assistants│                  │
│  │  (D3.js/React)   │  │  (iOS/Android)   │  │  (Alexa/Google)  │                  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘                  │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                      AI CLIENTS (NEW)                                       │    │
│  │                                                                             │    │
│  │  Claude Desktop │ ChatGPT │ Local LLMs │ Custom AI Agents                  │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                          │                                │
                          │ REST/WebSocket                 │ JSON-RPC (MCP)
                          │                                │
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                TRANSPORT LAYER                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌────────────────────────────────┐      ┌─────────────────────────────────┐       │
│  │  HTTP/HTTPS + Socket.IO        │      │  MCP Server (NEW)                │       │
│  │  (Express 4.17.1)              │      │  • stdio transport               │       │
│  │  • REST API v1/v2/v3           │      │  • HTTP/SSE transport            │       │
│  │  • WebSocket subscriptions     │      │  • WebSocket transport           │       │
│  └────────────────────────────────┘      └─────────────────────────────────┘       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                          │                                │
                          ▼                                ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              APPLICATION LAYER                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                      MCP ADAPTER LAYER (NEW)                                │    │
│  │                                                                             │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │  Resources   │  │    Tools     │  │   Prompts    │  │   Sessions   │  │    │
│  │  │              │  │              │  │              │  │              │  │    │
│  │  │ • Entries    │  │ • Query      │  │ • Daily      │  │ • Auth       │  │    │
│  │  │ • Treatments │  │   Stats      │  │   Review     │  │ • Trust      │  │    │
│  │  │ • Profiles   │  │ • Detect     │  │ • Pattern    │  │ • Rate       │  │    │
│  │  │ • Overrides  │  │   Patterns   │  │   Detective  │  │   Limiting   │  │    │
│  │  │ • Reports    │  │ • Log        │  │ • Bolus      │  │ • Confirm-   │  │    │
│  │  │              │  │   Treatment  │  │   Advisor    │  │   ations     │  │    │
│  │  │              │  │ • Suggest/   │  │              │  │              │  │    │
│  │  │              │  │   Activate   │  │              │  │              │  │    │
│  │  │              │  │   Override   │  │              │  │              │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                   AGENT CONTROL PLANE (NEW)                                 │    │
│  │                                                                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │    │
│  │  │                         EVENT STREAM                                  │  │    │
│  │  │  • EventEnvelope with cursor-based ordering                           │  │    │
│  │  │  • Issuer tracking (human, controller, agent)                         │  │    │
│  │  │  • Idempotency and deduplication                                      │  │    │
│  │  │  • WebSocket/SSE subscriptions                                        │  │    │
│  │  └──────────────────────────────────────────────────────────────────────┘  │    │
│  │                                                                             │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │  Config      │  │  Runtime     │  │  Computed    │  │  Delivery    │  │    │
│  │  │  Objects     │  │  Events      │  │  State       │  │  Tracking    │  │    │
│  │  │              │  │              │  │              │  │              │  │    │
│  │  │ • Profile    │  │ • Profile    │  │ • Policy     │  │ • Delivery   │  │    │
│  │  │   Definition │  │   Selection  │  │   Composition│  │   Request    │  │    │
│  │  │ • Override   │  │ • Override   │  │ • Capability │  │ • Delivery   │  │    │
│  │  │   Definition │  │   Instance   │  │   Snapshot   │  │   Observ-    │  │    │
│  │  │ • Controller │  │ • Controller │  │ • AI Insight │  │   ation      │  │    │
│  │  │   Kind       │  │   Reg.       │  │              │  │ • Reconcil-  │  │    │
│  │  │              │  │              │  │              │  │   iation     │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │    │
│  │                                                                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │    │
│  │  │           Authority & Conflict Resolution                             │  │    │
│  │  │  • Human > Agent > Controller hierarchy                               │  │    │
│  │  │  • Multi-writer semantics                                             │  │    │
│  │  │  • Supersession rules                                                 │  │    │
│  │  │  • Rate limiting & flip-flop prevention                               │  │    │
│  │  └──────────────────────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                          │                                          │
│                                          ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                   LEGACY APPLICATION LAYER                                  │    │
│  │                                                                             │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │ Authorization│  │    Plugin    │  │ Notification │  │     Data     │  │    │
│  │  │ (JWT/Shiro)  │  │    System    │  │    Engine    │  │    Loader    │  │    │
│  │  │              │  │  (30+ plugins)│  │              │  │              │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │    │
│  │                                                                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │    │
│  │  │                  EVENT BUS (lib/bus.js)                               │  │    │
│  │  │    Stream-based pub/sub: tick, data-update, notification             │  │    │
│  │  └──────────────────────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                        BRIDGE LAYER (NEW)                                   │    │
│  │                                                                             │    │
│  │  • devicestatus → EventEnvelope synthesis                                  │    │
│  │  • Profile hashing and change detection                                    │    │
│  │  • Override state diffing                                                  │    │
│  │  • Delivery extraction (enacted → DeliveryObservation)                     │    │
│  │  • Legacy API compatibility                                                │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   DATA LAYER                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │                            MongoDB Collections                              │    │
│  │                                                                             │    │
│  │  LEGACY (Existing):                                                        │    │
│  │  • entries (CGM readings)                                                  │    │
│  │  • treatments (insulin, carbs, corrections)                                │    │
│  │  • devicestatus (controller/pump snapshots)                                │    │
│  │  • profile (treatment profiles - legacy blob)                              │    │
│  │  • food (custom food database)                                             │    │
│  │  • activity (activity logs)                                                │    │
│  │                                                                             │    │
│  │  NEW (Control Plane):                                                      │    │
│  │  • events (EventEnvelope - append-only event stream)                       │    │
│  │  • profileDefinitions (versioned, content-hashed profiles)                 │    │
│  │  • profileSelections (profile activation events)                           │    │
│  │  • overrideDefinitions (reusable override templates)                       │    │
│  │  • overrideInstances (concrete override activations)                       │    │
│  │  • policyCompositions (computed effective therapy settings)                │    │
│  │  • deliveryRequests (intent to deliver insulin/basal)                      │    │
│  │  • deliveryObservations (confirmed delivery records)                       │    │
│  │  • reconciliations (request/observation matching)                          │    │
│  │  • controllerRegistrations (controller instances)                          │    │
│  │  • capabilitySnapshots (controller capability/status)                      │    │
│  │  • delegationGrants (authorization for agents/caregivers)                  │    │
│  │  • mcpSessions (AI client session tracking)                                │    │
│  │  • aiInsights (AI-generated patterns and recommendations)                  │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL INTEGRATIONS                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                  │
│  │  Loop / AAPS /   │  │  Dexcom Share /  │  │  Pushover /      │                  │
│  │  Trio / OpenAPS  │  │  LibreLink /     │  │  IFTTT /         │                  │
│  │  (Controllers)   │  │  Glooko          │  │  Discord         │                  │
│  │                  │  │  (CGM Bridges)   │  │  (Notifications) │                  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Examples

### Example 1: AI-Assisted Override via MCP

```
User (to Claude): "I'm going for a run in 30 minutes"
                       │
                       ▼
┌────────────────────────────────────┐
│  Claude Desktop (MCP Client)       │
│  • Understands context             │
│  • Retrieves current glucose       │
│  • Calls suggest_override tool     │
└────────────────────────────────────┘
                       │
                       │ JSON-RPC: callTool("suggest_override", {...})
                       ▼
┌────────────────────────────────────┐
│  MCP Server (Nightscout)           │
│  • Validates permissions           │
│  • Creates OverrideInstance        │
│  • Emits EventEnvelope             │
└────────────────────────────────────┘
                       │
                       │ POST /api/v3/events
                       ▼
┌────────────────────────────────────┐
│  Agent Control Plane               │
│  • Stores event with cursor        │
│  • Sets status: pending_approval   │
│  • Triggers confirmation workflow  │
└────────────────────────────────────┘
                       │
                       │ Mobile notification
                       ▼
┌────────────────────────────────────┐
│  User (Mobile App)                 │
│  "Approve exercise override?"      │
│  [Approve] [Modify] [Reject]       │
└────────────────────────────────────┘
                       │
                       │ User taps [Approve]
                       ▼
┌────────────────────────────────────┐
│  Control Plane                     │
│  • Creates confirmation event      │
│  • Updates override status: active │
│  • Notifies subscribed controllers │
└────────────────────────────────────┘
                       │
                       │ WebSocket event stream
                       ▼
┌────────────────────────────────────┐
│  Loop/AAPS Controller              │
│  • Receives override activation    │
│  • Adjusts automation accordingly  │
│  • Emits capability snapshot       │
└────────────────────────────────────┘
                       │
                       │ MCP notification
                       ▼
┌────────────────────────────────────┐
│  Claude Desktop                    │
│  "Override activated successfully" │
└────────────────────────────────────┘
```

### Example 2: Daily Review Query via MCP

```
User: "What was my time in range yesterday?"
                       │
                       ▼
┌────────────────────────────────────┐
│  AI Client (MCP)                   │
│  • Requests resource:              │
│    nightscout://entries/yesterday  │
└────────────────────────────────────┘
                       │
                       │ getResource("nightscout://entries/yesterday")
                       ▼
┌────────────────────────────────────┐
│  MCP Server                        │
│  • Checks read permission          │
│  • Queries control plane API       │
└────────────────────────────────────┘
                       │
                       │ GET /api/v3/entries?date=2026-01-17
                       ▼
┌────────────────────────────────────┐
│  Control Plane / Data Layer        │
│  • Queries entries collection      │
│  • Returns ~288 readings           │
└────────────────────────────────────┘
                       │
                       │ Raw entry data
                       ▼
┌────────────────────────────────────┐
│  MCP Server                        │
│  • Calls query_glucose_stats tool  │
│  • Calculates TIR, avg, stdDev     │
│  • Formats for AI consumption      │
└────────────────────────────────────┘
                       │
                       │ Structured summary
                       ▼
┌────────────────────────────────────┐
│  AI Client                         │
│  • Analyzes data                   │
│  • Generates natural language      │
│    response with insights          │
└────────────────────────────────────┘
                       │
                       ▼
User: "Yesterday's time in range was 72% (target: 70-180 mg/dL).
       Average glucose was 142 mg/dL. Great overnight control!"
```

### Example 3: Loop Controller Uploads Status

```
┌────────────────────────────────────┐
│  Loop App (iOS Controller)         │
│  • Runs automation loop            │
│  • Uploads devicestatus blob       │
└────────────────────────────────────┘
                       │
                       │ POST /api/v1/devicestatus
                       ▼
┌────────────────────────────────────┐
│  Bridge Layer                      │
│  • Receives devicestatus           │
│  • Extracts profile, override,     │
│    pump status, enacted action     │
│  • Diffs against last known state  │
└────────────────────────────────────┘
                       │
                       │ Synthesizes events
                       ▼
┌────────────────────────────────────┐
│  Control Plane                     │
│  Events created:                   │
│  • CapabilitySnapshot              │
│  • DeliveryObservation (if enacted)│
│  • OverrideInstance (if changed)   │
└────────────────────────────────────┘
                       │
                       │ Event stream subscription
                       ▼
┌────────────────────────────────────┐
│  MCP Clients (if subscribed)       │
│  • Receive real-time updates       │
│  • Can react to controller actions │
└────────────────────────────────────┘
                       │
                       │
                       ▼
┌────────────────────────────────────┐
│  Legacy Clients                    │
│  • Continue to work via v1 API     │
│  • No changes required             │
└────────────────────────────────────┘
```

---

## Key Architectural Principles

### 1. Separation of Concerns

| Layer | Responsibility |
|-------|----------------|
| **MCP Adapter** | AI protocol translation, natural language interface |
| **Control Plane** | Business logic, event sourcing, authority, conflict resolution |
| **Bridge Layer** | Legacy compatibility, devicestatus synthesis |
| **Data Layer** | Persistence, querying, indexing |

### 2. Authority Hierarchy

```
HUMAN (primary authority)
    ├─ Direct UI actions (highest priority)
    │
    ├─ HUMAN (caregiver, delegated)
    │   └─ Remote monitoring/intervention
    │
    ├─ AGENT (AI assistants, delegated)
    │   ├─ Suggestions (require approval)
    │   └─ Approved actions (within constraints)
    │
    └─ CONTROLLER (Loop/AAPS/Trio, automated)
        └─ Autonomous operation (within profile limits)
```

### 3. Event Sourcing

All state changes flow through the event stream:

```yaml
EventEnvelope:
  eventId: "uuid"
  eventType: "override.instance.activated"
  cursor: 12345  # Global ordering
  issuer: "mcp-client:claude-desktop:user-abc"
  timestamp: "2026-01-18T10:30:00Z"
  payload: { ... }
```

Benefits:
- **Audit trail** — Every action is recorded with who/what/when/why
- **Replay** — Can reconstruct state at any point in time
- **Multi-writer** — Conflict resolution via cursor ordering
- **Real-time** — Clients subscribe to event stream for live updates

### 4. Progressive Disclosure

AI agents discover capabilities dynamically:

1. **Initial connection** → Limited read-only access
2. **After trust buildup** → Suggestion capabilities
3. **With explicit delegation** → Action capabilities (with constraints)
4. **Emergency override** → Human can always revoke/supersede

### 5. Backward Compatibility

Legacy systems continue to work:

- **Bridge Layer** synthesizes events from devicestatus uploads
- **API v1/v2** remain functional alongside v3
- **Existing plugins** continue to use event bus
- **Gradual migration** path for controllers (Loop/AAPS/Trio)

---

## Migration Path

### Phase 1: Foundation (Parallel Systems)

```
Legacy System (existing)  ←→  Control Plane (new)
         │                           │
         │    Bridge Layer           │
         │    synthesizes events     │
         │                           │
         └───────────┬───────────────┘
                     │
                     ▼
              MongoDB (unified)
```

### Phase 2: MCP Integration

```
Legacy System  ←→  Control Plane  ←→  MCP Server
         │              │                  │
         │              │                  │
         └──────────────┴──────────────────┘
                        │
                        ▼
                   MongoDB
```

### Phase 3: Full Integration

```
                  Control Plane (primary)
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   MCP Server    Legacy Bridge    Direct API v3
        │               │               │
        ▼               ▼               ▼
     AI Clients    Old Clients    New Clients
```

---

## Security Model

### Authentication

```
User/AI Agent
      │
      │ OAuth 2.0 / OIDC
      ▼
Nightscout Identity Provider
      │
      │ Issues JWT with scopes
      ▼
Access Token with Delegation Grant
      │
      │ Validated on each request
      ▼
MCP Server / Control Plane
```

### Authorization Scopes

```yaml
Scope Hierarchy:
  - entries.read (low risk)
  - treatments.read (low risk)
  - profile.read (low risk)
  - analytics.run (low risk)
  
  - treatments.log (medium risk)
    └─ constraints: maxInsulin, requireConfirmation
  
  - override.suggest (medium risk)
    └─ constraints: requireApproval
  
  - override.activate (high risk)
    └─ constraints: allowedDefinitionIds, maxDuration
  
  - profile.modify (high risk)
    └─ constraints: requireMedicalReview
```

### Audit Trail

Every action creates an immutable event:

```yaml
EventEnvelope:
  eventId: "uuid"
  issuer: "mcp-client:claude:user-abc"
  timestamp: "2026-01-18T10:30:00Z"
  
  delegationGrant:
    grantId: "grant-123"
    grantedBy: "user-abc"
    scopes: ["override.suggest"]
  
  humanApproval:
    required: true
    approvedBy: "user-abc"
    approvedAt: "2026-01-18T10:35:00Z"
```

---

## Performance Considerations

### Event Stream Optimization

- **Cursor-based pagination** — Efficient incremental sync
- **Selective subscriptions** — Filter by event type/issuer
- **Event compaction** — Periodic snapshot + delta strategy
- **Caching** — PolicyComposition and CapabilitySnapshot are cached

### MCP Resource Optimization

- **Summarization** — Large datasets summarized for AI context efficiency
- **Sampling** — Representative samples instead of full datasets
- **Lazy loading** — Full data available on demand
- **Time-based caching** — Recent queries cached for quick re-access

---

## Monitoring & Observability

### Metrics to Track

1. **MCP Usage**
   - Tool invocations per client
   - Confirmation approval rates
   - Trust score progression
   - Error rates by tool

2. **Control Plane Health**
   - Event stream lag
   - Conflict resolution frequency
   - Authority override rate
   - Supersession patterns

3. **System Integration**
   - Controller upload latency
   - Bridge synthesis accuracy
   - WebSocket connection stability
   - API response times

### Logging Strategy

```yaml
LogEntry:
  level: "info" | "warning" | "error"
  component: "mcp-server" | "control-plane" | "bridge"
  
  context:
    userId: string
    sessionId: string
    requestId: string
  
  event:
    type: string
    details: object
  
  performance:
    duration_ms: number
    memory_mb: number
```

---

## Future Enhancements

### 1. Voice Assistant Integration

Enable hands-free diabetes management:
- "Alexa, what's my glucose?"
- "Hey Google, log 60 grams of carbs"

### 2. Predictive AI

Beyond reactive assistance:
- Proactive low/high predictions
- Meal bolus suggestions based on past patterns
- Sleep quality correlation with overnight glucose

### 3. Multi-Patient Support

For caregivers managing multiple people:
- Unified AI assistant across multiple Nightscout instances
- Cross-patient pattern recognition (anonymized)
- Family-wide insights

### 4. Integration with Wearables

Holistic health view:
- Heart rate variability + glucose correlation
- Sleep tracking + overnight patterns
- Exercise intensity + glucose response

---

## References

- [MCP Support Proposal](./proposals/mcp-support-proposal.md)
- [Agent Control Plane RFC](./proposals/agent-control-plane-rfc.md)
- [MCP-Agent Alignment Guide](./proposals/mcp-agent-control-plane-alignment.md)
- [OIDC Actor Identity Proposal](./proposals/oidc-actor-identity-proposal.md)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
- [Current Architecture Overview](./architecture-overview.md)

---

**Note:** This is a future-state architecture document. Implementation will occur in phases as outlined in the individual proposals. The current Nightscout architecture (documented in [architecture-overview.md](./architecture-overview.md)) remains the active system until migration phases are completed.
