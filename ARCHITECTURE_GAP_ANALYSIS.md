# Khora — Architecture Gap Analysis

> Audit date: 2026-02-08
> Codebase: `index.html` (1,825 lines, single-file React/Matrix SPA)
> Branch: `claude/analyze-codebase-gaps-uPftv`

---

## Executive Summary

Khora has a strong cryptographic and protocol foundation: Matrix as transport/identity, per-field AES-256-GCM encryption, EO operator adjacency with auto-intermediate emission, GIVEN/MEANT epistemic framing, structured observation prompts, anonymized metrics with k-anonymity goals, and a client-sovereign data model.

**The entity model is flat in ways that will break past a two-person demo.** The seven gaps below are ordered by dependency — each one unblocks the next.

---

## Current State: What Exists

| Layer | Status | Lines | Notes |
|-------|--------|-------|-------|
| Matrix transport (KhoraService) | Working | 242–373 | Login, room CRUD, state, events, messaging, invite/kick/tombstone |
| Per-field crypto (FieldCrypto) | Working | 112–135 | AES-256-GCM, generate/encrypt/decrypt |
| EO operations | Working | 137–211 | 9 operators, adjacency matrix, BFS pathfinding, auto-intermediates, state projection |
| Event type constants (EVT) | Defined | 220–239 | 18 custom event types across vault/bridge/roster/schema/metrics/network |
| Room classification | Working | 503–515 | 7 room types: vault, bridge, roster, schema, metrics, network, unknown |
| Client vault + observations | Working | 771–1331 | Vault CRUD, field-level sharing, observation recording, metrics consent, hard revoke |
| Provider dashboard + cases | Working | 1333–1798 | Case list from bridges, shared data decryption, messaging, info requests |
| Schema (prompts/defs/transforms) | Seeded | 377–457 | 5 default prompts, 1 HUD definition, anonymization transforms |
| Network view | Partial | 1716–1764 | Create/join by room ID, propagation model displayed but not enforced |
| Activity Stream | Working | 499–683 | DEV tab with EO event visualization, room topology, filters |
| Account type selection | Binary | 750–769 | Client or Provider — no Organization option |

---

## Gap 1: Organization Entity

### Problem
A "provider" is a Matrix user. There is no entity for "Nashville Rescue Mission" distinct from "@sarah:matrix.org who works there." The `AccountTypeScreen` (line 751) offers only `client` or `provider`. The `ProviderApp` component has no org context.

### What Exists
- `io.khora.identity` state event with `account_type` field (line 222–223)
- Supported values: `client`, `provider`, `schema`, `metrics`, `network` (lines 506–510)
- Provider roster room (lines 1372–1376) — owned by individual user, not org
- Network members store individual user IDs, not org references (line 1477)

### What's Missing
- **Org room type**: A Matrix room (or Space) with `account_type: 'organization'` in `io.khora.identity`
- **Org metadata**: name, type (shelter/outreach/legal-aid/clinic/CoC-lead), service area
- **Staff roster**: `io.khora.roster.index` within org room mapping user IDs to roles
- **Org creation flow**: AccountTypeScreen needs a third path or Provider path fork
- **Org context wrapper**: ProviderApp must operate within an org scope

### Required Event Types
```
io.khora.identity          → add account_type: 'organization'
io.khora.org.roster        → { staff: [{ userId, role, joined, assignments }] }
io.khora.org.metadata      → { name, type, service_area, languages, hours }
```

### Required Roles
`admin`, `case_manager`, `outreach_worker`, `intake_coordinator`, `read_only`

### EO Operations Emitted
- `DES` — designation of org entity on creation
- `INS` — first member added to roster
- `CON` — staff member joins org
- `NUL` — staff member leaves/removed

### Dependency
**Blocks everything.** Without orgs, bridges link to individual users not organizations. Network membership can't be organizational. Case assignment doesn't scale. Role-based access can't exist.

---

## Gap 2: Staff ↔ Org Lifecycle

### Problem
No mechanism for staff to join/leave organizations. No case assignment. No role-based field access.

### What Exists
- Bridge rooms already use `invite`/`kick` (lines 338–346)
- Tombstone pattern exists (line 348–350)
- Provider roster stores cases (line 1375) — but per-user, not per-org

