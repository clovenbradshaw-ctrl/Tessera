# Forms Inbox View — Design Document

## Problem

The current Schema Builder treats forms as isolated objects. The "Form Library" is a small modal listing saved forms with version numbers — there's no spatial sense of *where* a form sits relative to other forms, which organization authored it, what project it belongs to, or how versions relate over time. Users need to see **the forest and the tree simultaneously**: navigate a structured collection of forms while working on one in focused detail.

---

## Design Principles

1. **Inbox metaphor, not file-browser metaphor** — Forms arrive from networks, orgs, and collaborators. They're closer to messages than static files. Show recency, unread/changed status, and provenance.
2. **Context over chrome** — The active form gets most of the screen. The sidebar reveals *just enough* structure to orient you without stealing focus.
3. **Propagation is visible** — Forms flow down the hierarchy (Network → Org → Provider → Client). The inbox should make the *source* and *direction* of each form legible at a glance.
4. **Versions are a timeline, not a list** — Show version progression as a compact visual timeline, not a flat table.

---

## Layout: Three-Zone Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TOOLBAR (top bar — breadcrumb + actions)                               │
├──────────────┬──────────────────────────────────────────┬───────────────┤
│              │                                          │               │
│   SIDEBAR    │          MAIN CANVAS                     │  CONTEXT      │
│   (forms     │          (active form —                  │  PANEL        │
│    inbox)    │           Schema Builder                 │  (relations,  │
│              │           as it exists)                  │   lineage,    │
│   280px      │                                          │   versions)   │
│   fixed      │          flex: 1                         │               │
│              │                                          │   260px       │
│              │                                          │   collapsible │
│              │                                          │               │
└──────────────┴──────────────────────────────────────────┴───────────────┘
```

### Zone 1: Sidebar — Forms Inbox (left, 280px)

The sidebar replaces the current modal-based "Form Library." It is always visible when in Schema view, providing persistent navigation.

#### Structure

```
┌─────────────────────────┐
│ FORMS              [+]  │  ← header + new form button
├─────────────────────────┤
│ 🔍 Search / filter...   │  ← search bar
├─────────────────────────┤
│ ▸ INBOX (3)             │  ← incoming forms from network/org
│   ┌───────────────────┐ │
│   │ ● Status & Engag… │ │  ← dot = unreviewed change
│   │   Network · v2    │ │     source + version
│   │   normative       │ │     maturity badge
│   ├───────────────────┤ │
│   │ ● Intake Event    │ │
│   │   Network · v1.1  │ │
│   │   trial  ▲updated │ │  ← "updated" flag when newer
│   ├───────────────────┤ │     than local copy
│   │   Context Form    │ │
│   │   Network · v1    │ │
│   │   draft           │ │
│   └───────────────────┘ │
├─────────────────────────┤
│ ▸ MY FORMS (2)          │  ← locally authored
│   ┌───────────────────┐ │
│   │ ■ Housing Assess… │ │  ← filled square = active/editing
│   │   Local · v3      │ │
│   │   trial           │ │
│   ├───────────────────┤ │
│   │   Risk Screen     │ │
│   │   Local · v1      │ │
│   │   draft           │ │
│   └───────────────────┘ │
├─────────────────────────┤
│ ▸ ORG FORMS (4)         │  ← forms from my organization
│   (collapsed by default)│
├─────────────────────────┤
│ ▸ NETWORK COMMONS (8)   │  ← full schema commons catalog
│   (collapsed by default)│
└─────────────────────────┘
```

#### Grouping Rules

| Group | Source | What goes here |
|-------|--------|----------------|
| **INBOX** | Network/Org push | Forms with `propagation: required\|standard` that have been updated since last viewed. Auto-clears when opened. |
| **MY FORMS** | Local saves | Forms saved via the current "Save" button. These are the user's working copies. |
| **ORG FORMS** | Org room state | Forms published by the user's organization (from `io.khora.schema.form` events in org room). |
| **NETWORK COMMONS** | Network room state | All forms in the network schema room. Read-only unless user has network admin role. |

#### Item States

Each form row shows:
- **Dot indicator** (left edge): `●` unread/updated, `■` currently active, `○` read/unchanged, none for stale
- **Name** (truncated with ellipsis)
- **Source tag**: `Network` / `Org` / `Local` — color-coded (teal/blue/muted)
- **Version**: `v{n}` in mono font
- **Maturity badge**: `draft` / `trial` / `normative` / `deprecated` — same badges as current UI
- **Update flag** (conditional): `▲ updated` in gold when network/org version > local version

#### Interactions

- **Click** → loads form into Main Canvas (Schema Builder)
- **Right-click / long-press** → context menu: Duplicate, Delete, Export, Compare with...
- **Drag** → reorder within "My Forms" group
- **Collapse/expand** group headers with `▸` / `▾`
- **Search** filters across all groups by name, key, or field content

---

### Zone 2: Main Canvas (center, flex)

This is the **existing Schema Builder** (`FormBuilder` component) — compose/wire/preview modes. No structural change needed here. The only modification is:

- Remove the floating "Library" button from the toolbar (sidebar replaces it)
- Keep Save, Version Bump, History, and mode toggle buttons
- The form loaded from the sidebar populates this view
- Add a subtle breadcrumb at top: `My Forms / Housing Assessment / v3`

---

### Zone 3: Context Panel (right, 260px, collapsible)

This panel shows **relational context** for the currently active form. It answers: "Where does this form sit in the bigger picture?"

#### Sections

```
┌─────────────────────────┐
│ CONTEXT           [◀]   │  ← collapse toggle
├─────────────────────────┤
│                         │
│ LINEAGE                 │
│ ┌─────────────────────┐ │
│ │ Network: CoC Net    │ │  ← which network defined it
│ │ ↓ required          │ │     propagation level
│ │ Org: Harbor House   │ │  ← which org adopted it
│ │ ↓ extended (+2 fld) │ │     local extensions noted
│ │ Provider: You       │ │  ← current user's copy
│ │ ↓ active            │ │
│ │ → 12 clients        │ │  ← how many clients have it
│ └─────────────────────┘ │
│                         │
├─────────────────────────┤
│                         │
│ VERSION TIMELINE        │
│                         │
│  v1 ──○── Mar 2025     │  ← dot timeline, vertical
│         "Initial"       │     notes shown inline
│  v2 ──○── Jun 2025     │
│         "+housing q's"  │
│  v3 ──●── Feb 2026     │  ← filled dot = current
│         "Risk section"  │
│                         │
│  [Compare versions]     │  ← opens diff modal
│                         │
├─────────────────────────┤
│                         │
│ RELATED FORMS           │
│ ┌─────────────────────┐ │
│ │ Intake Event        │ │  ← shares fields or
│ │  2 shared fields    │ │     crosswalks with this form
│ │  1 crosswalk        │ │
│ ├─────────────────────┤ │
│ │ Additional Context  │ │
│ │  1 shared field     │ │
│ └─────────────────────┘ │
│                         │
├─────────────────────────┤
│                         │
│ USED BY ORGS            │
│ ┌─────────────────────┐ │
│ │ Harbor House (you)  │ │  ← orgs in the network
│ │ PATH Services       │ │     using this form
│ │ Salvation Army      │ │
│ │ 3 of 5 adopted      │ │  ← adoption count
│ └─────────────────────┘ │
│                         │
├─────────────────────────┤
│                         │
│ ACTIVITY                │
│  Feb 14 — You edited   │  ← recent EO operations
│  Feb 12 — v3 bumped    │     on this form
│  Jan 30 — Network sync │
│                         │
└─────────────────────────┘
```

#### Context Panel Collapse Behavior

- Default: **open** on desktop (>1100px), **closed** on narrow screens
- Toggle button `[◀]` / `[▶]` in panel header
- When collapsed, a thin 36px strip with icons remains visible for quick re-open

---

## Data Model Extensions

### New State Required

```javascript
// Per-form metadata (extend existing savedForms array items)
{
  id,
  name,
  key,
  version,
  maturity,
  savedAt,
  form,             // existing — full form snapshot
  frameworks,       // existing
  bindings,         // existing
  crosswalks,       // existing

  // ── NEW ──
  sourceType,       // 'local' | 'org' | 'network'
  sourceId,         // org room ID or network room ID (null for local)
  sourceName,       // display name of source
  propagation,      // 'required' | 'standard' | 'recommended' | 'optional'
  lastViewedAt,     // timestamp — for unread detection
  lastSyncedVersion,// version number last synced from upstream
  linkedFormKeys,   // [key] — form keys that share fields or have crosswalks
  adoptedByOrgs,    // [{orgId, orgName, localVersion, extensions}]
  clientCount,      // number of clients with this form active
  activityLog,      // [{ts, action, actor, detail}] — recent EO ops
}
```

### Derived Computations

```
isUnread(form)     = form.version > form.lastSyncedVersion
                     || form.savedAt > form.lastViewedAt

