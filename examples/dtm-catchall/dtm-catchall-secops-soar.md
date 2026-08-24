---
ucdd_version: 1.1
use_case_name: "Digital Threat Monitoring Catch All (dtm-catchall)"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-08-01
last_update: 2026-08-22
owner: "rodajrc"
status: "ACTIVE:IN_DEVELOPMENT"
related_flows:
    - "Generic Case Initialization"
    - "Case Initialization for GTI DTM Alerts"
    - "DTM Alert Score by Severity"
    - "Alert Prioritization by Alert Severity"
    - "Case Lifecycle Management by Severity"
    - "DTM Alert Notification"
---

# Digital Threat Monitoring Catch All Playbook

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Digital Threat Monitoring Catch All (DTM CatchAll) |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-08-01* |
| **Last update** | *2026-08-22* |
| **Owner** | rodajrc |
| **Status** | **Active**: Playbook running end-to-end: case initialization, severity-based scoring, prioritization, tier-based case lifecycle management. Email and Telegram notification are both supported. |
| **Related flows** | Generic Case Initialization, Case Initialization for GTI DTM Alerts, DTM Alert Score by Severity, Alert Prioritization by Alert Severity, Case Lifecycle Management by Severity, DTM Alert Notification. |

**This is UCDD Version 1.1**

## 1. Workflow Statement

