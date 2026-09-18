---
ucdd_version: 1.1
use_case_name: "DTM Alert Score by Severity"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-09
last_update: 2026-09-09
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
---

# DTM Alert Score by Severity

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | DTM Alert Score by Severity |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-09* (split out of the DTM CatchAll UCDD, where it was documented since 2026-08-19) |
| **Last update** | *2026-09-09* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the third block of DTM CatchAll. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md). Its output is consumed by [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md) and [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md). |

## 1. Workflow Statement

> *DTM Alert Score by Severity* reads the native `severity` of a Google Threat Intelligence DTM alert and turns it into two things every downstream block can read: a case-context tag keyed by a constant, and a weighted scoring entry that also sets the alert's `ALERT_SEVERITY` property. A missing or unrecognised severity scores as Medium.

## 2. Workflow Objective

> Convert the vendor's severity scale into the platform's alert scoring so that prioritization and lifecycle decisions can be made from one platform-native value.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Tools` | Google SecOps built-in toolkit | `Get Original Alert Json`, `Append to Context Value`, `Add Alert Scoring Information` | Required | |
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| CONST_CONTEXT_ALERT_SEVERITY | Input Constant | String | Case-context key under which the severity tag is appended. Downstream blocks reading the tag must use the same key. | CTX_ALERT_DTM_SEVERITY |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Weighted Alert Score**: a `Digital Threat Monitoring`-category scoring entry on the case wall, High, Medium or Low.
- **Severity context tag**: `[Alert.TicketId]:{severity}` appended to the case-scoped context property named by `CONST_CONTEXT_ALERT_SEVERITY`.
- **Alert severity property**: `[Alert.ALERT_SEVERITY]` set to the same High, Medium or Low value (an effect of the scoring action, see Section 9).

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:triage

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow whose alerts come from the `Google Threat Intelligence - DTM` connector. The block reads the Main Alert event's `severity` field from the original alert JSON; a missing field routes to the `else` branch.

## 7. Automation Strategy

Score the alert using the documented [Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity#prioritization-of-alerts): branch on the normalised `severity`, record the tag with *Tools - Append to Context Value* and the weighted score with *Tools - Add Alert Scoring Information*. The `else` branch enforces Medium so an unknown severity is never silently dropped from triage.

## 8. Workflow and Subflows

**Integrations and Actions**

- `Tools - Append to Context Value`: appends the alert severity to a context property under a caller-specified key and scope, readable downstream via `Tools - Get Context Value` on the same key.
- `Tools - Add Alert Scoring Information`: writes a weighted alert scoring entry to the case wall and updates the built-in Google SecOps alert priority.

> **Note**
> See [Add alert scoring information](https://docs.cloud.google.com/chronicle/docs/soar/marketplace/power-ups/tools#add-alert-scoring-information) for the documented behavior. The action also updates the alert-scoped `[Alert.ALERT_SEVERITY]` field with the calculated score, undocumented behavior confirmed only by inspecting the live action source (Section 9).

**Inputs and Outputs**

One input constant, `CONST_CONTEXT_ALERT_SEVERITY` (default `CTX_ALERT_DTM_SEVERITY`). No execution output.

**Steps**

1. `Tools - Get Original Alert Json` pulls the alert's raw JSON.
2. `Flow - IfFlowCondition` branches on the Main Alert event's `severity` field (lowercased, trimmed):
    - `high` branch:
        1. `Tools - Append to Context Value` tags the case context `[Alert.TicketId]:high`
        2. `Tools - Add Alert Scoring Information` records a High-severity scoring entry.
        3. *End branch*
    - `medium` branch:
        1. `Tools - Append to Context Value` tags the case context `[Alert.TicketId]:medium`
        2. `Tools - Add Alert Scoring Information` records a Medium-severity scoring entry.
        3. *End branch*
    - `low` branch:
        1. `Tools - Append to Context Value` tags the case context `[Alert.TicketId]:low`
        2. `Tools - Add Alert Scoring Information` records a Low-severity scoring entry.
        3. *End branch*
    - `else` branch (severity missing, malformed, or any other value):
        1. `Tools - Append to Context Value` tags the case context `[Alert.TicketId]:unknown`
        2. `Tools - Add Alert Scoring Information` records a Medium-severity scoring entry (hard enforced)
        3. *End branch*
3. *End*

**Outcomes and Effects**

1. `Tools - Append to Context Value` writes `[Alert.TicketId]:{severity}` (comma-separated across alerts) to a case-scoped context property keyed by `CONST_CONTEXT_ALERT_SEVERITY`.
2. `Tools - Add Alert Scoring Information` writes a `Digital Threat Monitoring`-category scoring entry to the case wall with severity High, Medium or Low depending on the branch.
3. The same `Add Alert Scoring Information` call also sets the alert-scoped `[Alert.ALERT_SEVERITY]` field to that same value, an undocumented effect confirmed only by inspection. This is what Prioritization and Alert Assessment and Disposition actually consume downstream.

The action's own documented description covers only the first line (`ALERT_SCORE_INFO`, "add an entry to the alert scoring database"). `ALERT_SCORE` and `ALERT_SEVERITY` are written in the same call but never mentioned in that description.

**Error Handling**

None explicit. All actions have `autoSkipOnFailure: false`. The `else` branch is the block's own guard against `MALFORMED_DATA` in the severity field.

## 9. Assumptions

- The block writes to both the case's context scope and the alert's scoring, keyed by `CONST_CONTEXT_ALERT_SEVERITY` (default `CTX_ALERT_DTM_SEVERITY`). Downstream blocks reading that value must reference the same key via this constant.
- Below is an excerpt (non-contiguous lines) from the source code of the `Tools - Add Alert Scoring Information` action. The action reads the `Severity` parameter, computes a composite score across every scoring entry on the alert, then writes three separate alert-context properties in the same call: the scoring database, a numeric score, and `ALERT_SEVERITY`. Only the scoring database is mentioned in the action's own documented description.

```python
ALERT_SCORE_INFO = "ALERT_SCORE_INFO"
ALERT_SCORE = "ALERT_SCORE"
ALERT_SEVERITY = "ALERT_SEVERITY"

# ...

severity = siemplify.extract_action_param("Severity")

# ... composite score computed across every scoring entry on the alert ...

alert_score = compute_score(total_scores)

siemplify.set_alert_context_property(ALERT_SCORE_INFO, current_score_str)
siemplify.set_alert_context_property(ALERT_SCORE, str(alert_score))
siemplify.set_alert_context_property(ALERT_SEVERITY, SEV_LIST[alert_score])
```

## 10. Improvements

- (TODO) Use the original alert JSON fields `confidence` and `has_analysis` alongside `severity`: auto-close alerts below a confidence and severity threshold, and route alerts with `has_analysis=true` straight to escalation, since threat analysts have appended context.

## 11. Workflow Simulation and Testing

Exercised in every end-to-end debug run of DTM CatchAll (see that UCDD, Section 11). The `[Alert.ALERT_SEVERITY]` side effect was confirmed by reading the live action source, not by a documented test.

## 12. Resources

- [GTI DTM Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity)
- [Tools power-up: Add alert scoring information](https://docs.cloud.google.com/chronicle/docs/soar/marketplace/power-ups/tools#add-alert-scoring-information)
- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None. The detection logic lives in each DTM monitor's own query configuration.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-09 | rodajrc | **Split** out of the DTM CatchAll UCDD (its version 14, Section 8.3 and the scoring-action assumption in Section 9) into a standalone subflow UCDD. Content unchanged; history before this date lives in that document. |