relatedForms(form) = allForms.filter(f =>
                       f.key !== form.key &&
                       (sharedFields(f, form).length > 0
                        || hasCrosswalk(f, form)))

sharedFields(a, b) = intersection of field keys between two forms

hasCrosswalk(a, b) = crosswalks.some(xw =>
                       xw.from === a.key && xw.to === b.key
                       || xw.from === b.key && xw.to === a.key)
```

---

## Interaction Flows

### Flow 1: Open Schema View (first load)

1. Sidebar loads with groups populated:
   - **INBOX**: forms from network/org with `savedAt > lastViewedAt`
   - **MY FORMS**: locally saved forms from IndexedDB
   - **ORG FORMS**: from org room state events
   - **NETWORK COMMONS**: from network room state events
2. If there's a form with unread updates, it auto-selects in the inbox.
3. Otherwise, most recently edited local form loads.
4. Context panel populates with selected form's lineage.

### Flow 2: Network pushes an updated form

1. New version of "Status & Engagement" arrives via Matrix state event.
2. INBOX group count increments, form shows `▲ updated` badge.
3. User clicks → form loads in canvas.
4. Context panel shows version timeline with new version highlighted.
5. If breaking change (major version bump), answer crosswalk modal triggers.
6. User reviews, optionally extends, saves to "My Forms."
7. INBOX clears the `●` indicator.

### Flow 3: Creating and linking forms

1. User clicks `[+]` in sidebar header → new blank form in "My Forms."
2. User builds form in Schema Builder as today.
3. After adding fields, Context Panel auto-detects shared field keys with existing forms.
4. "RELATED FORMS" section populates showing overlap.
5. User can click a related form to open a split-diff view.

### Flow 4: Comparing versions

1. User clicks a version dot in the Version Timeline.
2. Side-by-side diff opens (reuses existing `diffFormVersions` logic).
3. Changes highlighted: green = added, red = removed, gold = renamed.
4. "Restore this version" button available for rollback.

---

## Visual Design Notes

### Color Language

| Element | Color | Meaning |
|---------|-------|---------|
| Teal (`--teal`) | Network source | Form comes from network commons |
| Blue (`--blue`) | Org source | Form comes from organization |
| Muted (`--tx-2`) | Local source | Form is locally authored |
| Gold (`--gold`) | Update/attention | New version available, action needed |
| Green (`--green`) | Normative/stable | Maturity = normative |
| Purple (`--purple`) | Classification | MEANT-side interpretation link |
| Red (`--red`) | Deprecated/breaking | Deprecated maturity or major version break |

### Typography

- **Sidebar form names**: 12.5px, Manrope 600, `--tx-0`
- **Sidebar metadata**: 10px, IBM Plex Mono 400, `--tx-2`
- **Context panel headers**: 10px, IBM Plex Mono 600, `--tx-3`, letter-spacing 0.08em (matches existing `section-label` class)
- **Version timeline dates**: 10px, IBM Plex Mono 400, `--tx-2`
- **Breadcrumb**: 11px, Manrope 400, `--tx-2`, separator `/` in `--tx-3`

### Responsive Behavior

| Breakpoint | Sidebar | Context Panel | Canvas |
|------------|---------|---------------|--------|
| >1100px | 280px visible | 260px visible | flex |
| 900–1100px | 260px visible | collapsed (36px strip) | flex |
| <900px | overlay drawer (hamburger toggle) | hidden (accessible via tab) | full width |

### Animation

- Sidebar group expand/collapse: 150ms ease-out height transition
- Form selection: 100ms background-color fade
- Context panel slide: 200ms ease-out transform
- Unread dot pulse: 2s infinite subtle opacity pulse (0.6 → 1.0)

---

## Wireframe: Full View

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Schema Builder    My Forms / Housing Assessment / v3              [Save] [⋮]   │
├──────────┬───────────────────────────────────────────────────────┬───────────────┤
│ FORMS [+]│  Housing Assessment                                  │ CONTEXT    [◀]│
│ ┄┄┄┄┄┄┄┄ │  trial · v3 · 8 questions · 24 options · 2 fw       │               │
│ 🔍 search│  ┌─── Compose ─── Wire ─── Preview ───┐             │ LINEAGE       │
│          │  │                                      │             │ CoC Network   │
│ ▾ INBOX 3│  │  ▾ General                           │             │ ↓ required    │
│ ● Status…│  │    ┌──────────────────────────┐      │             │ Harbor House  │
│   Net v2 │  │    │ What is your current     │      │             │ ↓ +2 fields   │
│   normat…│  │    │ housing situation?       │      │             │ You (active)  │
│ ● Intake…│  │    │ ☐ Sheltered              │      │             │ → 12 clients  │
│   Net v1…│  │    │ ☐ Unsheltered            │      │             │               │
│   trial ▲│  │    │ ☐ At risk                │      │             │ VERSION       │
│   Context│  │    │ ☐ Stably housed          │      │             │ v1 ─○─ Mar 25 │
│   Net v1 │  │    └──────────────────────────┘      │             │ v2 ─○─ Jun 25 │
│          │  │                                      │             │ v3 ─●─ Feb 26 │
│ ▾ MY   2 │  │    ┌──────────────────────────┐      │             │               │
│ ■ Housing│  │    │ How long in current      │      │             │ RELATED       │
│   Loc v3 │  │    │ situation?               │      │             │ Intake Event  │
│   trial  │  │    │ ☐ <1 month               │      │             │  2 shared fld │
│   Risk Sc│  │    │ ☐ 1-6 months             │      │             │ Add'l Context │
│   Loc v1 │  │    │ ☐ 6-12 months            │      │             │  1 shared fld │
│          │  │    │ ☐ >1 year                │      │             │               │
│ ▸ ORG  4 │  │    └──────────────────────────┘      │             │ USED BY ORGS  │
│ ▸ NET  8 │  │                                      │             │ Harbor House  │
│          │  │  ▸ Risk Assessment                    │             │ PATH Services │
│          │  │  ▸ Demographics                       │             │ 2 of 5 adopted│
│          │  │                                      │             │               │
│          │  └──────────────────────────────────────┘             │ ACTIVITY      │
│          │                                                       │ Feb 14 edited │
│          │                                                       │ Feb 12 v3 bump│
└──────────┴───────────────────────────────────────────────────────┴───────────────┘
```

---

## Mapping to Existing Code

| Current Code | Change Needed |
|---|---|
| `FormBuilder` component (line 3416) | Wrap in new `SchemaView` layout container with sidebar + context panel |
| `savedForms` state (line 3431) | Extend with `sourceType`, `lastViewedAt`, `linkedFormKeys`, etc. |
| `showFormList` modal (line 4536-4593) | **Replace entirely** with sidebar component |
| `view==='schema'` render (line 8281) | Render `SchemaView` instead of bare `FormBuilder` |
| `DEFAULT_FORMS` (domain config) | Feed into NETWORK COMMONS group |
| Org room `io.khora.schema.form` events | Feed into ORG FORMS group |
| `diffFormVersions` function (line 3383) | Reuse for version comparison in context panel |
| `doSaveForm` / `doLoadForm` | Keep, but trigger sidebar refresh on save |
| Version history modal (line 4640) | Keep as modal, but also show compact timeline in context panel |

---

## Component Breakdown (Implementation Plan)

### New Components

1. **`SchemaView`** — Layout wrapper. Renders Sidebar + FormBuilder + ContextPanel in the three-zone grid.
2. **`FormsSidebar`** — Left panel. Manages groups (INBOX, MY FORMS, ORG, NETWORK), search, selection.
3. **`FormListItem`** — Individual form row in sidebar. Shows name, source, version, status indicators.
4. **`ContextPanel`** — Right panel. Sections: Lineage, Version Timeline, Related Forms, Used By, Activity.
5. **`VersionTimeline`** — Vertical dot-timeline of form versions. Clickable dots open diff.
6. **`LineageTree`** — Network → Org → Provider → Client chain visualization.
7. **`FormBreadcrumb`** — Top bar breadcrumb showing group / form name / version.

### Modified Components

1. **`FormBuilder`** — Remove Library button, accept `form` and `onFormChange` as props instead of owning state.
2. **`ProviderApp`** — `view==='schema'` renders `SchemaView` instead of `FormBuilder`.

---

## Open Questions

