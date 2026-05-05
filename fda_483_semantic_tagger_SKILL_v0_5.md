# SKILL: FDA 483 Semantic Tagger and Enterprise Matcher v0.5

> **Supersedes:** `fda_483_semantic_tagger_SKILL_v0_2.md` (WS-B acceptance; GXP-PLAN-PH2.2-001 §4.7).
> Historical v0.2 file is retained for audit trail. No further edits to v0.2.
>
> **Taxonomy dependency:** `flattened_quality_tag_taxonomy_v0_5.yaml` — minimum version required.
>
> **Plan reference:** GXP-PLAN-PH2.2-001 §4 (WS-B). Closes IMP-TAX-004, IMP-TAX-005,
> and the tagger-adjacent portion of IMP-TAX-011.

---

## Document Control

| Field | Value |
|---|---|
| Document ID | fda_483_semantic_tagger_SKILL |
| Version | 0.5 |
| Status | Active. Supersedes v0.2. Approved by Quality Domain Lead and Enrichment Lead per WS-B acceptance criterion 1. |
| Effective Date | 2026-04-21 |
| Scope | FDA 483 observation semantic tagging, enterprise matching, dual QSR↔QMSR vocabulary resolution, DMR→MDF alias-side emission, drug/biologic value_citations recognition, historical citation corpus backfill, and golden-set evaluation specification. |
| Depends on | `flattened_quality_tag_taxonomy_v0_5.yaml` (approved); `fda_483_scoring_policy_v0_2.md` (unchanged); `GXP-PLAN-PH2.2-001 v1.0` §4 (WS-B delivery vehicle). |
| Pipeline version string | `tagger_v0_5` |

---

## Revision History

| Version | Date | Summary |
|---|---|---|
| 0.2 | (prior release) | Initial skill document. Covered extraction, normalisation, enterprise matching, scoring, penalty/gate/cap rules, and taxonomy update policy. Taxonomy dependency: v0.2. Drug/biologic and laboratory-chromatography focus; no device vocabulary. |
| 0.5 | 2026-04-21 | Major update per WS-B (GXP-PLAN-PH2.2-001 §4). Five additions over v0.2: (1) Dual QSR↔QMSR vocabulary handling for medical-device citations — both vocabulary eras recognised and resolved to the same semantic value via taxonomy v0.5 aliases. (2) DMR→MDF alias-side emission — device_master_record_control and device_master_record_maintenance citations emit both the QSR tag and the alias-side ISO 13485 tag. (3) Drug/biologic value_citations gap-fills per IMP-TAX-011 — five additional anchors now recognised: record_type.batch_record, record_type.change_control_record, record_type.validation_report_record, evidence_type.validation_protocol_review, evidence_type.audit_trail_review. (4) Golden-set evaluation specification — 100-citation corpus (60 drug/biologic, 40 device) with precision/recall thresholds and dual-vocabulary correctness requirements. (5) Backfill-Strategy section — idempotent upsert semantics, upsert key, pipeline_version labelling, and batch activity events. No breaking changes to v0.2 scoring, penalty, or gate rules. All v0.2 behaviour is preserved and extended. |

---

## Purpose

Use a stable, flattened semantic tag taxonomy to:

1. Extract tags from raw FDA 483 observation text.
2. Normalize those tags into a canonical representation.
3. Match the 483 to enterprise quality objects.
4. Resolve dual QSR↔QMSR vocabulary to a single semantic value at match time.
5. Emit alias-side tags for the DMR→MDF family.
6. Propose controlled taxonomy updates when true gaps are found.

The primary objective is **traceable matching**, not opaque semantic similarity.

v0.5 extends v0.2 to cover the full medical-device vocabulary introduced in taxonomy v0.4 and the corrective additions in taxonomy v0.5, while preserving every v0.2 rule without modification.

---

## Tool Contract

### Required Tools

- `external_file_read(path)`
  - Reads the active taxonomy YAML file.
  - Must be called first in every session. The returned content is the authority for all category values, alias rules, value_citations anchors, and match policy.

### Recommended Tools

- `enterprise_search(filters_or_query)`
  - Retrieves candidate enterprise objects.
- `vector_search(text)`
  - Optional secondary recall mechanism; never the final ranking authority.
- `external_file_write(path, content)`
  - Only for approved, governed writes.
- `diff_tool(old, new)`
  - Used to review taxonomy changes.

---

## First Action

Always begin by reading the taxonomy file.

```text
external_file_read(path="{{taxonomy_path}}")
```

The taxonomy path must resolve to `flattened_quality_tag_taxonomy_v0_5.yaml` or a later version. Reject a taxonomy version below v0.5 — the dual-vocabulary aliases and drug/biologic value_citations required by this skill version are absent in earlier files.

Treat the returned file as the active source of truth for:

- Allowed categories and their canonical values.
- Alias rules (especially `aliases.control_object`, `aliases.process`, and `aliases.record_type`).
- Value_citations anchors (CFR / USP / ICH) used in anchor-matching.
- Match policy and scoring parameters.
- Taxonomy update policy.

Do not invent categories or values without checking the taxonomy first.

---

## Taxonomy Dependency — v0.5 Requirements

This skill requires taxonomy v0.5 because it depends on content that was absent or incomplete in earlier versions:

### Aliases added in v0.5

| Alias category | QSR-era key | Resolves to | Section |
|---|---|---|---|
| `aliases.control_object` | `device_master_record_control` | `medical_device_file_control` | ISO 13485 §4.2.3 |
| `aliases.process` | `device_master_record_maintenance` | `medical_device_file_maintenance` | ISO 13485 §4.2.3 |

These were promised in the taxonomy v0.4 revision history but not delivered in v0.4's YAML. v0.5 closes that gap.

### Aliases present in v0.4 (inherited in v0.5)

| Alias category | QSR-era key | Resolves to |
|---|---|---|
| `aliases.record_type` | `device_master_record` | `medical_device_file_record` |
| `aliases.dosage_form` | `tablet`, `capsule`, `suspension`, `sterile_injection`, `cream`, `ointment`, `powder` | Granular v0.3 successors |

### Value_citations gap-fills added in v0.5 (IMP-TAX-011)

The following drug/biologic entries now carry authoritative CFR/ICH anchors in `value_citations`. The tagger must recognise these when extracting evidence text that references the cited authority:

