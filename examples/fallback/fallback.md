---
ucdd_version: 1.1
use_case_name: "Fallback Playbook"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-12
last_update: 2026-09-18
owner: "rodajrc"
status: "ACTIVE:IN_DEVELOPMENT"
related_flows:
    - "Generic Case Initialization"
    - "Generic Alert Score by Entity Risk with GTI"
    - "RULE Alert Score by Severity"
    - "Generic Alert Prioritization by Alert Severity"
    - "Generic Alert Assessment and Disposition"
    - "Generic Alert Notification"
---

# Fallback Playbook

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Fallback Playbook |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-12* |
| **Last update** | *2026-09-18* |
| **Owner** | rodajrc |
| **Status** | **Active, in development**: enabled live at the platform's lowest priority with an All trigger. Six blocks and one condition of its own; notification channels disabled at the call site. Exercised end-to-end by the owner on live alerts on 2026-09-18. |
| **Related flows** | Chained subflows, each with its own UCDD under [`/examples/subflows/`](/examples/subflows/): Generic Case Initialization, Generic Alert Score by Entity Risk with GTI, RULE Alert Score by Severity, Generic Alert Prioritization by Alert Severity, Generic Alert Assessment and Disposition, Generic Alert Notification. |

**This is UCDD Version 1.1**

## 1. Workflow Statement

> *Fallback Playbook* triggers on any alert that no more specific playbook claimed. On execution, **`Generic Case Initialization`** runs the generic case setup and opens the `Triage` stage on the case's first alert. **`Generic Alert Score by Entity Risk with GTI`** enriches the alert's entities with Google Threat Intelligence and scores the alert from their threat scores, or writes a `Low` fallback score when there is nothing to enrich. If the alert comes from a Google SecOps SIEM rule (`Alert.DeviceProduct` is `RULE`), **`RULE Alert Score by Severity`** adds the rule's own severity to the score. **`Generic Alert Prioritization by Alert Severity`** maps the resulting severity onto the native `Alert.Priority`. **`Generic Alert Assessment and Disposition`** never disposes of an alert here, because the disposition list is left empty: on the case's first alert it moves the case to `Investigation` and assigns it to the investigation team, and for a high or critical severity it reassigns the case to the escalation team. Finally, **`Generic Alert Notification`** gates on `Alert.Priority` at high and above and, once a channel is enabled, notifies the configured contacts.

## 2. Workflow Objective

> Perform basic triage on any alert and determine whether the case is a candidate for a more specific playbook.