1. **Multi-form editing** — Should users be able to have multiple forms "open" in tabs within the canvas, or strictly one at a time? (Recommendation: one at a time, with fast switching via sidebar click.)
2. **Org form publishing** — When a user saves a form and they're an org admin, should there be an explicit "Publish to Org" action? (Recommendation: yes, separate from local save.)
3. **Network proposal flow** — Should proposing a form to the network schema happen from this view, or remain in the Network governance view? (Recommendation: add a "Propose to Network" action in the context menu, which opens the governance proposal flow.)
4. **Offline forms** — Since the app uses IndexedDB, should the sidebar show a connectivity indicator for forms that haven't synced? (Recommendation: yes, subtle icon.)

---

# DETAILED DESIGN — Pixel-Level Specifications

Everything below specifies exact styling, every interactive state, edge cases, and full component anatomy using Khora's existing design system tokens.

---

## 1. SIDEBAR — Full Anatomy

### 1.1 Sidebar Container

```
┌─────────────────────────────┐
│ bg: SWC.surface (#0e1218)   │
│ width: 280px                │
│ border-right: 1px solid     │
│   SWC.border (#1d2735)      │
│ display: flex               │
│ flex-direction: column      │
│ height: 100%                │
│ overflow: hidden            │
└─────────────────────────────┘
```

CSS mapping: mirrors existing `.inbox-list` pattern but vertical full-height.

### 1.2 Sidebar Header

```
┌───────────────────────────────────────────┐
│ ┌─────────────────────────────────────┐   │
│ │  ◇ SCHEMA          [+] [⋯]        │   │
│ │                                     │   │
│ │  padding: 14px 16px 10px            │   │
│ │  border-bottom: 1px solid border    │   │
│ └─────────────────────────────────────┘   │
└───────────────────────────────────────────┘
```

- **"SCHEMA"** label: `section-label` class (12px, IBM Plex Mono 500, `SWC.muted`, letter-spacing 0.07em)
- **◇ icon**: `I n="layers" s={13}` in `SWC.given` — small form stack icon
- **[+] button**: 24x24px, border-radius 6px, `background: transparent`, `border: 1px solid SWC.border`, `color: SWC.muted`. Hover: `background: SWC.hover`, `color: SWC.white`, `border-color: SWC.given`. Uses `I n="plus" s={12}`
- **[⋯] button**: same dimensions, opens dropdown menu (Import form, Import from clipboard, Bulk export). Uses `I n="more-h" s={12}`

### 1.3 Search Bar

```
┌───────────────────────────────────────────┐
│  ┌─────────────────────────────────────┐  │
│  │ 🔍  Search forms...                │  │
│  │                                     │  │
│  │  padding: 8px 12px                  │  │
│  │  margin: 8px 12px                   │  │
│  │  bg: SWC.bg (#07090d)              │  │
│  │  border: 1px solid SWC.border       │  │
│  │  border-radius: 6px                 │  │
│  │  font: 12px Manrope                 │  │
│  │  color: SWC.text                    │  │
│  │  placeholder-color: SWC.dim         │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

- **Search icon**: `I n="search" s={12} c={SWC.dim}` positioned absolute left 12px
- **Input**: left padding 30px to clear icon
- **Focus state**: `border-color: SWC.borderLit`, subtle `box-shadow: 0 0 0 2px ${SWC.given}14`
- **Clear button**: appears when text entered, `I n="x" s={10}` positioned absolute right 8px
- **Behavior**: filters all groups simultaneously. Matching characters highlighted with `background: ${SWC.given}28` on the matching substring within form names

### 1.4 Group Headers

```
┌───────────────────────────────────────────┐
│                                           │
│  ▾ INBOX                            3    │
│                                           │
│  padding: 10px 16px 6px                   │
│  cursor: pointer                          │
│  user-select: none                        │
│                                           │
└───────────────────────────────────────────┘
```

- **Chevron**: `I n="chevronDown" s={10}` rotates -90° when collapsed. Transition: `transform 150ms ease-out`
- **Group name**: 10.5px, IBM Plex Mono 600, `SWC.muted`, letter-spacing 0.08em, text-transform uppercase
- **Count badge**: 10px, IBM Plex Mono 500, right-aligned
  - INBOX count: `color: SWC.given`, `background: ${SWC.given}18`, padding 1px 6px, border-radius 8px — pulses if >0
  - Other counts: `color: SWC.dim`, no background
- **Hover**: entire header gets `background: SWC.hover` (150ms transition)
- **Divider**: 1px solid `SWC.border` below each group header

#### Group-specific styling:

| Group | Icon | Name color | Accent |
|-------|------|------------|--------|
| INBOX | `I n="inbox" s={11}` | `SWC.given` when has items, `SWC.dim` when empty | teal |
| MY FORMS | `I n="folder" s={11}` | `SWC.muted` | none |
| ORG FORMS | `I n="shieldCheck" s={11}` | `SWC.muted` | blue |
| NETWORK COMMONS | `I n="globe" s={11}` | `SWC.muted` | teal |

### 1.5 Form List Item — Every State

Each form item is a row inside a group. Anatomy:

```
┌───────────────────────────────────────────┐
│                                           │
│  ●  Status & Engagement Tracking    ⋯    │
│     Network · v2 · normative              │
│                                    ▲ new  │
│                                           │
│  height: auto (min 52px)                  │
│  padding: 8px 16px 8px 12px               │
│  border-left: 3px solid transparent       │
│  border-bottom: 1px solid SWC.border      │
│  cursor: pointer                          │
│  transition: all 150ms                    │
│                                           │
└───────────────────────────────────────────┘
```

#### Layout grid:

```
  3px    6px        flex-1                  24px
┌─────┬──────┬──────────────────────────┬──────┐
│left │ dot  │  content                 │ menu │
│bord │      │                          │      │
│     │      │  row1: name              │  ⋯   │
│     │      │  row2: source·ver·mature │      │
│     │      │  row3: update flag (opt) │      │
└─────┴──────┴──────────────────────────┴──────┘
```

#### Element specifications:

**Status dot** (left of name):
- Size: 7x7px, border-radius 50%
- States:
  - `unread`: `background: SWC.given`, with `animation: pulse 2s infinite` (0.6→1.0 opacity)
  - `active`: `background: SWC.white`, 7x7px square with border-radius 2px (■ shape)
  - `read`: `background: SWC.dim`, opacity 0.4
  - `stale`: hidden (display none)

**Form name** (row 1):
- 12.5px, Manrope 500, `color: SWC.text`
- Active form: Manrope 600, `color: SWC.white`
- `overflow: hidden`, `text-overflow: ellipsis`, `white-space: nowrap`
- Max width: fills available space minus dot and menu

**Metadata line** (row 2):
- 10px, IBM Plex Mono 400
- Source label: colored by source type
  - Network: `color: SWC.given` (#2dd4a0)
  - Org: `color: #60a5fa` (SWC.fw[0] blue)
  - Local: `color: SWC.dim`
- Separator: ` · ` in `SWC.dim`
- Version: `color: SWC.muted`
- Maturity: uses same color logic as existing `MaturityBadge`:
  - draft → `color: SWC.fw[0]` (blue)
  - trial → `color: SWC.sup` (gold)
  - normative → `color: SWC.given` (green)
  - deprecated → `color: SWC.red`

**Update flag** (row 3, conditional):
- Only shown when upstream version > local version
- `▲` arrow + "new" text
- 9px, IBM Plex Mono 600, `color: SWC.sup` (gold)
- `background: ${SWC.sup}10`, padding 1px 5px, border-radius 3px

**Context menu button** (⋯):
- Hidden by default, shown on hover
- 20x20px, `I n="more-v" s={11}`, `color: SWC.dim`
- Hover: `color: SWC.muted`

#### Interactive states:

**Default:**
```css
background: transparent;
border-left: 3px solid transparent;
```

**Hover:**
```css
background: SWC.hover;  /* #1b2330 */
border-left: 3px solid transparent;
/* Show ⋯ menu button */
```

**Active (selected):**
```css
background: SWC.raised;  /* #151b23 */
border-left: 3px solid SWC.given;  /* #2dd4a0 */
/* Name goes white + bold */
```

**Hover while active:**
```css
background: SWC.hover;
border-left: 3px solid SWC.given;
```

**Drag (MY FORMS only):**
```css
opacity: 0.7;
transform: scale(0.98);
box-shadow: 0 4px 12px rgba(0,0,0,0.3);
/* Drop placeholder: 2px solid SWC.given dashed */
```

### 1.6 Context Menu (right-click / ⋯ button)

