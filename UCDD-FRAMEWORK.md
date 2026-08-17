# Use Case Design Document (UCDD) — Template

A UCDD is written before a workflow is built, and stays alive after it ships. Each section answers one question a designer must not skip. Where a section is genuinely open or unresolved for this use case, say so explicitly rather than leaving it blank.

One UCDD per automation use case. A use case may end up implemented as any of the following combinations:
- A single standalone workflow (playbook); usually a short automation flow or a deterministic, single-purpose use case.
- A single standalone subflow (block); usually a general-use component meant to be reused by other use cases.
- A main workflow (playbook), plus the several subflows (blocks) it is made of; usually a comprehensive use case that resolves a full incident-response gap.

**This is UCDD Version 1.1**

## How to use this template

The structure of this document has been designed to account for multiple consumer roles; namely, **SOC Analysts** that review alerts affected by automation workflows, **Integration Engineers** that care about enabling an automation workflow quickly and effectively in their organizational environment, and **Automation Engineers** deeply intrigued about the technical implementation details of an automation workflow.

Therefore, when **reading** the framework, you're expected to follow along in the order as is. It's built so a reader can stop once their question is answered. For an Integration Engineer and a SOC Analyst, the first five sections tell what the workflow does, how it's configured, what it produces, and what kind of workflow it is. An automation engineer keeps reading into the design and implementation detail that follows to gather more technical insight about the playbook.

Conversely, **Writing order** doesn't have to match reading order, and usually won't. A workable draft sequence could begin by settling the Objective and Category first, then deciding the Outcomes and Analysis you're aiming for (deciding what the workflow should produce before deciding how it produces it is equivalent to Test-Driven Development, paired with workflow simulation and tests). Configuration can be drafted here too, if the use case's parameters are already clear at this point. Next, you would draft the Automation Strategy, followed by the next sections in order: Trigger Conditions, Workflow and Subflows, Assumptions, and Improvements, potentially in parallel while the workflow is actually being built, checked continually against the Objective and Category so the build doesn't drift from its own stated goal. Finally, Workflow Tests and Resources accumulate throughout, as things get tested and referenced. Once the workflow is finished, close with a cleanup pass on the first five sections so they read as the finished summary they're meant to be. The Workflow Statement in particular is usually only accurate once written last — it's your executive summary. This isn't a rule, though — you can write the document however is easiest for you.

---

## Document metadata

Track identity and status metadata for this UCDD, useful once you're maintaining more than one. Store it as a table:

| **Field** | **Value** |
|---|---|
| **Use case name** | *The name of your use case* |
| **SOAR platform** | *The name of your SOAR platform. For example: Google SecOps SOAR* |
| **Creation date** | *The date you created the UCDD* |
| **Last update** | *The date of your last update* |
| **Owner** | *The user or team that owns the playbook. You may include contact information as well.* |
| **Status** | *Any of the following: [`Draft`, `Active`, `Deployed`, `Decommissioned`]. You can define your own status enumeration as well.* |
| **Related flows** | *If a main workflow, include the subflows it is made of (flow chain). Also, depending on your platform features, your workflow may support flow attachment apart from flow chaining* |

Or as YAML frontmatter:

