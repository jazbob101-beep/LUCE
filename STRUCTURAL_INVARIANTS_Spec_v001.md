# Structural Invariants Spec v1 (Stop-the-Fracture Gate)

Created (UTC): 2026-01-03T23:46:13Z

This spec defines **canonical container shapes**, **required invariants**, and a **commit gate checklist** for Phase-4.

No semantic normalization is included here (taxonomy, naming, etc.). This is **shape + integrity only**.


## Global invariants (all datasets)

- Masters must be valid JSON and parse without loss.
- No dotted-path keys anywhere (keys containing '.').
- All top-level containers must use a single canonical type per spec (no list/dict duality).
- ID keys must be deterministic and stable: PIDs (`P\d+`), EIDs (`E\d+`), FAN IDs (per FAN spec).
- When an object is stored in a dict keyed by its ID, the object must include an `id` field equal to the dict key.
- All cross-file references must resolve (or the commit must fail).
- Serialization must be canonical: sorted dict keys; stable ordering for set-like arrays (tags, etc.).
- INDEX is derived; never hand-edit INDEX without regeneration from authoritative sources.

## TIMELINE canonical shape

- `events` MUST be a **dict keyed by EID** (never a list).
- Each event object MUST include: `id`, `event_type`, `participants` (array; may be empty only if contract allows), and at least one of `date` or `date_range` per the timeline schema.
- `event_type` MUST be a string and MUST be canonical per the timeline event taxonomy contract.
- Legacy fields (`type`, `subtype`, etc.) are allowed only as preserved history; they must not drive canonical logic.
- Participants MUST be objects with `person_id` and `role` at minimum (if participants is present).
- All `participants[].person_id` MUST exist in TREE + INDEX after commit (no pseudo IDs).
- Date model MUST be one of the approved forms (choose and enforce): ISO date string OR structured object; do not mix within the same master.

## TREE canonical shape

- `persons` MUST be a **dict keyed by PID** (never a list).
- Each person object MUST include `id` matching the dict key.
- All relationship edges that reference other PIDs MUST resolve to existing persons (no placeholders).
- Spouse/parent/child references must be symmetrical where the data model requires symmetry (Phase-4 should enforce).

## FAN canonical shape

- FAN master MUST use a single canonical container type for top-level FAN entries (dict keyed by FAN_ID recommended).
- All member/edge references to PIDs must resolve to TREE/INDEX after commit (or explicitly be flagged as unresolved; no pseudo IDs).

## YDNA canonical shape

- Testers MUST be a dict keyed by subject_id.
- If linked_person_id is present, it must resolve to TREE/INDEX.

## INDEX canonical shape (derived)

- INDEX MUST be regenerated from authoritative masters after any commit that changes TREE/TIMELINE/FAN/YDNA.
- `indexes.by_person` (or equivalent canonical location) MUST be a dict keyed by PID.
- `indexes.by_event` (if present) MUST be a dict keyed by EID.
- All indexed references must resolve; no orphan keys.

## Phase-4 commit gate (fail-loud checklist)

- Validate each master against its schema/contract and this structural spec.
- Confirm canonical container types (no list/dict duality).
- Confirm dict key/id equality for persons/events.
- Confirm no dot-key keys.
- Confirm cross-file referential integrity: Timeline participants -> Tree + Index; Tree edges -> Tree; Index keys -> authoritative sources.
- Regenerate INDEX and compare key counts for sanity (no unexpected collapses/balloons).
- If any check fails: do not bump rev; do not write masters; emit an audit report listing all violations.