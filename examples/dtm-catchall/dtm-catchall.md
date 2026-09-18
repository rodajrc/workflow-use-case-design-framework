---
ucdd_version: 1.2
use_case_name: "Digital Threat Monitoring Catch All (dtm-catchall)"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-08-01
last_update: 2026-09-18
owner: "rodajrc"
status: "ACTIVE:IN_DEVELOPMENT"
related_flows:
    - "Generic Case Initialization"
    - "DTM Case Initialization"
    - "DTM Track Externally"
    - "DTM Alert Score by Severity"
    - "Generic Alert Prioritization by Alert Severity"
    - "Generic Alert Assessment and Disposition"
    - "Generic Alert Notification"
---

# Digital Threat Monitoring Catch All Playbook

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Digital Threat Monitoring Catch All (DTM CatchAll) |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-08-01* |
| **Last update** | *2026-09-18* |
| **Owner** | rodajrc |
| **Status** | **Active**: Playbook running end-to-end: case initialization, severity-based scoring, prioritization, alert disposition and team assignment. Email and Telegram notification are both supported. |
| **Related flows** | Chained subflows, each with its own UCDD under [`/examples/subflows/`](/examples/subflows/): Generic Case Initialization, DTM Case Initialization, DTM Track Externally, DTM Alert Score by Severity, Generic Alert Prioritization by Alert Severity, Generic Alert Assessment and Disposition, Generic Alert Notification. |

**This is UCDD Version 1.2**

## 1. Workflow Statement

> *DTM CatchAll* triggers on any *Google Threat Intelligence* DTM alert that carries a non-empty `monitor_id` and `monitor_name`. On execution, **`Generic Case Initialization`** runs a generic case setup and opens the `Triage` stage on the case's first alert, followed by **`DTM Case Initialization`** that runs DTM-specific case enrichment, and **`DTM Track Externally`** that marks the source DTM alert as tracked externally in Google Threat Intelligence. Next, **`DTM Alert Score by Severity`** reads the alert's native severity and writes a weighted score into the case's and alert's context. Next, **`Generic Alert Prioritization by Alert Severity`** takes that severity as its own input and sets the case's real, native `Alert.Priority` field. **`Generic Alert Assessment and Disposition`** independently reads the same alert-severity value from context (not the `Alert.Priority` field Prioritization writes, and not through Prioritization's input) to either close a low-severity alert, or, on the case's first alert, move the case to the Investigation stage and assign it to the investigation team, and, for an escalation severity, reassign the case to the escalation team. Finally, **`Generic Alert Notification`** reads the Alert's assigned priority against a configurable gate and, if it clears the gate, notifies the configured contacts by email and, optionally, Telegram.*

## 2. Workflow Objective

> Notify all prioritized alerts from Google Threat Intelligence (GTI) Digital Threat Monitoring (DTM) to the optimal Incident Response Team, by assessing the incident through alert scoring by severity.

## 3. Configuration and Deployment

**Integrations**

The following table lists all integrations and actions using in the automation workflow.

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Google Threat Intelligence` | Alert source for DTM alerts; receives the alert status update | `Update DTM Alert` | Required | |
| `Tools` | Google SecOps built-in toolkit | `Get Original Alert Json`, `Add Context Value`, `Find First Alert`, `Assign Case to User` | Required | |
| `Siemplify` | Google SecOps native action integration | `Case Tag`, `Add General Insight`, `Instruction`, `Add Scoring Context Information`, `Change Alert Priority`, `Close Alert`, `Assign Case`, `Change Case Stage` | Required | |
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition`, used for every branch point in the playbook | Required | |
| `EmailV2` | Notification block's primary channel | `Send Email` | Required | |
| `Telegram` | Notification block's secondary channel | `Send Message` | Optional, per-environment | Requires per-environment Telegram integration instance and custom dynamic parameters to configure the Telegram chat ID. |

**Configurable Parameters**

The following table lists all parameters configurable across the automation workflow and subflows.

