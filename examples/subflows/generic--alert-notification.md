---
ucdd_version: 1.1
use_case_name: "Generic Alert Notification"
soar_platform: "GOOGLE_SECOPS_SOAR"
creation_date: 2026-09-09
last_update: 2026-09-17
owner: "rodajrc"
status: "ACTIVE"
related_flows:
    - "Digital Threat Monitoring Catch All (dtm-catchall)"
    - "Fallback Playbook"
---

# Generic Alert Notification

## Document metadata

| **Field** | **Value** |
|---|---|
| **Use case name** | Generic Alert Notification |
| **SOAR platform** | Google SecOps SOAR |
| **Creation date** | *2026-09-09* (split out of the DTM CatchAll UCDD, where it was documented since 2026-08-19) |
| **Last update** | *2026-09-17* |
| **Owner** | rodajrc |
| **Status** | **Active**: running live as the last block of DTM CatchAll and of the Fallback Playbook. Email and Telegram both supported. Named `DTM Alert Notification` until 2026-09-17. |
| **Related flows** | Invoked by chaining from [Digital Threat Monitoring Catch All](/examples/dtm-catchall/dtm-catchall.md) and from [Fallback Playbook](/examples/fallback/fallback.md). Reads the `Alert.Priority` set by [Generic Alert Prioritization by Alert Severity](/examples/subflows/generic--alert-prioritization-by-alert-severity.md). |

## 1. Workflow Statement

> *Generic Alert Notification* gates on the alert's priority and, when it clears the configured set, sends a branded HTML email to the environment's contact and, optionally, a branded Telegram message to a configured chat. It renders only native alert and case fields, so any main workflow can chain it. A delivery failure on either channel marks the case as Important and leaves an instruction on the case wall.

## 2. Workflow Objective

> Tell the Incident Response team about every alert that matters, from any product, on the channels they actually read, and never let a failed notification pass unnoticed.

## 3. Configuration and Deployment

**Integrations**

| Integration | Role | Actions Used | Required | Note |
|---|---|---|---|---|
| `Flow` | Google SecOps built-in control flow | `IfFlowCondition`, `Previous Actions Conditions` | Required | |
| `EmailV2` | Primary channel | `Send Email` | Required when `param_enable_email=1` | Sends to `[Environment.ContactEmail]`. |
| `Telegram` | Secondary channel | `Send Message` | Optional, per-environment | Requires a per-environment Telegram integration instance and the chat ID parameter. |
| `Siemplify` | Google SecOps native action integration | `Mark As Important`, `Instruction` | Required | Failure path only. |

**Configurable Parameters**

| Parameter Name | Type | Data Type | Description | Default Value |
|---|---|---|---|---|
| param_alert_priority_to_communicate | Input Parameter | String, comma-separated | The set of `Alert.Priority` values (lowercase) that clear the notification gate. | low,medium,high,critical |
| param_enable_email | Input Parameter | Integer (0,1) | Enables the EmailV2 channel. | 1 (enabled) |
| param_enable_telegram | Input Parameter | Integer (0,1) | Enables the Telegram channel. | 0 (disabled) |
| param_telegram_chat_id | Input Parameter | String | Target Telegram chat ID. Required only if `param_enable_telegram=1`. | — |
| param_branding_name | Input Parameter | String | Organization branding name shown in the email's HTML template. | Zevorus |
| param_playbook_name | Input Parameter | String | Playbook name shown in the email's footer. | Alert Notification Playbook |

## 4. Outcomes and Analysis

**Automated Outcomes**
- **Branded notification**: email and, optionally, Telegram, sent once the alert clears the `param_alert_priority_to_communicate` gate. Both render `[Alert.DeviceVendor]`, `[Alert.Product]`, `[Alert.Priority]`, `[Alert.Name]`, `[Alert.StartTime]`, `[Case.Id]` and a case link; branding and playbook name are parameters, so the template is never edited per deployment.

![Branded Notification: a redacted example of the notification you will receive when an alert is received, here from DTM CatchAll. The organization name ("Zevorus") and the playbook name ("DTM CatchAll") can be configured without modifying the HTML directly.](/examples/dtm-catchall/static/ss-notification-email-sample.png)

- **Important flag on failure**: the case is marked Important and receives a case-wall instruction when either channel fails.

**Human-in-the-Loop Actions**

None. The notification is the hand-off.

## 5. Workflow Category

subflow:case-management

## 6. Trigger Conditions