| Category | Value | Anchor |
|---|---|---|
| `record_type` | `batch_record` | 21 CFR 211.188; 21 CFR 211.186; ICH Q7 §6.7 |
| `record_type` | `change_control_record` | 21 CFR 211.100(b); 21 CFR 820.40; ICH Q10 §3.2.3; ICH Q7 §13 |
| `record_type` | `validation_report_record` | 21 CFR 211.100(a); FDA Process Validation Guidance 2011; ICH Q7 §12 |
| `evidence_type` | `validation_protocol_review` | 21 CFR 211.100(a); FDA Process Validation Guidance 2011; ICH Q7 §12 |
| `evidence_type` | `audit_trail_review` | 21 CFR 211.68(b); 21 CFR Part 11 §11.10(e); FDA Data Integrity Guidance 2018 |

These five entries existed as values in the taxonomy since v0.3 but lacked value_citations anchors. v0.5 back-fills those anchors. The tagger must now emit these tags when the evidence text cites the corresponding CFR sections or guidance documents, not only when the raw record-type label appears verbatim.

---

## Operating Model

The taxonomy has two layers.

### 1) Stable Semantic Ontology

These change slowly:

- system
- process
- quality_theme
- issue_pattern
- control_object
- record_type
- asset_type

### 2) Enterprise Registries

These can grow as the enterprise grows:

- site
- product
- sop_id
- policy_id
- event_id
- instrument_id
- procedure_id

Do not confuse new registry entries with ontology drift.

---

## Tagging Format

All tags must follow:

```text
category:value
```

Normalize values to:

- lowercase
- snake_case
- canonical terminology from the taxonomy

Each tag must include:

- `status`: `observed` or `inferred`
- `confidence`: `0.0` to `1.0`
- `evidence`: exact text span or precise excerpt
- `rationale`: one short explanation
- `alias_emitted` (v0.5 addition): `true` if this tag was generated as an alias-side emission alongside its QSR-era counterpart; omit or set `false` otherwise
- `vocabulary_era` (v0.5 addition for device tags): `qsr` (pre-2026-02-02), `qmsr` (2026-02-02 or later), or `dual` (tag applies to both eras); omit for non-device tags

---

## Extraction Process

For each observation in the FDA 483:

1. Identify explicit direct identifiers:
   - site
   - product
   - dosage_form
   - procedure_id
   - sop_id
   - instrument_id

2. Identify semantic tags:
   - system
   - quality_theme
   - process
   - control_object
   - issue_pattern

3. Identify context tags:
   - org_unit
   - record_type
   - asset_type
   - scope where appropriate

4. Identify supportive tags:
   - evidence_type
   - risk_domain

5. Keep observed and inferred tags separate.

6. Prefer a broader canonical value over an invented narrow one when text is ambiguous or redacted.

7. **[v0.5 addition]** Determine vocabulary era of device citations:
   - If the citation explicitly cites a QSR section (e.g., `§820.181`, `Subpart M`) and is known or dated pre-2026-02-02, tag `vocabulary_era: qsr`.
   - If the citation uses QMSR/ISO 13485 language (e.g., `§4.2.3 Medical Device File`, `Compliance Program 7382.850`) or is dated 2026-02-02 or later, tag `vocabulary_era: qmsr`.
   - If both vocabulary forms appear in the same observation, tag `vocabulary_era: dual`.

8. **[v0.5 addition]** For each device tag emitted, consult `aliases.control_object` and `aliases.process` in the taxonomy. If the emitted tag key is an alias source, perform alias-side emission (see §Alias Emission Rules below).

9. **[v0.5 addition]** For drug/biologic record_type and evidence_type tags, check the evidence text for citation of CFR sections or guidance documents listed in taxonomy v0.5 `value_citations`. Emit the corresponding tag even when the literal record-type label is absent, if the citation anchor is found.

---

## Dual QSR↔QMSR Vocabulary Handling

### Background

The FDA's Quality System Regulation (QSR, 21 CFR Part 820 pre-2026-02-02) was superseded on 2026-02-02 by the Quality Management System Regulation (QMSR), which incorporates ISO 13485:2016 by reference. The QMSR introduced new terminology (e.g., "Medical Device File" for what QSR called "Device Master Record"), added §820.45 for labeling visual verification, and replaced the Quality System Inspection Technique (QSIT) with Compliance Program 7382.850. All 149 Part 820 citations in the current corpus pre-date the transition and use QSR vocabulary. Forward citations will progressively adopt QMSR/ISO 13485 terminology.

### Matching Algorithm

The tagger handles dual vocabulary by treating QSR anchors and QMSR anchors as co-equal triggers for the same semantic tag.

**Step 1 — Anchor detection.** Scan the observation text for CFR section references (`§820.xx`), QSR subpart letters (`Subpart C`, `Subpart M`), ISO 13485 clause references (`§7.3.x`, `§4.2.3`), and QSIT/Compliance Program references.

**Step 2 — Vocabulary-era classification.** Classify each anchor as `qsr`, `qmsr`, or `dual`:
- QSR anchor patterns: `§820.30`, `§820.181`, `§820.184`, `§820.198`, `Device Master Record`, `QSIT`, subpart letters.
- QMSR anchor patterns: `§4.2.3`, `§7.3.x`, `Medical Device File`, `Compliance Program 7382.850`, `ISO 13485`.
- Dual pattern: text that uses both vocabularies or is unambiguously described by taxonomy values that carry dual QSR+ISO anchors in `value_citations`.

**Step 3 — Tag emission.** For each detected anchor, look up the matching taxonomy value in `value_citations`. Where the `value_citations` entry carries both a QSR and an ISO 13485 anchor (e.g., `design_verification: "21 CFR 820.30(f); ISO 13485:2016 §7.3.6"`), emit a single tag for that value regardless of which vocabulary era the observation text uses. Set `vocabulary_era` accordingly.

**Step 4 — Alias resolution.** Where taxonomy v0.5 `aliases` maps a QSR value to a QMSR equivalent (DMR→MDF family — see §Alias Emission Rules), perform alias-side emission in addition to the primary tag.

**Step 5 — Activity event.** When both QSR and QMSR anchors triggered during Steps 1–2 for the same citation, emit the activity event `DUAL_VOCABULARY_RESOLVED` (see §Activity Events).

### Same-Semantic-Value Guarantee

The dual-vocabulary mechanism guarantees that a citation worded in QSR language and a citation worded in QMSR language for the same regulatory obligation produce identical semantic tags. This is tested in the golden-set eval (see §Golden-Set Evaluation Specification). The alias at match-policy time is a matching-side hint, not a relabelling rule — both QSR and QMSR observations resolve to the same taxonomy value, making downstream SPEC-225 scoring vocabulary-era agnostic.

### Scope Constraint

