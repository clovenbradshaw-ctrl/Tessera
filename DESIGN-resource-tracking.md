# Resource Tracking Design

**Scope**: How resources (tangible, financial, service-based) are logged, allocated, and tracked, tagged to an individual, organization, or network.

---

## Problem

Khora tracks *observations about people* (GIVEN/MEANT) but has no concept of *resources that flow to or through* people and organizations. Case managers need to record what resources a client received, organizations need to track inventory and capacity, and networks need aggregate visibility into how resources distribute across their membership.

Resources are not identity data. They don't belong in the vault. They sit alongside bridge interactions as a parallel concern: "what did this person experience?" (observations) vs. "what did this person receive or use?" (resources).

---

## Key Principles

1. **Resources follow the existing hierarchy.** Network defines catalogs. Orgs manage inventory. Individuals receive allocations. This mirrors schema propagation.
2. **Resources are GIVEN-frame events.** A resource allocation is an observable fact — "client received bus voucher on Feb 14" — not an institutional interpretation. The MEANT layer can classify or evaluate resource events later (e.g., "meets HUD service requirement 4.03").
3. **Client sovereignty still applies.** Resource logs in bridge rooms are visible to both parties. Clients can see what was logged against them. Org-level inventory is org-scoped, not client-visible.
4. **No new room types.** Resources use state events in existing rooms (org rooms, bridge rooms, network rooms) rather than introducing new room topology.
5. **EO-traceable.** Every resource event emits EO operations, so the full history of designate → allocate → consume → deplete is auditable.

---

## Entity Model

### Resource Type (catalog entry)

A definition of *what kind of resource* exists. Authored at network or org level. Propagates downward like schema.

```
{
  id:           "rtype_bus_voucher",
  name:         "Bus Voucher",
  category:     "transportation",
  unit:         "voucher",          // unit of measure
  fungible:     true,               // interchangeable units (money, vouchers) vs. unique (housing unit #4B)
  perishable:   false,              // expires or depletes over time
  ttl_days:     90,                 // if perishable: days until expiry from allocation
  tags:         ["transit", "mobility"],
  source:       { level: "network", propagation: "standard" },
  maturity:     "normative",
  constraints:  {
    max_per_client:    10,          // optional: cap per individual per period
    period_days:       30,          // optional: rolling window for cap
    requires_approval: false,       // optional: allocation needs org admin sign-off
    eligible_roles:    ["case_manager", "outreach_worker"]  // who can allocate
  }
}
```

**Categories** (seeded, extensible by network):
- `housing` — beds, units, vouchers, rental assistance
- `financial` — emergency funds, deposits, stipends
- `transportation` — bus passes, ride vouchers, gas cards
- `food` — meals, pantry bags, grocery cards
- `health` — medical supplies, prescriptions, hygiene kits
- `employment` — job training slots, interview clothing, tools
- `legal` — legal aid hours, document preparation
- `education` — tutoring hours, school supplies, enrollment slots
- `general` — catch-all for org-specific items

### Resource Inventory (org-level stock)

Tracks how much of each resource type an org currently holds.

```
{
  resource_type_id:  "rtype_bus_voucher",
  total_capacity:    500,            // total ever provisioned
  available:         342,            // current unallocated
  allocated:         148,            // currently assigned to clients
  reserved:          10,             // held pending approval
  last_restocked:    1707900000000,  // timestamp
  funding_source:    "Grant #2024-A",// optional provenance
  notes:             ""
}
```

### Resource Allocation (individual-level event)

Records a specific resource given to a specific client, logged in the bridge room.

```
{
  id:                "ralloc_abc123",
  resource_type_id:  "rtype_bus_voucher",
  quantity:          2,
  unit:              "voucher",
  allocated_by:      "@provider:matrix.org",
  allocated_to:      "@client:matrix.org",    // redundant with bridge, explicit for clarity
  org_id:            "!orgroom:matrix.org",
  status:            "active",                 // active | consumed | expired | revoked
  allocated_at:      1707900000000,
  expires_at:        1715676000000,            // if perishable
  notes:             "For medical appointments this week",
  approval:          null                      // or { approved_by, approved_at } if requires_approval
}
```

### Resource Event (status change)

Tracks lifecycle transitions of an allocation.

```
{
  allocation_id:  "ralloc_abc123",
  event:          "consumed",        // allocated | consumed | expired | revoked | returned
  quantity:       1,                 // partial consumption supported
  recorded_by:    "@provider:matrix.org",
  recorded_at:    1708000000000,
  notes:          "Used for Tuesday appointment"
}
```

---

## Matrix Event Types

New entries in the `EVT` constant:

