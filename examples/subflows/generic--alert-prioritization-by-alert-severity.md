---
ucdd_version: 1.1
use_case_name: "Generic Alert Prioritization by Alert Severity"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-09
last_update: 2026-09-09
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
    - "Fallback Playbook"
---

# Generic Alert Prioritization by Alert Severity

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Generic Alert Prioritization by Alert Severity |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-09* (split out of the DTM CatchAll UCDD, where it was documented since 2026-08-19) |
| **Last update** | *2026-09-09* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the fourth block of DTM CatchAll. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md) and from [Fallback Playbook](/examples/fallback/fallback.md). Reads the severity written by a scoring block such as [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md). |

## 1. Workflow Statement

> *Generic Alert Prioritization by Alert Severity* maps a computed alert severity onto the platform's native `Alert.Priority` field, one branch per severity level, with Medium as the fallback for anything unrecognised. It is product-agnostic: any scoring block that leaves a severity in the input constant can feed it.

## 2. Workflow Objective

> Make the computed severity visible where the platform and the analysts already look: the alert's native priority, which the case inherits.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |
| `Siemplify` | Google SecOps native action integration | `Change Alert Priority` | Required | |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| CONST_ALERT_SEVERITY | Input Constant | String | The alert severity to map. Bound at the call site to the alert-scoped severity written by the scoring block (`[Alert.ALERT_SEVERITY]`). Normalised to lowercase by the block. | — |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Alert Priority Update**: native `Alert.Priority` set to Critical, High, Medium, Low or Informative from the severity.

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:triage

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow after a scoring block has run. Precondition: `CONST_ALERT_SEVERITY` resolves to a severity string; an empty or unknown value takes the fallback branch.

## 7. Automation Strategy

Use *Siemplify - Change Alert Priority* to select the platform's supported prioritization model from the computed severity. To know: Google SecOps cases inherit the priority level of the alert with the highest priority.

## 8. Workflow and Subflows

**Integrations and Actions**

- `Siemplify - Change Alert Priority`: sets the alert's native `Alert.Priority` field.

**Inputs and Outputs**

One input constant, `CONST_ALERT_SEVERITY`. No declared output.

**Steps**

1. `Flow - IfFlowCondition` branches on `[Input.CONST_ALERT_SEVERITY]` (lowercased):
    - `critical` branch: sets Priority Critical
    - `high` branch: sets Priority High
    - `medium` branch: sets Priority Medium
    - `low` branch: sets Priority Low
    - `informational` branch: sets Priority Informative
    - `else` branch: sets Priority Medium (fallback priority)
2. *End*.

**Outcomes and Effects**

Native `Alert.Priority` field is set to the resolved value.

**Error Handling**

None explicit. The `else` branch guards against `MALFORMED_DATA` in the input.

## 9. Assumptions

- The severity vocabulary is `critical`, `high`, `medium`, `low`, `informational`. A scoring block using other words lands on the fallback.

## 10. Improvements

None recorded.

## 11. Workflow Simulation and Testing

Exercised in every end-to-end debug run of DTM CatchAll (see that UCDD, Section 11).

## 12. Resources

- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None. The block is detection-agnostic.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-09 | rodajrc | **Split** out of the DTM CatchAll UCDD (its version 14, Section 8.4) into a standalone subflow UCDD. Content unchanged; history before this date, including the 2026-08-22 rename that resolved a community name collision and the 2026-08-24 move of lowercase normalisation into the block, lives in that document. |