### What's Missing

**Staff join flow:**
1. Org admin clicks "Invite Staff" → enters Matrix ID
2. Khora invites staff to org room with role in state event
3. Staff sees invite on login → accepts → joins org room
4. Dashboard shows org's cases filtered by role + assignments

**Multi-org membership:**
- Person can belong to multiple orgs (volunteer at shelter, full-time at outreach)
- App needs org switcher — sidebar shows which org context is active
- Case access, schema, and network membership flow from org

**Case assignment:**
- State event: `io.khora.roster.assignment` mapping bridge room IDs to staff user IDs
- Outreach worker sees only assigned cases
- Admin sees all org cases

**Bridge model change:**
- Currently: bridge is client ↔ individual provider user (line 897)
- Needed: bridge is client ↔ org, with specific staff granted access by org admin
- Bridge room invites org staff as needed, not just one person
- When staff leave org → access revoked via kick + tombstone

### Required State Events
```
io.khora.roster.assignment → { [bridgeRoomId]: [staffUserId, ...] }
io.khora.org.invite        → { userId, role, invited_by, ts }
```

### Dependency
Requires Gap 1 (Org Entity). Blocks Gap 3 (Org ↔ Network).

---

## Gap 3: Org ↔ Network Membership

### Problem
Network members are individual user IDs, not orgs. The `io.khora.network.members` state event stores `organizations` array but fills it with user IDs (line 1477). Schema propagation isn't enforced.

### What Exists
- Network room creation (lines 1472–1489): creates room with identity, members, seeds schema
- Join network (lines 1492–1503): joins by room ID
- Members displayed (lines 1740–1746): shows ID + role
- Propagation model displayed (lines 1748–1759): Required/Standard/Recommended/Optional levels shown in UI

### What's Missing

**Org-level membership:**
- `io.khora.network.members` should store org room IDs + org metadata
- Org admin is authorized signer; membership is organizational
- Individual staff inherit network context through their org

**Join flow:**
1. Network exists (created by CoC lead or coalition convener)
2. Org admin discovers network (directory, shared link, room ID)
3. Requests to join → network admin reviews and approves
4. Org room receives network schema via propagation
5. Org's `io.khora.identity` updated with network membership

**Schema propagation enforcement:**
- **Required**: prompts/definitions copied to org schema as immutable references
- **Standard**: can extend (add options) but not reduce
- **Recommended**: can modify or replace
- **Optional**: available, no expectation
- Currently these levels are displayed but never enforced (lines 1748–1759)
- Uses EO `REC` operator for governance — recursive descent through hierarchy

### Required State Events
```
io.khora.network.members   → { organizations: [{ org_room_id, org_name, org_type, role, joined }] }
io.khora.network.join_request → { org_room_id, org_name, requested_by, ts }
io.khora.schema.propagation → { prompt_key, level, source_network, version, immutable }
```

### Dependency
Requires Gap 1 (Org Entity) and Gap 2 (Staff Lifecycle). Blocks Gap 4 (Schema Versioning).

---

## Gap 4: Schema Versioning & Propagation Enforcement

### Problem
Schema prompts and definitions have no version tracking. Old observations don't reference the schema version they were recorded under. Network Required prompts can be modified at the org level.

### What Exists
- Default prompts (lines 378–401): 5 prompts with id, key, question, type, options, category
- Default definitions (lines 403–410): 1 HUD FY2025 housing status definition
- Schema seeded to rooms (lines 820–823, 1383–1389): prompts and definitions stored as state events
- Transforms defined (lines 412–416): dob→age_range, address→census_tract, etc.

### What's Missing

**Version tracking:**
```javascript
// Current prompt structure (no version)
{ id:'prompt_sleep_location', key:'sleep_location', question:'...', options:[...] }

// Needed
{ id:'prompt_sleep_location', key:'sleep_location', version: 3,
  question:'...', options:[...],
  previous_version: 2, changed_ts: Date.now(), changed_by: 'network_admin' }
```

**Observation → schema version linkage:**
```javascript
// Current observation (line 875)
{ id, category, value, date, prompt_id, free_text, ts }

// Needed
{ id, category, value, date, prompt_id, schema_version: 3, free_text, ts }
```