```
┌──────────────────────────┐
│ bg: SWC.raised (#151b23) │
│ border: 1px solid border │
│ border-radius: 8px       │
│ box-shadow: 0 8px 24px   │
│   rgba(0,0,0,0.4)        │
│ padding: 4px             │
│ min-width: 180px         │
│ z-index: 50              │
├──────────────────────────┤
│ ↗ Open in new tab        │  ← future: split view
│ ─────────────────────    │
│ ⊕ Duplicate              │
│ ↓ Export JSON             │
│ ─────────────────────    │
│ ⟷ Compare with...        │  ← opens picker
│ ─────────────────────    │
│ ↑ Publish to Org         │  ← if org admin + local form
│ ↑ Propose to Network     │  ← if org admin + org form
│ ─────────────────────    │
│ 🗑 Delete                 │  ← red text, SWC.red
└──────────────────────────┘
```

Each menu item:
- Padding: 8px 12px
- Font: 12.5px Manrope 500
- `color: SWC.text`
- Hover: `background: SWC.hover`, `color: SWC.white`
- Icons: 13px, left-aligned with 8px gap
- Dividers: 1px solid `SWC.border`, margin 4px 8px
- Delete: `color: SWC.red`, hover `background: ${SWC.red}10`

### 1.7 Empty States

**INBOX empty:**
```
┌───────────────────────────────────────┐
│                                       │
│       ✓  All caught up               │
│                                       │
│   No incoming form updates.           │
│   Forms from your network and org     │
│   will appear here when updated.      │
│                                       │
│   padding: 20px 16px                  │
│   text-align: center                  │
│   ✓ icon: I n="check" s={20}         │
│     color: SWC.given, opacity 0.5     │
│   heading: 12px Manrope 600 SWC.muted │
│   body: 11px Manrope 400 SWC.dim      │
│                                       │
└───────────────────────────────────────┘
```

**MY FORMS empty:**
```
┌───────────────────────────────────────┐
│                                       │
│       No saved forms yet              │
│                                       │
│   [+ Create your first form]          │
│                                       │
│   button: SwBtn style, SWC.given      │
│   ghost, padding 6px 14px, 11px font  │
│                                       │
└───────────────────────────────────────┘
```

**Search no results:**
```
┌───────────────────────────────────────┐
│                                       │
│    No forms matching "xyz"            │
│                                       │
│   padding: 24px 16px                  │
│   12px Manrope 400 SWC.dim            │
│   "xyz" in SWC.muted italic           │
│                                       │
└───────────────────────────────────────┘
```

### 1.8 Sidebar Scroll Behavior

- Group headers: **sticky** (`position: sticky; top: 0; z-index: 2; background: SWC.surface`)
- Scrollbar: 4px wide, `SWC.border` thumb, transparent track (matches existing `::-webkit-scrollbar` rules)
- Scroll region: everything below the search bar
- Groups collapse/expand independently; collapsed groups take ~32px (header only)

---

## 2. CONTEXT PANEL — Full Anatomy

### 2.1 Container

```css
width: 260px;
min-width: 260px;
border-left: 1px solid SWC.border;
background: SWC.surface;
display: flex;
flex-direction: column;
overflow-y: auto;
overflow-x: hidden;
transition: width 200ms ease-out, min-width 200ms ease-out, opacity 150ms;
```

**Collapsed state:**
```css
width: 36px;
min-width: 36px;
/* Show vertical icon strip */
```

### 2.2 Panel Header

```
┌──────────────────────────────┐
│                              │
│  CONTEXT                [◀]  │
│                              │
│  padding: 14px 14px 10px     │
│  border-bottom: 1px border   │
│  display: flex               │
│  justify-content: space-btw  │
│                              │
└──────────────────────────────┘
```

- **"CONTEXT"**: `section-label` class styling
- **[◀] collapse button**: 22x22px, `I n="chevronRight" s={12}` (flips to `chevronLeft` when collapsed), `color: SWC.dim`, `hover: SWC.muted`

### 2.3 Collapsed Strip

When collapsed, show a 36px vertical strip:

```
┌──┐
│▶ │  ← expand button, 14px from top
│  │
│🔗│  ← lineage icon (I n="gitBranch" s={14})
│  │
│◷ │  ← version icon (I n="clock" s={14})
│  │
│⟷│  ← related icon (I n="link" s={14})
│  │
│👥│  ← orgs icon (I n="users" s={14})
│  │
│⚡│  ← activity icon (I n="activity" s={14})
└──┘
```

Each icon: `color: SWC.dim`, `hover: SWC.muted`. Click on any icon expands the panel and scrolls to that section. The icons have a subtle left border accent when their section has interesting data:
- Lineage: always visible → no accent
- Version: accent `SWC.sup` if version history exists
- Related: accent `SWC.given` if related forms detected
- Orgs: accent `SWC.fw[0]` (blue) if multiple orgs use the form
- Activity: accent `SWC.given` if recent activity (< 24h)

### 2.4 Lineage Section

```
┌──────────────────────────────┐
│ LINEAGE                      │
│                              │
│  ┌────────────────────────┐  │
│  │ ◇ CoC Network          │  │
│  │   teal tag, I n="globe"│  │
│  │   font: 12px Man 600   │  │
│  │   color: SWC.given     │  │
│  ├────────────────────────┤  │
│  │ │  ↓                   │  │
│  │ │  required             │  │  ← propagation level
│  │ │  9px Mono, SWC.dim    │  │
│  ├────────────────────────┤  │
│  │ ◇ Harbor House          │  │
│  │   blue tag, I n="shield"│  │
│  │   font: 12px Man 600    │  │
│  │   color: SWC.fw[0]     │  │
│  ├────────────────────────┤  │
│  │ │  ↓                   │  │
│  │ │  extended (+2 fields) │  │
│  │ │  "+2" in SWC.given    │  │
│  ├────────────────────────┤  │
│  │ ◆ You (active)          │  │
│  │   white tag, I n="user" │  │
│  │   font: 12px Man 600    │  │
│  │   color: SWC.white     │  │
│  │   "active" in SWC.given │  │
│  ├────────────────────────┤  │
│  │   → 12 clients          │  │
│  │   I n="users" s={11}    │  │
│  │   11px Mono, SWC.muted  │  │
│  └────────────────────────┘  │
│                              │
└──────────────────────────────┘
```

**Lineage node styling:**
```css
padding: 8px 12px;
border-radius: 6px;
background: ${nodeColor}08;  /* 8% opacity of the node's accent color */
border: 1px solid ${nodeColor}20;
margin-bottom: 0;  /* no gap — connector fills it */
```

**Connector between nodes:**
```css
/* Vertical line segment */
width: 1px;
height: 20px;
background: SWC.border;
margin-left: 20px;  /* centered under the icon */

/* Propagation label sits next to the line */
font: 9px IBM Plex Mono;
color: SWC.dim;
margin-left: 8px;
```

**Local form (no upstream):** Lineage shows only one node:
```
┌──────────────────────────────┐
│ LINEAGE                      │
│                              │
│  ┌────────────────────────┐  │
│  │ ◆ Local form            │  │
│  │   Not linked to any     │  │
│  │   network or org schema │  │
│  │                         │  │
│  │   [Publish to Org →]    │  │
│  └────────────────────────┘  │
│                              │
└──────────────────────────────┘
```

### 2.5 Version Timeline Section

```
┌──────────────────────────────┐
│ VERSION TIMELINE             │
│                              │
│  ┌─ v3 ──●── Feb 14, 2026   │  ← current: filled dot
│  │         Risk assessment   │     notes: 11px Man 400
│  │         section added     │     SWC.muted, max 2 lines
│  │                           │
│  │         +2q -0q ±1o       │  ← diff summary
│  │         green/red/gold    │     9px Mono
│  │                           │
│  ├─ v2 ──○── Jun 15, 2025   │  ← past: hollow dot
│  │         Added housing     │
│  │         questions         │
│  │                           │
│  │         +4q -0q +12o      │
│  │                           │
│  ├─ v1 ──○── Mar 3, 2025    │  ← oldest
│  │         Initial version   │
│  │                           │
│  └───────────────────────────│
│                              │
│  [Compare versions]          │  ← SwBtn ghost, SWC.muted
│                              │
└──────────────────────────────┘
```

**Timeline structure (CSS):**