```javascript
// Resource tracking
RESOURCE_TYPE:      `${NS}.resource.type`,       // catalog entry (network or org room)
RESOURCE_INVENTORY: `${NS}.resource.inventory`,  // stock levels (org room)
RESOURCE_ALLOC:     `${NS}.resource.allocation`, // individual allocation (bridge room)
RESOURCE_EVENT:     `${NS}.resource.event`,      // lifecycle event (bridge room)
RESOURCE_POLICY:    `${NS}.resource.policy`,     // allocation rules (network or org room)
```

---

## Room Placement

Resources slot into the existing room topology without creating new rooms:

```
NETWORK ROOM
├── (existing: schema, members, governance, hash salt)
├── io.khora.resource.type     ← resource catalog (propagates to orgs)
└── io.khora.resource.policy   ← network-wide allocation policies

ORGANIZATION ROOM
├── (existing: identity, roster, metadata, settings)
├── io.khora.resource.type     ← local resource types (org-specific)
├── io.khora.resource.inventory← current stock per resource type
└── io.khora.resource.policy   ← org-level overrides / additional rules

BRIDGE ROOM (client ↔ provider)
├── (existing: bridge meta, refs, observations, ops, messages)
├── io.khora.resource.allocation ← what this client received
└── io.khora.resource.event      ← consumption/expiry/revocation log
```

Client vault rooms do **not** store resource data. Resources are relational (they happen *between* an org and a client via a provider), so they live in bridge rooms where both parties can see them.

---

## EO Integration

Resource lifecycle maps to EO operators:

| Action | EO Op | Target | Frame | Notes |
|--------|-------|--------|-------|-------|
| Define resource type in catalog | `DES` | `{ entity: rtype_id, designation: name }` | `{ type: 'GIVEN', role: 'network' }` | New resource type created |
| Restock inventory | `INS` | `{ field: 'inventory', entity: rtype_id }` | `{ type: 'GIVEN', role: 'org' }` | Stock added |
| Allocate to client | `INS` | `{ field: 'allocation', entity: ralloc_id }` | `{ type: 'GIVEN', epistemic: 'GIVEN' }` | Observable fact: client received resource |
| Partial consumption | `ALT` | `{ field: 'allocation', entity: ralloc_id }` | `{ type: 'GIVEN' }` | Quantity/status changed |
| Full consumption | `ALT` | `{ field: 'status' }` | `{ type: 'GIVEN' }` | `active` -> `consumed` |
| Expiry | `ALT` | `{ field: 'status' }` | `{ type: 'GIVEN' }` | `active` -> `expired` (can be automated) |
| Revocation | `NUL` | `{ field: 'allocation', entity: ralloc_id }` | `{ type: 'GIVEN' }` | Allocation nullified |
| Classify for reporting | `CON` + `SYN` | `{ source: ralloc_id, destination: 'hud:svc_4.03' }` | `{ type: 'MEANT' }` | Institutional interpretation of resource event |

This means the Activity Stream automatically picks up resource operations with no additional work.

---

## Tagging Model: Individual / Org / Network

The user's core ask: resources tagged to different entity levels. Here's how each level works:

### Individual-Tagged Resources

**Where**: Bridge room (`io.khora.resource.allocation`)

These are concrete allocations to a specific person. The provider records them in the bridge room shared with the client.

- Client can see all resources logged against them across all their bridges
- Provider sees resources they allocated, plus any from other providers in the same org (via org-level aggregation)
- Resource appears in client's case timeline alongside observations

**Privacy**: Bridge-scoped. Other orgs serving the same client via different bridges do not see these allocations unless the client shares that info or the network uses anonymized metrics.

### Org-Tagged Resources

**Where**: Org room (`io.khora.resource.inventory`, `io.khora.resource.type`)

These represent what the organization has, not what any individual received. Inventory management.

- Org admins and authorized staff can view and update inventory
- Restocking, write-offs, transfers between categories
- Aggregate view: "We distributed 148 bus vouchers this month across 67 clients"

**Privacy**: Org-scoped. Visible to org members based on role. Not visible to clients or other orgs.

### Network-Tagged Resources

**Where**: Network room (`io.khora.resource.type`, `io.khora.resource.policy`)

These represent the shared resource vocabulary and network-wide policies. Also where anonymized aggregate metrics flow.

- Network coordinator defines resource type catalog
- Propagation rules control whether orgs must, should, or may track specific resource types
- Anonymized metrics: "Across the network, 12,400 transportation resources were allocated in Q4" — no individual or org attribution unless opted in

**Privacy**: Network-scoped. Individual allocations never appear here. Only catalog definitions, policies, and opt-in anonymized aggregates.