> *DTM CatchAll* triggers on any *Google Threat Intelligence* DTM alert that carries a non-empty `monitor_id` and `monitor_name`. On execution, **`Generic Case Initialization`** runs a generic case setup, followed by **`Case Initialization for GTI DTM Alerts`** that runs DTM-specific case enrichment. Next, **`DTM Alert Score by Severity`** reads the alert's native severity and writes a weighted score into the case's and alert's context. Next, **`Alert Prioritization by Alert Severity`** takes that severity as its own input and sets the case's real, native `Alert.Priority` field. **`Case Lifecycle Management by Severity`** independently reads the same alert-severity value from context (not the `Alert.Priority` field Prioritization writes, and not through Prioritization's input) to either auto-close a benign alert or assign the case to the appropriate tier and move it to the Investigation or Incident case stage. Finally, **`DTM Alert Notification`** reads the Alert's assigned priority against a configurable gate and, if it clears the gate, notifies the configured contacts by email and, optionally, Telegram.*

## 2. Workflow Objective

> Notify all prioritized alerts from Google Threat Intelligence (GTI) Digital Threat Monitoring (DTM) to the optimal Incident Response Team, by assessing the incident through alert scoring by severity.

## 3. Configuration and Deployment

**Integrations**

The following table lists all integrations and actions using in the automation workflow.

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Google Threat Intelligence` | Alert Source for DTM Alerts | — | Required? | |
| `Tools` | Google SecOps built-in toolkit | `Get Original Alert Json`, `Add Context Value`, `Find First Alert`, `Assign Case to User` | Required | |
| `Siemplify` | Google SecOps native action integration | `Case Tag`, `Add General Insight`, `Instruction`, `Add Scoring Context Information`, `Change Alert Priority`, `Close Alert`, `Assign Case`, `Change Case Stage` | Required | |
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition`, used for every branch point in the playbook | Required | |
| `EmailV2` | Notification block's primary channel | `Send Email` | Required | |
| `Telegram` | Notification block's secondary channel | `Send Message` | Optional, per-environment | Requires per-environment Telegram integration instance and custom dynamic parameters to configure the Telegram chat ID. |

**Configurable Parameters**

The following table lists all parameters configurable across the automation workflow and subflows.

| Parameter Name | Data Type | Description | Default Value |
|---|---|---|---|
| param_is_slug_monitor_name | Integer (0,1) | Enables parsing `monitor_name` under the double-hyphen multitenancy slug convention for extra case tagging. **Monitor Name Format**: `{tenat-slug}--{monitor-type}[--{description}]`. E.g. `acme--custom-monitor`, `acme-cdmx--compromised-credentials--vip` | 0 (disabled) |
| param_alert_priority_to_communicate | String, comma-separated | The set of `Alert.Priority` values that clear the Notification gate. | low,medium,high,critical |
| param_enable_email | Integer (0,1) | Enables the EmailV2 notification channel. | 0 (disabled) |
| param_enable_telegram | Integer (0,1) | Enables the Telegram notification channel. | 0 (disabled) |
| param_telegram_chat_id | String | Target Telegram chat ID. Required only if `param_enable_telegram=1`. | — |
| param_investigation_team | String (SOC role) | Tier assigned when Case Lifecycle Management by Severity escalates a case to the Investigation stage. | @Tier1 |
| param_incident_team | String (SOC role) | Tier assigned when Case Lifecycle Management by Severity escalates a case to the Incident stage. | @Tier2 |
| param_branding_name | String | Organization branding name shown in the Notification email's HTML template. | Zevorus |
| param_playbook_name | String | Playbook name shown in the Notification email's HTML template. | DTM CatchAll |


> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Similar Cases UI Widget**: case-scoped enrichment about similar cases populated only on the first alert grouped into a case (Section 8.1).
- **Case Tags**: `gti:dtm` and `monitor:<monitor_name>` always included. When parameter `param_is_slug_monitor_name=1`, `monitor_tenant` and `monitor_type` tags are also included (Section 8.2).
- **DTM Alert Insight UI Widget**: a case-wall summary of the DTM finding, written by `Add General Insight` (Section 8.2).
- **Severity-definition Instruction**: a plain-text analyst note explaining DTM's severity scale, attached at case init visible in the case wall (Section 8.2).
- **Weighted Alert Score***: written to case and alert context (Section 8.3), 
- **Alert Priority Update**: native alert priority update by assessing the alert score (Section 8.4).
- **Case assignment and stage change**: Case Lifecycle Management by Severity assigns the case to parameters `param_investigation_team=@Tier1` or `param_incident_team=@Tier2` and moves it to the Investigation or Incident stage, or auto-closes the alert (Section 8.5).
- **Branded notification**: Sent once the alert clears the `param_alert_priority_to_communicate` gating parameter (Section 8.6). Email notification can be configurable.

![Branded Notification: a redacted example of the notification you will receive when a DTM alert is received. The organization name ("Zevorus") and the playbook name ("DTM CatchAll") can be configured without modifying the HTML directly.](/examples/dtm-catchall/static/ss-notification-email-sample.png)

**Human-in-the-Loop Actions**

None

## 5. Workflow Category

main-workflow:catch-all

## 6. Trigger Conditions

Execute the playbook when a *Google Threat Intelligence* alert from *Digital Threat Monitoring* of any sub-type (e.g. Document, Credential Exposure, etc.) is ingested into the SOAR platform.

### Technical Trigger Conditions

`Google Threat Intelligence` integration in Google SecOps is required to get the `Google Threat Intelligence - DTM` *connector* used pull the original alerts from the Threat Intelligence platform.

Some relevant fields from DTM Alerts (non-exhaustive, redacted) include:

```json
{ 
    "alert_type": "Compromised Credentials", // <-- Alert Type
    "title": "Leaked Web Service Credentials from \"redacted.tld\"", // redacted
    "severity": "low", // <-- Original Alert Severity
    "confidence": "0.27283120687581575", // <-- Monitor confidence level indicating the probability that the scraped content or event accurately matches the intent of the alert
    "monitor_name": "redacted-name--compromised-credentials--1", // <-- redacted, slug-ed name
    "has_analysis": "False", // <-- Unknown at the time of writing if this means Mandiant or GCTI analysis
    "monitor_version": "9",
    "event_type": "Main Alert", // <-- GTI DTM Alerts are broken down into events per entity AND the main alert of this alert type
    "startTime": "1234567890123", // redacted
    "endTime": "1234567890321" // redacted
}
```

See the [GTI DTM Compromised Credentials Alert Sample](/examples/dtm-catchall/gti-dtm-credential-alert-sample.json)

### Execution Policy

For Google SecOps, this playbook should have the following settings:

- **Priority**: 2
- **Reason**: PRODUCT_SPECIFIC workflow for general DTM alert handling

## 7. Automation Strategy

### Strategy Summary

**Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring (DTM)* (product) alert. **Score** the alert using DTM *"Alert Severity Definitions"*. **Prioritize** the alert using the calculated scores. **Manage** the case's lifecycle to the correct Incident Response SOC team depending on the final chosen priority. Finally, **Notify** the SOC team leveraging *Email* and, optionally, *Telegram* integrations.

### Technical Strategy

1. **Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring* (product) alert leveraging the original alert.
    - If the SOAR platform supports it, trigger by *Product Name* equal to `DTM Alert`
    - Alternatively, trigger if the alert contains the original fields `monitor_id` and `monitor_name` (product distinctive fields)
    - To reduce the likelihood of false-positive execution, include a condition to check the *Device Vendor* (`[Alert.DeviceVendor]`) equal to `Google Threat Intelligence`

2. **Score** the alert using the documented [Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity#prioritization-of-alerts).
    - Use *Tools - Add Context Value* integration action to store the alert severity in the case's scope
    - Use *Siemplify - Add Scoring Context Information* integration action to calculate a weighted score of the alert

3. **Prioritize** the alert using *Siemplify - Change Alert Priority* integration action to select the platform's supported prioritization model.
    - To know: *Google SecOps* cases inherit the priority level of the alert with highest priority

4. **Manage** the case's lifecycle using the alert scoring information and current case priority.
    - Use a *Flow - IfFlowCondition* (`Should Escalate Alert?`) to branch between the benign-close path and the escalation path
    - On the benign path, use *Siemplify - Close Alert* to auto-close, with a fixed reason, root cause, and `autoclose` tag
    - On the escalation path, use *Tools - Assign Case to User* to assign the case to an Incident Response SOC team, and *Siemplify - Change Case Stage* to modify the case's current stage to `Investigation` or `Incident`
    - Tier assignment is parameterized at the call site via `param_investigation_team`/`param_incident_team` inputs (By default `@Tier1`/`@Tier2`; *Google SecOps* built-in *SOC roles*).

5. **Notify** the alert leveraging *EmailV2 - Send Email* integration action, and optionally, *Telegram - Send Message*

### Additional Notes for Development

- Use the original alert JSON fields `severity`, `confidence` and `has_analysis` for alert triage.
    - (TODO) - Configure playbooks to auto-close alerts that have a `confidence` and `severity` below a certain threshold 
    - (TODO) Route alerts with `has_analysis=true` directly to an escalation queue, since it means threat analysts have appended valuable context

## 8. Workflow and Subflows

Documented block by block against the live working playbook.

The **DTM CatchAll** playbook has the following high-level structure:

> **(1)** *Case Initialization* -> **(2)** *Alert Score* -> **(3)** *Alert Prioritization* -> **(4)** *Case Lifecycle Management* -> **(5)** *Response (e.g. Alert Notification)*.

### 8.1 `Generic Case Initialization`

**Purpose**

Generic, cross-playbook baseline case setup. Determines whether the current alert is the first one grouped into its case and, only on that first alert, runs case-level `Siemplify - Get Similar Cases` enrichment. 

This whole block is an improvement over the stated objective rather than a requirement of it (see Section 10).

**Integrations and Actions**

`Tools`, `Flow`, `Siemplify`.

- `Siemplify - Get Similar Cases`: searches for similar cases matching Rule Generator, Category Outcome, and Entity Identifier, over a 14-day window, across both open and closed cases.

**Inputs and Outputs**

No declared inputs and no execution output. 

**Steps**
1. `Tools - Find First Alert` finds the identifier of the first alert grouped into the current case.
2. `Flow - IfFlowCondition` ("Is First Alert?") compares that result against `[Alert.Identifier]` placeholder.
    - *"Yes"* branch (current alert is the first alert)
        1. `Siemplify - Get Similar Cases` runs.
    - *"Else"* branch (current alert is NOT the first alert) 
        1. *End*
3. *End*

**Outcomes and Effects**

1. `Siemplify - Get Similar Cases` creates a case widget.

**Error Handling**

None explicit. All actions have `autoSkipOnFailure: false`.

### 8.2 `Case Initialization for GTI DTM Alerts`

**Purpose** 

DTM-specific case enrichment. Captures the raw alert JSON, tags the case, writes a case-level insight, and attaches a severity-triage instruction for DTM Alerts according to [DTM Alert Severity Definitions and Examples](https://gtidocs.virustotal.com/docs/dtm-alert-severity). 

Improvement beyond the objective since it is not required for the alert prioritization, case lifecycle management, and notification. This improvement is pertinent for post-automation handling for the analyst.

**Integrations and Actions**

`Tools`, `Siemplify`, `Flow`.

- `Siemplify - Add General Insight`: writes an HTML-rendered case-wall summary that provides extra details about the DTM alert and the monitor it created.

**Inputs and Outputs**

One declared input, `param_is_slug_monitor_name` (int, 0 or 1), gating whether the block attempts to parse `monitor_name` under the double-hyphen multitenancy slug convention `{tenant_slug}--{monitor_type}--{short_description}`. For more information about this, refer to [Multi-tenancy on a single Google SecOps](https://security.googlecloudcommunity.com/google-security-operations-2/multi-tenancy-on-a-single-google-secops-part-1-the-isolation-challenge-7789?tid=7789&fid=2).

No execution output.

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| param_is_slug_monitor_name | Input Parameter | Integer | Whether to enable processing of slug-ed DTM monitor names for extra enrichment. Set to 1 to enable. | 0 (DISABLED) |

**Steps**

1. `Tools - Get Original Alert Json` pulls the alert's raw JSON.
2. `Siemplify - Case Tag` ("Tag GTI DTM Alert") tags the case `gti:dtm,monitor:<monitor_name>`.
3. `Flow - IfFlowCondition` ("Is monitor_name Field Slug-ed?") branches on `[Input.param_is_slug_monitor_name] == 1`.
    - *"Yes"* branch
        1. `Siemplify - Case Tag` ("Tag Multi-Tenant Compatible GTI DTM Alert") splits `monitor_name` on double-dashes (`--`) and tags `monitor_tenant` (index 0) and `monitor_type` (index 1).
    - *"Else"* branch
        1. *No Action*
4. Both branches converge on `Siemplify - Add General Insight`.
5. `Siemplify - Instruction` attaches a plain-text severity-definitions message for the analyst.
6. *End*

**Outcomes and Effects**

1. `Siemplify - Instruction` writes a case-scoped message (currently only shown in the case wall)
2. `Siemplify - Add General Insight` writes a two-column table with the following information: Monitor ID, Monitor Name, Finding Type, Alert Name, Confidence, and Alert Severity, pulling every field from the Main Alert event.

**Error Handling**

None explicit. All actions have `autoSkipOnFailure: false`.

### 8.3 `DTM Alert Score by Severity`

**Purpose**

DTM-specific alert scoring. Reads the raw alert's native `severity` field (from the Main Alert event) and writes both a case-context tag and a weighted scoring entry reflecting it, keyed consistently by the constant context-variable name `CONST_CONTEXT_ALERT_SEVERITY` set to `CTX_ALERT_DTM_SEVERITY`.

**Integrations and Actions**

`Tools`, `Flow`.

- `Tools - Append to Context Value`: appends the alert severity to a context property under a caller-specified key and scope, readable downstream via `Tools - Get Context Value` on the same key.
- `Tools - Add Alert Scoring Information`: writes a weighteed alert scoring entry to the case wall and updates the built-in Google Secops Alert Priority.

> **Note**
> See [Add alert scoring information](https://docs.cloud.google.com/chronicle/docs/soar/marketplace/power-ups/tools#add-alert-scoring-information) for more detail about this integration action. It also updates the alert-scoped `[Alert.ALERT_SEVERITY]` field with the current calculated score, which is undocumented behavior confirmed only by inspecting the live block source code definition. Refer to [Assumptions](#9-assumptions) for more information.

**Inputs and Outputs**

One input constant, `CONST_CONTEXT_ALERT_SEVERITY` (default `CTX_ALERT_DTM_SEVERITY`).
No execution output.

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
2. `Tools - Add Alert Scoring Information` writes a `Digital Threat Monitoring`-category scoring entry to the case wall with severities set to High, Medium, or Low depending on the branch.
3. The same `Add Alert Scoring Information` call also sets the alert-scoped `[Alert.ALERT_SEVERITY]` field to that same High/Medium/Low value, an undocumented effect, confirmed only by inspection. This is what Prioritization (8.4) and Case Lifecycle Management by Severity (8.5) actually consume downstream.

The action's own documented description covers only the first line (`ALERT_SCORE_INFO`, "add an entry to the alert scoring database"). `ALERT_SCORE` and `ALERT_SEVERITY` are written in the same call but never mentioned in that description.

Refer to [Assumptions](#9-assumptions) for information about the constant parameter and the operation of the `Add Alert Scoring Information` action.

**Error Handling**

None explicit. All actions have `autoSkipOnFailure: false`.

### 8.4 `Generic Alert Prioritization by Alert Severity`

**Purpose**

Generic, catch-all-wide subflow that maps the computed severity into the Google SecOps native `Alert.Priority` property via `Siemplify - Change Alert Priority`.

**Integrations and Actions**

`Flow`, `Siemplify`.

- `Siemplify - Change Alert Priority`: sets the case's native `Alert.Priority` field using the weighted score from [8.3](#83-dtm-alert-score-by-severity).

**Inputs and Outputs**

One declared input `param_alert_severity` (string) defaulting to `[Alert.ALERT_SEVERITY]`.
No execution output

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| param_alert_severity | Input Parameter | String | The alert-severity value to prioritize by. | `[Alert.ALERT_SEVERITY]` |

**Steps**

1. `Flow - IfFlowCondition` branches on `[Input.param_alert_severity]`:
    - `critical` branch: sets Priority Critical
    - `high` branch: sets Priority High
    - `medium` branch: sets Priority Medium
    - `low` branch: sets Priority Low
    - `informational` branch: sets Priority Informative
    - `else` branch: sets Priority Medium (fallback priority)
2. *End*.

**Outcomes and Effects**

Outcome: the native `Alert.Priority` field is set to the resolved value.

**Error Handling**

None explicit

### 8.5 `Generic Case Lifecycle Management by Severity`

**Purpose**

Generic, catch-all-wide subflow that assigns the case to an Incident Response tier and moves its stage, or auto-closes a low-severity alert.

**Integrations and Actions**

`Flow`, `Siemplify`, `Tools`.

**Inputs and Outputs**

Two declared inputs. No execution output.

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| param_alert_severity | Input Parameter | String | The alert-severity value to prioritize by. | `[Alert.ALERT_SEVERITY]` |
| param_investigation_team | Input Parameter | String (SOC role) | Tier assigned when the case is escalated to Investigation. | @Tier1 |
| param_incident_team | Input Parameter | String (SOC role) | Tier assigned when the case is escalated to Incident. | @Tier2 |

**Steps**

1. `Flow - IfFlowCondition` branches on `[Alert.ALERT_SEVERITY | toLower()]`:
    - `critical` or `high` branch (escalate): `Siemplify - Change Case Stage` sets `Incident`, then `Tools - Assign Case to User` assigns to `param_incident_team`.
    - `medium` or `low` branch (investigate): `Siemplify - Change Case Stage` sets `Investigation`, then `Tools - Assign Case to User` assigns to `param_investigation_team`.
    - `else` branch (unmatched, effectively `informational`): `Siemplify - Close Alert`: Reason `NotMalicious`, Root Cause `Other`, tag `autoclose`, assigned to `@Tier1` for the record.
2. *End*.

**Outcomes and Effects**

Outcome: case stage change (`Investigation` or `Incident`) and assignee, or, on the benign path, the alert's closure status and `autoclose` tag.

**Dependencies**

None explicit.

**Error Handling**

None explicit.

### 8.6 `DTM Alert Notification`

**Purpose**

DTM-specific notification subflow: gates on the case's priority, then sends a branded HTML email and, optionally, a branded Telegram message.

**Integrations and Actions**

`Flow`, `EmailV2`, `Telegram`, `Siemplify`.

- `EmailV2 - Send Email`: Send a branded notification to the email contact configured in Google SecOps SOAR Environment settings.
- `Telegram - Send Message`: Send a branded notification message to a custom Telegram chat group using Telegram bot. Requires a per-environment integration instance.
- `Siemplify - Mark As Important`: flags the case as Important when any of the response integration actions fails.

**Inputs and Outputs**

Six declared inputs (see Section 3 for the full parameter table). 
No execution output.

**Steps**

1. Playbook validates first whether the alert's priority passes the `param_alert_priority_to_communicate` gate. If the condition fails, *End*
2. Check if notification via email is allowed. If not, continue to step 4.
3. `EmailV2 - Send Email` is executed with a branded HTML template. Error handling occurs immediately in case the action fails.
4. Check if notification via Telegram is allowed. If not, *End*.
5. `Telegram - Send Message` is executed. Error handling occurs immediately in case the action fails.

> **Warning**
> This is a simplification of the actual automation logic. Refer to the actual block to see exactly how it works.

**Outcomes and Effects**
1. `EmailV2 - Send Email` sends a branded HTML email (Zevorus palette) to `[Environment.ContactEmail]`, including a dynamic case link, `[General.HostUrl]cases/[Case.Id]` — no slash between the two, since `[General.HostUrl]` already resolves with a trailing slash.
2. `Telegram - Send Message` sends a branded message including the same elements as the email notification to `[Input.param_telegram_chat_id]`. The 
3. `Siemplify - Mark As Important` fires only on a delivery failure, on either channel — the only place in the whole playbook this action fires, a deliberate choice so a case whose automated notification failed doesn't go unnoticed.

**Error Handling**
When either the `EmailV2 - Send Email` or `Telegram - Send Message` action fails, instructions are sent to the case wall to provide details about the potential cause of the error (e.g. MISCONFIGURED_INTEGRATION, MISSING_INTEGRATION). Cases are marked as important as well.

## 9. Assumptions

- Every DTM alert, regardless of sub-type (e.g. Compromised Credentials, Document, etc.), carries `monitor_id` and `monitor_name`.
- [DTM Alert Score by Severity](#83-dtm-alert-score-by-severity) writes to both the case's context scope and the alert's scoring, keyed by `CONST_CONTEXT_ALERT_SEVERITY` (default `CTX_ALERT_DTM_SEVERITY`). Downstream blocks reading that value must reference the same key via this constant.
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

- **(BLOCK) Generic Case Initialization**: adds a case-scoped Similar Cases widget, populated only when the current alert is the first grouped into its case.

- **(BLOCK) Case Initialization for GTI DTM Alerts**: case-wall Insight redesigned as an honest-labels two-column table (Monitor Information, Alert Information), replacing fabricated fields with the closest real DTM signals.

- **(BLOCK) DTM Alert Notification**: Enhanced the notification with a HTML template for `EmailV2 - Send Email` integration with parameters `param_branding_name` and `param_playbook_name`.

- **(BLOCK) DTM Alert Notification**: Normalized the notification in the Telegram use case.

## 11. Workflow Simulation and Testing

Several real debug-mode runs have been executed against the live playbook, including a full end-to-end pass confirming Case Lifecycle Management by Severity assignment and Notification delivery.

## 12. Resources

- [GTI DTM Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity) — Sections 4

### Associated ADS

No detection-rule ADS is linked here. The alerts this playbook consumes originate from any GTI Digital Threat Monitoring, not a SecOps-native YARA-L rule, so the detection logic lives in each DTM monitor's own Lucene query configuration.

## Version history

Every non-trivial change to this document or to the playbook it describes. This is where continuous improvement after go-live gets recorded as well.

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-08-01 | rodajrc | **Initial Draft**: Objective defined |
| 1 | 2026-08-12 | rodajrc | **Initial Draft**: Categorization, Trigger and Execution Priority defined |
| 2 | 2026-08-14 | rodajrc | UCDD draft finalized. Wired `4 Any Alert Triage` block. |
| 3 | 2026-08-14 | rodajrc | Blocks renamed live to the `Any <stage>` / `GTI-DTM <stage>` convention. |
| 4 | 2026-08-14 | rodajrc | Decided `4 Any Alert Triage` should eventually split into a Case Stage Lifecycle block. |
| 5 | 2026-08-15 | rodajrc | **First working version:** Triage assignment and Notification's block both confirmed live end-to-end on sample case alert. Notification email rebuilt with a branded HTML template and a dynamic case link using `[General.HostUrl]`, which resolves to Google SecOps instance URL. |
| 6 | 2026-08-17 | rodajrc | Objective gained an Outcome/Scope/Dependencies breakdown. |
| 7 | 2026-08-18 | rodajrc | **Migrated to UCDD Framework v1.1**: Adopted the 12-section structure. |
| 8 | 2026-08-19 | rodajrc | **Completed Section 8**: documented 8.3 Alert Scoring, 8.4 Alert Prioritization, 8.5 Triage, and 8.6 Notification. |
| 9 | 2026-08-21 | rodajrc | **Completed UCDD**: fully documented DTM CatchAll playbook as of its current version |
| 10 | 2026-08-21 | rodajrc | **Fix**: Made alert notification DTM-specific. Fixed Telegram message template |
| 11 | 2026-08-22 | rodajrc | **Fix**: Renamed three subflows live and in the Content Hub package: `Alert Prioritization` -> `Alert Prioritization by Alert Severity`, fixing a confirmed name collision with an unrelated existing community contribution; `Case Initialization` -> `Generic Case Initialization` and `Case Lifecycle Management` -> `Case Lifecycle Management by Severity`, proactive disambiguation, no collision found for either. |
| 12 | 2026-08-24 | rodajrc | **improvment**: Add `Siemplify - Change Case Stage` action to main playbook to mark the beginning of the triaging step. Change 8.4 block name and moved normalization to lowercase from the input parameter. Converted `param_alert_severity` to constant. Change 8.5 block name and moved dependency to input constant instead. |
