# Khora Architecture Report

**Sovereign Case Management on the Matrix Protocol**

---

## Overview

Khora is an encryption-first, federated case management system built entirely on the [Matrix](https://matrix.org) protocol. It runs as a single-file React SPA (`index.html`, ~7,400 lines) deployed to GitHub Pages. There is no custom backend — Matrix homeservers (Synapse) serve as the data layer, and all business logic runs in the browser.

The core architectural idea is **client data sovereignty**: the person being served owns their data, controls who can see it field-by-field, and can revoke access at any time. Providers, organizations, and networks operate on top of this through a layered room topology, with a formal **Epistemic Operations (EO)** framework tracking how raw observations become institutional classifications.

---

## Table of Contents

1. [Technology Stack](#1-technology-stack)
2. [The Four Actor Types](#2-the-four-actor-types)
3. [Room Topology](#3-room-topology)
4. [The Client Vault](#4-the-client-vault)
5. [Bridges: Client ↔ Provider Relationships](#5-bridges-client--provider-relationships)
6. [Organizations](#6-organizations)
7. [Networks](#7-networks)
8. [Schema System](#8-schema-system)
9. [Governance](#9-governance)
10. [The GIVEN/MEANT Divide](#10-the-givenmeant-divide)
11. [Epistemic Operations (EO)](#11-epistemic-operations-eo)
12. [Encryption & Security Model](#12-encryption--security-model)
13. [Metrics & Anonymization](#13-metrics--anonymization)
14. [Data Import](#14-data-import)
15. [How It All Fits Together](#15-how-it-all-fits-together)
16. [Current Gaps](#16-current-gaps)

---

## 1. Technology Stack

| Layer | Technology |
|-------|-----------|
| **UI** | React 18, Babel standalone (in-browser JSX transpilation) |
| **Protocol** | Matrix (via matrix-js-sdk v34.11.1) |
| **Encryption** | Web Crypto API — AES-256-GCM per field, Megolm E2EE per room |
| **Local Storage** | IndexedDB (encrypted at rest via HKDF-derived key) |
| **Session** | sessionStorage (Matrix access token, device ID) |
| **Hosting** | GitHub Pages (static HTML, no server) |
| **Webhooks** | n8n (CSV/JSON import, email verification delivery) |

There is no build step. The browser loads React, Babel, and the Matrix SDK from CDNs, then transpiles the embedded JSX at runtime. The entire application ships as a single HTML file.

---

## 2. The Four Actor Types

Khora has four roles. They are **not selected at signup** — instead, the system infers them from Matrix room membership and state events at login.

### Client

The person receiving services. Owns a **vault room** containing their identity fields, case data, and observation history. Controls all access: which providers can see which fields, and can hard-revoke at any time.

**Detected by**: owning a room where `io.khora.identity` has `account_type: 'client'` and `owner` matches the user's Matrix ID.

### Provider

A case worker or service professional. Connects to clients through **bridge rooms**, records provider-side observations (assessments, referrals), and manages cases.

**Detected by**: owning a provider roster room (`account_type: 'provider'`) or being a member of an organization room.

### Org Admin

Manages an organization's staff, settings, and policies. Controls email verification requirements, staff roles, message access rules, and org-level opacity settings.

**Detected by**: appearing in an organization room's staff roster (`io.khora.org.roster`) with `role: 'admin'`.

### Network Coordinator

Operates at the federation layer. Creates and governs the shared schema that member organizations use, manages network membership, and runs governance processes.

**Detected by**: being a member of a network room (`account_type: 'network'`).

A single Matrix user can hold **multiple roles simultaneously** and switch between them without re-authenticating.

---

## 3. Room Topology

Everything in Khora is a Matrix room. There is no external database. Each room type stores specific state events under the `io.khora.*` namespace.

```
NETWORK ROOM
├── Schema (prompts, definitions, propagation levels)
├── Members (list of org room IDs)
├── Governance (proposals, consent rounds)
└── Hash salt (for deterministic matching tokens)

ORGANIZATION ROOM
├── Identity (name, type, service area)
├── Staff roster (user IDs, roles, email verification)
├── Schema (local copy, possibly extended from network)
├── Org settings (opacity, message access, email verification config)
└── Metrics room (anonymized aggregate data)

CLIENT VAULT ROOM (owned by client)
├── Identity (account_type: client)
├── Vault snapshot (all field values, encrypted)
├── Vault providers index (which providers have which keys)
├── Observations (client-recorded, GIVEN frame)
└── EO operation trace

BRIDGE ROOM (one per client↔provider pair)
├── Bridge metadata (shared field list)
├── Bridge refs (per-provider encryption keys)
├── EO operations (observation trace)
└── Messages (encrypted chat)

PROVIDER ROSTER ROOM
├── Identity (account_type: provider)
└── Provider profile
```

### Room Creation & Power Levels

- **Client vault**: Created by client on first login. Client has power level 100 (admin).
- **Bridge**: Created by client when connecting to a provider. Client is PL 100, provider is PL 50. The client can kick the provider; the provider cannot kick the client.
- **Organization room**: Created by an admin. Staff are invited and assigned roles via the roster state event.
- **Network room**: Created by a network coordinator. Organizations are listed by room ID in the members event.

---

## 4. The Client Vault

The vault is the client's sovereign data store. It contains 11 built-in fields across 5 categories:

| Category | Fields | Sensitive |
|----------|--------|-----------|
| **Identity** | Full Name, Date of Birth, ID Number | DoB, ID: yes |
| **Contact** | Email, Phone, Address | Address: yes |
| **Details** | Organization/Affiliation | No |
| **Case** | Case Notes, Documents, Case History | Documents: yes |
| **Sensitive** | Restricted Notes | Yes |

Clients can also define **custom fields** beyond these 11.

### Vault Snapshot

The vault state is persisted to the vault room as an `io.khora.vault.snapshot` state event:

```
{
  fields: { full_name: "...", dob: "...", ... },
  observations: [ { id, category, value, timestamp, ... } ],
  metrics_consent: { enabled: bool, categories: [...] },
  custom_fields: [ { key, label, category, sensitive } ]
}
```

### Field-Level Sharing

Each provider gets a **unique AES-256-GCM key**. When a client shares a field with a provider, the field value is encrypted with that provider's key and placed in the bridge room. Sharing is per-field: a client might share `full_name` and `phone` with Provider A, but only `case_notes` with Provider B.

The provider index (`io.khora.vault.providers`) tracks which providers have bridge rooms and which fields are shared:

```
{
  providers: [
    {
      userId: "@provider:matrix.org",
      bridgeRoomId: "!abc:matrix.org",
      sharedFields: { full_name: true, phone: true },
      providerKey: "base64-encoded-AES-key"
    }
  ]
}
```

---

## 5. Bridges: Client ↔ Provider Relationships

A **bridge** is a dedicated Matrix room connecting one client to one provider. It is the conduit through which encrypted field data, observations, and messages flow.

### Bridge Lifecycle

1. **Creation**: Client initiates. A new room is created with the client as admin (PL 100) and provider as moderator (PL 50).
2. **Field Sharing**: Client selects which vault fields to share. Each shared field is encrypted with the provider's unique key and sent as a bridge ref event.
3. **Messaging**: Both parties can send encrypted messages within the bridge room.
4. **Observations**: Provider records assessments (MEANT frame) as EO operations in the bridge.
5. **Soft Revoke**: Client un-shares specific fields. The provider loses access to those fields going forward but retains the bridge.
6. **Hard Revoke**: Client tombstones the entire bridge room, kicks the provider, and creates a fresh bridge with new encryption keys. The old room is archived and inaccessible.

### Hard Revoke Sequence

```
Client clicks "Hard Revoke"
  → Old bridge room receives m.room.tombstone event
  → Provider is kicked from old bridge
  → All encryption keys for old bridge are discarded
  → New bridge room created with fresh AES-256-GCM keys
  → Only still-authorized fields re-encrypted in new bridge
  → Old bridge visible as archived (tombstone points to new room)
```

**Trade-off**: Hard revoke prevents future access but cannot erase data the provider already cached locally. This is an inherent limitation of offline-capable encrypted systems.

---

## 6. Organizations

Organizations represent service agencies — shelters, clinics, outreach teams. An org is modeled as a Matrix room with specific state events.

### Org Identity & Metadata

```
io.khora.identity → { account_type: 'organization', owner, name, type }
io.khora.org.metadata → { name, type, service_area, languages, hours }
```

Organization types include: `direct_service`, `administrative`, `clinical`, and others.

### Staff Roster

The roster tracks who belongs to the org and what role they hold:

```
io.khora.org.roster → {
  staff: [
    { userId, role, joined, email_verification }
  ]
}
```

**Roles within an organization:**

| Role | Capabilities |
|------|-------------|
| `admin` | Full control: staff management, settings, schema |
| `case_manager` | Case access, data requests, observations |
| `field_worker` | Field-level case access |
| `intake_coordinator` | Initial assessments and intake |
| `read_only` | View-only access to authorized cases |

### Org Settings

Organizations configure several policies:

- **Opacity** (`io.khora.org.opacity`): Controls how much org identity is revealed in inter-org communication (full identity, pseudonymous, or anonymous).
- **Email Verification** (`io.khora.org.email_verification`): Optional domain-based email verification for staff, with configurable required domains, grace periods, and role requirements.
- **Message Access** (`io.khora.org.msg_access`): Rules for which staff roles can participate in inter-org messaging channels.

---

## 7. Networks

A network is the **federation layer** — a room where multiple organizations coordinate on shared schema, governance, and (eventually) cross-org client matching.

### Network Structure

```
io.khora.identity → { account_type: 'network' }

io.khora.network.members → {
  organizations: [
    { org_room_id, org_name, org_type, role, joined }
  ]
}

io.khora.network.hash_salt → {
  salt: "random-string",
  rotated: timestamp
}
```

The hash salt enables **deterministic matching tokens** — different organizations can independently compute the same hash for the same client without sharing plaintext PII. This is the foundation for the (not yet implemented) deduplication system.

### What Networks Govern

- **Schema**: The prompts and definitions that member organizations use. Networks author the canonical schema; orgs receive it according to propagation rules.
- **Governance**: Structured proposal and consent processes for schema changes.
- **Membership**: Which organizations belong and what roles they play.
- **Matching**: Cross-org client coordination via blind tokens (planned).

---

## 8. Schema System

The schema system defines **what data is collected and how it is classified**. It operates at three levels: network, organization, and client.

### Schema Components

**Prompts** (`io.khora.schema.prompt`): Questions asked to clients or providers.

```
{
  id: 'prompt_current_status',
  key: 'current_status',
  question: 'What is your current status?',
  type: 'single_select',
  audience: 'client',        // or 'provider'
  maturity: 'normative',     // draft | trial | normative | de_facto | deprecated
  source: { level: 'network', propagation: 'required' },
  options: [
    { v: 'active', l: 'Active — currently receiving services' },
    { v: 'on_hold', l: 'On hold — temporarily paused' },
    ...
  ]
}
```

**Definitions** (`io.khora.schema.definition`): Classification rules that derive institutional categories from raw observation values.

```
{
  key: 'priority_level',
  type: 'classification_rule',
  rules: [
    { category: 'priority_a', criteria: 'current_status IN (active) AND assessment_score >= 7' },
    { category: 'priority_b', criteria: 'intake_event=eligibility_confirmed AND engagement_type != none' }
  ]
}
```

**Authorities** (`io.khora.schema.authority`): External institutional standards that definitions reference (HUD definitions, CoC policies, local ordinances).

**Bindings** (`io.khora.schema.binding`): Mappings from local prompt option values to framework-specific codes. A single observation value (e.g., "vehicle") may map to different classifications under different authorities (e.g., HUD: "Literally Homeless Category 1", CoC: "Priority 1 Immediate Outreach").

**Transforms** (`io.khora.schema.transform`): Anonymization rules for metrics (age bucketing, geohashing, field blocking).

### Propagation Levels

When a network publishes schema, each element is tagged with a propagation level that controls how member organizations can interact with it:

| Level | Behavior |
|-------|----------|
| **Required** | Auto-applied to member orgs. Immutable at org level. |
| **Standard** | Auto-applied. Orgs can extend (additive changes only). |
| **Recommended** | Org reviews and decides whether to adopt. |
| **Optional** | Available in the network catalog. No expectation of adoption. |

---

## 9. Governance

Governance is the process by which networks make collective decisions about schema changes. It is built around **consent-based decision-making** rather than majority vote.

### Proposal Lifecycle

```
submitted → discussion → consent_round → resolved / adopted / blocked
```

A network coordinator creates a proposal (e.g., "Add a new prompt for housing stability"). Member organizations review it and submit a **consent position**:

| Position | Meaning |
|----------|---------|
| `adopt_as_is` | Accept the proposal unchanged |
| `adopt_with_extension` | Accept and add local variants |
| `needs_modification` | Accept only if specific changes are made |
| `cannot_adopt` | Block — this single position blocks adoption |

The `cannot_adopt` position functions as a **veto**. This reflects the consent model's principle that no organization should be forced to adopt schema it genuinely cannot implement.

### Governance Rhythms

Networks can define recurring decision cycles:

| Rhythm | Duration | Participants | Threshold |
|--------|----------|-------------|-----------|
| **Monthly Schema Review** | 5 days | Org admins | Consent |
| **Quarterly Alignment** | 5 days | Network stewards + affected orgs | Consent |
| **Annual Constitutional** | 14 days | All members | Supermajority |

### EO Tracing of Governance

Every governance decision is traced through EO operations:

- **DES**: A proposal is designated
- **CON**: The proposal is connected to an authority or framework
- **SEG**: The proposal's scope is segmented (e.g., excluding certain categories)
- **SYN**: A local value is synchronized with an external standard
- **SUP**: Org-level and network-level definitions coexist in superposition

---

## 10. The GIVEN/MEANT Divide

This is Khora's core conceptual innovation. Every piece of data in the system exists in one of two **epistemic frames**:

- **GIVEN** — What actually happened. The client's direct observation, unmediated by institutional interpretation. ("I slept in my car last night.")
- **MEANT** — What an institution classifies that observation as. The institutional meaning derived from the GIVEN via framework bindings. ("Under HUD 24 CFR 578.3, this person is Literally Homeless, Category 1.")

### Why This Matters

The same ground-truth observation produces **different institutional classifications** depending on which framework interprets it. A person sleeping in a vehicle is:

- **HUD**: Literally Homeless (Category 1) — eligible for CoC-funded services
- **CoC Local Policy**: Priority 1 — immediate outreach required
- **VA**: May not qualify under VA-specific homelessness criteria

Without the GIVEN/MEANT divide, these classifications are invisible. The system just records "homeless" without tracing which definition of "homeless" was applied, or that the client's actual statement was about sleeping in a car.

### Visualization

The UI renders a three-column layout:

```
┌─────────────────┬────────────┬───────────────────────┐
│     GIVEN       │  CROSSING  │        MEANT          │
│                 │            │                       │
│ Where did you   │  CON, SYN  │ Framework: HUD        │
│ sleep last      │     →      │ → Literally Homeless  │
│ night?          │            │                       │
│                 │            │ Framework: CoC        │
│ Answer: Vehicle │            │ → Priority 1 Outreach │
└─────────────────┴────────────┴───────────────────────┘
```

The center column shows which EO operators connect the GIVEN to each MEANT, making the interpretive chain explicit and auditable.

When frameworks **disagree** about the same observation, a **SUP (superposition)** badge appears — explicitly marking epistemic divergence.

---

## 11. Epistemic Operations (EO)

EO is a formal state machine that traces the lifecycle of every entity (field, observation, classification) in the system. Every change is an operation with a typed operator.

### Operators

| Operator | Name | Purpose |
|----------|------|---------|
| **NUL** | Nullify | Entity deleted or unset |
| **DES** | Designate | Entity created; new field defined |
| **INS** | Insert | Field populated for the first time |
| **SEG** | Segment | Scope narrowed (e.g., categories excluded) |
| **CON** | Connect | Linked to an external authority or framework |
| **SYN** | Synchronize | Local value mapped to an external standard code |
| **ALT** | Alter | Field value changed |
| **SUP** | Superposition | Multiple valid interpretations coexist |
| **REC** | Recurse | Hierarchical descent into sub-entity |

### Adjacency Matrix

Not all transitions are legal. The system enforces an adjacency matrix:

```
NUL → NUL, DES
DES → NUL, DES, INS
INS → DES, INS, SEG
SEG → INS, SEG, CON
CON → SEG, CON, SYN
SYN → CON, SYN, ALT
ALT → SYN, ALT, SUP
SUP → ALT, SUP, REC
REC → NUL, SUP, REC
```

If a requested transition is not directly adjacent, the system uses BFS to find the shortest valid path and **auto-emits intermediate operations**. For example, jumping from DES to CON would auto-emit INS and SEG in between.

### State Projection

Given a history of EO operations, the system can **project the current state** of any entity by replaying the operation chain. The `projectCurrentState()` function filters by epistemic frame (GIVEN or MEANT) and computes the latest value for each field, handling INS, ALT, NUL, SUP, and DES operations.

### Activity Stream

The Activity Stream view renders all EO events across the user's rooms with:

- Filters by room type, operator, and epistemic frame
- Color-coded room types (vault: teal, bridge: gold, org: blue, schema: purple, metrics: orange, network: green)
- Expandable event cards with raw operation content
- Running statistics (total events, GIVEN count, MEANT count)

---

## 12. Encryption & Security Model

Khora implements encryption at three layers, each defending against different threats.

### Layer 1: Matrix E2EE (Megolm)

All room events — state and messages — are encrypted via Matrix's built-in Megolm protocol. This protects against:

- Network eavesdroppers
- Curious homeserver administrators
- Any party not in the room's member list

### Layer 2: Per-Field AES-256-GCM

On top of Matrix E2EE, sensitive vault fields are encrypted with **per-provider AES-256-GCM keys**. Each provider connected to a client gets a unique key. This means:

- Even if two providers are in the same Matrix room, they cannot read each other's field data
- Revoking a provider means discarding their key — no re-encryption of the vault needed
- The client generates and distributes all keys

```
FieldCrypto.encrypt(plaintext, keyB64):
  → Generate 12-byte random IV
  → AES-256-GCM encrypt
  → Return { ciphertext: base64, iv: base64 }
```

### Layer 3: Local Storage Encryption

All data cached in IndexedDB is encrypted at rest using a key derived from the user's Matrix session:

```
LocalVaultCrypto.deriveKey(userId, accessToken, deviceId):
  → HKDF-SHA256(accessToken, salt=userId+deviceId)
  → Returns AES-256-GCM key
  → Key held in memory only; destroyed on logout
```

### Access Control

- **Room-level**: Matrix power levels control who can read/write state events in a room
- **Field-level**: Per-provider encryption keys control which specific fields a provider can decrypt
- **Role-level**: Org roster roles (admin, case_manager, field_worker, etc.) determine UI capabilities

### Threat Model

**Protected against**: Network eavesdropping, homeserver admin snooping, unauthorized room members, device theft (when locked/logged out).

**Not protected against**: Compromised device with active session (key is in memory), metadata analysis (room membership and message timing are visible to the homeserver), data already cached by a revoked provider.

---

## 13. Metrics & Anonymization

Khora separates **operational data** (client-owned, encrypted, per-field access) from **anonymized metrics** (aggregate, de-identified, org-level).

### Anonymization Pipeline

When a client consents to metrics, their observations are transformed before emission:

| Field | Method | Output |
|-------|--------|--------|
| `dob` | Age range bucketing | `"25-34"` |
| `address` | Area geohash | `"area_abc123"` |
| `full_name` | Blocked | Removed entirely |
| `id_number` | Blocked | Removed entirely |
| `phone` | Blocked | Removed entirely |
| `email` | Blocked | Removed entirely |

### Cohort Hashing

For cross-org coordination, Khora uses **deterministic cohort hashes**:

```
cohortHash(userId, networkSalt) → SHA-256(userId + salt)
```

This produces a stable identifier for the same person across organizations without revealing their Matrix ID. The network salt is rotated periodically and stored in the network room.

### Metric Events

Anonymized metrics are emitted to the organization's metrics room as `io.khora.metric` events containing only the cohort hash, anonymized demographics, and observation categories — never raw PII.

---

## 14. Data Import

Providers can bulk-import client records from CSV or JSON files via the data import engine.

### Process

1. **Parse**: Read CSV/JSON file in the browser
2. **Auto-map**: Column names are matched to vault fields using an alias table (e.g., "First Name", "fname", "given_name" all map to `full_name`)
3. **Per-record encryption**: Each imported record gets a unique AES-256-GCM key
4. **Room creation**: A `client_record` room is created for each row (separate from client vault rooms — these are provider-managed records, not client-sovereign data)
5. **Metrics emission**: Only anonymized demographics are emitted; all PII stays encrypted in the record room

### Integration

The import engine connects to an n8n webhook endpoint for external data sources (Airtable, Postgres) via:

- `POST /api/tables` — Fetch available table schemas
- `GET /api/records/{table_id}` — Fetch records from a table

---

## 15. How It All Fits Together

### Complete Data Flow: Client Observation → Institutional Classification

```
1. CLIENT records an observation
   "Where did you sleep last night?" → "Vehicle"
   ↓
2. EO operation emitted to vault room
   { op: 'INS', target: { field: 'sleep_location' },
     value: 'vehicle', frame: { epistemic: 'GIVEN' } }
   ↓
3. FRAMEWORK BINDINGS activate
   Local value 'vehicle' looked up against all bound frameworks
   → HUD 578.3: code_7 → "Literally Homeless (Category 1)"
   → CoC Policy: Priority_1 → "Immediate Outreach"
   ↓
4. PROVIDER sees the case update
   GIVEN/MEANT divide shows the original answer alongside
   every institutional interpretation, with EO operators
   connecting them (CON, SYN)
   ↓
5. METRICS (if client consents)
   Anonymized observation + demographics (age_range, geo_hash)
   emitted to org metrics room. No PII leaves the client device.
```

### Complete Flow: Organization Joins a Network

```
1. NETWORK exists with published schema
   Prompts, definitions, authorities, propagation levels
   ↓
2. ORG ADMIN discovers network (via room ID or referral)
   Requests membership
   ↓
3. NETWORK COORDINATOR approves
   Org added to io.khora.network.members
   ↓
4. SCHEMA PROPAGATION
   Network schema copied to org room
   - Required prompts: read-only at org level
   - Standard: org can extend (additive only)
   - Recommended: org can adopt or ignore
   - Optional: available in catalog
   ↓
5. GOVERNANCE BEGINS
   Network proposes schema change
   Status: submitted → discussion → consent_round
   ↓
6. ORGS RESPOND with consent positions
   adopt_as_is / adopt_with_extension / needs_modification / cannot_adopt
   ↓
7. RESOLUTION
   If no org blocks: schema version propagates to all members
   Full EO trace documents the decision chain
```

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    NETWORK ROOM                         │
│  Schema (canonical) · Members · Governance · Hash Salt  │
└────────────┬──────────────────────────┬─────────────────┘
             │                          │
    ┌────────▼────────┐       ┌────────▼────────┐
    │   ORG ROOM A    │       │   ORG ROOM B    │
    │ Staff roster    │       │ Staff roster    │
    │ Local schema    │       │ Local schema    │
    │ Settings/policy │       │ Settings/policy │
    │ Metrics room    │       │ Metrics room    │
    └──┬─────────┬────┘       └──┬──────────────┘
       │         │               │
  ┌────▼───┐ ┌──▼─────┐    ┌───▼────┐
  │BRIDGE 1│ │BRIDGE 2│    │BRIDGE 3│
  │OrgA↔C1 │ │OrgA↔C2 │    │OrgB↔C1 │
  └────┬───┘ └──┬─────┘    └───┬────┘
       │        │               │
  ┌────▼────────▼───┐    ┌────▼─────────┐
  │ CLIENT 1 VAULT  │    │ CLIENT 2     │
  │ Fields (enc)    │    │ VAULT        │
  │ Observations    │    │              │
  │ EO trace        │    │              │
  └─────────────────┘    └──────────────┘

  Legend:
  ──── Matrix room membership
  All rooms use Megolm E2EE
  Bridge rooms add per-field AES-256-GCM
  Client vaults add IndexedDB encryption at rest
```

### Authentication Flow

```
Browser loads index.html
  → User enters Matrix homeserver + credentials
  → matrix-js-sdk authenticates via password
  → Access token stored in sessionStorage
  → Device crypto initialized (Megolm key exchange)
  → KhoraService.scanRooms() fetches state from all joined rooms
  → Role inference: check io.khora.identity events across rooms
  → Route to ClientApp or ProviderApp based on detected roles
```

---

## 16. Current Gaps

The system defines several subsystems that are architecturally sound but not yet fully implemented in code.

### Organization lifecycle (Critical)
No UI for creating organizations, inviting staff, or managing the join/leave workflow. Bridges currently connect to individual provider users, not to organizations.

### Schema propagation enforcement
Propagation levels (required/standard/recommended/optional) are displayed but not enforced. An org can currently ignore a "required" prompt without system-level prevention.

### Client ↔ Provider discovery
Connecting requires entering a raw Matrix ID. No QR codes, no referral links, no directory — impractical for field use.

### Provider-side structured observations
Provider observations (the MEANT frame) are defined in the schema but the UI only supports freeform case notes, not structured prompt-based assessments.

### Cross-org deduplication
The cohort hashing infrastructure exists but blind matching (deterministic tokens, fuzzy matching, resolution workflow) is not yet implemented.

### Governance enforcement
Proposals and consent positions are defined but the governance workflow does not automatically apply or block schema changes based on voting outcomes.

### Test coverage
Zero automated tests. The cryptographic layer, EO state machine, and role inference logic all lack test coverage.

---

*This document describes the architecture as of the current codebase. Khora is under active development; subsystems marked as gaps are designed but awaiting implementation.*