---

## Propagation

Resource types propagate like schema forms:

```
Network defines:  rtype_bus_voucher (propagation: standard)
                  rtype_emergency_fund (propagation: required)
                  rtype_legal_consult (propagation: optional)

Org A receives:   rtype_bus_voucher    ← auto-adopted, can extend locally
                  rtype_emergency_fund ← auto-adopted, immutable
                  rtype_legal_consult  ← visible in catalog, org decides

Org A adds:       rtype_interview_clothes (propagation: local)
                  ← org-specific, not in network catalog
```

The `source` field on each resource type tracks provenance:
- `{ level: 'network', propagation: 'required' }` — from network, must adopt
- `{ level: 'network', propagation: 'standard' }` — from network, expected to adopt
- `{ level: 'org' }` — org-specific, not propagated upward
- `{ level: 'local' }` — provider-specific (rarely used)

---

## Constraints & Policies

Resource policies prevent over-allocation and enforce rules without central authority:

```
{
  resource_type_id:   "rtype_emergency_fund",
  rules: [
    {
      type:           "cap_per_client",
      max_quantity:   500,
      unit:           "dollars",
      period_days:    365,
      scope:          "org"          // per-org cap (vs. "network" for cross-org cap)
    },
    {
      type:           "approval_required",
      threshold:      200,           // amounts above this need approval
      approver_roles: ["admin", "case_manager_senior"]
    },
    {
      type:           "eligible_roles",
      roles:          ["case_manager", "outreach_worker"]
    }
  ]
}
```

**Enforcement is client-side.** The UI checks constraints before allowing allocation. Since Matrix state events are append-only and the EO audit trail captures everything, violations are detectable after the fact even if someone bypasses the UI.

---

## Anonymized Resource Metrics

Follows the existing metrics anonymization pipeline:

```
Individual allocation (bridge room)
  ↓ client opts into resource metrics
  ↓ transform: strip PII, bucket amounts, hash cohort
  ↓
Org metrics room: io.khora.metric (type: 'resource')
  {
    cohort_hash:     "abc...",
    resource_type:   "transportation",     // category, not specific type
    quantity_bucket: "1-5",                // bucketed, not exact
    period:          "2024-Q1",
    ts:              1707900000000
  }
  ↓ org opts into network aggregation
  ↓
Network aggregate: io.khora.metric.aggregate (type: 'resource')
  {
    org_count:       8,                    // number of orgs contributing
    resource_type:   "transportation",
    total_bucket:    "100-500",
    period:          "2024-Q1"
  }
```

k-anonymity: suppress if fewer than 5 clients in a bucket. Same rules as existing observation metrics.

---

## UI Sketch (Conceptual)

### Provider View (Bridge Room)

New tab or section alongside existing observations/messages:

```
┌─────────────────────────────────────────┐
│  Resources for Jane Doe                 │
│─────────────────────────────────────────│
│  [+ Allocate Resource]                  │
│                                         │
│  Feb 14  Bus Voucher ×2      active     │
│          "For medical appts"            │
│          [Mark Used] [Revoke]           │
│                                         │
│  Feb 10  Emergency Fund $150  consumed  │
│          "Security deposit assist"      │
│          Approved by @admin             │
│                                         │
│  Jan 28  Hygiene Kit ×1     consumed    │
│          "Standard intake kit"          │
└─────────────────────────────────────────┘
```

### Org Admin View (Org Room)

Dashboard panel showing inventory and aggregate allocation:

```
┌─────────────────────────────────────────┐
│  Resource Inventory                     │
│─────────────────────────────────────────│
│  Bus Vouchers     342/500    ██████░░   │
│  Emergency Fund   $8,200     ██████░░   │
│  Hygiene Kits     47/100     ████░░░░   │
│                                         │
│  This Month                             │
│  ─────────                              │
│  58 allocations across 34 clients       │
│  Top category: Transportation (62%)     │
│  Pending approvals: 3                   │
│                                         │
│  [Manage Catalog] [Restock] [Export]    │
└─────────────────────────────────────────┘
```

### Client View (Vault)

Read-only summary of resources received across all bridges:

```
┌─────────────────────────────────────────┐
│  My Resources                           │
│─────────────────────────────────────────│
│  From: Nashville Rescue Mission         │
│  Feb 14  Bus Voucher ×2       active    │
│  Feb 10  Emergency Fund $150  used      │
│                                         │
│  From: Legal Aid Society                │
│  Jan 15  Legal Consult ×2hr   used      │
└─────────────────────────────────────────┘
```

### Network View

Aggregate dashboard (no individual data):