```css
/* Vertical line running down the left side */
.vt-line {
  position: absolute;
  left: 16px;
  top: 0;
  bottom: 0;
  width: 1px;
  background: SWC.border;
}

/* Version node */
.vt-node {
  position: relative;
  padding: 8px 0 16px 36px;  /* left space for dot + line */
}

/* Dot — positioned over the vertical line */
.vt-dot {
  position: absolute;
  left: 12px;  /* centered on the line */
  top: 10px;
  width: 9px;
  height: 9px;
  border-radius: 50%;
}

/* Current version dot */
.vt-dot.current {
  background: SWC.given;
  box-shadow: 0 0 6px ${SWC.given}40;
}

/* Past version dot */
.vt-dot.past {
  background: transparent;
  border: 2px solid SWC.dim;
}

/* Hovered past dot */
.vt-dot.past:hover {
  border-color: SWC.muted;
  background: ${SWC.muted}20;
  cursor: pointer;
}
```

**Version label**: `v{n}` in 11px IBM Plex Mono 600, `SWC.white` (current) or `SWC.muted` (past)

**Date**: 10px IBM Plex Mono 400, `SWC.dim`, right-aligned

**Notes**: 11px Manrope 400, `SWC.muted`, `line-height: 1.4`, max 2 lines with ellipsis

**Diff summary**: inline badges
- `+2q`: `color: SWC.given` (questions added)
- `-1q`: `color: SWC.red` (questions removed)
- `±3o`: `color: SWC.sup` (options changed)
- Font: 9px IBM Plex Mono 500, separated by 6px gap

**Click on past version**: opens existing `diffFormVersions` modal with that version vs current

**No version history:**
```
┌──────────────────────────────┐
│ VERSION TIMELINE             │
│                              │
│  ● v1 (current)              │
│    No prior versions.        │
│    Bump the version to start │
│    tracking history.         │
│                              │
│    [Bump version →]          │
│                              │
└──────────────────────────────┘
```

### 2.6 Related Forms Section

```
┌──────────────────────────────┐
│ RELATED FORMS                │
│                              │
│  ┌────────────────────────┐  │
│  │ ⟷ Intake Event         │  │
│  │   2 shared fields       │  │
│  │   1 crosswalk           │  │
│  │                         │  │
│  │   Shared: housing_stat, │  │
│  │   duration              │  │
│  └────────────────────────┘  │
│  ┌────────────────────────┐  │
│  │ ⟷ Additional Context   │  │
│  │   1 shared field        │  │
│  │   0 crosswalks          │  │
│  └────────────────────────┘  │
│                              │
└──────────────────────────────┘
```

**Related form card:**
```css
padding: 8px 12px;
border-radius: 6px;
border: 1px solid SWC.border;
background: SWC.bg;
margin-bottom: 6px;
cursor: pointer;
transition: all 150ms;

/* Hover */
border-color: SWC.borderLit;
background: SWC.hover;
```

- **Link icon**: `I n="link" s={11}` in `SWC.dim`
- **Form name**: 12px Manrope 600, `SWC.text`
- **Stats line**: 10px IBM Plex Mono 400, `SWC.dim`
  - "2 shared fields" — count in `SWC.given` if >0
  - "1 crosswalk" — count in `SWC.sup` if >0
- **Shared field names** (expandable): 9.5px IBM Plex Mono 400, `SWC.dim`, italic. Collapsed by default, expand on hover or click. Show as comma-separated `key` names.
- **Click**: navigates to that form in the sidebar (loads it in canvas), previous form can be reached with browser-style back

**No related forms:**
```
┌──────────────────────────────┐
│ RELATED FORMS                │
│                              │
│  No related forms detected.  │
│  Forms sharing field keys    │
│  or with crosswalk mappings  │
│  will appear here.           │
│                              │
│  11px Man 400, SWC.dim       │
│  padding: 12px               │
│  border: 1px dashed border   │
│  border-radius: 6px          │
│  text-align: center          │
└──────────────────────────────┘
```

### 2.7 Used By Orgs Section

```
┌──────────────────────────────┐
│ ADOPTION (3 of 5 orgs)       │
│                              │
│  ████████████░░░░░  60%      │  ← adoption bar
│                              │
│  ┌────────────────────────┐  │
│  │ ✓ Harbor House (you)   │  │  ← adopted, you're here
│  │   v2 · +2 extensions   │  │
│  ├────────────────────────┤  │
│  │ ✓ PATH Services        │  │  ← adopted
│  │   v2 · standard        │  │
│  ├────────────────────────┤  │
│  │ ✓ Salvation Army       │  │  ← adopted
│  │   v1 · outdated        │  │     outdated version flag
│  ├────────────────────────┤  │
│  │ ○ City Services        │  │  ← not adopted
│  ├────────────────────────┤  │
│  │ ○ Housing Authority    │  │  ← not adopted
│  └────────────────────────┘  │
│                              │
└──────────────────────────────┘
```

**Adoption progress bar:**
```css
height: 4px;
border-radius: 2px;
background: SWC.border;
margin: 6px 0 10px;
overflow: hidden;

/* Fill */
.fill {
  height: 100%;
  background: SWC.given;
  border-radius: 2px;
  transition: width 300ms ease-out;
}
```

**Org row:**
- Adopted: `I n="check" s={10}` in `SWC.given`, name in `SWC.text`
- Not adopted: `I n="circle" s={10}` in `SWC.dim`, name in `SWC.dim`
- Your org: append "(you)" in `SWC.muted`, 10px italic
- Outdated version: `SwTag` with `SWC.sup`, text "outdated", 8px
- Extension count: 10px Mono, `SWC.given` if >0, `SWC.dim` if 0

**Only visible for network-sourced forms.** For local/org forms, section header shows "LOCAL FORM" and body shows:
```
This form isn't published to a
network. Publish to track adoption.
```

### 2.8 Activity Section

```
┌──────────────────────────────┐
│ ACTIVITY                     │
│                              │
│  ● Feb 14  You edited        │
│  │         2 fields modified │
│  │                           │
│  ○ Feb 12  Version bumped    │
│  │         v2 → v3 (minor)  │
│  │                           │
│  ○ Jan 30  Network sync      │
│  │         Propagated from   │
│  │         CoC Network       │
│  │                           │
│  ○ Jan 15  Created           │
│  │         Initial save      │
│  │                           │
│                              │
│  [Show all activity →]       │
│                              │
└──────────────────────────────┘
```

**Activity item:**
- Dot: 5px, `SWC.given` for recent (<24h), `SWC.dim` for older
- Vertical connector: 1px `SWC.border`, same pattern as version timeline
- **Date**: 10px IBM Plex Mono 600, `SWC.muted`
- **Action**: 11px Manrope 500, `SWC.text`
- **Detail**: 10px Manrope 400, `SWC.dim`
- Show max 5 items by default, "Show all →" expands

**Maps to EO operations**: Each activity entry corresponds to an `io.khora.op` event:
| EO Op | Activity Label |
|-------|---------------|
| DES | Created |
| INS | Field added |
| ALT | Field modified / Edited |
| SEG | Section reorganized |
| CON | Linked to authority |
| SYN | Synced with network |
| SUP | Conflict detected |
| NUL | Deleted |

---

## 3. TOOLBAR (Top Bar) — Full Anatomy

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                    │
│  ◇ Schema    ❯  My Forms  ❯  Housing Assessment  ❯  v3      [Save] [Publish] [⋮]  │
│                                                                                    │
│  height: 44px                                                                      │
│  padding: 0 20px                                                                   │
│  border-bottom: 1px solid SWC.border                                               │
│  background: SWC.bg                                                                │
│  display: flex                                                                     │
│  align-items: center                                                               │
│  justify-content: space-between                                                    │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Breadcrumb segments:**
- Each segment: 12px Manrope 400, `SWC.dim`
- Active segment (last): 12px Manrope 600, `SWC.white`
- Separator `❯`: 10px, `SWC.dim`, padding 0 6px
- Clickable segments: `cursor: pointer`, `hover: color SWC.muted`, clicking a group segment scrolls sidebar to that group
- `◇` icon at start: `I n="layers" s={13}` in `SWC.given`

**Right-side actions:**
- Same buttons as current FormBuilder header (`Save`, `History`, `Publish`)
- Moved from inline form header to toolbar
- `[⋮]` more menu: Version Bump, Export, Import, Settings

---

## 4. INTERACTION DETAILS

### 4.1 Form Selection Flow (click sidebar item)

```
t=0ms    User clicks form item in sidebar
t=0ms    Sidebar: previous item loses active state (instant)
t=0ms    Sidebar: clicked item gains active border-left (instant)
t=0ms    Canvas: fade-out current form (opacity 1→0, 80ms)
t=80ms   Canvas: load new form data into FormBuilder state
t=80ms   Canvas: fade-in new form (opacity 0→1, 150ms)
t=80ms   Context: all sections update with new form's data
t=80ms   Breadcrumb: updates to reflect new form path
t=230ms  Complete — new form fully visible
```