Only the DMR→MDF concept pair is a genuine QSR→QMSR rename. Design History File (DHF, §820.30(j)) and Device History Record (DHR, §820.184) are preserved conceptually across QSR and QMSR — ISO 13485 §7.3.10 and §4.2.5 are refinements, not renames. Taxonomy v0.5 does not introduce ISO-side target values for DHF or DHR; therefore no alias-side emission applies to DHF or DHR tags. If a future taxonomy version introduces dedicated ISO-side targets for those concepts, alias emission will apply then.

---

## Alias Emission Rules

Alias-side emission applies whenever a QSR-worded citation triggers a tag whose key is listed as an alias source in taxonomy v0.5.

### Trigger Conditions

Alias-side emission fires when ALL of the following are true:

1. The observation text uses QSR-era vocabulary referencing the Device Master Record concept, i.e., any of:
   - Explicit citation of `21 CFR 820.181`
   - Text containing "Device Master Record" or "DMR" in a QSR context
   - QSR Subpart M (§820.180–186) reference combined with device-master-record language

2. The tagger would emit at least one of the DMR-family tags:
   - `control_object:device_master_record_control`
   - `process:device_master_record_maintenance`
   - `record_type:device_master_record`

### Emission Procedure

For each DMR-family tag emitted, emit an additional sibling tag for the alias target:

| Primary tag emitted | Alias-side tag emitted | Alias source |
|---|---|---|
| `control_object:device_master_record_control` | `control_object:medical_device_file_control` | `aliases.control_object.device_master_record_control` |
| `process:device_master_record_maintenance` | `process:medical_device_file_maintenance` | `aliases.process.device_master_record_maintenance` |
| `record_type:device_master_record` | `record_type:medical_device_file_record` | `aliases.record_type.device_master_record` |

### Alias-Side Tag Fields

The alias-side tag carries:

```json
{
  "tag": "control_object:medical_device_file_control",
  "status": "inferred",
  "confidence": 0.85,
  "evidence": "<same evidence span as the primary DMR tag>",
  "rationale": "Alias-side emission: device_master_record_control (QSR §820.181) resolves to medical_device_file_control (ISO 13485 §4.2.3) per taxonomy v0.5 aliases.control_object. MDF is a superset of DMR; retain the QSR-anchored primary tag for the §820.181 citation.",
  "alias_emitted": true,
  "vocabulary_era": "qsr"
}
```

Set confidence on alias-side tags to at most `0.85` unless the citation explicitly mentions ISO 13485 §4.2.3 alongside the QSR section, in which case set confidence to match the primary tag. The alias is a best-effort mapping — the MDF is a superset of the DMR, not an identity — and the lower ceiling reflects that asymmetry.

### Scope Disambiguation

Both the QSR-anchored primary tag and the ISO 13485-anchored alias-side tag must coexist in the output for the same observation. Downstream systems consuming the tags must retain both. Match-policy resolution at SPEC-225 time uses the alias to treat both as pointing to the same semantic intent, but the record of which vocabulary era was used is preserved in `vocabulary_era` on each tag.

Do NOT perform alias-side emission for DHF or DHR tags. `control_object:design_history_file_control`, `process:design_history_file_maintenance`, and `record_type:design_history_file_record` have no alias target in taxonomy v0.5 and must not have one fabricated.

### Activity Event

When alias-side emission fires for any DMR-family tag, emit the activity event `TAG_EMITTED_UNDER_ALIAS` (see §Activity Events).

---

## Drug/Biologic Value_Citations Anchors

### Background

Audit v1.2 §5.3 (IMP-TAX-011) identified that several drug/biologic values in `record_type` and `evidence_type` carried thin or absent `value_citations` anchors relative to their device-side equivalents. Taxonomy v0.5 back-fills five entries. This section specifies how the tagger uses those anchors.

### Anchor-Driven Emission

For the five anchored values, the tagger must emit the tag when the observation text cites the regulatory reference, even if the literal label (e.g., "batch record") does not appear verbatim. The following trigger patterns must be recognised:

#### record_type:batch_record

- Regulatory citation triggers: `21 CFR 211.188`, `21 CFR 211.186`, `ICH Q7 §6.7`
- Text triggers: "master production and control record", "batch production record", "batch record", "manufacturing record" in a 21 CFR 211 context

#### record_type:change_control_record

- Regulatory citation triggers: `21 CFR 211.100(b)`, `21 CFR 820.40`, `ICH Q10 §3.2.3`, `ICH Q7 §13`
- Text triggers: "change control record", "change control documentation", "written procedure for change control", "document controls" in a 21 CFR 211 or 820 context

#### record_type:validation_report_record

- Regulatory citation triggers: `21 CFR 211.100(a)`, FDA Process Validation Guidance 2011, `ICH Q7 §12`
- Text triggers: "validation report", "process validation report", "validation documentation", "IQ/OQ/PQ report"

#### evidence_type:validation_protocol_review

- Regulatory citation triggers: `21 CFR 211.100(a)`, FDA Process Validation Guidance 2011, `ICH Q7 §12`
- Text triggers: "reviewed the validation protocol", "protocol review", "review of validation protocol", "validation protocol was reviewed", "reviewed the IQ/OQ/PQ protocol"

#### evidence_type:audit_trail_review

- Regulatory citation triggers: `21 CFR 211.68(b)`, `21 CFR Part 11 §11.10(e)`, FDA Data Integrity Guidance 2018
- Text triggers: "reviewed the audit trail", "audit trail was reviewed", "audit trail review", "reviewed audit trail entries", "21 CFR Part 11 audit trail", "reviewed electronic records audit trail"

### Confidence Calibration for Anchor-Driven Emission

When the tag is emitted because a regulatory anchor was detected in text rather than because the literal record-type label appeared verbatim:

- Set `status: inferred`
- Set `confidence: 0.75` to `0.90` depending on specificity of the anchor citation
- Include both the anchor text and the inferred label in `evidence`
- Set `rationale` to identify the anchor that triggered the inference

When the literal label appears verbatim AND the anchor citation appears:

- Set `status: observed`
- Set `confidence: 0.90` to `1.0`

---

## Device Vocabulary Handling

### Two-Era Coverage Requirement

Every device-domain 483 observation must be processed through both QSR and QMSR anchor tables simultaneously. Do not route device citations exclusively through QSR vocabulary even though the current corpus is entirely pre-transition. New citations from 2026-02-02 onward will use QMSR language, and the tagger must handle them without reconfiguration.

### QSR Vocabulary — Primary Anchor Patterns

The 149 Part 820 citations in the current corpus use these QSR subpart references:

| QSR Subpart | Sections | Dominant subject | Primary taxonomy tags |
|---|---|---|---|
| B | §820.20–25 | Management responsibility, quality audit, personnel | `process:management_review_execution`, `process:internal_quality_audit_execution`, `issue_pattern:management_review_not_performed`, `issue_pattern:quality_audit_not_performed_or_inadequate` |
| C | §820.30–39 | Design controls | `system:device_design_control_system`, `process:design_planning` through `process:design_history_file_maintenance`, `control_object:design_input_control` through `control_object:design_history_file_control` |
| D | §820.40–50 | Document controls | `control_object:qms_document_control` |
| E | §820.50 | Purchasing controls | `quality_theme:supplier_quality_management` |
| F | §820.60–70 | Identification and traceability | `process:device_identification_marking`, `process:device_udi_assignment`, `process:device_traceability_assignment`, `control_object:device_udi_control`, `control_object:device_traceability_control` |
| G | §820.70–96 | Production and process controls | `process:device_manufacturing_operation`, `process:device_labeling_visual_verification` (§820.45 QMSR addition), `control_object:device_nonconforming_material_control` |
| H | §820.100 | Acceptance activities | `process:incoming_device_acceptance`, `process:in_process_device_acceptance`, `process:final_device_acceptance` |
| I | §820.120–130 | Nonconforming product | `control_object:device_nonconforming_material_control` |
| J | §820.100 | CAPA | `control_object:capa_execution_control` |
| L | §820.140–160 | Handling, storage, distribution, installation | `process:device_installation_operation` |
| M | §820.180–186 | Records — **DMR/DHR** | `control_object:device_master_record_control` (alias → `medical_device_file_control`), `process:device_master_record_maintenance` (alias → `medical_device_file_maintenance`), `record_type:device_master_record` (alias → `medical_device_file_record`), `control_object:device_history_record_control`, `record_type:device_history_record` |
| N | §820.198–200 | Complaints and servicing | `process:complaint_intake`, `process:complaint_investigation`, `process:device_mdr_reportability_assessment`, `process:device_servicing_operation` |
| O | §820.250 | Statistical techniques | (use existing `quality_theme:analytical_reliability`) |

### QMSR Vocabulary — Primary Anchor Patterns

QMSR citations use ISO 13485:2016 clause references. Map these to the same semantic tags as their QSR equivalents:

| ISO 13485:2016 clause | Subject | Maps to same taxonomy tag as |
|---|---|---|
| §4.2.3 | Medical Device File | `control_object:medical_device_file_control` (no alias needed — this IS the QMSR-primary value) |
| §5.5.2 | Management representative | `control_object:management_representative_designation_control` |
| §5.6 | Management review | `process:management_review_execution` |
| §7.3.2–7.3.9 | Design controls | `process:design_planning` through `process:design_change_control_execution` |
| §7.3.10 | Design and development records | `record_type:design_history_file_record` (no DMR alias; DHF preserved) |
| §7.4.3 | Incoming acceptance | `process:incoming_device_acceptance` |
| §7.5 | Production and service provision | `process:device_manufacturing_operation` |
| §7.5.3 | Installation | `process:device_installation_operation` |
| §7.5.4 | Servicing | `process:device_servicing_operation` |
| §7.5.8 | Identification | `process:device_identification_marking` |
| §7.5.9 | Traceability | `process:device_traceability_assignment` |
| §8.2.2 | Complaint handling | `process:complaint_intake`, `process:complaint_investigation` |
| §8.2.4 | Internal audit | `process:internal_quality_audit_execution` |
| §8.2.5 | Acceptance activities | `process:in_process_device_acceptance`, `process:final_device_acceptance` |
| §8.3 | Nonconforming product | `control_object:device_nonconforming_material_control` |

### Adjacent Regulations — Part 803, 806, 807, 812, 830

| Regulation | Subject | Primary tags |
|---|---|---|
| 21 CFR Part 803 | Medical Device Reporting | `process:device_mdr_reportability_assessment`, `process:medical_device_report_submission`, `control_object:device_mdr_reportability_control`, `record_type:mdr_record`, `issue_pattern:mdr_not_submitted_timely`, `issue_pattern:complaint_not_evaluated_for_mdr_reportability` |
| 21 CFR Part 806 | Corrections and Removals | `process:device_corrections_and_removals_execution`, `control_object:device_corrections_removals_control`, `record_type:corrections_removals_report_record`, `issue_pattern:corrections_removals_not_reported` |
| 21 CFR Part 807 | Establishment Registration and Listing | `system:medical_device_quality_management_system` (supporting context) |
| 21 CFR Part 812 | IDE | `record_type:ide_record` |
| 21 CFR Part 830 | UDI | `process:device_udi_assignment`, `control_object:device_udi_control`, `issue_pattern:udi_not_in_dhr` |
| 21 CFR §820.45 | Labeling/packaging visual verification (QMSR addition) | `process:device_labeling_visual_verification`, `control_object:device_labeling_verification_control`, `quality_theme:labeling_accuracy` |

---

## Matching Process

After tagging the source document, retrieve candidate enterprise objects and score them using the taxonomy's match policy.

### Retrieval Order

1. Direct identifier filters first, when present.
2. Semantic retrieval second.
3. Vector retrieval only as a recall aid, never as the final ranking authority.

### Matching Priority

Use category priority in this order:

1. Direct identifiers
2. Semantic overlap
3. Context overlap
4. Supportive overlap

### Required Behaviour

- If an explicit direct identifier exists in the source, do not allow a purely semantic match to outrank a candidate with a true direct observed match unless that candidate is blocked by conflict or cap rules.
- Show the exact shared tags in the output.
- Show critical missing tags.
- Explain caps and penalties.

### Alias Resolution at Match Time

At match time, the alias mechanism allows a tag on either side of an alias pair to count as a match against the other. Specifically:

- A source tag of `control_object:device_master_record_control` (QSR era) matches a candidate tag of `control_object:medical_device_file_control` (QMSR era) via `aliases.control_object`.
- A source tag of `control_object:medical_device_file_control` matches a candidate tag of `control_object:device_master_record_control`.
- The same bidirectional matching applies to `process:device_master_record_maintenance` ↔ `process:medical_device_file_maintenance` and `record_type:device_master_record` ↔ `record_type:medical_device_file_record`.

This guarantees that an enterprise quality object authored under QSR vocabulary and a 483 citation worded in QMSR language (or vice versa) score the same semantic match as a same-era pair.