**Propagation enforcement:**
- When org joins network, Required prompts become read-only at org level
- Standard prompts: `org_options = network_options + org_extensions` (additive only)
- Schema room needs `io.khora.schema.propagation` state events tracking source + level
- On network schema update, propagation pushes to member orgs

**Transform rules as network governance:**
- Transforms should be centrally defined at network level
- Ensures consistent anonymization across member orgs for metrics comparability

### Required Changes
- Add `version` field to all schema event types
- Add `schema_version` to observation events
- Add `io.khora.schema.propagation` state event type
- Enforce immutability for Required-level propagated prompts in UI
- Add version diff / changelog view

### Dependency
Requires Gap 3 (Org ↔ Network). Independent of Gaps 5–7 but improves data quality for all.

---

## Gap 5: Client ↔ Provider Discovery

### Problem
Client-provider connection requires entering a Matrix ID (line 1286). This is a non-starter for field use. The provider "Find Client" also requires a Matrix ID (line 1775).

### What Exists
- Client adds provider by Matrix ID (lines 893–908): creates bridge, invites
- Provider discovers client by Matrix ID (lines 1449–1469): searches existing bridges

### What's Missing

**Client → Provider discovery paths:**

1. **QR code at service points** (highest priority)
   - Shelter intake desk, outreach van, day center
   - QR encodes: org Matrix room ID + short invite token
   - Client scans → auto-creates bridge invite to org
   - One scan, one tap — frictionless

2. **Provider directory within network**
   - Client browses providers scoped to network
   - "Show me shelters near me" or "show me legal aid"
   - Requires orgs to publish metadata (type, location, hours, languages) to network room

3. **Referral links**
   - Provider sends client a link (SMS, printed card)
   - Pre-populates bridge creation flow

**Provider → Client discovery:**

4. **In-person encounter (QR scan)**
   - Outreach worker meets someone → asks "do you have Khora?"
   - Client shows their QR code (generated from vault room ID)
   - Worker scans → bridge invite created

5. **Provisional encounter record** (no Khora account)
   - Worker creates MEANT observation: "met someone at [location] on [date]"
   - No vault exists yet — provisional record in org roster
   - If person later creates Khora, provisional record can be linked

### Required Components
- QR code generation (client-side, using vault room ID or org room ID + token)
- QR scanner component (camera access via `getUserMedia`)
- Invite token system: short-lived tokens stored as state events
- Provider directory: org metadata published to network room
- Provisional encounter model: new event type `io.khora.encounter.provisional`

### Required Event Types
```
io.khora.invite.token      → { token, org_room_id, created, expires, single_use }
io.khora.encounter.provisional → { location, date, description, created_by, linked_vault: null }
```

### Dependency
QR scanning is independent (can be built now). Directory requires Gap 1 + Gap 3. Provisional encounters require Gap 1.

---

## Gap 6: Provider-Side Observations in Bridge Rooms

### Problem
The code conflates vault schema (client-owned identity fields) and case schema (provider assessments, progress notes, service plans). Provider communication is freeform messages (line 1436–1438), not structured EO events.

### What Exists
- Client observations: structured with prompts, GIVEN frame, stored in vault (lines 873–891)
- Provider messages: freeform `m.room.message` with `io.khora.type: 'note'` or `'request'` (lines 1434–1446)
- Bridge room: two-way encrypted channel (client shares vault fields, provider sends messages)

### What's Missing

**Provider observations as structured EO events:**
- Case notes → `io.khora.op` with MEANT frame, op type INS
- Assessment scores (VI-SPDAT, PHQ-9) → structured prompts with provider-specific schema
- Service plan milestones → structured events with status tracking
- Referral status → structured events with org-level routing

**Provider prompts (distinct from client prompts):**
```javascript
// Client prompt: "Where did you sleep last night?" (GIVEN, self-report)
// Provider prompt: "What is this person's VI-SPDAT score?" (MEANT, professional assessment)
// Provider prompt: "Housing placement status?" (MEANT, case management)
```

**Schema room distinction:**
- Client-facing prompts: questions clients answer about their experience
- Provider-facing prompts: questions providers answer about their assessments
- Schema room needs `audience` field: `'client'` or `'provider'`

