---
ucdd_version: 1.1
use_case_name: "DTM Case Initialization"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-09
last_update: 2026-09-17
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
---

# DTM Case Initialization

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | DTM Case Initialization |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-09* (split out of the DTM CatchAll UCDD, where it was documented since 2026-08-14) |
| **Last update** | *2026-09-09* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the second block of DTM CatchAll. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md). |

## 1. Workflow Statement

> *DTM Case Initialization* enriches a case created from a Google Threat Intelligence DTM alert. It captures the raw alert JSON, tags the case with the alert's monitor, optionally derives tenant and monitor-type tags from a slug-formatted monitor name, writes a case-wall insight summarizing the finding, and attaches the DTM severity definitions as an analyst instruction.

## 2. Workflow Objective

> Give the analyst, on the case wall, everything DTM knows about the alert and its monitor before any triage happens.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Tools` | Google SecOps built-in toolkit | `Get Original Alert Json` | Required | |
| `Siemplify` | Google SecOps native action integration | `Case Tag`, `Add General Insight`, `Instruction` | Required | |
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition` | Required | |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| param_is_slug_monitor_name | Input Parameter | Integer (0,1) | Enables parsing `monitor_name` under the double-hyphen multitenancy slug convention for extra case tagging. **Monitor Name Format**: `{tenant-slug}--{monitor-type}[--{description}]`. E.g. `acme--custom-monitor`, `acme-cdmx--compromised-credentials--vip`. Set to 1 to enable. | 0 (disabled) |

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Case Tags**: `gti:dtm` and `monitor:<monitor_name>` always. When `param_is_slug_monitor_name=1`, `monitor_tenant` and `monitor_type` tags as well.
- **DTM Alert Insight UI Widget**: a case-wall summary of the DTM finding written by `Add General Insight`.
- **Severity-definition Instruction**: a plain-text analyst note explaining DTM's severity scale, visible in the case wall.

**Human-in-the-Loop Actions**

None.

## 5. Workflow Category

subflow:enrichment

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow whose alerts come from the `Google Threat Intelligence - DTM` connector. The block reads the Main Alert event fields `monitor_id`, `monitor_name`, `alert_type`, `id`, `confidence` and `severity` from the original alert JSON; see the [GTI DTM Compromised Credentials Alert Sample](/examples/dtm-catchall/gti-dtm-credential-alert-sample.json).

## 7. Automation Strategy

Pull the original alert once, tag from its monitor fields, branch on the slug parameter for the multitenancy tags, then write the insight and the instruction unconditionally. Everything here is analyst-facing context; nothing downstream depends on it.

## 8. Workflow and Subflows

**Inputs and Outputs**

One declared input, `param_is_slug_monitor_name` (Section 3), gating whether the block parses `monitor_name` under the double-hyphen multitenancy slug convention `{tenant_slug}--{monitor_type}--{short_description}`. For the convention itself, refer to [Multi-tenancy on a single Google SecOps](https://security.googlecloudcommunity.com/google-security-operations-2/multi-tenancy-on-a-single-google-secops-part-1-the-isolation-challenge-7789?tid=7789&fid=2).

No execution output.

**Steps**

1. `Tools - Get Original Alert Json` pulls the alert's raw JSON.
2. `Siemplify - Case Tag` ("Tag GTI DTM Alert") tags the case `gti:dtm,monitor:<monitor_name>`.
3. `Flow - IfFlowCondition` ("Is monitor_name Field Slug-ed?") branches on `[Input.param_is_slug_monitor_name] == 1`.
    - *"Yes"* branch
        1. `Siemplify - Case Tag` ("Tag Multi-Tenant Compatible GTI DTM Alert") splits `monitor_name` on double-dashes (`--`) and tags `monitor_tenant` (index 0) and `monitor_type` (index 1).
    - *"Else"* branch
        1. *No Action*
4. Both branches converge on `Siemplify - Add General Insight`.
5. `Siemplify - Instruction` attaches a plain-text severity-definitions message for the analyst, following [DTM Alert Severity Definitions and Examples](https://gtidocs.virustotal.com/docs/dtm-alert-severity).
6. *End*

**Outcomes and Effects**

1. `Siemplify - Instruction` writes a case-scoped message (currently only shown in the case wall).
2. `Siemplify - Add General Insight` writes a two-column table with Monitor ID, Monitor Name, Finding Type, Alert ID, Confidence and Alert Severity, pulling every field from the Main Alert event. The HTML template is at [template-html-add-insight.html](/examples/dtm-catchall/static/template-html-add-insight.html).

**Error Handling**

None explicit. All actions have `autoSkipOnFailure: false`.

Known failure, fixed 2026-08-29: the insight template applied `substring("0", "4")` to the Confidence cell and errored ("Invalid substring indices") whenever `confidence` rendered shorter than 4 characters (e.g. `0.5`). The `substring()` call was removed; the cell now renders the raw value. Failure class: `MALFORMED_DATA` in the template's own assumptions about field length.

## 9. Assumptions

- Every DTM alert, regardless of sub-type (Compromised Credentials, Document, etc.), carries `monitor_id` and `monitor_name`.
- When the slug convention is enabled, `monitor_name` has at least two double-hyphen-separated tokens; a name off-convention tags whatever index 0 and index 1 resolve to.

## 10. Improvements

- Case-wall Insight redesigned as an honest-labels two-column table (Monitor Information, Alert Information), replacing fabricated fields with the closest real DTM signals.
- The block is an improvement beyond the objective of DTM CatchAll: prioritization, lifecycle management and notification do not depend on it. It matters for post-automation handling by the analyst.

## 11. Workflow Simulation and Testing

Exercised in every end-to-end debug run of DTM CatchAll (see that UCDD, Section 11). The substring failure was diagnosed from production case walls (case 7468) and its fix verified against the following day's run statistics.

## 12. Resources

- [GTI DTM Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity)
- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.

### Associated ADS

None. The detection logic lives in each DTM monitor's own query configuration.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-09 | rodajrc | **Split** out of the DTM CatchAll UCDD (its version 14, Section 8.2) into a standalone subflow UCDD. Content unchanged; history before this date lives in that document. |
| 1 | 2026-09-17 | rodajrc | **Change**: `Add General Insight` table shows the Alert ID (`id`) instead of the Alert Name (`title`) in the first row of the Alert Information column; the alert title already names the alert object in the case, so the cell repeated it. Template updated and verified against the live block; it also still carried the `substring()` call removed in the parent's version 13, now dropped. |