```yaml
---
ucdd_version: 1.1
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

Every non-trivial change to this document or to the playbook it describes, including continuous improvement recorded after go-live.

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0.0 | *Today's date* | *Your name* | *Initial draft* |

Suggested versioning procedure:

<!-- TODO: versioning procedure under revision -->

- **Minor versioning** logs changes to individual automated actions or workflow settings.
- **Major versioning** logs entire subflows or big improvements that heavily change the automation logic.

You can add **tags** in the *summary of changes* column. For example: `config` for config changes, `new_subflow` when creating a new subflow, `improvement` when updating and improving a subflow, etc.

---

## 1. Playbook Statement

> **Once the subflows are named, how do they actually compose into the automation described in the [Automation Strategy](#4-automation-strategy) section?**

A short paragraph describing the automation strategy in terms of its trigger conditions and subflows that abstract the technical details of the automation workflow.

Example from [DTM CatchAll](/examples/dtm-catchall/dtm-catchall-secops-soar.md): 

*DTM CatchAll* triggers on any *Google Threat Intelligence* DTM alert that carries a non-empty `monitor_id` and `monitor_name`. On execution, **`1 Any Case Initialization`** runs a generic case setup, followed by **`1 GTI-DTM Alert Case Initialization`** that runs DTM-specific case enrichment. Next, **`2 GTI-DTM Alert Score by Severity`** reads the alert's native severity and writes a weighted score into the case's context. Next, **`3 Any Alert Prioritization`** reads that score and sets the case's real, native `Alert.Priority` field. **`4 Any Alert Triage`** reads the resulting priority to either auto-close a benign alert or assign the case to the appropriate tier and move it to the Investigation or Incident case stage. Finally, **`5 Any Alert Notification`** reads the Alert's assigned priority against a configurable gate and, if it clears the gate, notifies the configured contacts by email and, optionally, Telegram.*

> **TIP**
> If it answers *how* the objective gets solved using named pieces, it's doing its job.

## 2. Objective

> **What (security) challenge do we have that can be solved through automation?**

State the goal in plain language and be precise about the verbs. 

**Vague goals produce vague playbooks**. For example: *"Handle all alerts"* is a bad objective, not because *"all alerts"* is incorrect (that's a *fallback* workflow) but because the word *"handle"* hinders what you really mean: Is it closing the alert automatically? or communicating it to someone? or prioritizing it for triage? 

A good objective either picks one of those precisely, or explicitly scopes in all of them as distinct outcomes. For example: *"Notify the incident responder of all their tenant scoped alerts"* is slightly better because it clearly defines the **scope** (all tenant-bound alerts) and the **outcome** (notification specifically to the incident responder) of the use case.

Even better is the following example: *"Notify the incident response team of all their prioritized tenant scoped alerts through alert scoring and prioritization"*. This example is great because it clearly states the **scope** (all prioritized tenant-bound alerts), the **outcome** (notification specifically to the incident response team), and the **dependencies** that are needed to meet the result (alert triage requires alert scoring and prioritization).

Here's the full objective I would write for an automation use case:

> Notify the suitable incident response team tier through alert assessment and prioritization of alerts originated by Google Threat Intelligence Digital Threat Monitoring (DTM).

Unfortunately, there's no single formula to write down the objective. However, you don't have to write it as a sentence either. You can simply list the aspects of your objective. For example:

- **Outcome**: Notification of all prioritized alerts to the triaged IR team
- **Scope**: All GTI DTM alerts
- **Dependencies**: Triage of DTM alerts requires assessing the alerts (two methods: Built-in DTM Alert Severity or IoC from Entity Enrichment).

> **Note**
> The list method allowed us to describe further the assessment methodology as well (built-in severity and IoC).

In summary, a well written objective must describe precisely one or many concrete **outcomes** that will lead you throughout the playbook design phase. A clearly defined **scope** that hints at the boundaries of your playbook logic. And finally, the **dependencies**, ideally explicit, that are needed to achieve your outcomes within your scope.

## 3. Categorization

> **What kind of automation is this?**

For consistency and sharing, I suggest categorizing playbooks as follows:

- **By structural role:** A *Workflow*, usually the entry point for an incident handling playbook or a complex automation use case (also receives the name of *Playbook*) or a *Subflow*, usually invoked by another playbook, chained or attached, with its own input/output contract (also receives the name of *Block*). Most SOAR platforms distinguish these two roles in some form at the object level, even where the exact terminology differs.
- **By functional goal:** Some examples include:
    - `enrichment` (attaching context before a human or another flow sees the alert)
    - `triage` (assessing and scoring severity)
    - `containment` or `response` (isolating a host, disabling an account, blocking an indicator)
    - `case-management` (creating tickets, notifying, escalating)
    - `catch-all` or `fallback` (a deliberately generic net for alert types that don't yet have a dedicated playbook, recognized as a distinct pattern in its own right rather than an afterthought).

You can use these keywords when categorizing your playbooks in the form of `{structural_role}:{main_functional_goal}`. E.g. `workflow:catch-all` or `subflow:enrichment`.

## 4. Automation Strategy

> **How is the objective going to be automated?**

Free text. A short plan of the mechanism, before it gets broken into implementation pieces.

Here you can describe a bit more precisely the scope and dependencies of your automation goal, without being too rigorous about the specific technical details of your SOAR platform. Why? This is the *"north star"* you (and others) will use when mapping the idea to an actual SOAR platform.

The following example is a strategy summary for a DTM CatchAll playbook example I built in Google SecOps SOAR.

> **Trigger** on any *Google Threat Intelligence* (vendor) *Digital Threat Monitoring (DTM)* (product) alert. **Score** the alert using DTM *"Alert Severity Definitions"*. **Prioritize** the alert using the calculated scores. **Triage** the case to the correct Incident Response SOC team depending on the final chosen priority. Finally, **Notify** the SOC team leveraging *Email* and, optionally, *Telegram* integrations.

When writing your strategy, use text formatting to your own advantage to make the strategy clearer. Moreover, use one sentence to describe a potential subflow within the workflow.

1. I used **bold text** to represent **verbs** that lead to **outcomes** in the automation. In the example above, I have the following verbs that define each stage of the workflow:
    - Trigger condition
    - Score
    - Prioritize
    - Triage
    - Notify

2. I used *Italics* to highlight *data sources* and *SOAR components*:
    - Google Threat Intelligence (data source vendor)
    - Digital Threat Monitoring (data source product)
    - Alert Severity Definitions (GTI's built-in alert severity)
    - Email Integration
    - Telegram Integration

3. Finally, each sentence describes a potential subflow:
    - *"Score the alert using DTM's Alert Severity Definitions"* is an alert scoring subflow.
    - *"Notify the SOC team leveraging EmailV2 and, optionally, Telegram integrations"* is a notification subflow that explicitly states that Telegram is optional (parameterizable to be enabled or not)

> **Tip**
> Begin by writing your *"Strategy Summary"*, as shown above. This is your base strategy, SOAR platform independent. 
> After the overall idea is set, you can write a more concrete version of that summary: A *"Technical Strategy"*. This is a more precise and rigorous version of your strategy that will deliberately include your SOAR platform vocabulary. But still not as detailed as in Section 6, Modular Technical Implementation. Similar to the standard *Strategy Summary* that guides you on ANY SOAR platform, you want the *Technical Summary* to guide you on your particular SOAR platform.
> The benefit of dividing your strategy in two layers is big! If you end up discovering that a technical claim you made was not supported by your platform, you can simply update that section in your *Technical Strategy* while keeping the base strategy intact! Also, your base strategy is portable between SOAR platforms.

## 5. Trigger and Execution Priority

> **What conditions have to occur for your workflow to run?**

State the trigger conditions. Usually, a workflow will run after receiving an alert under the following scopes:

- Trigger by *ANY* alert
- Trigger by a *VENDOR_SPECIFIC* alert
- Trigger by a *PRODUCT_SPECIFIC* alert
- Trigger by an *SPECIFIC_ALERT* 

If the workflow is unconditioned, your playbook MUST not depend on any original alert field.

If the workflow is conditioned on a particular vendor alert, you should include a sample of the standardized fields of the original alerts that can be originated by all products offered by that vendor. This is often too hard and time-consuming to do and often is too broad; hence, I'd suggest moving to the next trigger scope unless you have a very particular reason to remain vendor scoped.

If the workflow is conditioned on a particular product or alert, you should include an example of an original alert of that product.

Finally, trigger conditions should account for false-positive execution as well. You can reduce false-positives by checking unique identifying fields. E.g. `vendor_name`, `product_id`, or `alert_id` field. 

Workflows can also be prioritized. Usually, the more specific the playbook, the narrower is the scope.

- **Priority 1** — (`SPECIFIC_ALERT`) Only for deterministic automation or alert-specific incident handling. The most targeted match available for this exact condition. Anything is more specific than this playbook!
- **Priority 2** — (`PRODUCT_SPECIFIC`) This is for product-specific automation. Matches a whole product or alert family, broader than Priority 1 but not a last resort.
- **Priority 3** — (`VENDOR_SPECIFIC` or `ANY`) For fallback or catch-all logic. Fires only when nothing more specific claimed the alert.

You can write this section as shown below:

```md
**Trigger Conditions**
Explain briefly the trigger conditions here. Include a sample of the original alert or alerts that are relevant for your workflow.

