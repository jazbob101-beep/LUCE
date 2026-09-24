# LUCE

LUCE is the canonical living workshop repository for Chris + Luce projects, beginning with TomTom Revival.

## Purpose

This repository stores current operational truth, lightweight technical reference, runbooks, architecture notes, device profiles, environment definitions, reusable scripts, and pointers to external evidence.

It is designed to support an exploratory, improvisational engineering workflow without turning Chris into the project documentarian.

## Authority hierarchy

1. **LUCE `main`** — current operational/project truth for material that has been migrated here.
2. **Dropbox / NAS** — evidence warehouse for large captures, firmware, images, logs, immutable experiment packages, and other bulky artifacts.
3. **Working source trees / Codex workspaces** — active implementation and investigation areas. Their contents are not automatically canonical documentation.
4. **Chat memory / thread handoffs** — convenience context only. They do not override current Git documentation.

Historical evidence remains evidence even when current Git documentation describes a later state.

## Anti-Hydra rules

- A durable fact should have one canonical home whenever practical.
- Other documents should link to that home rather than maintain competing copies.
- Living canonical documents are updated **in place**.
- Do not create `v2`, `final`, dated replacement, or similar filenames merely to preserve revisions. Git history is the version archive.
- Obsolete current instructions should be removed or clearly marked historical when a newer proven workflow supersedes them.
- Large/raw evidence stays outside Git unless there is a specific reason to store it here.

## Stewardship model

Chris drives experiments, makes design decisions, and reports/corrects bench observations. Luce is responsible for keeping durable project knowledge coherent when asked to preserve it, including reading the current canonical file first, reconciling the new information, updating the existing living document, and committing the change.

Codex may be used for large mechanical consolidation, source-tree inspection, repetitive reconciliation, and migration work. Its output becomes canonical only after review/adjudication and commit to this repository.

## Repository shape

The structure grows only when real material needs a home. See [`docs/README.md`](docs/README.md) for documentation buckets and [`evidence/INDEX.md`](evidence/INDEX.md) for external evidence conventions.

Start with [`CURRENT_STATE.md`](CURRENT_STATE.md) for the concise present-tense project entry point.