If the form being left has unsaved changes:
```
t=0ms    Dirty indicator: subtle dot appears next to form name in sidebar
         (no blocking modal — Khora philosophy is non-destructive by default,
          forms auto-persist to IndexedDB, explicit "Save" is for Matrix persistence)
```

### 4.2 Form Update Arrival (live Matrix event)

```
t=0ms    Matrix state event arrives with updated schema.form
t=0ms    Compare incoming version with local lastSyncedVersion
         If incoming > local:
t=0ms      INBOX group: form appears (or updates)
t=0ms      Item gets unread dot (●), pulse animation starts
t=0ms      INBOX count badge increments
t=0ms      If form is currently active in canvas:
t=0ms        Gold banner slides down from toolbar:
             ┌──────────────────────────────────────────┐
             │ ⚠ Updated upstream (v2 → v3)  [Review]  │
             │   12px Man, SWC.sup bg, 36px height      │
             └──────────────────────────────────────────┘
             [Review] loads the incoming version for diff comparison
```

### 4.3 Drag-to-Reorder (MY FORMS group only)

```
t=0ms    mousedown/touchstart on a MY FORMS item (not on ⋯ button)
t=150ms  If still holding: enter drag mode
         - Item lifts (scale 0.98, opacity 0.8, shadow)
         - Drop zones appear between other MY FORMS items
           (2px horizontal lines, SWC.given, dashed)
t=???    User drags over a drop zone
         - Drop zone expands to 8px, solid SWC.given
t=???    User releases
         - Item animates to new position (150ms ease-out)
         - Order persisted to IndexedDB
```

Drag is **not** available for INBOX, ORG, or NETWORK groups (those are sorted by system rules).

### 4.4 Compare Versions Modal

Triggered from: Version Timeline click, or context menu "Compare with..."

```
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  Compare Versions — Housing Assessment                         [×]   │
│                                                                       │
│  ┌─ v2 (Jun 2025) ────────────┐  ┌─ v3 (current) ────────────────┐  │
│  │                             │  │                                │  │
│  │  General                    │  │  General                       │  │
│  │  ─────                      │  │  ─────                         │  │
│  │  What is your current       │  │  What is your current          │  │
│  │  housing situation?         │  │  housing situation?            │  │
│  │  ☐ Sheltered                │  │  ☐ Sheltered                   │  │
│  │  ☐ Unsheltered              │  │  ☐ Unsheltered                 │  │
│  │  ☐ At risk                  │  │  ☐ At risk                     │  │
│  │                             │  │  ☐ Doubled up        ← +green │  │
│  │  ☐ Stably housed            │  │  ☐ Stably housed               │  │
│  │                             │  │                                │  │
│  │                             │  │  Risk Assessment      ← +green│  │
│  │                             │  │  ─────                         │  │
│  │                             │  │  Have you experienced  ← +grn │  │
│  │                             │  │  violence recently?            │  │
│  │                             │  │  ☐ Yes  ☐ No                   │  │
│  │                             │  │                                │  │
│  └─────────────────────────────┘  └────────────────────────────────┘  │
│                                                                       │
│  Summary: +1 section, +2 questions, +5 options, 0 removed             │
│                                                                       │
│  [Restore v2]                     [Close]                             │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

- Uses existing `fb-adopt-modal` and `fb-adopt-panel` patterns
- Left panel: older version, `border-top: 3px solid SWC.dim`
- Right panel: newer version, `border-top: 3px solid SWC.given`
- Added elements: `background: ${SWC.given}08`, left border `3px solid SWC.given`
- Removed elements: `background: ${SWC.red}08`, left border `3px solid SWC.red`, strikethrough text
- Changed elements: `background: ${SWC.sup}08`, left border `3px solid SWC.sup`
- Summary bar: 11px Mono, counts colored per type

### 4.5 Search Behavior — Detailed

**Input:**
- Debounce: 150ms after last keystroke
- Minimum 2 characters to trigger search
- Search fields: `form.name`, `form.key`, `form.description`, `form.sections[].title`, `form.sections[].questions[].prompt`

**Results display:**
- All groups remain visible but only show matching forms
- Empty groups auto-collapse during search
- Matching text highlighted inline: replace matching chars with `<mark>` styled as `background: ${SWC.given}28; color: SWC.white; border-radius: 2px; padding: 0 1px`
- Show what matched under the form name if the match is in a field other than name:
  ```
  Status & Engagement Tracking
  Network · v2 · normative
  ↳ matched: "housing" in question        ← 9px Mono, SWC.dim, italic
  ```

**Keyboard navigation during search:**
- `↑` / `↓`: move selection through results
- `Enter`: load selected form
- `Escape`: clear search, restore sidebar

---

## 5. RESPONSIVE BEHAVIOR — Detailed

### 5.1 Breakpoint: >1100px (Full desktop)

```
┌──────────┬──────────────────────────────────────┬──────────────┐
│ Sidebar  │          Canvas                       │ Context      │
│ 280px    │          flex: 1                      │ 260px        │
│ visible  │                                       │ visible      │
└──────────┴──────────────────────────────────────┴──────────────┘
```

All three zones visible. No overlays.

### 5.2 Breakpoint: 900–1100px (Narrow desktop / tablet landscape)

```
┌──────────┬──────────────────────────────────────────┬────┐
│ Sidebar  │          Canvas                           │ ◀  │
│ 240px    │          flex: 1                          │36px│
│ visible  │                                           │    │
│ compact  │                                           │    │
└──────────┴──────────────────────────────────────────┴────┘
```

- Sidebar: shrinks to 240px, search bar input placeholder truncated
- Context panel: auto-collapsed to 36px icon strip
- Click ▶ on context strip: panel slides open as 260px overlay on top of canvas (not pushing it), with `box-shadow: -8px 0 32px rgba(0,0,0,0.3)`

### 5.3 Breakpoint: 700–900px (Tablet portrait)

```
┌──┬───────────────────────────────────────────────────────┐
│☰ │          Canvas (full width)                          │
│  │                                                       │
│56│          Context panel: accessible via bottom sheet   │
│px│                                                       │
└──┴───────────────────────────────────────────────────────┘
```

- Sidebar: collapses to 56px icon-only strip (mirrors existing `.app-sidebar` behavior at <700px)
  - Shows only: group icons stacked vertically
  - Click hamburger `☰` (top): opens sidebar as full overlay drawer
  - Overlay: slides from left, `width: 300px`, `z-index: 40`, `backdrop: rgba(0,0,0,0.5)`
  - Close: click outside, swipe left, or press Escape
- Context panel: hidden entirely. Accessible via a pull-up bottom sheet:
  - Grab handle: 40px wide, 4px tall, `SWC.border`, centered, border-radius 2px
  - Pull up to reveal context panel in bottom sheet format
  - Max height: 60vh
  - `border-radius: 12px 12px 0 0` on top corners

### 5.4 Breakpoint: <700px (Mobile)

```
┌────────────────────────────────────────┐
│ ☰  Schema  Housing Assess…  [Save]    │  ← toolbar compressed
├────────────────────────────────────────┤
│                                        │
│  Canvas (full width, full height)      │
│                                        │
│  Sidebar: overlay drawer               │
│  Context: bottom sheet                 │
│                                        │
└────────────────────────────────────────┘
```

- Toolbar: breadcrumb truncated to current form name only
- Save button only (other actions in ⋮ menu)
- Sidebar: full-screen overlay drawer
- Context: bottom sheet with swipe-up gesture

---

## 6. CSS CLASS STRUCTURE

New classes following existing naming conventions:

```css
/* ═══ Schema Inbox Layout ═══ */
.si-wrap { display: flex; height: calc(100vh - 56px); overflow: hidden; }
.si-sidebar { width: 280px; min-width: 280px; border-right: 1px solid var(--border-0);
  display: flex; flex-direction: column; background: var(--bg-2); }
.si-canvas { flex: 1; min-width: 0; overflow: auto; padding: 20px; }
.si-context { width: 260px; min-width: 260px; border-left: 1px solid var(--border-0);
  display: flex; flex-direction: column; background: var(--bg-2); overflow-y: auto;
  transition: width 200ms ease-out, min-width 200ms ease-out; }
.si-context.collapsed { width: 36px; min-width: 36px; }

/* Sidebar internals */
.si-header { padding: 14px 16px 10px; border-bottom: 1px solid var(--border-0);
  display: flex; align-items: center; justify-content: space-between; }
.si-search { margin: 8px 12px; position: relative; }
.si-search input { width: 100%; padding: 8px 12px 8px 30px; border-radius: 6px;
  border: 1px solid var(--border-0); background: var(--bg-0); color: var(--tx-0);
  font-size: 12px; font-family: var(--sans); }