Alias-resolved matches score at the same weight as direct same-value matches. They are not penalised as partial matches. Document the alias resolution in the `rationale` field of the match result.

---

## Scoring Model

Use the following scoring shape (inherited unchanged from v0.2):

```text
score =
    5 * direct_exact_matches
  + 3 * semantic_matches
  + 2 * context_matches
  + 1 * supportive_matches
  - 4 * direct_conflicts
  - 2 * semantic_conflicts
  + 2 * has_direct_match_boost
  + 1 * semantic_cluster_boost
```

### Observed vs. Inferred Weighting

```text
observed = 1.0
inferred = 0.6
```

Recommended per-tag contribution:

```text
contribution = base_weight * min(source_tag_weight, candidate_tag_weight)
```

Alias-side tags carry `status: inferred` and are subject to the `0.6` certainty weight unless the original primary tag was observed AND the alias resolves to an exact stable-category match, in which case the alias-side tag inherits the primary tag's weight.

### Category Buckets

#### Direct

- site
- product
- dosage_form
- procedure_id
- sop_id
- instrument_id

#### Semantic

- system
- quality_theme
- process
- control_object
- issue_pattern

#### Context

- org_unit
- record_type
- asset_type
- scope

#### Supportive

- evidence_type
- risk_domain

---

## Penalties

### Direct Conflicts

Penalize heavily for conflicting explicit identifiers:

- Conflicting site
- Conflicting product
- Conflicting dosage_form
- Conflicting procedure_id
- Conflicting sop_id
- Conflicting instrument_id

### Semantic Conflicts

Penalize semantically incompatible matches, such as:

- Calibration SOP returned for batch record review
- Facility cleaning document returned for chromatography data issue without fit
- Equipment maintenance procedure returned for QC method execution issue without appropriate scope
- [v0.5 addition] QSR-era design-control document returned for a blood/plasma CGMP observation
- [v0.5 addition] Drug/biologic CAPA procedure returned for a device MDR reporting observation without cross-domain scope evidence

---

## Gating and Caps

### Site Gate

If the source contains an explicit observed site tag and the candidate:

- Does not share that site, AND
- Does not have `scope:global`,

then cap the score.

```text
max_score = 3
```

### Product Gate

If the source contains an explicit observed product tag and the candidate does not share that product, cap the score.

```text
max_score = 4
```

### Inference-Only Gate

If the candidate shares only inferred tags and no observed shared tags, cap the score.

```text
max_score = 2
```

### Workflow Compatibility Gate

If the retrieval workflow expects a specific `record_type` or `process` and the candidate is incompatible, cap the score.

```text
max_score = 3
```

---

## Boost Rules

### Direct Match Boost

Add `+2` when at least one direct category has an exact shared observed tag.

### Semantic Cluster Boost

Add `+1` when:

- At least two semantic tags match, AND
- At least one `control_object` or `issue_pattern` tag matches.

---

## Tie-Breakers

When scores are tied or close, break ties in this order:

1. More observed shared tags
2. More direct shared tags
3. Stronger evidence density
4. Current status over superseded status
5. Recency if relevant

---

## Output Contract

Return JSON with this structure:

```json
{
  "taxonomy_version_used": "0.5",
  "pipeline_version": "tagger_v0_5",
  "document_metadata": {},
  "observation_tags": [
    {
      "observation_id": "",
      "vocabulary_era_detected": "qsr|qmsr|dual|non_device",
      "dual_vocabulary_resolved": false,
      "tags": [
        {
          "tag": "category:value",
          "status": "observed|inferred",
          "confidence": 0.0,
          "evidence": "",
          "rationale": "",
          "alias_emitted": false,
          "vocabulary_era": "qsr|qmsr|dual|null"
        }
      ]
    }
  ],
  "document_level_tags": [],
  "candidate_matches": [
    {
      "object_id": "",
      "object_type": "",
      "score": 0,
      "shared_tags": [],
      "alias_resolved_matches": [],
      "missing_critical_tags": [],
      "penalties_applied": [],
      "caps_applied": [],
      "rationale": ""
    }
  ],
  "taxonomy_update_proposals": [
    {
      "category": "",
      "proposed_value": "",
      "proposal_type": "new_value|new_alias|new_registry_entry|new_category",
      "reason": "",
      "evidence": "",
      "expected_reuse": "high|medium|low",
      "decision": "proposed|rejected"
    }
  ],
  "coverage_gaps": [],
  "activity_events": []
}
```

### v0.5 Output Field Additions

- `taxonomy_version_used`: must be `"0.5"` or later.
- `pipeline_version`: must be `"tagger_v0_5"` for all outputs produced by this skill version.
- `observation_tags[].vocabulary_era_detected`: the vocabulary era detected for this observation.
- `observation_tags[].dual_vocabulary_resolved`: `true` when both QSR and QMSR anchors were detected in the same observation.
- `tags[].alias_emitted`: `true` when this tag was generated as an alias-side emission.
- `tags[].vocabulary_era`: the vocabulary era applicable to this specific tag.
- `candidate_matches[].alias_resolved_matches`: list of tag pairs that were matched via alias resolution at match time (e.g., `{source: "control_object:device_master_record_control", candidate: "control_object:medical_device_file_control", alias_source: "aliases.control_object"}`).
- `activity_events`: list of activity event objects emitted during this run (see §Activity Events).

---

## Golden-Set Evaluation Specification

### Corpus Composition

The golden-set evaluation corpus consists of 100 hand-labelled FDA 483 citations:

| Segment | Count | Description |
|---|---|---|
| Drug/biologic | 60 | Covers sterile manufacturing, solid-dose, API/biologics, laboratory/data integrity, packaging/labeling, change control, CAPA, OOS, stability, environmental monitoring |
| Device — QSR-worded | at least 15 | Citations dated pre-2026-02-02, using QSR vocabulary (§820.xx sections, QSR subpart letters, QSIT references, "Device Master Record", "Device History Record") |
| Device — QMSR-worded | at least 15 | Citations dated 2026-02-02 or later, using QMSR vocabulary (ISO 13485 clause references, "Medical Device File", Compliance Program 7382.850 references) |
| Device — DMR-family | at least 3 | Citations within the device subset that specifically reference the Device Master Record / Medical Device File concept (§820.181, §4.2.3, or DMR/MDF text) — used for alias-side emission verification |

The device subset total is 40 citations. Segments overlap: DMR-family citations are a subset of QSR-worded or QMSR-worded citations.

### Evaluation Metrics

#### Tag-Level Precision

```text
precision = (correctly_emitted_tags) / (total_emitted_tags)
```