```
┌─────────────────────────────────────────┐
│  Network Resource Overview (Q1 2024)    │
│─────────────────────────────────────────│
│  8 orgs reporting · 1,247 allocations   │
│                                         │
│  By Category                            │
│  Transportation  ████████████  42%      │
│  Financial       ██████████   34%       │
│  Health          ██████       18%       │
│  Other           ██            6%       │
│                                         │
│  [View Trends] [Export Report]          │
└─────────────────────────────────────────┘
```

---

## Encryption Behavior

Resources follow the same encryption model as other bridge data:

| Data | Encryption | Rationale |
|------|-----------|-----------|
| Resource type catalog (network/org) | Matrix Megolm only | Not PII, shared vocabulary |
| Resource inventory (org) | Matrix Megolm only | Org-internal, no client PII |
| Resource allocation (bridge) | Megolm + per-field AES-256-GCM | Contains client context, follows bridge encryption model |
| Resource metrics (anonymized) | Megolm only | Already anonymized, no PII |

Allocation `notes` fields may contain sensitive context. They get the same per-provider AES-256-GCM treatment as bridge refs — the client's key encrypts them, and the provider has a copy of that key for their bridge.

---

## Dependencies & Sequencing

Resource tracking depends on:

1. **Organization Entity (Gap 1)** — inventory lives in org rooms. Without orgs, there's nowhere to put stock levels.
2. **Staff Lifecycle (Gap 2)** — `eligible_roles` constraints need org roster roles to exist.
3. **Schema Propagation (existing)** — resource type propagation uses the same mechanism.

Resource tracking does **not** depend on:
- Cross-org matching/dedup
- Provider-side assessments
- Client discovery

**Recommended sequencing**: implement resource tracking after Gaps 1-2 are resolved, alongside or after schema versioning.

---

## Seed Data

Default resource types for initial deployment (similar to how forms have default prompts):

```javascript
const DEFAULT_RESOURCE_TYPES = [
  { id: 'rtype_bus_voucher',       name: 'Bus Voucher',         category: 'transportation', unit: 'voucher',  fungible: true,  perishable: true,  ttl_days: 90 },
  { id: 'rtype_gas_card',          name: 'Gas Card',            category: 'transportation', unit: 'card',     fungible: true,  perishable: false },
  { id: 'rtype_emergency_fund',    name: 'Emergency Fund',      category: 'financial',      unit: 'dollars',  fungible: true,  perishable: false },
  { id: 'rtype_rental_assist',     name: 'Rental Assistance',   category: 'financial',      unit: 'dollars',  fungible: true,  perishable: false },
  { id: 'rtype_shelter_bed',       name: 'Shelter Bed Night',   category: 'housing',        unit: 'night',    fungible: true,  perishable: true,  ttl_days: 1 },
  { id: 'rtype_housing_voucher',   name: 'Housing Voucher',     category: 'housing',        unit: 'voucher',  fungible: false, perishable: true,  ttl_days: 120 },
  { id: 'rtype_hygiene_kit',       name: 'Hygiene Kit',         category: 'health',         unit: 'kit',      fungible: true,  perishable: false },
  { id: 'rtype_meal',              name: 'Meal',                category: 'food',           unit: 'meal',     fungible: true,  perishable: true,  ttl_days: 1 },
  { id: 'rtype_grocery_card',      name: 'Grocery Card',        category: 'food',           unit: 'card',     fungible: true,  perishable: false },
  { id: 'rtype_legal_consult',     name: 'Legal Consultation',  category: 'legal',          unit: 'hour',     fungible: true,  perishable: false },
  { id: 'rtype_job_training_slot', name: 'Job Training Slot',   category: 'employment',     unit: 'slot',     fungible: false, perishable: true,  ttl_days: 30 },
];
```

---

## Open Questions

1. **Resource transfers between orgs**: Should orgs be able to transfer inventory to each other? This adds complexity (inter-org rooms, transfer events, reconciliation). Could defer to a later iteration.

2. **Client self-reporting**: Should clients be able to log resources they received outside the system? E.g., "I got food stamps from DHS." This would be a GIVEN observation, not a formal allocation. Probably handled by existing observation forms rather than the resource system.

3. **Recurring allocations**: Some resources are ongoing (weekly meal program, monthly bus pass). Should the system model recurring schedules, or treat each as a discrete allocation? Discrete is simpler and more accurate for audit purposes.

4. **Resource requests**: Should clients be able to request resources through the system? This flips the flow (client → provider instead of provider → client). Could be a future feature layered on top.

5. **Non-fungible resource identity**: For unique items like housing units, should each unit have its own identity and state history? This starts resembling asset management. Worth considering for housing-focused deployments but adds significant complexity.
