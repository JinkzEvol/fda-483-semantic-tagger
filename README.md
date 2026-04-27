# FDA 483 Semantic Tagger

Reference repository for the FDA 483 semantic tagging skill and its flattened quality tag taxonomy.

## Contents

- `fda_483_semantic_tagger_SKILL_v0_5.md`: active skill specification for FDA 483 observation tagging, enterprise matching, dual QSR/QMSR vocabulary resolution, alias-side emission, and golden-set evaluation.
- `flattened_quality_tag_taxonomy_v0_5.yaml`: taxonomy source of truth for canonical tag values, aliases, value-citation anchors, normalization rules, and update policy.

## Scope

This repository documents a traceable tagging model for FDA 483 observations. The current v0.5 release covers:

- semantic extraction and normalization of FDA 483 observation text
- enterprise matching using a flattened quality ontology
- dual QSR to QMSR vocabulary handling for medical device citations
- DMR to MDF alias-side emission support
- drug and biologic citation-anchor recognition for record and evidence tags
- golden-set evaluation requirements and backfill semantics

The taxonomy is intended to support explainable matching workflows rather than opaque similarity-only approaches.

## Current Version

- Skill version: `0.5`
- Taxonomy version: `0.5`
- Effective date: `2026-04-21`
- Pipeline version string: `tagger_v0_5`

## Usage

1. Read `flattened_quality_tag_taxonomy_v0_5.yaml` first.
2. Use the taxonomy as the authority for canonical values, aliases, value-citation anchors, and matching policy.
3. Apply the rules in `fda_483_semantic_tagger_SKILL_v0_5.md` to extract, normalize, and score tags from FDA 483 observations.

## Repository Notes

- The skill document supersedes the earlier v0.2 specification referenced in its revision history.
- The taxonomy remains additive relative to v0.4 and preserves prior stable category values.
- This repository currently contains the normative specification artifacts only.