A tag is "correct" when it appears in the hand-labelled ground truth for the same observation.

**Acceptance threshold: ≥ 90% precision.**

#### Tag-Level Recall

```text
recall = (correctly_emitted_tags) / (total_ground_truth_tags)
```

**Acceptance threshold: ≥ 85% recall.**

#### Dual-Vocabulary Resolution Correctness

For each citation that contains both a QSR anchor and a QMSR anchor, verify that:

1. Both anchors were detected.
2. The `dual_vocabulary_resolved` flag is `true` in the output.
3. The emitted semantic tags for both anchor types are identical (same `category:value` string).

**Acceptance threshold: 100% of such citations must pass all three conditions.**

#### Alias-Side Emission Completeness

For each DMR-family citation (i.e., a citation referencing Device Master Record, §820.181, or Medical Device File, §4.2.3):

1. The primary QSR-anchored tag must be emitted (e.g., `control_object:device_master_record_control`).
2. The alias-side tag must be emitted alongside it (e.g., `control_object:medical_device_file_control`).
3. The alias-side tag must carry `alias_emitted: true`.

**Acceptance threshold: 100% of DMR-family citations must satisfy all three conditions.**

### Evaluation Procedure

1. Prepare the 100-citation corpus as a JSONL file with one object per citation containing `citation_id`, `raw_text`, `observation_date` (or best estimate), `program_area`, and `ground_truth_tags` (the hand-labelled ground-truth tag set).
2. Run each citation through the tagger with `taxonomy_version_used = "0.5"` and `pipeline_version = "tagger_v0_5"`.
3. Compare emitted tags to `ground_truth_tags` at the `category:value` level. Alias-side tags are scored separately from primary tags.
4. Compute precision, recall, dual-vocabulary resolution correctness, and alias-side emission completeness.
5. Record results in the golden-set eval report artefact. Cross-reference back to WS-B acceptance criterion 2 (GXP-PLAN-PH2.2-001 §4.7).

### Failure Handling

- If precision or recall falls below threshold, diagnose whether the failure is concentrated in a specific domain (device QSR, device QMSR, drug/biologic, or a specific CFR subpart).
- If dual-vocabulary resolution correctness is below 100%, identify which anchor types failed to trigger.
- If alias-side emission is below 100%, identify whether the trigger conditions in §Alias Emission Rules were not detected or were detected but emission was suppressed.
- If the QMSR corpus (post-2026-02-02 citations) is too small to evaluate naturally, supplement with synthetic QMSR-worded examples. Document the supplement in the eval report and note it as a provisional acceptance pending organic corpus growth (Risk R2 per GXP-PLAN-PH2.2-001 §11.1).

---

## Backfill Strategy

This section specifies the strategy for re-tagging the historical `public.citations_483` corpus and writing results to `enriched.citation_tags`.

### Rationale for Full Overwrite

Phase 2.1 v1.3 CT-079 gates SPEC-225 on `pipeline_version = 'tagger_v0_5'` rows in `enriched.citation_tags`. Retaining pre-v0.5 rows alongside v0.5 rows requires every SPEC-223/225 query to filter on `pipeline_version`, adding permanent query complexity and creating a silent-bug category (a filter omission produces mixed-version output). Taxonomy v0.5 is strictly additive over v0.4: no value was removed or renamed, so retagging under v0.5 produces a superset of prior tags with no information loss. The `source_payload_hash` stale-write guard makes the upsert idempotent.

**Decision (OQ3, GXP-PLAN-PH2.2-001 §11.3.3): idempotent upsert, overwrite pre-v0.5 rows.**

### Upsert Key

```sql
ON CONFLICT (citation_id, cfr_section, category, value)
  DO UPDATE SET
    confidence         = EXCLUDED.confidence,
    tagged_by          = EXCLUDED.tagged_by,
    pipeline_version   = EXCLUDED.pipeline_version,
    source_payload_hash = EXCLUDED.source_payload_hash,
    tagged_at          = EXCLUDED.tagged_at,
    updated_at         = now()
```

If the table's existing unique constraint uses a different column set, use that constraint. The intent is that a (citation, tag) pair is unique and re-running the tagger updates the row in place.

### Pipeline Version Labelling

All rows written or updated by the v0.5 backfill must carry:

```sql
pipeline_version = 'tagger_v0_5'
```

This is the value CT-079 checks. Do not use any other format or casing variant.

### Role and Permission

All writes must be performed under role `enrichment_writer` per WS-A Option A pattern:

```sql
SET LOCAL ROLE enrichment_writer;
```

This is applied per-transaction by the shared DB-wrapper helper introduced in WS-A. Direct writes under `service_role` will fail with `insufficient_privilege` on `enriched.*`.

### Batch Processing

Process the corpus in batches to avoid Cloudflare Worker memory and subrequest limits:

- Recommended batch size: 100 citations per Worker invocation.
- Claim and process each batch atomically: read from `public.enrichment_outbox`, tag the citations, write to `enriched.citation_tags`, update `enrichment_outbox` status to `processed`.
- At the end of each batch, emit the activity event `CITATION_BACKFILL_BATCH_COMPLETE` with the batch size and row count written.

### Stale-Write Guard

Before writing a tag row, compare `source_payload_hash` of the current `public.citations_483` row against the hash stored in `enriched.citation_tags` for the same citation. If the hash matches and the existing row already carries `pipeline_version = 'tagger_v0_5'`, skip the write to avoid unnecessary I/O. If the hash differs or the existing row carries an older `pipeline_version`, perform the upsert.

### Resumability

The backfill run must be resumable from the last successfully processed batch. Use `public.enrichment_outbox` status transitions (`pending` → `claimed` → `processed`) to track which citations have been tagged. If the run is interrupted, re-run from the first unclaimed batch.

### Verification

After the backfill completes:

1. Run the distinct tag-value enumeration query (audit v1.2 §5.6 Query 1) against `enriched.citation_tags` filtered to `pipeline_version = 'tagger_v0_5'`.
2. Verify that: (a) no tag values appear that are absent from taxonomy v0.5 `stable_categories`; (b) the expected device-side tag values appear for device-program-area citations; (c) alias-side tags (`alias_emitted = true`) appear for DMR-family citations.
3. Spot-check 50 random citations against expected output from the golden-set eval spec.
4. Confirm CT-MON-001 remains green throughout the backfill run.

---

## Activity Events

The tagger emits activity events as structured log entries in the `activity_events` array of the output JSON and into the enrichment pipeline activity log (`mon-logs.md` activity enum).

### Event Schema

