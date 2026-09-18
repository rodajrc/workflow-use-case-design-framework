---
ucdd_version: 1.2
use_case_name: "Generic Alert Score by Entity Risk with GTI"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-18
last_update: 2026-09-18
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Fallback Playbook"
---

# Generic Alert Score by Entity Risk with GTI

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Generic Alert Score by Entity Risk with GTI |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-18* |
| **Last update** | *2026-09-18* |
| **Owner** | rodajrc |
| **Status** | **Active**: enabled and chained live as the second block of the Fallback Playbook. |
| **Related flows** | Invoked by chaining from [Fallback Playbook](/examples/fallback/fallback.md). Its scoring entries are consumed by [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md) and [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md). Sibling of [RULE Alert Score by Severity](/examples/subflows/rule--alert-score-by-severity.md) and [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md), which write to the same scoring key. |

## 1. Workflow Statement

> *Generic Alert Score by Entity Risk with GTI* enriches every entity of the alert with Google Threat Intelligence and writes one alert scoring entry per entity, from Informational to High, according to the entity's GTI threat score. When there are no entities, no GTI integration, or the enrichment fails, it leaves a case comment saying which, and optionally writes a single Low scoring entry so that downstream blocks still have a severity to read.

## 2. Workflow Objective

> Turn the reputation of the alert's entities into the platform's alert score for any alert, whatever product raised it, so that prioritization and assessment can run on alerts that carry no severity of their own.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition`, `For Each Loop` | Required | |
| `Tools` | Google SecOps built-in toolkit | `Get Integration Instances`, `Add Alert Scoring Information` | Required | |
| `GoogleThreatIntelligence` | Entity reputation | `Enrich Entities` | Optional, per-environment | Detected at run time; a missing or unconfigured instance takes the no-integration path. |
| `Siemplify` | Google SecOps native action integration | `Case Comment` | Required | Explains why no per-entity scoring happened. |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| CONST_CONTEXT_ALERT_SEVERITY | Input Constant | String | Name of the scoring entries written. Shared with every other scoring block, so their entries compose into one alert score. | CTX_ALERT_SEVERITY |
| param_enable_fallback_severity | Input Parameter | Integer (0,1) | When 1, writes one `Low` scoring entry on the paths where no entity was scored (no entities, no GTI integration, failed enrichment). | 1 |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Entity enrichment**: every entity of the alert is enriched with GTI (score threshold 30, comments retrieved, no sandbox resubmission), which also populates the entity's GTI widget.
- **Per-entity alert scoring**: one `Entity Threat Score` scoring entry per entity, whose description names the entity and its threat score. `Tools - Add Alert Scoring Information` composes the entries into the alert's score and sets `[Alert.ALERT_SEVERITY]` (see the [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md) UCDD, Section 9, for that undocumented effect).
- **Case comment on the empty paths**: `No entities to enrich`, `Missing GTI integration`, or `Enrichment action failed`, followed by the optional `Low` fallback scoring entry.

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:triage

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow before prioritization. No original alert field is read; the block works from the alert's entities and the environment's integration instances.

## 7. Automation Strategy

Check the preconditions in order, entities first and integration second, and give every failed precondition the same exit: a case comment naming the cause, then the shared fallback-severity decision. On the happy path, enrich all entities in one call, then loop over the result and route each entity by its GTI threat score into one of five scoring entries, so that the alert score reflects the worst entity through the platform's own composition rules rather than through logic in the block.

## 8. Workflow and Subflows

**Inputs and Outputs**

One input constant and one declared input (Section 3). No execution output.

**Steps**

1. If the alert has no entities, or the environment has no configured Google Threat Intelligence instance, a case comment says which and the block skips to step 4.
2. `GoogleThreatIntelligence - Enrich Entities` enriches every entity. If it fails, a case comment says so and the block skips to step 4.
3. For each enriched entity, `Tools - Add Alert Scoring Information` writes one entry from its GTI threat score: above 69 `High`, 30 to 69 `Medium`, 2 to 29 `Low`, 1 or below `Informational`. *End*.
4. If `param_enable_fallback_severity` is on, `Tools - Add Alert Scoring Information` writes one `Low` entry so downstream blocks still have a severity. *End*.

> **Warning**
> This is a simplification of the actual automation logic. Refer to the actual block to see exactly how it works.

**Outcomes and Effects**

Scoring entries named by `CONST_CONTEXT_ALERT_SEVERITY` under category `Entity Threat Score`, one per entity or one fallback entry; a case comment on the three empty paths; GTI enrichment data on the entities.

**Error Handling**

`Enrich Entities` is the only action with `autoSkipOnFailure: true`, so its failure reaches the `Successful Enrichment?` check instead of halting the run (`MISSING_INTEGRATION` and `MISCONFIGURED_INTEGRATION` are caught earlier by the instance check; a runtime failure such as a quota error lands here). Everything else halts the run on failure.

## 9. Assumptions

- Every enriched entity carries `gti_assessment.threat_score.value`; an entity without it is routed as Benign, by owner decision (2026-09-18).
- `[Enrich All Entities with GTI.is_success]` renders as the string `true` on success, so the plain string equality in `Successful Enrichment?` holds (owner-verified on a live run, 2026-09-18).
- Scoring entries compose by the platform's ratio (5 Low = 1 Medium, 3 Medium = 1 High, 2 High = 1 Critical, per the action's own description), so many Informational entities do not raise the alert score.

## 10. Improvements

- Built for the Fallback Playbook on 2026-09-17 and fixed the same day: the fallback entry's name placeholder (`_KEY` suffix) corrected, and the three empty paths converged on one fallback-severity decision with a distinguishing case comment each.
- Candidates: the score thresholds (69, 29, 1) as inputs; a `Critical` band; treating a missing `gti_assessment` as unknown rather than benign.

## 11. Workflow Simulation and Testing

Exercised by the owner on live alerts on 2026-09-18 inside the Fallback Playbook, including the enrichment-success check. No recorded debug transcript.

## 12. Resources

- [Fallback Playbook UCDD](/examples/fallback/fallback.md), the workflow this block was written for.
- [DTM Alert Score by Severity UCDD](/examples/subflows/dtm-catchall--alert-score-by-severity.md), Section 9: source excerpt showing what `Add Alert Scoring Information` writes.

### Associated ADS

None.

> **NOTE**
> The version-history table this document used to end with was removed on 2026-09-18. The repository's git history already records every change to this file, so keeping a changelog by hand duplicated that job.