**Execution Policy**
- **Priority**: State your playbook priority
- **Reason**: Why that priority?
```

## 6. Modular Technical Implementation

> **What's the skeleton of your automation workflow?**

This section is about breaking the final playbook into its overall compounding elements.

For each subflow, name it and state what it does. This is also where platform-specific details belong. For example, on Google SecOps SOAR, that means naming the actual integration actions and configurations each subflow will call.

> **Note**
> As you develop your playbook, you may determine that another subflow is needed, or is not needed but good to have. You can also include those extra subflows as long as they don't break your original strategy. You should document those *"extra"* blocks in the *Assumptions and Improvements* section as well.

You must document all input and output parameters of your subflows, if any. This is very important for the parameterization principle. Importantly, many SOAR platforms support outputting data in two main ways:
- **Execution output**: The subflow specifies the data model will output. You usually use this output when you need to feed another block or action downstream in the workflow logic.
- **Case or Alert database**: Most SOAR and DFIR platforms support adding context in case or alert scoped variables. In comparison to programming terms, these would be like global variables. You usually use this type of output when you want to store something accessible for the analyst as well.

> **Tip**
> A consistent naming convention makes a parameter's role obvious at a glance. As one example, not a requirement: the DTM CatchAll reference implementation prefixes call-site-configurable inputs with `param_` (e.g. `param_investigation_team`) and fixed, rarely-changed constants with `CONST_` (e.g. `CONST_CONTEXT_ALERT_SEVERITY`). Pick whatever scheme keeps intent legible in your own UCDDs.

> **Tip**
> I found that the *Strategy Summary* and *Technical Strategy* in the previous sections are enough (if written properly) to start working on your playbook.
> You can safely skip this section until you have a working version of your playbook. At that point, you walkthrough every one of your blocks to describe their functionality.
> You can also fill this section as you build the playbook, making sure you are aligned with what you described in your *Strategy Summary*. At the same time, work on **Section 7** to document any Assumption you made and Improvement you added while making your playbook.

Finally, for each action in your subflows, you MUST consider **error handling**.

**Which actions in this workflow and subflows are subject to failure, and what happens when they fail?**

List the actions that can realistically error, and what the playbook does about it. Throwing an error and halting the operation of the playbook is acceptable only if there's a good reason for it. 

For your reference, here is a list of recurring failure classes worth checking against every action: 
- `API_RATE_LIMIT`, when your integration action requires talking to a third-party tool and you reach an API rate limit or block.
- `MISSING_INTEGRATION`, when your platform lacks the integration instance needed to talk to a third-party platform.
- `MISCONFIGURED_INTEGRATION`, when your platform has the integration available but the configuration is incorrect.
- `MALFORMED_DATA`, when the data you expect from an alert or a previous integration action is not what you expected.
- `SYSTEM_ERROR`, when the SOAR platform or the third-party platform fails.

> **Note**
> A shared taxonomy of failure classes across use cases would make this section more consistent and is worth developing once enough UCDDs exist to generalize from.

## 7. Assumptions and Improvements

> **What did the Modular Technical Implementation take for granted?**

List them explicitly. An assumption that later turns out false is expected, but an assumption nobody wrote down is a design gap.

You can also list any added logic you included to optimize, improve, or enhance the main outcome. For example, baseline case enrichment is a typical flow that is added to every workflow that is not inherently necessary to achieve the main goal.

## 8. Simulation

> **What actual test runs have been done, and what did they show?**

Record every simulation run and every finding it produced, on whatever mechanism the platform provides for reproducible test execution. For example, Google SecOps SOAR's Playbook Simulator, run against a synthetic or real test case.

Reproducible test runs against a real or synthetic alert or case serve to: 
1. Prove the playbook does what Sections 1–6 claim it does, deterministically.
2. Identify hardcoded settings that can be parameterized.
3. Find issues that can feed back into earlier sections rather than living only as a test log.

When testing, make sure to save the synthetic alert or cases you used.

> **Warning**
> If using real cases and alerts for testing, make sure to always validate you are not sharing sensitive information. 

## 9. Resources

> **What did you use to build this, and where can someone verify it?**

Any reference used to design or implement this use case:

- Vendor documentation for the specific actions and platform features involved.
- Community guidance and posts.
- Related UCDDs this one depends on or was adapted from.

Cross-reference by section where useful, rather than leaving one undifferentiated list at the end.

## 10. Associated ADS

Palantir's [ADS-Framework](https://github.com/palantir/alerting-detection-strategy-framework/blob/master/ADS-Framework.md) has a **Response** section that can be linked from this file.

Together, a rule ADS and a workflow UCDD can fill the documentation gap and help you design end-to-end detection and incident response workflows!
