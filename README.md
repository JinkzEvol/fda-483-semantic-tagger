# FDA 483 Semantic Tagger

Public reference repository for an FDA 483 semantic tagging specification and its flattened quality tag taxonomy.

## What This Is

FDA Form 483 observations document inspection findings identified by FDA investigators. This repository defines a traceable tagging model that turns those observations into structured semantic tags so they can be normalized, explained, and matched against enterprise quality-system objects.

The repository is specification-first. It contains normative documentation artifacts, not an application or service implementation.

## Who This Is For

- regulatory, quality, and compliance teams working with FDA 483 observations
- knowledge engineering or enrichment teams building structured tagging pipelines
- implementers who need a documented taxonomy for explainable matching across drug, biologic, and medical-device inspection content

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

## Status

- Document type: reference specification
- Coverage: drug, biologic, and medical-device inspection vocabulary
- Implementation content: none in this repository beyond normative specification artifacts
- Maintenance state: current files reflect the active v0.5 release noted in the source documents

## Current Version

- Skill version: `0.5`
- Taxonomy version: `0.5`
- Effective date: `2026-04-21`
- Pipeline version string: `tagger_v0_5`

## Usage

1. Read `flattened_quality_tag_taxonomy_v0_5.yaml` first.
2. Use the taxonomy as the authority for canonical values, aliases, value-citation anchors, and matching policy.
3. Apply the rules in `fda_483_semantic_tagger_SKILL_v0_5.md` to extract, normalize, and score tags from FDA 483 observations.

## Conformance Notes

Implementations should treat the taxonomy YAML as the source of truth for canonical values and matching policy, then apply the skill document's extraction, normalization, alias-resolution, and evaluation rules. The skill document also defines golden-set evaluation expectations for validating tagging behavior.

## Repository Notes

- The skill document supersedes the earlier v0.2 specification referenced in its revision history.
- The taxonomy remains additive relative to v0.4 and preserves prior stable category values.
- This repository currently contains the normative specification artifacts only.
- Internal change IDs and plan references are preserved inside the source documents for traceability. They are retained as historical document metadata, not as required external dependencies.