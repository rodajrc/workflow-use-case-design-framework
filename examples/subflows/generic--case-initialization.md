---
ucdd_version: 1.1
use_case_name: "Generic Case Initialization"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-09
last_update: 2026-09-09
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
---

# Generic Case Initialization

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Generic Case Initialization |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-09* (split out of the DTM CatchAll UCDD, where it was documented since 2026-08-14) |
| **Last update** | *2026-09-09* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the first block of DTM CatchAll. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md). |

## 1. Workflow Statement

> *Generic Case Initialization* runs as the first block of any main workflow. It determines whether the current alert is the first one grouped into its case and, only on that first alert, enriches the case with `Siemplify - Get Similar Cases`. Every later alert grouped into the same case passes through without action.

## 2. Workflow Objective

> Give the analyst a Similar Cases widget on every new case, computed once, without repeating the lookup for every alert the case later absorbs.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Tools` | Google SecOps built-in toolkit | `Find First Alert` | Required | |
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |
| `Siemplify` | Google SecOps native action integration | `Get Similar Cases` | Required | |

**Configurable Parameters**

None. The block declares no inputs.

> **IMPORTANT**
> The case must belong to a SOAR environment that exists. Alerts ingested into a non-existent environment are stored, but `Tools - Find First Alert` fails on their cases (see Section 8, Error Handling).

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Similar Cases UI Widget**: case-scoped enrichment listing cases that share Rule Generator, Category Outcome and Entity Identifier over a 14-day window, open and closed. Populated only on the first alert grouped into the case.

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:enrichment

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow as its first block, on any alert. The block is product-agnostic and expects nothing from the alert beyond the case it belongs to.

## 7. Automation Strategy

Find the first alert of the case and compare it with the current alert. Enrich only when they are the same alert, so the case-level widget is built exactly once per case.

## 8. Workflow and Subflows

**Inputs and Outputs**

No declared inputs and no execution output.

**Steps**

1. `Tools - Find First Alert` finds the identifier of the first alert grouped into the current case.
2. `Flow - IfFlowCondition` ("Is First Alert?") compares that result against the `[Alert.Identifier]` placeholder.
    - *"Yes"* branch (current alert is the first alert)
        1. `Siemplify - Get Similar Cases` runs.
    - *"Else"* branch (current alert is NOT the first alert)
        1. *End*
3. *End*

**Outcomes and Effects**

1. `Siemplify - Get Similar Cases` creates a case widget.

**Error Handling**

None explicit. All actions have `autoSkipOnFailure: false`.

When the case belongs to a SOAR environment that does not exist, `Tools - Find First Alert` fails immediately with the error message `Api Key for filter (x.Name == apiKeyName) wasn't found`. The message misleads: the cause is the non-existent environment, which prevents the playbook action from launching. Failure class: `SYSTEM_ERROR` on the platform side, prevented by deployment discipline rather than handled in the block.

## 9. Assumptions

- `[Alert.Identifier]` and the identifier returned by `Tools - Find First Alert` are comparable as plain strings.

## 10. Improvements

- The whole block is an improvement over the objective of the workflows that use it, not a requirement of them: it adds the Similar Cases widget, populated only when the current alert is the first grouped into its case.

## 11. Workflow Simulation and Testing

Exercised in every end-to-end debug run of DTM CatchAll (see that UCDD, Section 11). The non-existent-environment failure was reproduced from production case walls on 2026-08-30.

## 12. Resources

- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None. The block is detection-agnostic.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-09 | rodajrc | **Split** out of the DTM CatchAll UCDD (its version 14, Section 8.1) into a standalone subflow UCDD. Content unchanged; history before this date lives in that document. |
