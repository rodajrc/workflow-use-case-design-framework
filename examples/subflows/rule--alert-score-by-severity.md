---
ucdd_version: 1.1
use_case_name: "RULE Alert Score by Severity"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-18
last_update: 2026-09-18
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Fallback Playbook"
---

# RULE Alert Score by Severity

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | RULE Alert Score by Severity |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-18* |
| **Last update** | *2026-09-18* |
| **Owner** | rodajrc |
| **Status** | **Active**: enabled and chained live as the third block of the Fallback Playbook, on the branch for Google SecOps SIEM rule alerts. |
| **Related flows** | Invoked by chaining from [Fallback Playbook](/examples/fallback/fallback.md) when the alert's product is `RULE`. Its scoring entry is consumed by [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md) and [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md). Sibling of [Generic Alert Score by Entity Risk with GTI](/examples/subflows/generic--alert-score-by-entity-risk-with-gti.md) and [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md). |

## 1. Workflow Statement

> *RULE Alert Score by Severity* reads the severity a Google SecOps SIEM detection rule declared for its alert and writes it as an alert scoring entry, so that an alert produced by a YARA-L rule is prioritized from the rule author's own judgement. A rule without a severity gets, optionally, a `Medium` fallback entry.

## 2. Workflow Objective

> Carry the detection rule's severity into the platform's alert score without an analyst re-deciding it.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |
| `Tools` | Google SecOps built-in toolkit | `Add Alert Scoring Information` | Required | |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| CONST_CONTEXT_ALERT_SEVERITY | Input Constant | String | Name of the scoring entry written. Shared with every other scoring block. | CTX_ALERT_SEVERITY |
| param_enable_fallback_severity | Input Parameter | Integer (0,1) | When 1, writes a `Medium` entry for a rule alert whose severity is missing or unrecognised. | 1 |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Alert scoring entry** under category `Detection Rule`, severity Critical, High, Medium, Low or Informational, inherited from the rule; or the `Medium` fallback entry. `Tools - Add Alert Scoring Information` composes it into the alert's score and sets `[Alert.ALERT_SEVERITY]`.

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:triage

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow that has already established the alert is a SIEM rule alert (the Fallback Playbook tests `[Alert.DeviceProduct]` equal to `RULE` before chaining it). Reads the original event field `detection_ruleLabels_severity`, the rule's `severity` label.

## 7. Automation Strategy

One *Flow - IfFlowCondition* routes on the lowercased rule severity label, one branch per level, accepting the spellings rule authors actually use (`moderate` for medium, anything starting with `info` for informational), and each branch writes exactly one *Tools - Add Alert Scoring Information* entry. The default branch, taken when the field is empty or carries an unknown value, goes through the same optional fallback decision as the entity-scoring block, so a rule without a severity is still scored when the caller wants it to be.

## 8. Workflow and Subflows

**Inputs and Outputs**

One input constant and one declared input (Section 3). No execution output.

**Steps**

1. Route on the rule's severity label, `[Event.detection_ruleLabels_severity]` lowercased, and write one `Tools - Add Alert Scoring Information` entry for it: `critical`, `high`, `medium` or `moderate`, `low`, and anything starting with `info` for `Informational`. *End*.
2. If the label is missing or unrecognised and `param_enable_fallback_severity` is on, write one `Medium` entry instead. *End*.

> **Warning**
> This is a simplification of the actual automation logic. Refer to the actual block to see exactly how it works.

**Outcomes and Effects**

One scoring entry named by `CONST_CONTEXT_ALERT_SEVERITY`, category `Detection Rule`, description `Inherited RULE severity` or the fallback text.

**Error Handling**

None explicit; every action has `autoSkipOnFailure: false`. An unrecognised severity value is not an error: it takes the fallback path.

## 9. Assumptions

- `[Event.detection_ruleLabels_severity]` is where the Google SecOps connector exposes the rule's `severity` meta label, in any case. A rule whose label uses another word than the ones matched above takes the fallback path.

## 10. Improvements

- Built for the Fallback Playbook on 2026-09-17 and reworked the same day: the alert-scope `Append to Context Value` steps were dropped (their key referenced an undeclared input), the scoring entries were renamed from the literal `ALERT_SEVERITY` to the shared constant, and the fallback-severity decision was added.

## 11. Workflow Simulation and Testing

Verified by the owner on a live alert from a high-severity rule on 2026-09-18: the `High` branch fired and the scoring entry was written. No recorded run for the other branches yet.

## 12. Resources

- [Fallback Playbook UCDD](/examples/fallback/fallback.md), the workflow this block was written for.

### Associated ADS

None. The block reads whatever severity any rule declares; it is not tied to one detection.

> **NOTE**
> The version-history table this document used to end with was removed on 2026-09-18. The repository's git history already records every change to this file, so keeping a changelog by hand duplicated that job.