| Parameter Name | Data Type | Description | Default Value | Declared by |
|---|---|---|---|---|
| param_is_slug_monitor_name | Integer (0,1) | Enables parsing `monitor_name` under the double-hyphen multitenancy slug convention for extra case tagging. **Monitor Name Format**: `{tenant-slug}--{monitor-type}[--{description}]`. E.g. `acme--custom-monitor`, `acme-cdmx--compromised-credentials--vip` | 0 (disabled) | [DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md) |
| param_alert_priority_to_communicate | String, comma-separated | The set of `Alert.Priority` values that clear the Notification gate. | low,medium,high,critical | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_enable_email | Integer (0,1) | Enables the EmailV2 notification channel. | 0 (disabled) | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_enable_telegram | Integer (0,1) | Enables the Telegram notification channel. | 0 (disabled) | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_telegram_chat_id | String | Target Telegram chat ID. Required only if `param_enable_telegram=1`. | — | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_disposition_severities | String, comma-separated | Severities that Alert Assessment and Disposition closes without assignment. | informational,low | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_escalation_severities | String, comma-separated | Severities that Alert Assessment and Disposition hands to the escalation team. | high,critical | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_investigation_team | String (SOC role) | Team assigned, with the `Investigation` stage, on the case's first non-disposed alert. | @Tier1 | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_escalation_team | String (SOC role) | Team the case is reassigned to for an escalation severity. | @Tier2 | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_branding_name | String | Organization branding name shown in the Notification email's HTML template. | Zevorus | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_playbook_name | String | Playbook name shown in the Notification email's HTML template. | DTM CatchAll | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |


> **IMPORTANT**
> Before alerts arrive, create every SOAR environment they can land in: the environment name, or one of its aliases, must match the environment value the connector assigns at ingestion. Alerts ingested into a non-existent environment are stored, but their cases are hidden from the UI and API, and playbook runs against them fail at their first action step (see the [Generic Case Initialization](/examples/subflows/generic--case-initialization.md) UCDD, Error Handling).

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Similar Cases UI Widget**: case-scoped enrichment about similar cases populated only on the first alert grouped into a case ([Generic Case Initialization](/examples/subflows/generic--case-initialization.md)).
- **Triage stage**: the case moves to the `Triage` stage on its first alert ([Generic Case Initialization](/examples/subflows/generic--case-initialization.md)).
- **Case Tags**: `gti:dtm` and `monitor:<monitor_name>` always included. When parameter `param_is_slug_monitor_name=1`, `monitor_tenant` and `monitor_type` tags are also included ([DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md)).
- **DTM Alert Insight UI Widget**: a case-wall summary of the DTM finding, written by `Add General Insight` ([DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md)).
- **Severity-definition Instruction**: a plain-text analyst note explaining DTM's severity scale, attached at case init, visible in the case wall ([DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md)).
- **DTM alert status sync**: the source alert is set to `Tracked Externally` in Google Threat Intelligence ([DTM Track Externally](/examples/subflows/dtm-catchall--track-externally.md)).
- **Playbook view `DTM Alert`**: the playbook's default view, visible to the `Tier1`, `Tier2` and `Tier3` SOC roles, with one half-width widget: the Google Threat Intelligence integration's own `Update DTM Alert` action widget, rendering that step's JSON result from [DTM Track Externally](/examples/subflows/dtm-catchall--track-externally.md) and hidden when the result is empty.
- **Weighted Alert Score**: written to case and alert context ([DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md)).
- **Alert Priority Update**: native alert priority update by assessing the alert score ([Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md)).
- **Alert disposition, assignment and escalation**: [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) closes alerts whose severity is in `param_disposition_severities`; otherwise, on the case's first alert, moves the case to the Investigation stage and assigns it to `param_investigation_team=@Tier1`, and for severities in `param_escalation_severities` reassigns the case to `param_escalation_team=@Tier2`.
- **Branded notification**: Sent once the alert clears the `param_alert_priority_to_communicate` gating parameter ([Generic Alert Notification](/examples/subflows/generic--alert-notification.md)).

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

**Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring (DTM)* (product) alert. **Mark** the source alert as tracked externally. **Score** the alert using DTM *"Alert Severity Definitions"*. **Prioritize** the alert using the calculated scores. **Assess** the alert against the configured severity lists: dispose of it, or assign the case to the investigation team and, when warranted, escalate it. Finally, **Notify** the SOC team leveraging *Email* and, optionally, *Telegram* integrations.

