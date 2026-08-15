---
ucdd_version: 1
use_case_name: "Digital Threat Monitoring Catch All (dtm-catchall)"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-08-01
last_update: 2026-08-15
owner: "rodajrc"
status: "ACTIVE:IN_DEVELOPMENT"
related_flows:
    - "Any Case Initialization"
    - "GTI-DTM Alert Case Initialization"
    - "GTI-DTM Alert Score by Severity"
    - "Any Alert Prioritization"
    - "Any Alert Triage"
    - "Any Alert Notification"
---

# Digital Threat Monitoring Catch All Playbook

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Digital Threat Monitoring Catch All (DTM CatchAll) |
| **SOAR platform** | Google SecOps SOAR (Formerly, Siemplify) |
| **Creation date** | *2026-08-01* |
| **Last update** | *2026-08-15* |
| **Owner** | rodajrc |
| **Status** | **Active, first working version**: Playbook confirmed end-to-end: Case initialization, severity-based scoring, prioritization, tier-based triage, and email notification with a branded HTML template and a dynamic case link. Telegram notification is supported but currently disabled by config. Entity-based scoring (beyond native DTM severity) is a planned improvement, not required by the stated objective. |
| **Related flows** | Any Case Initialization, GTI-DTM Alert Case Initialization, GTI-DTM Alert Score by Severity, Any Alert Prioritization, Any Alert Triage, Any Alert Notification. |

**This is UCDD Version 1**

## Version history

Every non-trivial change to this document or to the playbook it describes. This is where continuous improvement after go-live gets recorded as well.

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0.0 | 2026-08-01 | rodajrc | **Initial Draft**: Objective defined |
| 0.1 | 2026-08-12 | rodajrc | **Initial Draft**: Categorization, Trigger and Execution Priority defined |
| 0.2 | 2026-08-14 | rodajrc | UCDD draft finalized. Wired `4 Any Alert Triage` block. |
| 0.3 | 2026-08-14 | rodajrc | Blocks renamed live to the `Any <stage>` / `GTI-DTM <stage>` convention. Section 5 rewritten around my standing five-stage playbook framework (Case Init -> Scoring -> Prioritization -> Triage -> Specific Response). |
| 0.4 | 2026-08-14 | rodajrc | Decided `4 Any Alert Triage` should eventually split into a Case Stage Lifecycle block and a Team/Queue Routing block instead of collapsing both through one branch (5.5.1, not built yet). Wrote down the variable sets I need to weigh for each future block. |
| 0.5 | 2026-08-15 | rodajrc | **First working version**. Triage assignment and Notification's block both confirmed live end-to-end on sample case alert. Notification email rebuilt with a branded HTML template and a dynamic case link using `[General.HostUrl]`, which resolves to Google SecOps instance URL. |
| 0.6 | 2026-08-15 | rodajrc | Began Section 5 block-by-block walkthrough against the live playbook: 5.1 `1 Any Case Initialization` and 5.2 `1 GTI-DTM Alert Case Initialization` documented, including the live `param_is_slug_monitor_name = 0` config-vs-intent gap. |

## 1. Objective

> Notify the suitable incident response team tier through alert assessment and prioritization of events originated by Google Threat Intelligence Digital Threat Monitoring (DTM).

**Main Outcome**
- RESPONSE: Notification Action

**Dependencies**
- Alert assessment depends on scoring the alert's severity leveraging either (or both) the DTM alert assigned severity (DONE) or advanced IoC entity enrichment (PENDING) workflow.
- Case triage through continuous alert assessment to the suitable Incident Response SOC Team (e.g. Tier 1, Tier 2, and Tier 3).

**Considerations**
- Tenants may have differing IR SOC tiers; important candidate for parameterization
- Advanced IoC entity enrichment is more complex than just mapping the SOAR case priority to the highest DTM alert assigned severity
- Confirmed (high-confidence) benign (low severity) alerts can be auto-closed

## 2. Categorization

main-workflow:catch-all

## 3. Trigger and Execution Priority

**Trigger Conditions**
Execute the playbook when a *Google Threat Intelligence* alert from *Digital Threat Monitoring* of any sub-type (e.g. Document, Credential Exposure, etc.) is ingested into the SOAR platform.

Some relevant original fields from product (non-exhaustive, redacted):
```json
{ 
    "alert_type": "Compromised Credentials",
    "title": "Leaked Web Service Credentials from \"redacted.tld\"", // redacted
    "severity": "low",
    "confidence": "0.27283120687581575",
    "monitor_name": "redacted-name--compromised-credentials--1", // redacted
    "has_analysis": "False",
    "monitor_version": "9",
    "event_type": "Main Alert",
    "startTime": "1234567890123", // redacted
    "endTime": "1234567890321" // redacted
}
```

