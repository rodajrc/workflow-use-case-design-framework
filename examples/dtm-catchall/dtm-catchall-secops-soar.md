---
ucdd_version: 1
use_case_name: "Digital Threat Monitoring Catch All (dtm-catchall)"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-08-01
last_update: 2026-08-14
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
| **Last update** | *2026-08-14* |
| **Owner** | rodajrc |
| **Status** | **Active, in development**: Playbook handles case initialization, alert assessment through severity scores, alert prioritization, and alert notification with Email and Telegram 3P integrations. Triage block (`4 Any Alert Triage`) is live but I've flagged it as incomplete as being too simple, not genuinely parameterized, and isn't product alert specific. Missing Score block for GTI-DTM entities and artifacts. |
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
    - Alternatively, trigger if the alert contains the orginal fields `monitor_id` and `monitor_name` (product distinctive fields)
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

5. **Notificate** the alert leveraging *EmailV2 - Send Email* integration action, and optionally, *Telegram - Send Message*

## 5. Modular Technical Implementation

Technical Strategy is enough for now

## 6. Playbook Statement

> TODO
> Still under development

*DTM CatchAll* triggers on any Google Threat Intelligence DTM alert (any monitor, any alert sub-type) that carries a non-empty `monitor_id` and `monitor_name`. On execution, **`1 Any Case Initialization`** runs generic, cross-playbook case setup (first-alert / similar-case detection); **`1 GTI-DTM Alert Case Initialization`** then runs DTM-specific enrichment (original-alert-JSON capture, case-wall insight, monitor-slug-aware tagging). **`2 GTI-DTM Alert Score by Severity`** reads the alert's native severity and writes a weighted score into case context. **`3 Any Alert Prioritization`** reads that score and sets the case's real, native `Alert.Priority` field. **`4 Any Alert Triage`** reads the resulting priority to either auto-close a benign alert or assign the case to the appropriate tier and move it to the Investigation or Incident case stage. Finally, **`5 Any Alert Notification`** reads `Alert.Priority` against a configurable gate and, if it clears the gate, notifies the configured contacts by email and, optionally, Telegram.

## 7. Assumptions and Improvements

- I'm assuming every DTM alert (regardless of sub-type — Compromised Credentials, Document, etc.) carries `monitor_id` and `monitor_name`; I deliberately didn't enumerate sub-types in the trigger condition (Section 3), accepting the residual risk that a future non-DTM GTI alert type could coincidentally carry monitor-shaped fields.

## 8. Simulation

No formal Playbook Simulator run is on record for this build.
Formal simulation is deferred until first version is completed.

## 9. Resources

- [GTI DTM Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity) — Sections 4

## 10. Associated ADS

No detection-rule ADS is linked here. The alerts this playbook consumes originate from any GTI Digital Threat Monitoring, not a SecOps-native YARA-L rule, so the detection logic lives in each DTM monitor's own Lucene query configuration.
