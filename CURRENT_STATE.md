# Current State

> **Bootstrap status:** LUCE is being established as the canonical living project repository. This file is intentionally sparse until legacy source sets are distilled and adjudicated. Do not infer missing facts from omissions here.

## Active project

**TomTom Revival**

Primary active target at repository bootstrap: **TT3 “Tomi” / TomiDock**.

## Current design baseline

The current TomiDock design is operational. GPIO4 is no longer artificially held HIGH and is electrically unconnected in the current no-GPIO design. Normal boot, USB enumeration, CDC-ECM networking, Wi-Fi routing/NAPT, and TCP/2323 forwarding have passed post-removal smoke testing.

## Current documentation status

Canonical topic documents will be added incrementally as legacy Dropbox/Codex/handoff material is reconciled into living Git documents.

Initial planned migrations include:

- TT3/Tomi device profile
- Tomi runtime / BusyBox ABI
- Tomi file-transfer workflow
- TomiDock architecture and network topology
- machine/environment roles
- critical bench safety constraints
- lightweight evidence index/provenance pointers

## Rule for fresh threads

Read this file first, then follow links to the relevant canonical topic document. Use chat handoffs only for ephemeral conversational context that is not yet represented in Git.
