---
ucdd_version: 1.1
use_case_name: "DTM Track Externally"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-17
last_update: 2026-09-18
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
---

# DTM Track Externally

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | DTM Track Externally |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-17* |
| **Last update** | *2026-09-18* |
| **Owner** | rodajrc |
| **Status** | **Active**: enabled and chained live as the third block of DTM CatchAll. Named `DTM Handle Externally` until 2026-09-18. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md), right after [DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md). |

## 1. Workflow Statement

> *DTM Track Externally* reads the original DTM alert and sets that alert's status to `Tracked Externally` in Google Threat Intelligence, so the DTM console reflects that the alert is now handled from the SOAR.

## 2. Workflow Objective

> Keep the alert source in step with the SOAR: once a DTM alert has a case, nobody working in the DTM console should treat it as untouched.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Tools` | Google SecOps built-in toolkit | `Get Original Alert Json` | Required | Source of the DTM alert `id`. |
| `Google Threat Intelligence` | Alert source; receives the status update | `Update DTM Alert` | Required | Runs on the environment's integration instance (`AutomaticEnvironment`). |

**Configurable Parameters**

None. The block declares no inputs; it reads the alert it needs from the case.

## 4. Outcomes and Analysis

**Automated Outcomes**

- **DTM alert status sync**: the source alert's status in Google Threat Intelligence becomes `Tracked Externally`.

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:case-management

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow whose alerts come from the `Google Threat Intelligence - DTM` connector. Preconditions: the original alert JSON carries a Main Alert event with an `id` field (see the [GTI DTM Compromised Credentials Alert Sample](/examples/dtm-catchall/gti-dtm-credential-alert-sample.json)), and the environment has a Google Threat Intelligence integration instance.

## 7. Automation Strategy

Mark the alert early and unconditionally, right after enrichment and before any triage decision, so the DTM side shows SOAR ownership whatever triage concludes later. The block resolves the alert `id` itself instead of taking it as an input, which keeps it callable from any DTM main workflow with no call-site binding.

## 8. Workflow and Subflows

**Integrations and Actions**

- `Tools - Get Original Alert Json`: returns the raw alert JSON, from which the Main Alert event's `id` is read.
- `GoogleThreatIntelligence - Update DTM Alert`: updates the DTM alert identified by `Alert ID`, setting `Status` to `Tracked Externally`.

**Inputs and Outputs**

No declared inputs. The block ends in an `Output` step with no output value.

**Steps**

1. `Tools - Get Original Alert Json` runs.
2. `GoogleThreatIntelligence - Update DTM Alert` runs with `Alert ID` = `[Tools_Get Original Alert Json_1.JsonResult | "Events._rawDataFields" | filter("event_type", "=", "Main Alert") | "id"]` and `Status` = `Tracked Externally`.
3. `Output` returns to the main workflow.

**Outcomes and Effects**

1. The DTM alert's status changes in Google Threat Intelligence. Nothing is written to the case.

**Error Handling**

Both actions run with `autoSkipOnFailure: false` and no retries, so a failed update stops the main workflow at this block. No dedicated error path.

## 9. Assumptions

- `Tracked Externally` is a status the `Update DTM Alert` action accepts for the alert's current state.
- The integration instance selected for the environment has permission to update DTM alerts.
- `Tools - Get Original Alert Json` runs a second time per playbook run (DTM Case Initialization already calls it); the cost is accepted in exchange for a block with no inputs.

## 10. Improvements

- Documented from the live definition on 2026-09-17; no changes since the block was created.

## 11. Workflow Simulation and Testing

No run is recorded in this document yet.

## 12. Resources

- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-17 | rodajrc | **New**: block documented from its live definition (three steps, no inputs). Chained by DTM CatchAll as block 3 from that document's version 16. |
| 1 | 2026-09-18 | rodajrc | **Docs**: the `Update DTM Alert` step's JSON result now feeds the `DTM Alert` playbook view of DTM CatchAll (the integration's action widget, visible to Tier1 to Tier3). No block change. |
| 2 | 2026-09-18 | rodajrc | **Renamed** live and in this repository from `DTM Handle Externally` to `DTM Track Externally`, after the `Tracked Externally` status it sets in Google Threat Intelligence (file `dtm-catchall--track-externally.md`). No step change. |
