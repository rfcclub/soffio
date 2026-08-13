---
name: loomkit-archive
description: Merge specs, archive the change, and update living documentation.
tags: [loomkit, archive, phase-6]
---

# LoomKit Archive Phase (Hermes)

## Purpose
Finalize the change after successful verification.

## Actions
1. Merge specs into living `specs/` directory
2. Move change folder to `archive/`
3. Update any cross-references
4. Clean up temporary files

## Output
- Archived change in `archive/<name>/`
- Updated living specs
- Final traceability report

## Rules
- Only archive if `loomkit-verify` passed.
- Never delete history — only move to archive.