[GTI DTM Compromised Credentials Alert Sample](/examples/dtm-catchall/ignore-gti-dtm-credential-alert-sample.json) (currently git-ignored until sensitive content is removed)

**Execution Policy**
- **Priority**: 2
- **Reason**: PRODUCT_SPECIFIC workflow for general DTM alert handling: generic alert and case management, DTM alert scoring and prioritization, generic triage, and alert notification

## 4. Automation Strategy

### Strategy Summary

**Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring (DTM)* (product) alert. **Score** the alert using DTM *"Alert Severity Definitions"*. **Prioritize** the alert using the calculated scores. **Triage** the case to the correct Incident Response SOC team depending on the final chosen priority. Finally, **Notify** the SOC team leveraging *Email* and, optionally, *Telegram* integrations.

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

4. **Triage** the alert using the alert scoring information and current case priority.
    - Use a *Flow - IfFlowCondition* (`Should Escalate Alert?`) to branch between the benign-close path and the escalation path
    - On the benign path, use *Siemplify - Close Alert* to auto-close, with a configurable `autoclose_message` and `autoclose_tag`
    - On the escalation path, use *Siemplify - Assign Case* (or *Tools - Assign Case to User*) to assign the case to an Incident Response SOC team, and *Siemplify - Change Case Stage* to modify the case's current stage to `Investigation` or `Incident`
    - Tier assignment is parameterized at the call site via `param_investigation_team`/`param_incident_team` inputs (By default `@Tier1`/`@Tier2`; *Google SecOps* built-in *SOC roles*).

5. **Notify** the alert leveraging *EmailV2 - Send Email* integration action, and optionally, *Telegram - Send Message*

## 5. Modular Technical Implementation

Documented block by block against the live working playbook. Blocks not yet walked through are marked pending below.

The **DTM CatchAll** playbook has the following linear history:

> **(1)** *Case Initialization* -> **(2)** *Alert Scoring* -> **(3)** *Alert Prioritization* -> **(4)** *Triage* -> **(5)** *Alert Notification*.

Blocks corresponding to stages **1** and **2** have product-specific subflows. The remaining stages use *catch all* subflows.

Depending on the role and impact each block puts on the use case, precision here is deliberately uneven by design. For instance, the block named `1 Any Case Initialization` is a generic, reusable improvement rather than a requirement of this use case's stated objective, so it gets a lighter pass here. 

The remaining blocks are either DTM-specific or, in the case of `3 Any Alert Prioritization` and `4 Any Alert Triage`, pre-existing generic building blocks this playbook depends on directly, so both of those get full treatment because they are relevant. 

### 5.1 `1 Any Case Initialization`

**Purpose**
Generic, cross-playbook baseline case setup, not specific to DTM. Determines whether the current alert is the first one grouped into its case and, only on that first alert, runs case-level "`Siemplify - Get Similar Cases`" enrichment. This whole block is an improvement over the stated objective rather than a requirement of it (see Section 7).

**Steps.**
1. `Tools - Find First Alert` finds the identifier of the first alert grouped into the current case.
2. `Flow - IfFlowCondition` ("Is First Alert?") compares that result against `[Alert.Identifier]`.
    - *"Yes"* branch (current alert is the first alert)
        1. `Siemplify - Get Similar Cases` searches for similar cases matching Rule Generator, Category Outcome, and Entity Identifier, over a 14-day window, across both open and closed cases.
    - *"Else"* branch (current alert is NOT the first alert) 
        1. *No action*
3. *End*

**Input/Output**
No declared inputs and no execution output. 

**Effect**
The block's only effect is a case-scoped side effect, the Similar Cases widget, populated only when this is the first alert in the case.

**Error handling**
None explicit. All actions have `autoSkipOnFailure: false`, so a failure in either action halts this block, and with it the whole playbook.

### 5.2 `1 GTI-DTM Alert Case Initialization`