### Technical Strategy

1. **Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring* (product) alert leveraging the original alert.
    - If the SOAR platform supports it, trigger by *Product Name* equal to `DTM Alert`
    - Alternatively, trigger if the alert contains the original fields `monitor_id` and `monitor_name` (product distinctive fields)
    - To reduce the likelihood of false-positive execution, include a condition to check the *Device Vendor* (`[Alert.DeviceVendor]`) equal to `Google Threat Intelligence`

2. **Mark** the source alert as `Tracked Externally` using *GoogleThreatIntelligence - Update DTM Alert*, so the DTM console reflects SOAR ownership before triage starts.

3. **Score** the alert using the documented [Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity#prioritization-of-alerts).
    - Use *Tools - Add Context Value* integration action to store the alert severity in the case's scope
    - Use *Siemplify - Add Scoring Context Information* integration action to calculate a weighted score of the alert

4. **Prioritize** the alert using *Siemplify - Change Alert Priority* integration action to select the platform's supported prioritization model.
    - To know: *Google SecOps* cases inherit the priority level of the alert with highest priority

5. **Assess** the alert using the alert scoring information, in three decisions that each run only for alerts that survived the previous one.
    - Use a *Flow - IfFlowCondition* (`Dispose Alert?`) against `param_disposition_severities`; on the disposition path, use *Siemplify - Close Alert* with a fixed reason and root cause
    - Use *Tools - Find First Alert* and a *Flow - IfFlowCondition* (`First Alert?`) so that *Siemplify - Change Case Stage* (`Investigation`) and *Tools - Assign Case to User* (`param_investigation_team`) run once per case
    - Use a *Flow - IfFlowCondition* (`Escalate Alert?`) against `param_escalation_severities`; on the escalation path, use *Tools - Assign Case to User* to reassign the case to `param_escalation_team`
    - Severity lists and teams are parameterized at the call site (By default `informational,low` / `high,critical` and `@Tier1` / `@Tier2`; *Google SecOps* built-in *SOC roles*).

6. **Notify** the alert leveraging *EmailV2 - Send Email* integration action, and optionally, *Telegram - Send Message*

### Additional Notes for Development

- Use the original alert JSON fields `severity`, `confidence` and `has_analysis` for alert triage.
    - (TODO) - Configure playbooks to auto-close alerts that have a `confidence` and `severity` below a certain threshold 
    - (TODO) Route alerts with `has_analysis=true` directly to an escalation queue, since it means threat analysts have appended valuable context

## 8. Workflow and Subflows

Every subflow is a reusable block with its own UCDD under [`/examples/subflows/`](/examples/subflows/). This section records how DTM CatchAll chains them and what each link relies on; the blocks themselves (steps, actions, outcomes, error handling) are documented in their own files and not repeated here.

The **DTM CatchAll** playbook has the following high-level structure:

> **(1)** *Case Initialization* -> **(2)** *Alert Score* -> **(3)** *Alert Prioritization* -> **(4)** *Alert Assessment and Disposition* -> **(5)** *Response (e.g. Alert Notification)*.

**Subflow chain**

| # | Subflow | Category | Bound at the call site | Reads | Writes |
|---|---|---|---|---|---|
| 1 | [Generic Case Initialization](/examples/subflows/generic--case-initialization.md) | subflow:enrichment | — | case alerts | Similar Cases widget, case stage `Triage` |
| 2 | [DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md) | subflow:enrichment | `param_is_slug_monitor_name` | original alert JSON | case tags, insight, instruction |
| 3 | [DTM Track Externally](/examples/subflows/dtm-catchall--track-externally.md) | subflow:case-management | — | original alert `id` | DTM alert status in Google Threat Intelligence |
| 4 | [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md) | subflow:triage | `CONST_CONTEXT_ALERT_SEVERITY` = `CTX_ALERT_SEVERITY` | original alert `severity` | case context key, scoring entry, `[Alert.ALERT_SEVERITY]` |
| 5 | [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md) | subflow:triage | `CONST_ALERT_SEVERITY` = `[Alert.ALERT_SEVERITY]` | alert severity | native `Alert.Priority` |
| 6 | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) | subflow:case-management | `CONST_ALERT_SEVERITY` = `[Alert.ALERT_SEVERITY]`, `param_disposition_severities`, `param_escalation_severities`, `param_investigation_team`, `param_escalation_team` | alert severity, first alert of the case | alert closure, or case stage and assignee |
| 7 | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) | subflow:case-management | `param_alert_priority_to_communicate`, `param_enable_email`, `param_enable_telegram`, `param_telegram_chat_id`, `param_branding_name`, `param_playbook_name` | `Alert.Priority`, original alert JSON | email, Telegram message, Important flag on failure |