.si-search input:focus { border-color: var(--border-2); }
.si-search-icon { position: absolute; left: 10px; top: 50%; transform: translateY(-50%); }

.si-group-header { padding: 10px 16px 6px; cursor: pointer; user-select: none;
  display: flex; align-items: center; justify-content: space-between;
  border-bottom: 1px solid var(--border-0); transition: background 150ms; }
.si-group-header:hover { background: var(--bg-3); }
.si-group-label { font-size: 10.5px; font-family: var(--mono); font-weight: 600;
  color: var(--tx-1); letter-spacing: 0.08em; display: flex; align-items: center; gap: 6px; }
.si-group-count { font-size: 10px; font-family: var(--mono); color: var(--tx-2); }
.si-group-count.alert { color: var(--teal); background: rgba(62,201,176,0.12);
  padding: 1px 6px; border-radius: 8px; }

.si-form-item { padding: 8px 16px 8px 12px; cursor: pointer;
  border-left: 3px solid transparent; border-bottom: 1px solid var(--border-0);
  transition: all 150ms; display: flex; gap: 6px; align-items: flex-start; }
.si-form-item:hover { background: var(--bg-3); }
.si-form-item.active { background: var(--bg-4); border-left-color: var(--teal); }
.si-form-item .si-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0;
  margin-top: 5px; }
.si-form-item .si-dot.unread { background: var(--teal); animation: pulse 2s infinite; }
.si-form-item .si-dot.active { background: var(--tx-0); border-radius: 2px; }
.si-form-item .si-dot.read { background: var(--tx-3); opacity: 0.4; }
.si-form-name { font-size: 12.5px; font-weight: 500; overflow: hidden;
  text-overflow: ellipsis; white-space: nowrap; }
.si-form-item.active .si-form-name { font-weight: 600; color: var(--tx-0); }
.si-form-meta { font-size: 10px; font-family: var(--mono); color: var(--tx-2);
  margin-top: 2px; }
.si-form-update { font-size: 9px; font-family: var(--mono); font-weight: 600;
  color: var(--gold); background: rgba(201,163,82,0.10); padding: 1px 5px;
  border-radius: 3px; margin-top: 2px; display: inline-block; }

/* Context panel internals */
.si-ctx-section { padding: 12px 14px; border-bottom: 1px solid var(--border-0); }
.si-ctx-label { font-size: 10px; font-family: var(--mono); font-weight: 600;
  color: var(--tx-1); letter-spacing: 0.07em; margin-bottom: 8px; }

/* Version timeline */
.si-vt { position: relative; padding-left: 20px; }
.si-vt::before { content: ''; position: absolute; left: 16px; top: 0; bottom: 0;
  width: 1px; background: var(--border-0); }
.si-vt-node { position: relative; padding: 8px 0 14px 20px; }
.si-vt-dot { position: absolute; left: -8px; top: 10px; width: 9px; height: 9px;
  border-radius: 50%; }
.si-vt-dot.current { background: var(--teal); box-shadow: 0 0 6px rgba(62,201,176,0.3); }
.si-vt-dot.past { background: transparent; border: 2px solid var(--tx-3); cursor: pointer; }
.si-vt-dot.past:hover { border-color: var(--tx-1); }

/* Lineage chain */
.si-lineage-node { padding: 8px 12px; border-radius: 6px; margin-bottom: 0; }
.si-lineage-connector { width: 1px; height: 16px; margin-left: 20px; position: relative; }
.si-lineage-connector::before { content: ''; position: absolute; left: 0; top: 0;
  bottom: 0; width: 1px; background: var(--border-1); }

/* Adoption bar */
.si-adopt-bar { height: 4px; border-radius: 2px; background: var(--border-0);
  overflow: hidden; margin: 6px 0 10px; }
.si-adopt-fill { height: 100%; background: var(--teal); border-radius: 2px;
  transition: width 300ms ease-out; }

/* Responsive */
@media(max-width:1100px) {
  .si-sidebar { width: 240px; min-width: 240px; }
  .si-context { width: 36px; min-width: 36px; }
}
@media(max-width:900px) {
  .si-sidebar { position: fixed; left: -300px; top: 56px; bottom: 0; width: 300px;
    z-index: 40; transition: left 250ms ease-out; box-shadow: 8px 0 32px rgba(0,0,0,0.4); }
  .si-sidebar.open { left: 0; }
  .si-context { display: none; }
  .si-canvas { padding: 14px; }
}
@media(max-width:700px) {
  .si-sidebar { width: 100%; left: -100%; }
  .si-sidebar.open { left: 0; }
}
```

---

## 7. STATE MANAGEMENT CHANGES

### 7.1 New State Shape (extends FormBuilder)

```javascript
// SchemaView wraps FormBuilder and manages:
const [sidebarOpen, setSidebarOpen] = useState(true);       // responsive toggle
const [contextOpen, setContextOpen] = useState(true);        // right panel toggle
const [selectedFormId, setSelectedFormId] = useState(null);   // which form is loaded
const [searchQuery, setSearchQuery] = useState('');           // sidebar search
const [expandedGroups, setExpandedGroups] = useState(
  new Set(['inbox', 'my-forms'])                              // which groups are open
);
const [formOrder, setFormOrder] = useState([]);               // MY FORMS sort order (IDs)
const [dirtyForms, setDirtyForms] = useState(new Set());      // forms with unsaved changes

// Computed from existing data:
const inboxForms = useMemo(() =>
  allForms.filter(f =>
    f.sourceType !== 'local' &&
    f.version > (f.lastSyncedVersion || 0)
  ), [allForms]);

const myForms = useMemo(() =>
  allForms.filter(f => f.sourceType === 'local')
    .sort((a, b) => {
      const ai = formOrder.indexOf(a.id);
      const bi = formOrder.indexOf(b.id);
      if (ai >= 0 && bi >= 0) return ai - bi;
      return (b.savedAt || 0) - (a.savedAt || 0);
    }), [allForms, formOrder]);

const orgForms = useMemo(() =>
  allForms.filter(f => f.sourceType === 'org'), [allForms]);

const networkForms = useMemo(() =>
  allForms.filter(f => f.sourceType === 'network'), [allForms]);

const filteredForms = useMemo(() => {
  if (!searchQuery || searchQuery.length < 2) return null;
  const q = searchQuery.toLowerCase();
  return allForms.filter(f =>
    f.name.toLowerCase().includes(q) ||
    f.key.toLowerCase().includes(q) ||
    (f.description || '').toLowerCase().includes(q) ||
    f.form.sections.some(s =>
      s.title.toLowerCase().includes(q) ||
      s.questions.some(qn => qn.prompt.toLowerCase().includes(q))
    )
  );
}, [allForms, searchQuery]);
```

### 7.2 Dirty Form Detection

```javascript
// Track if current form has diverged from last saved snapshot
const formFingerprint = (form) =>
  JSON.stringify({
    name: form.name, key: form.key, maturity: form.maturity,
    sections: form.sections.map(s => ({
      title: s.title,
      questions: s.questions.map(q => ({
        prompt: q.prompt, type: q.type,
        options: (q.options || []).map(o => o.label)
      }))
    }))
  });

const lastSavedFingerprint = useRef(null);

