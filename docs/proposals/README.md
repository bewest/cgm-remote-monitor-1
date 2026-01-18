# Nightscout Enhancement Proposals

This directory contains proposals for significant enhancements and architectural changes to Nightscout.

## Active Proposals

### Agent-Based & AI Integration

#### [Agent Control Plane RFC](./agent-control-plane-rfc.md)
**Status:** Draft  
**Created:** 2026-01-01

Proposes a clean separation between control plane (policy, configuration, intent) and data plane (observations, telemetry, delivery) for Nightscout and automated insulin delivery (AID) systems. Enables agentic collaboration where AI agents, caregivers, and automation systems can safely participate in therapy management.

**Key Concepts:**
- Event-driven architecture with EventEnvelope
- ProfileDefinition and OverrideDefinition configurations
- PolicyComposition for effective therapy settings
- DeliveryRequest/Observation tracking
- Authority hierarchy (Human > Agent > Controller)
- Multi-writer conflict resolution

#### [MCP Support Proposal](./mcp-support-proposal.md)
**Status:** Draft  
**Created:** 2026-01-18

Proposes adding Model Context Protocol (MCP) support to enable standardized AI agent integration with Nightscout. MCP provides a universal interface for AI models to securely access glucose monitoring data, treatment information, and automation controls.

**Key Features:**
- MCP Server implementation with Resources, Tools, and Prompts
- Read-only analytics tools (glucose stats, pattern detection)
- Write tools with safety guardrails (treatment logging, override suggestions)
- OAuth-based authentication with delegation grants
- Progressive trust system
- Human-in-the-loop confirmations for high-risk actions

#### [MCP and Agent Control Plane Alignment](./mcp-agent-control-plane-alignment.md)
**Status:** Draft  
**Created:** 2026-01-18

Detailed mapping between MCP implementation and Agent Control Plane architecture. Clarifies how MCP acts as a protocol layer enabling AI agents to interact with the control plane's event-driven architecture.

**Key Alignments:**
- MCP as protocol adapter above control plane
- MCP clients as agent issuers with delegated authority
- Event mapping (MCP tools → control plane events)
- Resource-to-collection mapping
- Shared safety guardrails and conflict resolution
- Coordinated implementation phases

### Identity & Security

#### [OIDC Actor Identity Proposal](./oidc-actor-identity-proposal.md)
**Status:** Draft  
**Created:** 2026-01-01

Proposes OpenID Connect (OIDC) based identity system for Nightscout to support human users, automated controllers, AI agents, and caregiver delegation.

**Key Features:**
- OIDC-based authentication
- Actor types (human, controller, agent, caregiver)
- Role-based access control (RBAC)
- Delegation and impersonation
- Audit trails
- Multi-tenancy support

### Data & API

#### [API Query Normalization](./api-query-normalization.md)
**Status:** Draft  
**Created:** Prior to 2026-01-01

Proposes normalization of API query patterns across Nightscout endpoints for consistency and predictability.

#### [Bridge Rules](./bridge-rules.md)
**Status:** Draft  
**Created:** Prior to 2026-01-01

Documents rules for bridging legacy devicestatus uploads to the new event-driven control plane model.

**Key Concepts:**
- Profile hashing for change detection
- Override diffing for state transitions
- Delivery extraction (suggested → requested → confirmed)
- Idempotency handling

#### [Conflict Resolution](./conflict-resolution.md)
**Status:** Draft  
**Created:** Prior to 2026-01-01

Defines conflict resolution strategies for multi-writer scenarios in Nightscout.

**Key Rules:**
- Authority hierarchy enforcement
- Override composition when multiple active
- Supersession rules
- Rate limiting and flip-flop prevention

### Testing & Quality

#### [Testing Modernization Proposal](./testing-modernization-proposal.md)
**Status:** Draft  
**Created:** Prior to 2026-01-01

Proposes modernization of Nightscout's testing infrastructure and practices.

### Integration

#### [Integration Questionnaire](./integration-questionnaire.md)
**Status:** Draft  
**Created:** Prior to 2026-01-01

Questionnaire for Loop, AAPS, Trio, and other AID systems to understand their capabilities and integration points with the Agent Control Plane.

**Topics:**
- Profiles & overrides representation
- Policy composition
- Delivery fidelity (suggested vs. confirmed)
- Timing & ordering
- Minimal event set commitment