### Required Changes
- Add `audience` field to schema prompts: `'client'` (GIVEN) vs `'provider'` (MEANT)
- Create default provider prompts (VI-SPDAT, PHQ-9, housing placement, service plan)
- Formalize provider observations as structured EO events in bridge rooms
- Add provider observation recording UI in case view
- Bridge room becomes two-way structured channel: client GIVENs + provider MEANTs

### Required Event Type Extensions
```
io.khora.schema.prompt     → add { audience: 'client' | 'provider' }
io.khora.observation       → new event type for bridge-level observations
                               { prompt_id, value, frame: 'MEANT', author, schema_version }
```

### Dependency
Independent of Gaps 1–5. Can be built in parallel. Improves case management quality significantly.

---

## Gap 7: Dedup Without Plaintext PII

### Problem
If two orgs encounter the same person, there's no way to know without putting plaintext PII in a shared index. This is the hardest problem and the one that makes or breaks scale.

### What Exists
- `io.khora.network.hash_salt` event type defined (line 238)
- SHA-256 cohort hashing for metrics (lines 213–218)
- Metrics anonymization pipeline (lines 441–469)
- K-anonymity enforcement goal (mentioned in UI but not algorithmically enforced)

### What's Missing

**Layer 1: Client-controlled linking (zero infrastructure)**
- Client knows who they're connected to
- Client enables "coordination mode" → flag in each bridge: "I'm also connected to [org X]"
- Emits `CON` operation to both bridges
- No server-side matching — client is the oracle
- Handles common case: active case manager + day center visits

**Layer 2: Blind deterministic token (opt-in, for coordinated entry)**
```javascript
token = HMAC-SHA256(
  normalize(first_name) + "|" + normalize(last_name) + "|" + dob,
  network_salt
)
// normalize = lowercase, strip whitespace/punctuation, Unicode NFKD
// Truncated to first 16 bytes, base64
// Generated client-side using vault plaintext + network salt
// Only the token crosses the bridge — network matching room never sees PII
```
- Token stored in network-level matching room
- On new bridge creation + client consent → compare against index
- Collision → both providers notified: "This client may also be receiving services from another member org"
- No PII revealed; providers coordinate through client
- **Vulnerability**: brute-force with candidate list + network salt → mitigate with Argon2id + quarterly salt rotation

**Layer 3: Fuzzy matching (advanced, opt-in)**
- For approximate DOB ("about 40") or name variants ("Muhammad"/"Mohammed")
- Multiple tokens from phonetic encoding (Soundex, Double Metaphone) + DOB buckets (year ± 1)
- 2+ token collisions → flag for human review by network-level case manager
- Emits `SUP` (superposition) operation: "these records may be the same entity"

### Required Event Types
```
io.khora.match.token       → { token_hash, network_id, created_ts }
io.khora.match.hit         → { token_a, token_b, orgs_involved, confidence }
io.khora.match.resolve     → { hit_id, resolution: 'confirmed' | 'dismissed', resolved_by }
```

### Required Vault Extension
```javascript
// Add to vault snapshot alongside metrics_consent
matching_consent: {
  enabled: false,
  layers: [],        // ['deterministic', 'fuzzy']
  network_id: null,
  last_token_ts: null
}
```

### Dependency
Layer 1 is independent. Layers 2–3 require Gap 1 (Org) + Gap 3 (Network). Should be built last — most complex, most security-sensitive.

---

## Target Room Topology (At Scale)

```
NETWORK ROOM (CoC / Coalition)
├── io.khora.identity        (account_type: 'network')
├── io.khora.network.members (org IDs + roles)
├── io.khora.network.hash_salt (rotating quarterly)
├── io.khora.schema.prompt   (network-level prompts, versioned)
├── io.khora.schema.definition (network-level defs, versioned)
├── io.khora.match.token     (blind matching index)
│
├── ORG ROOM (Provider Organization)
│   ├── io.khora.identity        (account_type: 'organization')
│   ├── io.khora.org.metadata    (name, type, service_area, hours)
│   ├── io.khora.org.roster      (staff + roles)
│   ├── io.khora.roster.assignment (case → staff mapping)
│   ├── io.khora.schema.prompt   (org extensions, respecting propagation)
│   │
│   └── METRICS ROOM (per-org, anonymized)
│       └── io.khora.metric      (anonymized events, k-anonymity enforced)
│
├── CLIENT VAULT ROOM (client-owned)
│   ├── io.khora.identity        (account_type: 'client')
│   ├── io.khora.vault.snapshot  (encrypted fields + observations + consents)
│   ├── io.khora.vault.providers (bridge index)
│   └── CLIENT SCHEMA ROOM (client-facing prompts)
│
└── BRIDGE ROOM (per client↔org pair)
    ├── io.khora.bridge.meta     (client ↔ org link, status)
    ├── io.khora.bridge.refs     (per-field encrypted refs)
    ├── io.khora.op              (EO operations from both sides)
    ├── io.khora.observation     (structured provider observations, MEANT)
    └── m.room.message             (encrypted communication)
```