- **Outcome**: every alert reaches an analyst's queue, prioritized and assigned, with nothing closed automatically.
- **Scope**: alerts of any vendor and product that no other playbook attached to, in any environment.
- **Dependencies**: the generic blocks, a Google Threat Intelligence integration instance for entity scoring (detected at run time, not required), and, for SIEM rule alerts only, the rule severity field.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition`, `Previous Actions Conditions` | Required | In every block. |
| `Tools` | Google SecOps built-in toolkit | `Find First Alert`, `Get Integration Instances`, `Add Alert Scoring Information`, `Assign Case to User` | Required | Blocks 1, 2, 3 and 5. |
| `GoogleThreatIntelligence` | Entity reputation | `Enrich Entities` | Optional, per-environment | Block 2 detects the instance at run time and comments on the case when it is missing. |
| `Siemplify` | Google SecOps native action integration | `Get Similar Cases`, `Change Case Stage`, `Case Comment`, `Change Alert Priority`, `Close Alert`, `Mark As Important`, `Instruction` | Required | Blocks 1 to 6. `Close Alert` is present in block 5 but unreachable at this call site. |
| `EmailV2` | Notification channel | `Send Email` | Optional | Only when `param_enable_email=1`. |
| `Telegram` | Notification channel | `Send Message` | Optional | Only when `param_enable_telegram=1`. |

**Configurable Parameters**

The following table lists the values bound at each block's call site. Values that differ from the block's own default are marked.

| Parameter Name | Data Type | Description | Value at the call site | Declared by |
|---|---|---|---|---|
| CONST_CONTEXT_ALERT_SEVERITY | String | Name of the scoring entries the two scoring blocks write; one shared key. | CTX_ALERT_SEVERITY | [Generic Alert Score by Entity Risk with GTI](/examples/subflows/generic--alert-score-by-entity-risk-with-gti.md), [RULE Alert Score by Severity](/examples/subflows/rule--alert-score-by-severity.md) |
| param_enable_fallback_severity | Integer (0,1) | Writes a fallback scoring entry when a block has nothing to score from (`Low` in block 2, `Medium` in block 3). | 1 | [Generic Alert Score by Entity Risk with GTI](/examples/subflows/generic--alert-score-by-entity-risk-with-gti.md), [RULE Alert Score by Severity](/examples/subflows/rule--alert-score-by-severity.md) |
| CONST_ALERT_SEVERITY | String | Severity read by prioritization and assessment. Bound to `[Alert.ALERT_SEVERITY]`, which the scoring blocks set. | `[Alert.ALERT_SEVERITY]` | [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md), [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_disposition_severities | String, comma-separated | Severities the assessment block would close outright. **Set to empty** (block default `informational,low`): the fallback never closes an alert. | — | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_escalation_severities | String, comma-separated | Severities that reassign the case to the escalation team. | high,critical | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_investigation_team | String (SOC role) | Team assigned, with the `Investigation` stage, on the case's first alert. | @Tier1 | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_escalation_team | String (SOC role) | Team the case is reassigned to for an escalation severity. | @Tier2 | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) |
| param_alert_priority_to_communicate | String, comma-separated | `Alert.Priority` values that clear the notification gate. **Set to `high,critical`** (block default `low,medium,high,critical`). | high,critical | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_enable_email | Integer (0,1) | Enables the EmailV2 channel. **Set to 0** (block default 1). | 0 (disabled) | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_enable_telegram | Integer (0,1) | Enables the Telegram channel. | 0 (disabled) | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_telegram_chat_id | String | Target Telegram chat ID. Required only if `param_enable_telegram=1`. | — | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_branding_name | String | Organization branding name in the email template. | Zevorus | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |
| param_playbook_name | String | Playbook name in the email footer. **Set to `Fallback Playbook`** (block default `Alert Notification Playbook`). | Fallback Playbook | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) |

> **NOTE**
> Parameters prefixed with the keyword `CONST` should not be modified.

## 4. Outcomes and Analysis

This playbook captures and responds to any alert type. In general, if this playbook is seen running frequently against a particular incident or case type, that is a signal that a more specific playbook is needed to automate the response.

For cases produced by native platform alerts (e.g. Google SecOps YARA-L rules), you can trust the automated assessment with moderate confidence. For cases produced by third-party platform alerts, you must trust the automated assessment with lower confidence.

**Automated Outcomes**
- **Similar Cases UI Widget** and **Triage stage**, on the case's first alert ([Generic Case Initialization](/examples/subflows/generic--case-initialization.md)).
- **Entity enrichment and alert score**: every entity enriched with GTI and one scoring entry per entity, or a case comment plus a `Low` fallback entry when there is nothing to enrich ([Generic Alert Score by Entity Risk with GTI](/examples/subflows/generic--alert-score-by-entity-risk-with-gti.md)); for SIEM rule alerts, one more entry carrying the rule's severity ([RULE Alert Score by Severity](/examples/subflows/rule--alert-score-by-severity.md)).
- **Alert Priority Update**: native `Alert.Priority` set from the composed severity, `Medium` when no scoring entry was written ([Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md)).
- **Assignment and escalation**: on the case's first alert, stage `Investigation` and assignee `@Tier1`; for a high or critical severity, reassignment to `@Tier2`. No alert is closed ([Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md)).
- **Notification** for priorities high and critical, once `param_enable_email` or `param_enable_telegram` is set at the call site; with both at `0`, the block runs and ends without sending ([Generic Alert Notification](/examples/subflows/generic--alert-notification.md)).

**Human-in-the-Loop Actions**

None. The assigned case is the hand-off; every automated action here is one an analyst can revise.

## 5. Workflow Category

main-workflow:fallback

## 6. Trigger Conditions

Any alert that has not been attached to a more specific playbook, in any environment. The playbook uses the platform's *All* trigger, which has no condition, so no original alert field is read to decide execution.

### Execution Policy

- **Priority**: Lowest priority. On Google SecOps SOAR the scale runs from 1 (highest) to 3 (lowest), with 2 as the default, and only one playbook attaches automatically per alert; this playbook runs at 3, while the product-specific [DTM CatchAll](/examples/dtm-catchall/dtm-catchall.md) runs at 2. The framework's five-level scale (Section 6 of `UCDD-FRAMEWORK.md`) maps its *ANYTHING* level onto the platform's 3.
- **Reason**: The fallback playbook exists to fill automation gaps by notifying users about potential cases that may require a more specific playbook.

## 7. Automation Strategy

### Strategy Summary

**Trigger** on any alert. **Initialize** the case once. **Score** the alert from its entities' reputation and, for SIEM rule alerts, from the rule's severity. **Prioritize** the alert from that score. **Assess** it without closing anything: assign every case to the investigation team and escalate the severe ones. **Notify** the SOC team, once a channel is enabled. Everything but one routing condition is done by generic blocks, so every improvement to a block reaches the fallback.

### Technical Strategy

1. **Trigger** with the platform's *All* trigger at the lowest priority, so any more specific playbook wins the automatic attachment.
2. **Initialize** the case with *Generic Case Initialization*: Similar Cases widget and `Triage` stage on the first alert only.
3. **Score** the alert with *Generic Alert Score by Entity Risk with GTI* (fallback severity on), then test `[Alert.DeviceProduct]` for `RULE` and, when it matches, add *RULE Alert Score by Severity* (fallback severity on). Both write scoring entries under the shared key, which the platform composes into one alert score and `[Alert.ALERT_SEVERITY]`.
4. **Prioritize** the alert with *Generic Alert Prioritization by Alert Severity*, bound to `[Alert.ALERT_SEVERITY]`; its `else` branch (`Medium`) is only reached when neither scoring block wrote an entry.
5. **Assess** the alert with *Generic Alert Assessment and Disposition*, with the disposition list empty so that nothing is closed on incomplete information; the first alert gets `Investigation` and `@Tier1`, high and critical severities get `@Tier2`.
6. **Notify** with *Generic Alert Notification* at `high,critical`, channels off until a deployment turns one on.

### Additional Notes for Development

The owner's design notes not yet implemented in the live playbook:

- Tag the alert's grouping case as fallback.
- Instruct how to handle an alert or case modified by the fallback playbook: the fallback cannot assess with high confidence the severity and priority of all alerts, so analysts must treat any automated action with low confidence.
- Identify similar cases by comparing the alert's shared entities and properties, and suggest creating a specific playbook if many instances exist.
- Calculate a risk level. (Severity scoring and entity scoring, which the original notes listed here, were implemented on 2026-09-17 as blocks 2 and 3, with Google Threat Intelligence instead of VirusTotalV3.)

## 8. Workflow and Subflows

Every subflow is a reusable block with its own UCDD under [`/examples/subflows/`](/examples/subflows/). This section records how this playbook chains them and what each link relies on; the blocks themselves (steps, actions, outcomes, error handling) are documented in their own files and not repeated here.

The **Fallback Playbook** has the following high-level structure:

> **(1)** *Case Initialization* -> **(2)** *Alert Score by Entity Risk* -> **(3)** *Alert Score by Rule Severity* (SIEM rule alerts only) -> **(4)** *Alert Prioritization* -> **(5)** *Alert Assessment and Disposition* -> **(6)** *Alert Notification*.

**Subflow chain**

| # | Subflow | Category | Bound at the call site | Reads | Writes |
|---|---|---|---|---|---|
| 1 | [Generic Case Initialization](/examples/subflows/generic--case-initialization.md) | subflow:enrichment | — | case alerts | Similar Cases widget, case stage `Triage` |
| 2 | [Generic Alert Score by Entity Risk with GTI](/examples/subflows/generic--alert-score-by-entity-risk-with-gti.md) | subflow:triage | `CONST_CONTEXT_ALERT_SEVERITY` = `CTX_ALERT_SEVERITY`, `param_enable_fallback_severity` = 1 | alert entities, environment integration instances | GTI enrichment, scoring entries, case comment on the empty paths |
| 3 | [RULE Alert Score by Severity](/examples/subflows/rule--alert-score-by-severity.md) | subflow:triage | `CONST_CONTEXT_ALERT_SEVERITY` = `CTX_ALERT_SEVERITY`, `param_enable_fallback_severity` = 1 | `[Event.detection_ruleLabels_severity]` | scoring entry |
| 4 | [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md) | subflow:triage | `CONST_ALERT_SEVERITY` = `[Alert.ALERT_SEVERITY]` | composed alert severity | native `Alert.Priority` |
| 5 | [Generic Alert Assessment and Disposition](/examples/subflows/generic--alert-assessment-and-disposition.md) | subflow:case-management | `CONST_ALERT_SEVERITY` = `[Alert.ALERT_SEVERITY]`, `param_disposition_severities` = empty, `param_escalation_severities`, `param_investigation_team`, `param_escalation_team` | alert severity, first alert of the case | case stage and assignee |
| 6 | [Generic Alert Notification](/examples/subflows/generic--alert-notification.md) | subflow:case-management | `param_alert_priority_to_communicate` = `high,critical`, `param_enable_email` = 0, `param_enable_telegram` = 0, `param_telegram_chat_id`, `param_branding_name`, `param_playbook_name` = `Fallback Playbook` | `Alert.Priority` | email, Telegram message, Important flag on failure |

**Main-workflow actions outside the blocks**

One condition, `Is RULE alert?`: `Flow - IfFlowCondition` on `[Alert.DeviceProduct]` equal to `RULE`, between blocks 2 and 4. Its `YES` branch chains block 3; its default branch goes straight to block 4, where both paths converge.

**Contract between the blocks**

- Blocks 2 and 3 are the writers of the severity every later block consumes: each `Tools - Add Alert Scoring Information` call composes all entries on the alert and sets `[Alert.ALERT_SEVERITY]` (see the [DTM Alert Score by Severity](/examples/subflows/dtm-catchall--alert-score-by-severity.md) UCDD, Section 9). Both write under the same `CTX_ALERT_SEVERITY` name so their entries compose into one score; a rule severity and an entity score therefore add up rather than override each other.
- Blocks 4 and 5 read `[Alert.ALERT_SEVERITY]`. With fallback severity on in both scoring blocks, it is set on every path; if it were ever unset, block 4 sets `Medium` and block 5 takes the `else` branch of both list checks (owner-verified 2026-09-18), so the alert is neither disposed nor escalated.
- Block 6 reads the native `Alert.Priority` that block 4 wrote, not the severity.
- Blocks 1, 2, 4, 5 and 6 read no original alert field. Block 3 reads `[Event.detection_ruleLabels_severity]`, but only runs behind the `Is RULE alert?` guard, so the workflow still triggers on anything and depends on no field to execute.
- Blocks 1 and 5 each call `Tools - Find First Alert` and compare its result with `[Alert.Identifier]`, so a case's first alert is the only one that gets the `Triage` stage, the Similar Cases widget, the `Investigation` stage and the `@Tier1` assignment; later alerts grouped into the case are scored, prioritized, possibly escalated, and notified.

**Error Handling**

No block halts the workflow explicitly. Every action runs with `autoSkipOnFailure: false` except `Enrich Entities` in block 2, the two assignment actions in block 5 and the two send actions in block 6, so a failure elsewhere stops the run at that step. Block 2 turns a missing GTI instance or a failed enrichment into a case comment and a fallback score rather than a halt.

## 9. Assumptions

- `[Alert.DeviceProduct]` carries the value `RULE` for alerts generated by Google SecOps SIEM rules (owner-verified on live alerts, 2026-09-18) and never for third-party products.
- Every GTI-enriched entity carries a `gti_assessment.threat_score`; an entity without one scores as Benign (block 2, Section 9).
- When `[Alert.ALERT_SEVERITY]` is unset, both list checks in block 5 take their `else` branch (owner-verified, 2026-09-18).
- The SOC roles `@Tier1` and `@Tier2` exist in the target instance.
- `[Environment.ContactEmail]` is populated before `param_enable_email` is turned on.

## 10. Improvements

- **Chain built from generic blocks** (2026-09-17): the fallback reuses the four generic blocks that DTM CatchAll chains and adds two scoring blocks of its own, both generic in turn. The notification block was briefly duplicated as a second "Generic Alert Notification" with an older Telegram text; the duplicate was deleted and the fallback rebound to the block DTM CatchAll uses.
- **Scoring from two sources** (2026-09-17): entity reputation for any alert, plus the rule's declared severity for SIEM rule alerts, composed by the platform under one shared key. Same-day fixes closed an undeclared-input key, a misspelt constant placeholder, literal entry names, and a first-alert gate that compared against `true`.
- **Disposition disabled at the call site** (2026-09-17): the block default `informational,low` would have auto-closed alerts on unverified severities, the wrong call for a workflow that runs on alert types nobody has looked at yet.
- Candidates from the owner's notes and from Google's reference fallback structure, not yet implemented: case tags marking fallback handling, a case-wall instruction about the low confidence of the automated assessment, a similar-case suggestion, a risk level, and an analyst question or handover step.

## 11. Workflow Simulation and Testing

Exercised end-to-end by the owner on live alerts on 2026-09-18 with no issue found; no recorded debug transcript. The same checks confirmed the `RULE` product value, the `else` path of block 5 when the severity is unset, and the enrichment-success check in block 2. Blocks 1, 4, 5 and 6 were exercised inside DTM CatchAll (see that UCDD, Section 11); blocks 2 and 3, the routing condition, the empty-disposition configuration and the All trigger are new to this playbook.

## 12. Resources

- [Create a fallback playbook](https://docs.cloud.google.com/chronicle/docs/soar/respond/working-with-playbooks/create-a-catch-all-triage-playbook), Google Security Operations documentation: priority scale, *All* trigger, and the reference structure a fallback playbook is meant to follow. Sections 6 and 10.
- [DTM CatchAll UCDD](/examples/dtm-catchall/dtm-catchall.md), the product-specific catch-all that shares all four blocks.

### Associated ADS

None. This playbook is detection-agnostic by definition.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-12 | rodajrc | Initial scaffold from `TEMPLATE_FULL_UCDD.md`, on v1.1 |
| 1 | 2026-09-17 | rodajrc | **Completed from the live playbook**: four generic blocks chained (Generic Alert Assessment and Disposition replaces the planned Generic Case Lifecycle Management by Severity; Generic Alert Notification added), call-site values recorded, All trigger and priority 3 documented, owner's design notes moved to Section 7 development notes and Section 10. |
| 2 | 2026-09-18 | rodajrc | **Change**: two scoring blocks added ahead of prioritization, `Generic Alert Score by Entity Risk with GTI` for every alert and `RULE Alert Score by Severity` behind an `Is RULE alert?` condition in the main workflow; notification gate raised to `high,critical`; the duplicate notification block replaced by the shared one. Eleven defects found in the export were fixed live the same day and are recorded in the blocks' UCDDs. |