useEffect(() => {
  const current = formFingerprint(form);
  if (lastSavedFingerprint.current && current !== lastSavedFingerprint.current) {
    setDirtyForms(prev => new Set([...prev, form.id]));
  }
}, [form]);
```

### 7.3 IndexedDB Persistence (auto-save)

```javascript
// Auto-persist sidebar state every 2 seconds if dirty
useEffect(() => {
  const timer = setInterval(() => {
    if (dirtyForms.size > 0) {
      // Save to IndexedDB encrypted store
      localVault.put('schema-sidebar-state', {
        formOrder,
        expandedGroups: [...expandedGroups],
        lastSelectedFormId: selectedFormId,
      });
      // Auto-save dirty forms to IndexedDB (not Matrix — that requires explicit Save)
      dirtyForms.forEach(id => {
        const f = allForms.find(x => x.id === id);
        if (f) localVault.put(`schema-form-${id}`, f);
      });
      setDirtyForms(new Set());
    }
  }, 2000);
  return () => clearInterval(timer);
}, [dirtyForms, formOrder, expandedGroups, selectedFormId]);
```

---

## 8. KEYBOARD SHORTCUTS

| Key | Context | Action |
|-----|---------|--------|
| `Ctrl+N` | Schema view | New blank form (same as [+] button) |
| `Ctrl+S` | Schema view | Save current form to library |
| `Ctrl+Shift+S` | Schema view | Publish to Matrix (same as Publish button) |
| `Ctrl+[` | Schema view | Toggle sidebar |
| `Ctrl+]` | Schema view | Toggle context panel |
| `/` | Sidebar focused | Focus search bar |
| `Escape` | Search focused | Clear search |
| `↑` / `↓` | Sidebar focused | Navigate form list |
| `Enter` | Sidebar focused | Load selected form |
| `Ctrl+Shift+V` | Schema view | Open version bump modal |

---

## 9. ACCESSIBILITY

- **Sidebar**: `role="navigation"`, `aria-label="Form library"`
- **Group headers**: `role="button"`, `aria-expanded="true/false"`, `aria-controls="group-{id}"`
- **Form list**: `role="listbox"`, items are `role="option"`, `aria-selected="true/false"`
- **Context panel**: `role="complementary"`, `aria-label="Form context"`
- **Version timeline**: `role="list"`, each node `role="listitem"`
- **Focus trap**: when sidebar opens as overlay (mobile), focus traps inside
- **All interactive elements**: visible focus rings (`box-shadow: 0 0 0 2px ${SWC.given}40`)
- **Color-blind safe**: all states use shape + text in addition to color (dot shape, text labels, icons)

---

## 10. WIREFRAMES — Additional Views

### 10.1 Sidebar During Search

```
┌─────────────────────────────┐
│ ◇ SCHEMA           [+] [⋯] │
├─────────────────────────────┤
│ 🔍 housi                [×] │
├─────────────────────────────┤
│ ▾ MY FORMS (1 match)        │
│ ■ [Housi]ng Assess…         │  ← match highlighted
│   Local · v3 · trial        │
├─────────────────────────────┤
│ ▾ NETWORK (1 match)         │
│   [Housi]ng Stability       │
│   Network · v1 · draft      │
│   ↳ "housing" in question   │  ← match context
├─────────────────────────────┤
│ ▸ INBOX (0)    ── hidden ── │
│ ▸ ORG (0)      ── hidden ── │
└─────────────────────────────┘
```

### 10.2 Context Panel — Local Form (no lineage)

```
┌──────────────────────────────┐
│ CONTEXT                 [◀]  │
├──────────────────────────────┤
│                              │
│ LINEAGE                      │
│ ┌────────────────────────┐   │
│ │ ◆ Local form            │   │
│ │                         │   │
│ │ This form exists only   │   │
│ │ in your local library.  │   │
│ │                         │   │
│ │ [Publish to Org →]      │   │
│ │ [Propose to Network →]  │   │
│ └────────────────────────┘   │
│                              │
├──────────────────────────────┤
│                              │
│ VERSION TIMELINE             │
│                              │
│ ● v1 (current)               │
│   Created Feb 14, 2026       │
│   No prior versions.         │
│                              │
│   [Bump version]             │
│                              │
├──────────────────────────────┤
│                              │
│ RELATED FORMS                │
│ ┌────────────────────────┐   │
│ │ No related forms.       │   │
│ │ Forms sharing field     │   │
│ │ keys will appear here.  │   │
│ └────────────────────────┘   │
│                              │
├──────────────────────────────┤
│                              │
│ ACTIVITY                     │
│ ● Feb 14  Created            │
│                              │
└──────────────────────────────┘
```

### 10.3 Upstream Update Banner (in canvas)

```
┌────────────────────────────────────────────────────────────────────┐
│ ⚠  "Status & Engagement" was updated in the network (v1 → v2)    │
│    +2 questions, +4 options added                                  │
│                                           [Review changes] [Later] │
│                                                                    │
│    height: auto (collapsed ~44px, expanded ~70px)                  │
│    bg: rgba(SWC.sup, 0.06)                                         │
│    border-bottom: 1px solid ${SWC.sup}30                           │
│    padding: 10px 20px                                              │
│    animation: slideDown 200ms ease-out                             │
└────────────────────────────────────────────────────────────────────┘
```

### 10.4 Mobile Sidebar Overlay

```
┌────────────────────────────────────────┐
│ ╳               SCHEMA                 │  ← close button + title
├────────────────────────────────────────┤
│ 🔍 Search forms...                     │
├────────────────────────────────────────┤
│ ▾ INBOX (2)                            │
│ ● Status & Engagement Tracking         │
│   Network · v2 · normative             │
│ ● Intake Event Tracking                │
│   Network · v1.1 · trial     ▲ new     │
├────────────────────────────────────────┤
│ ▾ MY FORMS (3)                         │
│ ■ Housing Assessment                   │
│   Local · v3 · trial                   │
│   Risk Screening Tool                  │
│   Local · v1 · draft                   │
│   Daily Check-in                       │
│   Local · v2 · trial                   │
├────────────────────────────────────────┤
│ ▸ ORG FORMS (4)                        │
│ ▸ NETWORK COMMONS (8)                  │
│                                        │
│              (scrollable)              │
└────────────────────────────────────────┘

Backdrop: rgba(0,0,0,0.5)
Width: 300px (or 100% on <700px)
Slide from left: transform translateX(-100%→0) 250ms
```

### 10.5 Full Desktop — Network Admin View

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  ◇ Schema  ❯  Network Commons  ❯  Status & Engagement  ❯  v2         [Save] [⋮]    │
├──────────┬───────────────────────────────────────────────────────────┬───────────────┤
│ ◇ SCHEMA │  Status & Engagement Tracking                            │ CONTEXT    [◀]│
│ [+] [⋯]  │  normative · v2 · 6 questions · 18 options              │               │
│ ┄┄┄┄┄┄┄┄ │  ┌─── Compose ─── Wire ─── Preview ───┐                │ LINEAGE       │
│ 🔍 search│  │                                      │                │ ┌───────────┐ │
│          │  │  ⚠ This form is locked (normative).  │                │ │ ◇ CoC Net │ │
│ ▾ INBOX 0│  │  Changes require a governance        │                │ │   origin   │ │
│ ✓ caught │  │  proposal.  [Start proposal]         │                │ └───────────┘ │
│   up     │  │                                      │                │ ↓ required    │
│          │  │  ▾ General                            │                │ 4 of 5 orgs   │
│ ▾ MY   3 │  │    What is your current housing...   │                │               │
│   Housing│  │    ☐ Sheltered                        │                │ VERSION       │
│   Risk Sc│  │    ☐ Unsheltered  🔒 normative        │                │ v1 ─○─ Mar 25 │
│   Check… │  │    ☐ At risk                          │                │ v2 ─●─ Jun 25 │
│          │  │    ☐ Stably housed                    │                │               │
│ ▸ ORG  4 │  │                                      │                │ ADOPTION      │
│          │  │    How long in current...             │                │ ████████░░ 80%│
│ ▾ NET  8 │  │    ☐ <1 month                        │                │ ✓ Harbor House│
│ ■ Status…│  │    ☐ 1-6 months                      │                │ ✓ PATH Svc    │
│   Net v2 │  │    ☐ 6-12 months                     │                │ ✓ Salvation A │
│   normat…│  │    ☐ >1 year                         │                │ ✓ City Svc    │
│   Intake…│  │                                      │                │ ○ Housing Auth│
│   Net v1 │  │  ▸ Demographics                      │                │               │
│   trial  │  │  ▸ Engagement Metrics                │                │ ACTIVITY      │
│   Context│  │                                      │                │ Jun 15 v2 bump│
│   Net v1 │  └──────────────────────────────────────┘                │ Jun 10 2q add │
│   draft  │                                                          │ Mar 3  Created│
│   ...    │                                                          │               │
└──────────┴───────────────────────────────────────────────────────────┴───────────────┘
```

Note: normative forms show a lock banner in canvas and disable direct editing. Changes route through governance proposals.

---

## 11. EDGE CASES

| Scenario | Behavior |
|----------|----------|
| User has no org or network | INBOX, ORG FORMS, and NETWORK COMMONS groups show empty placeholders with explanatory text and CTAs |
| 50+ forms in a single group | Virtualized list rendering (render only visible items ±5 buffer). Show "Showing 12 of 53" at group header |
| Two forms share the same `key` | Group them visually in sidebar under one expandable row (like current Form Library groups by key) |
| Network form deleted upstream | Show in INBOX with `⚠ removed` flag in red, allow user to keep local copy or acknowledge deletion |
| Form with no questions | Show in sidebar normally, canvas shows empty state: "Add your first section to get started" |
| Simultaneous edit + incoming update | Gold banner in canvas; user's local changes are preserved, they choose to merge or ignore |
| Browser tab goes offline | Sidebar shows subtle offline indicator next to INBOX header; auto-retries when connection restores |
| User is read-only staff role | Sidebar shows all groups but context menu hides "Delete", "Publish", "Propose"; canvas is view-only |
