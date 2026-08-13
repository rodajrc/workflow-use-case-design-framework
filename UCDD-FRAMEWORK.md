# Use Case Design Document (UCDD) — Template

A UCDD is written before a playbook is built, and kept alive after it ships. Each section answers a specific question a designer must not skip. Where a section is genuinely open or unresolved for this use case, say so explicitly rather than leaving it blank.

One UCDD per automation use case. A use case may end up implemented as any of the following combinations:
- A single standalone workflow (playbook); usually short automation flows or priority one deterministic, specific use cases.
- A single standalone subflow (block); usually a general use subflow that can later be used for other automation use cases.
- A main workflow (playbook), plus the several subflows (blocks) it is made of; usually a comprehensive use case that seeks to resolve an incident response gap.

**This is UCDD Version 1**

---

## Document metadata

You would likely want to keep a record of your UCDD as you work in more playbooks. 

You can store them in a format like this.

| **Field** | **Value** |
|---|---|
| **Use case name** | *The name of your use case* |
| **SOAR platform** | *The name of your SOAR platform. For example: Google SecOps SOAR* |
| **Creation date** | *The date you created the UCDD* |
| **Last update** | *The date of your last update* |
| **Owner** | *The user or team that owns the playbook. You may include contact information as well.* |
| **Status** | *Any of the following: [`Draft`, `Active`, `Deployed`, `Decommissioned`]. You can define your own status enumeration as well.* |
| **Related flows** | *If a main workflow, include the subflows it is made of (flow chain). Also, depending on your platform features, your workflow may support flow attachment apart from flow chaining* |

Or like this:

```yaml
---
ucdd_version: 1
use_case_name: "The name of your use case"
soar_platform: "The name of your SOAR platform"
creation_date: yyyy-mm-dd
last_update: yyyy-mm-dd
owner: "Your name or your team"
status: STATUS_ENUM
related_flows: ["related_chained_subflow_1", "related_chained_subflow_2", "related_attached_workflow_a"]
---
```

> **Tip**
> YAML Frontmatters are really effective for coding agents and advanced file-system searching via custom scripts

## Version history

Every non-trivial change to this document or to the playbook it describes. This is where continuous improvement after go-live gets recorded as well.

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0.0 | *Today's date* | *Your name* | *Initial draft* |

When versioning, I suggest the following procedures:

<!-- TODO: versioning procedure under revision -->

- **Minor versioning**: 
    - If working on a single standalone workflow or subflow, you increase the minor version when you modify individual automation actions and reconfigure main flow settings.
    - If working on a main workflow, you increase the minor version when modifying individual subflow actions or main workflow settings (e.g. description or priority)
- **Major versioning**: When the playbook includes a whole new subflow
    - If working on a single standalone workflow or subflow, you increase the major version when you add new integration actions or you change heavily the logic of the automation
    - If working on a main workflow, you increase the major version when modifying or adding new subflows entirely

You can also add tags or keywords in the *summary of changes* depending on the type of addition or modification you did. Some examples include: `config` for config changes to automation actions, `flow_creation` when you add a new subflow to a main workflow, or `error_handling` when you create conditional logic to handle potential errors in your automation workflow.

---

## 1. Objective

**What has to be done through automation?**

State the goal in plain language, and be precise about the verbs. 

