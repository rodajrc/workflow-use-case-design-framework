---
ucdd_version: 1.1
use_case_name: "Generic Case Lifecycle Management by Severity"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-09
last_update: 2026-09-09
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
---

# Generic Case Lifecycle Management by Severity

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Generic Case Lifecycle Management by Severity |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-09* (split out of the DTM CatchAll UCDD, where it was documented since 2026-08-19) |
| **Last update** | *2026-09-09* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the fifth block of DTM CatchAll. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md). Reads the severity written by a scoring block such as [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md). |

## 1. Workflow Statement

> *Generic Case Lifecycle Management by Severity* takes a computed alert severity and either escalates the case, moving it to the Incident or Investigation stage and assigning it to the matching Incident Response tier, or auto-closes the alert as benign. The tiers are parameters, so the same block serves any SOC structure.

## 2. Workflow Objective

> Put every case in front of the right team at the right stage, or out of the queue, without an analyst touching it first.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |
| `Siemplify` | Google SecOps native action integration | `Change Case Stage`, `Close Alert` | Required | |
| `Tools` | Google SecOps built-in toolkit | `Assign Case to User` | Required | |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| CONST_ALERT_SEVERITY | Input Constant | String | The alert severity to act on. Bound at the call site to `[Alert.ALERT_SEVERITY]`. Normalised to lowercase by the block. | — |
| param_investigation_team | Input Parameter | String (SOC role) | Tier assigned when the case is escalated to the Investigation stage. | @Tier1 |
| param_incident_team | Input Parameter | String (SOC role) | Tier assigned when the case is escalated to the Incident stage. | @Tier2 |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Case assignment and stage change**: the case moves to `Investigation` or `Incident` and is assigned to `param_investigation_team` or `param_incident_team`.
- **Benign closure**: on the fallback path the alert is closed with a fixed reason and root cause and tagged `autoclose`.

**Human-in-the-Loop Actions**

None. The escalated case is the hand-off to a human.

## 5. Workflow Category

subflow:case-management

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow after a scoring block has run. Precondition: `CONST_ALERT_SEVERITY` resolves to a severity string; anything outside `critical`, `high`, `medium`, `low` is treated as benign.

## 7. Automation Strategy

Use a *Flow - IfFlowCondition* (`Should Escalate Alert?`) to branch between the benign-close path and the escalation path. On the benign path, use *Siemplify - Close Alert* with a fixed reason, root cause and `autoclose` tag. On the escalation path, use *Siemplify - Change Case Stage* to move the case to `Investigation` or `Incident` and *Tools - Assign Case to User* to hand it to the tier. Tier assignment is parameterised at the call site (`param_investigation_team`, `param_incident_team`; by default the Google SecOps built-in SOC roles `@Tier1` and `@Tier2`).

## 8. Workflow and Subflows

**Inputs and Outputs**

One input constant and two declared inputs (Section 3). No execution output.

**Steps**

1. `Flow - IfFlowCondition` branches on `[Input.CONST_ALERT_SEVERITY | toLower()]`:
    - `critical` or `high` branch (escalate): `Siemplify - Change Case Stage` sets `Incident`, then `Tools - Assign Case to User` assigns to `param_incident_team`.
    - `medium` or `low` branch (investigate): `Siemplify - Change Case Stage` sets `Investigation`, then `Tools - Assign Case to User` assigns to `param_investigation_team`.
    - `else` branch (unmatched, effectively `informational`): `Siemplify - Close Alert`: Reason `NotMalicious`, Root Cause `Other`, tag `autoclose`, assigned to `@Tier1` for the record.
2. *End*.

**Outcomes and Effects**

Outcome: case stage change (`Investigation` or `Incident`) and assignee, or, on the benign path, the alert's closure status and `autoclose` tag.

**Error Handling**

None explicit. The `else` branch is a closure, not a guard: a severity that fails to match because of `MALFORMED_DATA` closes the alert as benign. Callers must make sure the scoring block never leaves the severity empty.

## 9. Assumptions

- The SOC roles named in the tier parameters exist in the target instance.
- Google SecOps cases inherit the priority of their highest-priority alert, so stage and assignment decided per alert are coherent at case level.

## 10. Improvements

- Split out of the original `4 Any Alert Triage` block on 2026-08-14, when triage assignment and case-stage lifecycle were separated.

## 11. Workflow Simulation and Testing

Exercised in every end-to-end debug run of DTM CatchAll; the first full pass confirming tier assignment was 2026-08-15 (see that UCDD, Section 11).

## 12. Resources

- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None. The block is detection-agnostic.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-09 | rodajrc | **Split** out of the DTM CatchAll UCDD (its version 14, Section 8.5) into a standalone subflow UCDD. Content unchanged; history before this date lives in that document. |