**Purpose** 
DTM-specific case enrichment. Captures the raw alert JSON, tags the case, writes a case-level insight, and attaches a severity-triage instruction for DTM Alerts according to [DTM Alert Severity Definitions and Examples](https://gtidocs.virustotal.com/docs/dtm-alert-severity). Improvement beyond the objective since it is not required for the alert prioritization, triage, and notification; however, it is pertinent for post automation handling for the analyst.

**Input/output**
One declared input, `param_is_slug_monitor_name` (int, 0 or 1), gating whether the block attempts to parse `monitor_name` under the double-hyphen multitenancy slug convention (a custom implementation documented in Google Cloud Security Community [Multi-tenancy on a single Google SecOps](https://security.googlecloudcommunity.com/google-security-operations-2/multi-tenancy-on-a-single-google-secops-part-1-the-isolation-challenge-7789?tid=7789&fid=2)). 

No execution output. All effects are case-scoped writes: the two tag calls, the insight, and the instruction message.

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| param_is_slug_monitor_name | Input Parameter | Integer | Whether to enable processing of slug-ed DTM monitor names for extra enrichment. Set to 1 to enable. | 0 (DISABLED) |

**Steps**
1. `Tools - Get Original Alert Json` pulls the alert's raw JSON. Every downstream field reference in this block reads from its result, so this step is load-bearing for everything after it.
2. `Siemplify - Case Tag` ("Tag GTI DTM Alert") tags the case `gti:dtm,monitor:<monitor_name>` (trimmed, lowercased), unconditionally.
3. `Flow - IfFlowCondition` ("Is monitor_name Field Slug-ed?") branches on `[Input.param_is_slug_monitor_name] == 1`.
    - *"Yes"* branch
        1. `Siemplify - Case Tag` ("Tag Multi-Tenant Compatible GTI DTM Alert") splits `monitor_name` on double-dashes (`--`) and tags `monitor_tenant` (index 0) and `monitor_type` (index 1).
    - *"Else"* branch
        1. *No Action*
4. Both branches converge on `Siemplify - Add General Insight`, which writes a case HTML insight, as shown below:

![HTML Insight example](/examples/dtm-catchall/static/ss-case-html-insight.png)

5. `Siemplify - Instruction` attaches a plain-text severity-definitions message for the analyst.
6. *End*

**Error handling** 
None explicit. All actions have `autoSkipOnFailure: false`, so a failure in either action halts this block, and with it the whole playbook.

### 5.3–5.6 — pending

`2 GTI-DTM Alert Score by Severity`, `3 Any Alert Prioritization`, `4 Any Alert Triage`, and `5 Any Alert Notification` are documented in the following sessions.

## 6. Playbook Statement

*DTM CatchAll* triggers on any Google Threat Intelligence DTM alert (any monitor, any alert sub-type) that carries a non-empty `monitor_id` and `monitor_name`. On execution, **`1 Any Case Initialization`** runs generic, cross-playbook case setup (first-alert / similar-case detection); **`1 GTI-DTM Alert Case Initialization`** then runs DTM-specific enrichment (original-alert-JSON capture, case-wall insight, monitor-slug-aware tagging). **`2 GTI-DTM Alert Score by Severity`** reads the alert's native severity and writes a weighted score into case context. **`3 Any Alert Prioritization`** reads that score and sets the case's real, native `Alert.Priority` field. **`4 Any Alert Triage`** reads the resulting priority to either auto-close a benign alert or assign the case to the appropriate tier and move it to the Investigation or Incident case stage. Finally, **`5 Any Alert Notification`** reads `Alert.Priority` against a configurable gate and, if it clears the gate, notifies the configured contacts by email and, optionally, Telegram.

## 7. Assumptions and Improvements

**Assumptions**

- I'm assuming every DTM alert (regardless of sub-type — Compromised Credentials, Document, etc.) carries `monitor_id` and `monitor_name`; I deliberately didn't enumerate sub-types in the trigger condition (Section 3), accepting the residual risk that a future non-DTM GTI alert type could coincidentally carry monitor-shaped fields.
- I'm assuming the Scoring subflow's output field name and the Prioritization subflow's input field name stay in agreement going forward. That's `Alert.ALERT_SEVERITY`, written by Scoring and read by Prioritization's `Prioritize by Alert Severity` condition. Neither subflow declares this as a formal contract yet, so a future edit to either one's field naming could silently break the link again without either subflow raising an error.

**Improvements**

- **(BLOCK) Any Alert Notification**: Added HTML template for `EmailV2 - Send Email` integration with parameters `param_branding_name` and `param_playbook_name`.

![Email sent example](/examples/dtm-catchall/static/ss-notification-email-sample.png)

## 8. Simulation

No formal Playbook Simulator run is written up here yet. Several real debug-mode runs have been executed against the live playbook, including a full end-to-end pass confirming Triage assignment and Notification delivery.

## 9. Resources

- [GTI DTM Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity) — Sections 4

## 10. Associated ADS

No detection-rule ADS is linked here. The alerts this playbook consumes originate from any GTI Digital Threat Monitoring, not a SecOps-native YARA-L rule, so the detection logic lives in each DTM monitor's own Lucene query configuration.