**Vague goals produce vague playbooks**. For example: *"Handle all alerts"* is a bad objective, not because *"all alerts"* is incorrect (that's a *catch-all* scope) but because the word *"handle"* hinders what you really mean: Is it closing the alert? or communicating it to someone? or prioritizing it for triage? 

A good objective either picks one of those precisely, or explicitly scopes in all of them as distinct outcomes. For example: *"Notify the incident responder of all their tenant scoped alerts"* is slightly better because it clearly defines the **scope** (all tenant-bound alerts) and the **outcome** (notification specifically to the incident responder) of the use case.

Even better is the following example: *"Notify the incident response team of all their prioritized tenant scoped alerts"*. This example is great because it clearly states the **scope** (all prioritized tenant-bound alerts), the **outcome** (notification specifically to the incident response team), and the **dependencies** that are needed to meet the result (alert prioritization requires alert scoring and triage).

Here's the full objective I would write for an automation use case:

> Notify the suitable incident response team tier through alert assessment and prioritization of events originated by Google Threat Intelligence Digital Threat Monitoring (DTM).

Unfortunately, there's no single formula to write down the objective. However, you don't have to write it as a sentence either. You can simply list the aspects of your objective. For example:

- **Outcome**: Notification of all prioritized alerts to the triaged IR team
- **Scope**: All GTI DTM alerts
- **Dependencies**: Prioritization of DTM alerts requires assessing the alerts (two methods: Built-in severity or entity enrichment).

Notice this second way of writing the objective allowed us to also describe the assessment methodology!

In summary, a well written objective must describe precisely one or many concrete **outcomes** that will lead you throughout the playbook design phase. A clearly defined **scope** that hints at the boundaries of your playbook logic. And finally, the **dependencies**, ideally explicit, that are needed to achieve your outcomes within your scope.

## 2. Categorization

> **Note**
> Not yet a settled taxonomy. Treat the following as a proposed starting point.

This section answers the question: *What kind of automation is this, on the axes that matter for this program?* I have identified two:

- **Structural role** — is this a **Main Workflow** (usually the entry point for an incident handling playbook or a complex automation use case. Also receives the name of *Playbook*) or a **Subflow** (usually invoked by another playbook, chained or attached, with its own input/output contract. Also receives the name of *Block*)? Most SOAR platforms distinguish these two roles in some form at the object level, even where the exact terminology differs.
- **Functional goal** — what is this automation actually *for*? Common recurring categories across the industry: 
    - `enrichment` (attaching context before a human or another flow sees the alert)
    - `triage` (assessing and scoring severity)
    - `containment` or `response` (isolating a host, disabling an account, blocking an indicator)
    - `case-management` (creating tickets, notifying, escalating)
    - `catch-all` or `fallback` (a deliberately generic net for alert types that don't yet have a dedicated playbook, recognized as a distinct pattern in its own right rather than an afterthought).

You can use these keywords when categorizing your playbooks in the form of `{structural_role}:{main_functional_goal}`. E.g. `main-workflow:catch-all`.

## 3. Trigger and Execution Priority

State the trigger condition as a single sentence. It should align with your **scope** in the **objective** section. 

You can also define a priority tier if supported by your SOAR platform to resolve collisions when more than one playbook's trigger matches the same alert. In a 1-2-3 priority model, you would assign the priorities as follows:

- **Priority 1** — Only for deterministic automation or event-specific incident handling. The most targeted match available for this exact condition. Anything is more specific than this playbook!
- **Priority 2** — This is for product-specific automation. Matches a whole product or alert family, broader than Priority 1 but not a last resort.
- **Priority 3** — For fallback or catch-all logic. Fires only when nothing more specific claimed the alert.

You can write this section as shown below:

```md
**Trigger Conditions**
Explain briefly the trigger conditions here.

**Execution Policy**
- **Priority**: State your playbook priority
- **Reason**: Why that priority?
```

## 4. Automation Strategy

**How is the objective going to be automated?**

Free text. A short plan of the mechanism, before it gets broken into implementation pieces.

Here you can describe more precisely the scope and dependencies of your automation goal, and you definitely start using product specific terminology to describe how you are going to solve the automation challenge.

The following example is a strategy summary for a DTM CatchAll playbook example I built in Google SecOps SOAR.

> **Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring (DTM)* (product) alert. **Score** the alert using DTM's *"Alert Severity Definitions"*. **Prioritize** the alert and grouping case using the calculated score and alert severity record leveraging *Siemplify* and *Tools* integration actions. **Triage** the case to the correct IR SOC team depending on the final priority level. Finally, **Notify** the SOC team leveraging *EmailV2* and, optionally, *Telegram* integrations.

When writing your strategy, do not be afraid of using text formatting to your own advantage to make it clearer. Also, use sentences to describe the workflow architecture.

1. I used **bold text** to represent **verbs** that lead to **outcomes** in the automation. In the example above, I have the following verbs that define each stage of the workflow:
    - Trigger
    - Score
    - Prioritize
    - Triage
    - Notify
2. I used *Italics* to highlight platform specific components and features.
    - Google Threat Intelligence
    - Digital Threat Monitoring
    - [DTM] Alert Severity Definitions
    - Siemplify Integration
    - Tools Integration
    - EmailV2 Integration
    - Telegram Integration
3. Each sentence describes a subflow within the main workflow. For example:
    - *"Score the alert using DTM's Alert Severity Definitions"* is an alert scoring subflow.
    - *"Notify the SOC team leveraging EmailV2 and, optionally, Telegram integrations"* is a notification subflow that explicitly states that Telegram is optional (parameterizable to be enabled or not)

> **Tip**
> You can begin by defining your *"Strategy Summary"*, as shown above. This is your base strategy. After the overall idea is set, you write a more concrete version of that summary: A *"Technical Strategy"*. 
> If you end up discovering that a technical claim you made was not supported by your platform, you can simply update that section in your *Technical Strategy* while keeping the base strategy intact! (unless your base strategy has a problem)

## 5. Modular Technical Implementation

Read back the Automation Strategy and break it into its smaller goals or milestones.

If you used the sentence technique to break down the workflow into clearly defined outcomes, this will become very easy for you to scrutinize. 

For each subflow, name it and state what it does. This is also where platform-specific detail belongs. For example, on Google SecOps SOAR, that means naming the actual integration actions each subflow will call.

Other users can take your UCDD and translate your logic into their own platforms as well.

> **Important**
> Do not forget specifying the input/output parameters, if any, of your subflows. This is very important for the Playbook Parameterization principle.

> **Note**
> As you develop your playbook, you may determine that another subflow is needed, or is not needed but good to have. You can also include those extra subflows as long as they don't break your original strategy.
> I'll suggest leaving these additional flows at the end though.

Also, for each action you MUST consider **error handling**.

**Which actions in this workflow and subflows are subject to failure, and what happens when they fail?**

List the actions that can realistically error, and what the playbook does about it. Throwing an error and halting the operation of the playbook is acceptable only if there's a good reason for it. 

For your reference, here is a list of recurring failure classes worth checking against every action: 
- `API_RATE_LIMIT`, when your integration action requires talking to a third-party tool and you reach an API rate limit or block.
- `MISSING_INTEGRATION`, when your platform lacks the integration instance needed to talk to a third-party platform.
- `MISCONFIGURED_INTEGRATION`, when your platform has the integration available but the configuration is incorrect.
- `MALFORMED_DATA`, when the data you expect from an alert or a previous integration action is not what you expected.
- `SYSTEM_ERROR`, when the SOAR platform fails.

> **Note**
> A shared taxonomy of failure classes across use cases would make this section more consistent and is worth developing once enough UCDDs exist to generalize from.

## 6. Playbook Statement

**Once the subflows are named, how do they compose into the automation described in the Automation Strategy?**

A short paragraph, written using the subflow names or identifiers from Section 5, describing the automation strategy in terms of those blocks. If it answers *how* the objective gets solved using named pieces, it's doing its job.

*Example (DTM CatchAll): Automation Playbook that triggers on DTM Alert detection. On execution, the playbook runs a generic case initialization followed by a DTM-alert-specific case initialization subflow to handle case enrichment. Next, the flow runs a DTM-specific alert scoring subflow to assess the severity of the alert, from which an alert prioritization subflow assigns native SOAR prioritization for effective case management and triage...*

While not a hard rule, the playbook statement should state the main playbook as a linear execution of abstract or complex actions.

## 7. Assumptions and Improvements

**What did the Modular Technical Implementation take for granted?**

List them explicitly. An assumption that later turns out false is expected, but an assumption nobody wrote down is a design gap.

You can also list any added logic you included to optimize, improve, or enhance the main outcome. For example, baseline case enrichment is a typical flow that is added to every workflow that is not inherently necessary to achieve the main goal.

## 8. Simulation

**What actual test runs have been done, and what did they show?**

Record every simulation run and every finding it produced, on whatever mechanism the platform provides for reproducible test execution. For example, Google SecOps SOAR's Playbook Simulator, run against a synthetic or real test case. 

Reproducible test runs against a real or synthetic alert or case serve to: 
1. Prove the playbook does what Sections 1–6 claim it does, deterministically.
2. Identify hardcoded settings that can be parameterized.
3. Find issues that can feed back into earlier sections rather than living only as a test log.

When testing, make sure to save the synthetic alert or cases you used.

> **Warning**
> If using real cases and alerts for testing, make sure to always validate you are not sharing sensitive information. 

## 9. Resources

**What did you use to build this, and where can someone verify it?**

Any reference used to design or implement this use case:

- Vendor documentation for the specific actions and platform features involved.
- Community guidance and posts.
- Related UCDDs this one depends on or was adapted from.

Cross-reference by section where useful, rather than leaving one undifferentiated list at the end.

## 10. Associated ADS

Palantir's [ADS-Framework](https://github.com/palantir/alerting-detection-strategy-framework/blob/master/ADS-Framework.md) has a **Response** section that can be linked from this file.

Together, a rule ADS and a workflow UCDD can fill the documentation gap and help you design end-to-end detection and incident response workflows!