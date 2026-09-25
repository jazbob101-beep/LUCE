# Documentation Buckets

The documentation tree is intentionally small and should grow only when real material needs a canonical home.

## Buckets

### `devices/`
**Question answered:** What is this physical device?

Stable device identity, hardware facts, firmware/runtime identity, important device-specific paths, and durable capabilities/limitations.

### `architecture/`
**Question answered:** How is this system designed, and why?

System relationships, topology, component roles, design decisions, interfaces, and current architectural constraints.

### `runbooks/`
**Question answered:** How do I actually do X on the bench?

Copy/paste operational procedures that are current and proven. When a workflow changes, update the living runbook rather than creating a version-suffixed replacement.

### `reference/`
**Question answered:** What stable technical facts or compatibility rules must we remember?

Examples: BusyBox/app command surfaces, protocol facts, ABI constraints, addresses or identifiers whose duplication elsewhere would create drift.

### `environment/`
**Question answered:** Where does work happen and what is each machine/tool responsible for?

Host roles, build environments, canonical paths, source-tree boundaries, toolchain locations, and execution-environment rules.

## Classification rule

Prefer one canonical home for each durable fact. Other documents should link to that home rather than reproduce a second independently maintained copy.

When material genuinely spans buckets, put the authoritative detail where it fits best and keep only the minimum cross-reference elsewhere.

## Migration rule

Legacy documents are source material, not automatically canonical truth. Before material from them is adopted, reconcile its claims as current/supported, historical only, superseded, duplicate, contradictory, or unresolved. Only the adjudicated living result belongs here.
