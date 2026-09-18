---
ucdd_version: 1.1
use_case_name: "Generic Alert Assessment and Disposition"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-17
last_update: 2026-09-18
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
    - "Fallback Playbook"
---

# Generic Alert Assessment and Disposition

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Generic Alert Assessment and Disposition |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-17* |
| **Last update** | *2026-09-18* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the sixth block of DTM CatchAll, where it replaced `Generic Case Lifecycle Management by Severity` on 2026-09-17. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md) and from [Fallback Playbook](/examples/fallback/fallback.md). Reads the severity written by a scoring block such as [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md). Successor of `Generic Case Lifecycle Management by Severity`, whose UCDD was removed with the block (see the repository history before 2026-09-17). |

## 1. Workflow Statement

> *Generic Alert Assessment and Disposition* takes a computed alert severity and decides what happens to the alert: severities in a disposition list close the alert outright; every other severity puts the case in front of the investigation team once, on the case's first alert; severities in an escalation list additionally hand the case to the escalation team. Both lists and both teams are parameters, so the same block serves any SOC structure.

## 2. Workflow Objective

> Dispose of alerts that do not deserve an analyst, and put every other case in front of the right team at the right stage, without an analyst touching it first.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |
| `Siemplify` | Google SecOps native action integration | `Close Alert`, `Change Case Stage` | Required | |
| `Tools` | Google SecOps built-in toolkit | `Find First Alert`, `Assign Case to User` | Required | |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| CONST_ALERT_SEVERITY | Input Constant | String | The alert severity to act on. Bound at the call site to `[Alert.ALERT_SEVERITY]`. Normalised to lowercase by the block. | — |
| param_disposition_severities | Input Parameter | String, comma-separated | Severities that close the alert without assignment. Compared lowercase. | informational,low |
| param_escalation_severities | Input Parameter | String, comma-separated | Severities that hand the case to `param_escalation_team`. Compared lowercase. | high,critical |
| param_investigation_team | Input Parameter | String (SOC role) | Team assigned, and stage `Investigation` set, on the case's first alert when the alert is not disposed. | @Tier1 |
| param_escalation_team | Input Parameter | String (SOC role) | Team the case is reassigned to when the severity is in `param_escalation_severities`. | @Tier2 |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Alert disposition**: an alert whose severity is in the disposition list is closed with a fixed reason and root cause.
- **First-alert assignment and stage**: on the case's first alert, the case moves to `Investigation` and is assigned to `param_investigation_team`.
- **Escalation**: an alert whose severity is in the escalation list reassigns the case to `param_escalation_team`, on every such alert.

**Human-in-the-Loop Actions**

None. The assigned case is the hand-off to a human.

## 5. Workflow Category

subflow:case-management

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow after a scoring block has run. Precondition: `CONST_ALERT_SEVERITY` resolves to a severity string. A severity in neither list (for example `medium`) is neither disposed nor escalated: the case gets the first-alert assignment and nothing else.

## 7. Automation Strategy

Decide disposition first, then assignment, then escalation, so that each later decision only runs for alerts that survived the earlier one. Use a *Flow - IfFlowCondition* (`Dispose Alert?`) that checks whether the disposition list contains the severity; on the disposition path, use *Siemplify - Close Alert* with a fixed reason and root cause. On the surviving path, use *Tools - Find First Alert* and a second *IfFlowCondition* (`First Alert?`) so that *Siemplify - Change Case Stage* (`Investigation`) and *Tools - Assign Case to User* (`param_investigation_team`) run once per case, not once per alert. Then use a third *IfFlowCondition* (`Escalate Alert?`) that checks whether the escalation list contains the severity and, when it does, *Tools - Assign Case to User* reassigns the case to `param_escalation_team`. Lists and teams are parameterised at the call site; by default the teams are the Google SecOps built-in SOC roles `@Tier1` and `@Tier2`.

## 8. Workflow and Subflows

**Inputs and Outputs**

One input constant and four declared inputs (Section 3). No execution output.

**Steps**

1. If the severity is in `param_disposition_severities`, `Siemplify - Close Alert` closes the alert as inconclusive. *End*.
2. If the current alert is the first alert of its case, `Siemplify - Change Case Stage` sets `Investigation` and `Tools - Assign Case to User` assigns the case to `param_investigation_team`.
3. If the severity is in `param_escalation_severities`, `Tools - Assign Case to User` reassigns the case to `param_escalation_team`.
4. *End*.

> **Warning**
> This is a simplification of the actual automation logic. Refer to the actual block to see exactly how it works.

**Outcomes and Effects**

Outcome: the alert's closure status (disposition path), or the case stage `Investigation` and assignee `param_investigation_team` (first alert only), or the assignee `param_escalation_team` (escalation severities, every alert). On a case's first alert with an escalation severity, both assignments run in sequence: the case is assigned to the investigation team and immediately reassigned to the escalation team, which is what the case's activity log will show.

**Error Handling**

The two `Tools - Assign Case to User` steps run with `autoSkipOnFailure: true`, so a failed assignment does not stop the run. Every other action has `autoSkipOnFailure: false`. The two list checks are substring checks with `setIfEmpty("")` on the list side; an empty severity is not guarded against by the block, so the scoring block must never leave it empty (its own `else` branch does that).

## 9. Assumptions

- The SOC roles named in the team parameters exist in the target instance.
- `Tools - Find First Alert` returns the identifier of the first alert grouped into the case, comparable as a plain string with `[Alert.Identifier]`; the same gate the [Generic Case Initialization](/examples/subflows/generic--case-initialization.md) block uses.
- Severity values are whole tokens that do not appear inside one another. The list checks are substring checks, so a list entry `informational` would also match a severity `info`.

## 10. Improvements

- Replaced `Generic Case Lifecycle Management by Severity` on 2026-09-17. Compared with it: the severity thresholds moved from hardcoded branch literals into two list parameters; the stage and the investigation assignment now happen once per case instead of on every alert; the `Incident` stage is no longer set, escalation is an assignment only; the closure reason changed from `NotMalicious` with tag `autoclose` to `Inconclusive` with no tag.
- The `Assign To User` on the disposition path is the literal `@Tier1` rather than `param_investigation_team`.

## 11. Workflow Simulation and Testing

Not yet exercised end-to-end in a recorded debug run since the replacement. Owner checks on the live instance (2026-09-17 and 2026-09-18) covered the condition operators, the first-alert gate, and the behaviour when `CONST_ALERT_SEVERITY` is unset: both list checks take their `else` branch, so an unscored alert is neither disposed nor escalated.

## 12. Resources

- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None. The block is detection-agnostic.

> **NOTE**
> The version-history table this document used to end with was removed on 2026-09-18. The repository's git history already records every change to this file, so keeping a changelog by hand duplicated that job.