```json
{
  "event_type": "EVENT_TYPE_CONSTANT",
  "timestamp": "ISO 8601",
  "citation_id": "",
  "observation_id": "",
  "details": {}
}
```

### TAG_EMITTED_UNDER_ALIAS

**Trigger:** An alias-side tag was emitted during processing of an observation.

**When emitted:** Once per alias-side tag emitted. If one observation produces three alias-side tags (e.g., all three DMR-family aliases fire), emit three separate `TAG_EMITTED_UNDER_ALIAS` events.

```json
{
  "event_type": "TAG_EMITTED_UNDER_ALIAS",
  "timestamp": "...",
  "citation_id": "...",
  "observation_id": "...",
  "details": {
    "primary_tag": "control_object:device_master_record_control",
    "alias_tag": "control_object:medical_device_file_control",
    "alias_source": "aliases.control_object",
    "alias_target": "medical_device_file_control",
    "vocabulary_era": "qsr",
    "primary_confidence": 0.95,
    "alias_confidence": 0.85
  }
}
```

### DUAL_VOCABULARY_RESOLVED

**Trigger:** Both QSR and QMSR anchors were detected in the same observation, and both resolved to the same semantic tag value.

**When emitted:** Once per observation where `dual_vocabulary_resolved = true`. Not emitted for observations where only one vocabulary era is detected.

```json
{
  "event_type": "DUAL_VOCABULARY_RESOLVED",
  "timestamp": "...",
  "citation_id": "...",
  "observation_id": "...",
  "details": {
    "qsr_anchor_detected": "21 CFR 820.181",
    "qmsr_anchor_detected": "ISO 13485:2016 §4.2.3",
    "resolved_tag": "control_object:medical_device_file_control",
    "qsr_side_tag": "control_object:device_master_record_control",
    "qmsr_side_tag": "control_object:medical_device_file_control",
    "alias_applied": true
  }
}
```

### CITATION_BACKFILL_BATCH_COMPLETE

**Trigger:** A backfill batch has been fully processed and written to `enriched.citation_tags`.

**When emitted:** Once at the end of each batch during a backfill run. Not emitted during real-time tagging of individual citations.

```json
{
  "event_type": "CITATION_BACKFILL_BATCH_COMPLETE",
  "timestamp": "...",
  "citation_id": null,
  "observation_id": null,
  "details": {
    "batch_index": 0,
    "batch_size": 100,
    "citations_processed": 100,
    "tags_written": 412,
    "alias_tags_emitted": 7,
    "dual_vocabulary_resolved_count": 3,
    "upsert_key": "(citation_id, cfr_section, category, value)",
    "pipeline_version": "tagger_v0_5",
    "enriched_table": "enriched.citation_tags",
    "stale_write_guard_skips": 0
  }
}
```

### Pre-existing Events (from v0.2, inherited)

The following events defined by consuming systems (`mon-logs.md`) are recorded in the activity log by the enrichment pipeline, not directly by the tagger output JSON. They are listed here for cross-reference:

- `ROLE_ESCALATION_APPLIED` — emitted by the DB-wrapper helper per transaction (WS-A deliverable).
- `OUTBOX_DRAIN_LAG_DETECTED` — emitted by CT-MON-001 (WS-A deliverable).
- `MATRIX_RULE_APPROVED` — emitted by WS-C matrix authoring workflow.
- `MATRIX_COVERAGE_GAP_DETECTED` — emitted by SPEC-223 when no matrix rule matches (WS-C / WS-D scope).
- `ALIGNMENT_SCORE_COMPUTED` — emitted by SPEC-225 (WS-D scope).
- `CITATION_BACKFILL_GATE_DEFERRED` — emitted by CT-079 when `pipeline_version` check fails (Phase 2.1 v1.3 scope).

---

## Taxonomy Update Policy

### Default Mode

Operate in `propose_only` mode unless explicitly told otherwise.

### Can Be Auto-Added as Registry Entries

- site
- product
- sop_id
- policy_id
- event_id
- instrument_id
- procedure_id
- document_id
- software

### Must Stay Proposal-Only by Default

- scope
- system
- org_unit
- process
- quality_theme
- issue_pattern
- control_object
- record_type
- asset_type
- evidence_type
- risk_domain

### New Category Rule

Do not propose a new top-level category unless repeated tagging failures across multiple documents prove that no existing category can represent the concept. `taxonomy_update_policy.top_level_category_change.allowed: false` in the taxonomy file is the authoritative constraint.

### Alias Proposal Policy

Before proposing a new alias, verify:

1. Both the source key and the target value exist in taxonomy v0.5 `stable_categories` or `enterprise_registry_categories`.
2. The proposed alias is not an identity mapping (identical source and target).
3. The proposed alias does not create a cycle with existing aliases.
4. The disambiguation note accurately characterises why the mapping is approximate rather than exact (following the model of the DMR→MDF alias notes).

All new alias proposals require Quality Domain Lead approval before production use. `approval_status: propose_only` until that approval is granted.

---

## Prompt Template

Use this prompt when executing the skill:

```text
You are an FDA 483 semantic tagging and enterprise matching agent operating under
fda_483_semantic_tagger_SKILL v0.5.

Your job is to:
1. Read the active tag taxonomy from an external file.
2. Extract normalized tags from raw FDA 483 observation text.
3. Handle both QSR (pre-2026-02-02) and QMSR (post-2026-02-02) device vocabulary,
   resolving both to the same semantic value via taxonomy v0.5 aliases.
4. Emit alias-side tags for DMR-family observations per the alias emission rules.
5. Match the 483 to relevant enterprise entities and documents.
6. Propose careful taxonomy updates only when truly needed.

First action:
external_file_read(path="{{taxonomy_path}}")

Verify that the taxonomy version is 0.5 or later. If the version is below 0.5,
stop and report that the taxonomy version is insufficient for this skill version —
the dual-vocabulary aliases and drug/biologic value_citations required by v0.5
are absent in earlier files.

Treat the taxonomy file as the active source of truth.

DUAL VOCABULARY HANDLING:
- For each device observation, detect whether it uses QSR vocabulary (§820.xx sections,
  QSR subpart letters, QSIT, "Device Master Record") or QMSR vocabulary (ISO 13485
  clause references, "Medical Device File", Compliance Program 7382.850).
- When both QSR and QMSR anchors appear in the same observation, set
  dual_vocabulary_resolved = true and emit DUAL_VOCABULARY_RESOLVED event.
- Use taxonomy value_citations dual anchors to ensure both vocabulary eras map to
  the same semantic tag.

DMR→MDF ALIAS EMISSION:
- When you emit control_object:device_master_record_control, also emit
  control_object:medical_device_file_control with alias_emitted = true.
- When you emit process:device_master_record_maintenance, also emit
  process:medical_device_file_maintenance with alias_emitted = true.
- When you emit record_type:device_master_record, also emit
  record_type:medical_device_file_record with alias_emitted = true.
- Do NOT emit alias-side tags for DHF or DHR — those have no alias target in v0.5.
- Emit TAG_EMITTED_UNDER_ALIAS event for each alias-side tag emitted.

DRUG/BIOLOGIC VALUE_CITATIONS:
- Recognise the five backfilled anchors: record_type.batch_record (21 CFR 211.188),
  record_type.change_control_record (21 CFR 211.100(b)), record_type.validation_report_record
  (21 CFR 211.100(a)), evidence_type.validation_protocol_review (21 CFR 211.100(a)),
  evidence_type.audit_trail_review (21 CFR 211.68(b)).
- Emit these tags when the evidence text cites the regulatory reference, even if
  the literal label is absent.

Output pipeline_version = "tagger_v0_5" on every response.

Use tags in the format category:value.
Normalize values to lowercase snake_case.

For every tag, return:
- observed or inferred status
- confidence
- evidence text
- rationale
- alias_emitted (true/false)
- vocabulary_era (qsr / qmsr / dual / null)

When matching candidates, use this priority:
1. Direct identifiers
2. Semantic overlap (alias-resolved matches count at full weight)
3. Context overlap
4. Supportive overlap

Use this scoring model:
score =
  5 * direct_exact_matches
+ 3 * semantic_matches
+ 2 * context_matches
+ 1 * supportive_matches
- 4 * direct_conflicts
- 2 * semantic_conflicts
+ 2 * has_direct_match_boost
+ 1 * semantic_cluster_boost

Observed tag weight = 1.0
Inferred tag weight = 0.6
Alias-side tag weight = 0.6 (inferred) unless alias target is an exact stable-category match

Apply these gates and caps:
- if explicit_site_present and no_matching_site and not_global_scope: cap score at 3
- if explicit_product_present and no_matching_product: cap score at 4
- if only_inferred_matches and no_observed_matches: cap score at 2
- if workflow record_type or process is incompatible: cap score at 3

Alias resolution at match time:
- device_master_record_control ↔ medical_device_file_control (bidirectional)
- device_master_record_maintenance ↔ medical_device_file_maintenance (bidirectional)
- device_master_record ↔ medical_device_file_record (bidirectional, record_type)
- Document alias-resolved matches in candidate_matches[].alias_resolved_matches

Tie-break on:
1. More observed shared tags
2. More direct shared tags
3. Stronger evidence density
4. Current status over superseded
5. Recency if relevant

Before proposing a taxonomy update, check:
1. Can this map to an existing canonical value?
2. Is this only a synonym or alias?
3. Is this just a registry entry?
4. Is the concept reusable across multiple records?
5. Does it materially improve matching or explainability?

Unless explicitly authorized, operate in propose_only mode.

Return JSON using the output contract defined by this skill:
- taxonomy_version_used: "0.5" (minimum)
- pipeline_version: "tagger_v0_5"
- Include vocabulary_era_detected and dual_vocabulary_resolved per observation
- Include alias_emitted and vocabulary_era per tag
- Include alias_resolved_matches per candidate match
- Include activity_events list
```

---

## Implementation Notes

### Sequencing with WS-A

The tagger must not begin corpus backfill writes to `enriched.citation_tags` until WS-A acceptance criterion 4 is satisfied — specifically, manual proof that a drain of one outbox message produces a corresponding `enriched.citation_tags` row within 15 minutes. Attempting backfill writes before WS-A completes will result in `insufficient_privilege` errors from Postgres (the IMP-TAX-012 failure class). The shared DB-wrapper helper from WS-A (`/packages/shared/db-client/`) must be in place before the tagger backfill worker is invoked.

### Pipeline Version Gating

Phase 2.1 v1.3 CT-079 gates SPEC-225 execution on `pipeline_version = 'tagger_v0_5'` rows. Until the backfill completes, SPEC-225 will decline to produce alignment output. This is the intended behaviour — it prevents the "working system producing uninformative outputs" failure mode identified in audit v1.2 §6.1.

### Schema Shape Decision

At audit v1.2 §5.6.6, the `enriched.citation_tags` table was observed to have a single `gmp_subsystem` column rather than a `(category, value)` shape. The v0.5 tagger writes `(category, value)` tuples. If the current live schema still has the `gmp_subsystem` shape, a migration must be applied before the backfill run to add `category` and `value` columns and update the unique constraint to `(citation_id, cfr_section, category, value)`. The upsert key in §Backfill-Strategy is conditional on this migration being applied.

### Mixed-Version Corpus During Backfill

During the backfill run, the corpus will contain rows tagged under prior tagger versions (`gmp_subsystem` shape or `pipeline_version = 'tagger_v0_4'`) alongside newly upserted rows. CT-079 handles this correctly by filtering on `pipeline_version`. Do not attempt to run SPEC-225 against the mixed corpus — wait for backfill completion before enabling SPEC-225.

### Synthetic QMSR Citations (Risk R2 Mitigation)

If the golden-set eval finds fewer than 15 organic post-2026-02-02 device citations in the corpus, supplement the QMSR-worded segment with synthetic citations constructed from real QSR observations reworded in QMSR/ISO 13485 language. Document the supplement count in the golden-set eval report. Accept the eval provisionally, noting the synthetic supplement, and commit to re-evaluation when the organic post-transition corpus reaches 15 citations.

### alias_write_mode_default Constraint

All aliases in taxonomy v0.5 carry `approval_status: propose_only`. The tagger may EMIT alias-side tags in its output (this is within the tagger's scope and is required by WS-B). The tagger may NOT write new alias entries to the taxonomy YAML without Quality Domain Lead approval. Use `external_file_write` for taxonomy updates only when the Quality Domain Lead has explicitly approved a specific new alias.

### CT-MON-001 Alert Window During Backfill

During a declared backfill period, the CT-MON-001 monitor worker's effective alert window should be extended to 60 minutes per Risk R4 mitigation (GXP-PLAN-PH2.2-001 §11.1). This is configured via the `alert_window_override_minutes` feature flag in the monitor worker's `wrangler.toml`. Set this flag before starting the backfill run and revert it after the backfill completes and CT-MON-001 has run clean for 24 hours.

---

*End of Document — fda_483_semantic_tagger_SKILL v0.5*