No trigger of its own. Chained by a main workflow as its last block, after prioritization. Precondition: `Alert.Priority` is set. No product-specific field is read.

## 7. Automation Strategy

Gate first, so the expensive channel actions never run for priorities nobody wants to hear about. Then treat each channel independently behind its own enable flag, with a *Previous Actions Conditions* check attached directly to each send action, so one channel failing does not stop the other and every failure becomes visible on the case. Render only native placeholders, so the block needs no knowledge of the alert's product.

## 8. Workflow and Subflows

**Integrations and Actions**

- `EmailV2 - Send Email`: sends a branded notification to the email contact configured in the Google SecOps SOAR environment settings.
- `Telegram - Send Message`: sends a branded notification to a custom Telegram chat through a Telegram bot. Requires a per-environment integration instance.
- `Siemplify - Mark As Important`: flags the case as Important when any of the response actions fails.

**Inputs and Outputs**

Six declared inputs (Section 3). No execution output.

**Steps**

1. Check whether the alert's priority is in `param_alert_priority_to_communicate`. If not, *End*.
2. If email is enabled, `EmailV2 - Send Email` sends the branded HTML to the environment's contact. If the send fails, an instruction is written to the case wall and the case is marked Important.
3. If Telegram is enabled, `Telegram - Send Message` sends the branded message to the configured chat. If the send fails, an instruction is written to the case wall and the case is marked Important.
4. *End*.

> **Warning**
> This is a simplification of the actual automation logic. Refer to the actual block to see exactly how it works.

**Outcomes and Effects**

1. `EmailV2 - Send Email` sends a branded HTML email (Zevorus palette) to `[Environment.ContactEmail]`, including a dynamic case link, `[General.HostUrl]cases/[Case.Id]`. There is no slash between the two placeholders because `[General.HostUrl]` already resolves with a trailing slash.
2. `Telegram - Send Message` sends a branded message with the same elements as the email to `[Input.param_telegram_chat_id]`.
3. `Siemplify - Mark As Important` fires only on a delivery failure, on either channel. It is the only place in DTM CatchAll and in the Fallback Playbook where this action fires, a deliberate choice so a case whose automated notification failed does not go unnoticed.

**Error Handling**

Both send actions run with `autoSkipOnFailure: true`, which is what lets the failure branch run instead of halting the playbook; every other step has `autoSkipOnFailure: false`. When either `EmailV2 - Send Email` or `Telegram - Send Message` fails, an instruction is written to the case wall describing the likely cause (`MISCONFIGURED_INTEGRATION`, `MISSING_INTEGRATION`) and the case is marked Important.

## 9. Assumptions

- `[Environment.ContactEmail]` is populated in every environment the workflow serves.
- The Telegram integration instance is configured per environment when `param_enable_telegram=1`.
- The priority gate is a substring check on the lowercase priority; the list values are whole priority names.

## 10. Improvements

- Notification enhanced with an HTML template for `EmailV2 - Send Email`, parameterised by `param_branding_name` and `param_playbook_name`.
- Telegram message normalised to carry the same elements as the email.
- Named DTM-specific on 2026-08-21, but its templates only ever rendered native alert placeholders, so on 2026-09-17 it was renamed `Generic Alert Notification` and chained by the Fallback Playbook as well. A short-lived duplicate block of the same name, carrying an older plain Telegram text, was deleted the same day.

## 11. Workflow Simulation and Testing

Exercised in every end-to-end debug run of DTM CatchAll; the first live delivery on a sample case alert was 2026-08-15 (see that UCDD, Section 11). Not yet exercised from the Fallback Playbook, where both channels are disabled at the call site.

## 12. Resources

- [Digital Threat Monitoring Catch All UCDD](/examples/dtm-catchall/dtm-catchall.md), the workflow this block was written for.
- [Fallback Playbook UCDD](/examples/fallback/fallback.md), its second call site.

### Associated ADS

None.

## Version history

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0 | 2026-09-09 | rodajrc | **Docs**: out of the DTM CatchAll UCDD (its version 14, Section 8.6) into a standalone subflow UCDD, as `dtm-catchall--alert-notification.md`. Content unchanged. |
| 1 | 2026-09-17 | rodajrc | **Renamed** live and in this repository to `Generic Alert Notification` (file `generic--alert-notification.md`); the duplicate generic block deleted; steps documented from the export; email-channel default corrected to enabled; stale claim that DTM fields are rendered removed. |