---

## Proposal Relationships

```
┌────────────────────────────────────────────────────────────┐
│                  AI & Agent Integration                     │
│                                                             │
│  ┌──────────────────────┐     ┌──────────────────────┐    │
│  │  MCP Support         │────▶│  Agent Control       │    │
│  │  Proposal            │     │  Plane RFC           │    │
│  └──────────────────────┘     └──────────────────────┘    │
│            │                            │                  │
│            │                            │                  │
│            └────────┬───────────────────┘                  │
│                     │                                      │
│                     ▼                                      │
│         ┌──────────────────────┐                          │
│         │  MCP-Agent Control   │                          │
│         │  Plane Alignment     │                          │
│         └──────────────────────┘                          │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│              Supporting Infrastructure                      │
│                                                             │
│  ┌──────────────────────┐     ┌──────────────────────┐    │
│  │  OIDC Actor Identity │     │  Conflict Resolution │    │
│  │  Proposal            │     │                      │    │
│  └──────────────────────┘     └──────────────────────┘    │
│                                                             │
│  ┌──────────────────────┐     ┌──────────────────────┐    │
│  │  Bridge Rules        │     │  API Query           │    │
│  │                      │     │  Normalization       │    │
│  └──────────────────────┘     └──────────────────────┘    │
│                                                             │
│  ┌──────────────────────┐     ┌──────────────────────┐    │
│  │  Testing Modern-     │     │  Integration         │    │
│  │  ization             │     │  Questionnaire       │    │
│  └──────────────────────┘     └──────────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

### Dependency Graph

```
MCP Support Proposal
    ├─ Requires: Agent Control Plane RFC (Phase 1+)
    ├─ Requires: OIDC Actor Identity Proposal (delegation)
    ├─ Uses: Bridge Rules (for legacy compatibility)
    └─ Uses: Conflict Resolution (for multi-writer safety)

Agent Control Plane RFC
    ├─ Requires: OIDC Actor Identity Proposal (issuer identity)
    ├─ Requires: Conflict Resolution (authority hierarchy)
    ├─ Uses: Bridge Rules (devicestatus → events)
    └─ Uses: API Query Normalization (consistent queries)

MCP-Agent Control Plane Alignment
    ├─ Requires: MCP Support Proposal
    ├─ Requires: Agent Control Plane RFC
    └─ Documents: Integration patterns and shared components
```

---

## Implementation Status

| Proposal | Status | Phase | Target Date |
|----------|--------|-------|-------------|
| Agent Control Plane RFC | Draft | Planning | Q2 2026 |
| MCP Support Proposal | Draft | Planning | Q2 2026 |
| MCP-Agent Alignment | Draft | Planning | Q2 2026 |
| OIDC Actor Identity | Draft | Planning | Q2 2026 |
| Bridge Rules | Draft | Planning | Q2 2026 |
| Conflict Resolution | Draft | Planning | Q2 2026 |
| API Query Normalization | Draft | Planning | TBD |
| Testing Modernization | Draft | Planning | TBD |
| Integration Questionnaire | Active | Data Gathering | Ongoing |

---

## Contributing

To propose a new enhancement:

1. Create a new markdown file in this directory following the template below
2. Add an entry to this README under "Active Proposals"
3. Open a pull request for community review
4. Present in community meetings if significant architectural change

### Proposal Template

```markdown
# RFC: [Title]

**Status:** Draft | Active | Accepted | Implemented | Superseded  
**Authors:** [Name(s)]  
**Created:** YYYY-MM-DD  
**Last Updated:** YYYY-MM-DD  
**Related:** [Links to related proposals]

## Abstract

[1-2 paragraph summary]

## Motivation

[Why is this needed? What problems does it solve?]

## Design

[Detailed technical design]

## Implementation Plan

[Phases, milestones, dependencies]

## Alternatives Considered

[What other approaches were considered and why were they rejected?]

## References

[Links to relevant documentation, specifications, prior art]
```

---

## Questions?

For questions about these proposals:

1. Open an issue in the GitHub repository
2. Join the discussion in Discord (#development channel)
3. Attend community meetings (schedule on nightscout.github.io)

---

## License

All proposals in this directory are released under the same license as Nightscout (AGPL-3.0).
