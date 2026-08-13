---
ucdd_version: 1
use_case_name: "Digital Threat Monitoring Catch All (dtm-catchall)"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-08-01
last_update: 2026-08-12
owner: "rodajrc"
status: "ACTIVE:IN_DEVELOPMENT"
related_flows:
    - "Base Case Initialization"
    - "DTM Alert Case Initialization"
    - "DTM Alert Score by DTM Severity"
    - "Alert Prioritization"
    - "Alert Notification"
---

# Digital Threat Monitoring Catch All Playbook

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Digital Threat Monitoring Catch All (DTM CatchAll) |
| **SOAR platform** | Google SecOps SOAR (Formerly, Siemplify) |
| **Creation date** | *2026-08-01* |
| **Last update** | *2026-08-12* |
| **Owner** | rodajrc |
| **Status** | **Active, in development**: Playbook handles case initialization, alert assessment through severity scores, alert prioritization, and alert notification with Email and Telegram 3P integrations. Missing: Case triaging workflow. |
| **Related flows** | Base Case Initialization, DTM Alert Case Initialization, DTM Alert Score by DTM Severity, Alert Prioritization, Alert Notification. |

**This is UCDD Version 1**

## Version history

Every non-trivial change to this document or to the playbook it describes. This is where continuous improvement after go-live gets recorded as well.

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0.0 | 2026-08-01 | rodajrc | **Initial Draft**: Objective defined |
| 0.1 | 2026-08-12 | rodajrc | **Initial Draft**: Categorization, Trigger and Execution Priority defined |

## 1. Objective

> Notify the suitable incident response team tier through alert assessment and prioritization of events originated by Google Threat Intelligence Digital Threat Monitoring (DTM).

**Main Outcome**
- Notification action

**Dependencies**
- Alert assessment depends on scoring the alert's severity leveraging either (or both) the DTM alert assigned severity or advanced IoC entity enrichment workflow.
- Case triage through continuous alert assessment to the suitable Incident Response SOC Team (e.g. Tier 1, Tier 2, and Tier 3).

**Considerations**
- Tenants may have differing IR SOC tiers; important candidate for parameterization
- Advanced IoC entity enrichment is more complex than just mapping the SOAR case priority to the highest DTM alert assigned severity
- Confirmed benign alerts can be auto-closed

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

1. **Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring* (product) alert leveraging the Original JSON alert schema.
    - If the SOAR platform supports triggering by *Product Name* equal to `DTM Alert`
    - Alertnatively, trigger if the alert contains the orginal fields `monitor_id` and `monitor_name`.
    - To reduce the likelihood of false-positive execution, include a trigger condition by `[Alert.DeviceVendor]` equal to `Google Threat Intelligence`

2. **Score** the alert using the documented [Alert Severity Definitions](https://gtidocs.virustotal.com/docs/dtm-alert-severity#prioritization-of-alerts).
    - Use *Tools - Add Context Value* integration action to store the alert severity in the case's scope
    - Use *Siemplify - Add Scoring Context Information* integration action to calculate a weighted score of the alert

3. **Prioritize** the alert using *Siemplify - Change Alert Priority* integration action.

4. **Triage** the alert using the alert scoring information and current case priority.
    - Use *Siemplify - Change Case Stage* to modify the case's current stage to `Investigation` or `incident`
    - Use *Siemplify - Assign Case* to assign the case to an Incident Response SOC team (e.g. Tier 1)

5. **Notificate** the alert leveraging *EmailV2 - Send Email* integration action, and optionally, *Telegram - Send Message*