---

## Implementation Priority Matrix

| Priority | Gap | Effort | Impact | Blocks |
|----------|-----|--------|--------|--------|
| **P0** | Gap 1: Organization Entity | Medium | Critical | Gaps 2, 3, 5 (partial), 7 |
| **P0** | Gap 6: Provider Observations | Medium | High | Nothing (parallel safe) |
| **P1** | Gap 2: Staff ↔ Org Lifecycle | Medium | High | Gap 3 |
| **P1** | Gap 5: QR Discovery | Low–Medium | High | Nothing (QR is independent) |
| **P2** | Gap 3: Org ↔ Network | Medium | Medium | Gap 4, Gap 7 (L2/L3) |
| **P2** | Gap 4: Schema Versioning | Low | Medium | Nothing |
| **P3** | Gap 7: Dedup/Matching | High | Critical at scale | — |

---

## Cross-Cutting Concerns (Not Gaps, But Notable)

### Offline-first capability
No service worker, no IndexedDB cache, no offline queue. Matrix SDK handles some reconnection, but vault data isn't accessible offline.

### Audit logging
EO operations provide an implicit audit trail, but there's no dedicated audit log view showing who accessed what data when. For HMIS compliance, this matters.

### Error recovery
No retry logic for failed Matrix API calls. No optimistic UI updates. No conflict resolution for concurrent state writes.

### Testing
Zero automated tests. For a system handling PII with cryptographic guarantees, this is a significant risk.

### File structure
Single 1,825-line HTML file. Functional for a prototype but blocks team development, code review, and modular testing. Consider decomposition before implementing Gaps 1–7.

---

## Appendix: Current EVT Constants (line 220–239)

```javascript
const EVT = {
  IDENTITY:        'io.khora.identity',
  OP:              'io.khora.op',
  VAULT_SNAPSHOT:  'io.khora.vault.snapshot',
  VAULT_PROVIDERS: 'io.khora.vault.providers',
  BRIDGE_META:     'io.khora.bridge.meta',
  BRIDGE_REFS:     'io.khora.bridge.refs',
  ROSTER_INDEX:    'io.khora.roster.index',
  SCHEMA_PROMPT:   'io.khora.schema.prompt',
  SCHEMA_DEF:      'io.khora.schema.definition',
  SCHEMA_TRANSFORM:'io.khora.schema.transform',
  SCHEMA_SUB:      'io.khora.schema.subscription',
  METRIC:          'io.khora.metric',
  METRIC_AGG:      'io.khora.metric.aggregate',
  METRIC_SUPPRESS: 'io.khora.metric.suppression',
  NET_MEMBERS:     'io.khora.network.members',
  NET_HASH_SALT:   'io.khora.network.hash_salt',
};
```

### Proposed Additions

```javascript
// Organization
ORG_ROSTER:      'io.khora.org.roster',
ORG_METADATA:    'io.khora.org.metadata',
ORG_INVITE:      'io.khora.org.invite',
ROSTER_ASSIGN:   'io.khora.roster.assignment',

// Discovery
INVITE_TOKEN:    'io.khora.invite.token',
ENCOUNTER_PROV:  'io.khora.encounter.provisional',

// Provider observations
OBSERVATION:     'io.khora.observation',

// Schema versioning
SCHEMA_PROP:     'io.khora.schema.propagation',

// Matching / dedup
MATCH_TOKEN:     'io.khora.match.token',
MATCH_HIT:       'io.khora.match.hit',
MATCH_RESOLVE:   'io.khora.match.resolve',
```
