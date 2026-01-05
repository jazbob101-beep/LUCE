# MASTER_LOADER_MANIFEST_v001

Created UTC: 2026-01-05T02:06:12.055061Z

## What this is
- Single source of truth for *how to load* masters and *which contracts* govern them.
- Contracts are **cumulative**. Newer contracts must not drop global gates.

## Preflight (mandatory)
1. Load this manifest.
2. Load latest masters by prefix in `/mnt/data`.
3. Validate container shapes at canonical paths.
4. If any shape check fails: **blocked_contract_mismatch** and stop.

## Canonical paths
- TREE: people_path=/people, indexes_path=/indexes
- TIMELINE: events_path=/events, indexes_path=/indexes
- FAN: fan_sets_path=/fan_sets, indexes_path=/indexes
- INDEX: by_person_path=/indexes/by_person
- YDNA: testers_path=/testers, indexes_path=/indexes

## Global gates (always enforced)
- no_dotted_path_keys_anywhere
- no_TMP_IDs_in_phase3_outputs_or_phase4_commits
- leaf_ops_only_json_pointer
- composite_overwrite_rejection
- regression_guard_no_collateral_key_loss

## Contract fingerprints (sha256)
- STRUCTURAL_INVARIANTS_Spec_v001.md: `9a30062208c8366109f5573186c5be46d600af671a674f0023adf8abd15b829a`
- TREE_Contract_v004.md: `4debf38d0c1dee250d436b6f8ad742f37ebd5ff73e09019cd05ee4dfb6587247`
- TIMELINE_Contract_v003.md: `9ba322b9641e46aab3500b58c0bb901b62473ad19ebd7a2131b09a3cf8b4cb18`
- FAN_Contract_v001.md: `d170abe581d89c9617cf067ee176220f456feff41d7a955f40c908deea680f95`
- INDEX_Contract_v001.md: `08635211a2c4939958ce1e1142aeb8d159e9b9739fa8e30936f61855ad02e341`
- SCHEMA_CONTRACT_TIMELINE_EventTaxonomy_v001.json: `f188623f37fee9a30e3cf6842cab3cf35ea49bdfb1c525ebe2ad032fb16289a0`