**Main-workflow actions outside the blocks**

None. `Siemplify - Change Case Stage`, which the main playbook ran itself before the triage blocks from 2026-08-24, moved into Generic Case Initialization on 2026-09-17.

**Contract between the blocks**

- Block 4 is the only writer of the severity every later block consumes. Blocks 5 and 6 read it through `[Alert.ALERT_SEVERITY]`, an effect of `Tools - Add Alert Scoring Information` that is undocumented by the vendor (see the scoring block's Section 9). Block 6 does not read block 5's output; both read block 4.
- Block 7 reads the native `Alert.Priority` that block 5 wrote, not the severity.
- Block 3 writes nothing into the case; its only effect is on the alert source, and no later block depends on it.
- Blocks 1, 5, 6 and 7 are product-agnostic and can be chained by any main workflow (the [Fallback Playbook](/examples/fallback/fallback.md) chains all four); blocks 2, 3 and 4 read DTM alert fields and are specific to this product.

**Error Handling**

None of the blocks halts the workflow explicitly; every action runs with `autoSkipOnFailure: false`, so a failed action stops the run at that step. The two failures seen in production, the non-existent-environment failure at block 1 and the insight template failure at block 2 (fixed), are documented in those blocks' UCDDs.

## 9. Assumptions

- Every DTM alert, regardless of sub-type (e.g. Compromised Credentials, Document, etc.), carries `monitor_id` and `monitor_name`.
- [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md) writes to both the case's context scope and the alert's scoring, keyed by `CONST_CONTEXT_ALERT_SEVERITY`. The block's default is `CTX_ALERT_SEVERITY` since 2026-09-17, shared by every scoring block, and DTM CatchAll binds the same value at the call site since 2026-09-18. Downstream blocks reading that value must reference the same key via this constant. The scoring action's undocumented `ALERT_SEVERITY` side effect, including the source excerpt that proves it, is recorded in that block's Section 9.

## 10. Improvements

- **(BLOCK) [Generic Case Initialization](/examples/subflows/generic--case-initialization.md)**: adds a case-scoped Similar Cases widget, populated only when the current alert is the first grouped into its case.

- **(BLOCK) [DTM Case Initialization](/examples/subflows/dtm-catchall--case-initialization.md)**: case-wall Insight redesigned as an honest-labels two-column table (Monitor Information, Alert Information), replacing fabricated fields with the closest real DTM signals.

- **(BLOCK) [Generic Alert Notification](/examples/subflows/generic--alert-notification.md)** (as DTM Alert Notification): Enhanced the notification with a HTML template for `EmailV2 - Send Email` integration with parameters `param_branding_name` and `param_playbook_name`.

- **(BLOCK) [Generic Alert Notification](/examples/subflows/generic--alert-notification.md)** (as DTM Alert Notification): Normalized the notification in the Telegram use case.

## 11. Workflow Simulation and Testing

Several real debug-mode runs have been executed against the live playbook, including a full end-to-end pass on 2026-08-15 confirming tier assignment (then done by Generic Case Lifecycle Management by Severity) and Notification delivery. Block 6's replacement has not yet had a recorded end-to-end run.

## 12. Resources

- [GTI DTM Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity) — Sections 4

### Associated ADS

No detection-rule ADS is linked here. The alerts this playbook consumes originate from any GTI Digital Threat Monitoring, not a SecOps-native YARA-L rule, so the detection logic lives in each DTM monitor's own Lucene query configuration.

> **NOTE**
> The version-history table this document used to end with was removed on 2026-09-18. The repository's git history already records every change to this file, so keeping a changelog by hand duplicated that job.
