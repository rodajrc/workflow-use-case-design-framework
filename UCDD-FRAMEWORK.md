# Use Case Design Document (UCDD) — Template

A UCDD is written before a workflow is built, and stays alive after it ships. Each section answers one question a designer must not skip. Where a section is genuinely open or unresolved for this use case, say so explicitly rather than leaving it blank.

One UCDD per automation use case. A use case may end up implemented as any of the following combinations:
- A single standalone workflow (playbook); usually a short automation flow or a deterministic, single-purpose use case.
- A single standalone subflow (block); usually a general-use component meant to be reused by other use cases.
- A main workflow (playbook), plus the several subflows (blocks) it is made of; usually a comprehensive use case that resolves a full incident-response gap.

**This is UCDD Version 1.2**

## Document Structure

### Who is this document for?

The structure of this document has been designed to account for multiple consumer roles; namely, **SOC Analysts** that review alerts affected by automation workflows, **Integration Engineers** that care about enabling an automation workflow quickly and effectively in their organizational environment, and **Automation Engineers** deeply intrigued about the technical implementation details of an automation workflow.

### Sections

1. Workflow Statement
2. Workflow Objective
3. Configuration and Deployment
4. Outcomes and Analysis
5. Workflow Category
6. Trigger Conditions
7. Automation Strategy
8. Workflow and Subflows
9. Assumptions
10. Improvements
11. Workflow Simulation and Testing
12. Resources

### Short guide for reading and writing a UCDD

When **reading** the framework, you're expected to follow along in the order as is. It's built so a reader can stop once their question is answered. For an Integration Engineer and a SOC Analyst, the first five sections tell what the workflow does, how it's configured, what it produces, and what kind of workflow it is. An automation engineer keeps reading into the design and implementation detail that follows to gather more technical insight about the playbook.

Conversely, **Writing order** doesn't have to match reading order, and usually won't. A workable draft sequence could begin by settling the Objective and Category first, then deciding the Outcomes and Analysis you're aiming for (deciding what the workflow should produce before deciding how it produces it is equivalent to Test-Driven Development, paired with workflow simulation and tests). Configuration can be drafted here too, if the use case's parameters are already clear at this point. Next, you would draft the Trigger Conditions, since the original alert schema they surface is what the Automation Strategy that follows can then draw on, followed by the remaining sections in order: Workflow and Subflows, Assumptions, and Improvements, potentially in parallel while the workflow is actually being built, checked continually against the Objective and Category so the build doesn't drift from its own stated goal. Finally, Workflow Tests and Resources accumulate throughout, as things get tested and referenced. Once the workflow is finished, close with a cleanup pass on the first five sections so they read as the finished summary they're meant to be. The Workflow Statement in particular is usually only accurate once written last — it's your executive summary. This isn't a rule, though — you can write the document however is easiest for you.

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
ucdd_version: 1.2
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

Recommended only when the document is not tracked in git or another version-control system. When it is, the repository history already records every change to the file, and a hand-kept table duplicates that job; a short note saying so, in place of the table, is enough. Without version control, log every non-trivial change to this document or to the playbook it describes, including continuous improvement recorded after go-live. You can place this section at the end of the document as well.

| **Version** | **Date** | **Author** | **Summary of changes** |
|---|---|---|---|
| 0.0 | *Today's date* | *Your name* | *Initial draft* |

Suggested versioning procedure:

<!-- TODO: versioning procedure under revision -->

- **Minor versioning** logs changes to individual automated actions or workflow settings.
- **Major versioning** logs entire subflows or big improvements that heavily change the automation logic.

You can add **tags** in the *summary of changes* column. For example: `config` for config changes, `new_subflow` when creating a new subflow, `improvement` when updating and improving a subflow, etc.

---

## 1. Workflow Statement

> **How the named subflows compose into the automation described in [Automation Strategy](#7-automation-strategy).**

A short paragraph describing the automation workflow in plain terms, abstracted away from technical detail. This is the *Executive Summary* of the workflow.

Here is an example of a Workflow Statement:

> *The workflow* triggers on any *{Threat-Intelligence feed or platform name}* alert that carries a non-empty `monitor_id` and `monitor_name` field on its original alert. On execution, **`Generic Case Initialization`** runs a generic case setup, followed by **`Product-Specific Alert Case Initialization`** that runs source-specific case enrichment. Next, **`Product-Specific Alert Score by Severity`** reads the alert's native severity and writes a weighted score into the case's context. Next, **`Generic Alert Prioritization`** reads that score and sets the case's SOAR-platform-native *Alert Priority*. **`Generic Alert Triage`** reads the resulting priority to either auto-close a benign alert or assign the case to the appropriate tier and move it to the Investigation or Incident case's SOAR-platform-native stage. Finally, **`Generic Alert Notification`** reads the Alert's assigned priority against a configurable gate that evaluates the alert severity and analyst queue and, if it clears the gate, notifies the configured contacts by email and, optionally, Telegram.

> **Note**
> This *Workflow Statement* was adapted from a [worked example](/examples/dtm-catchall/dtm-catchall.md).

## 2. Workflow Objective

> **The (security) challenge this automation solves.**

State the goal in plain language, precise about the verbs, covering three things: one or more concrete **outcomes** the automation delivers, the **scope** that bounds it, and the **dependencies**, ideally explicit, needed to achieve those outcomes within that scope.

### How to write a clear automation workflow objective?

**Vague goals produce vague playbooks!**

For example: *"Handle all alerts"* is a bad objective, not because *"all alerts"* is incorrect (that's a *fallback* workflow) but because the word *"handle"* hinders what you really mean: Is it closing the alert automatically? or communicating it to someone? or prioritizing it for triage? 

A good objective either picks one of those **"outcomes"** precisely, or explicitly takes in all of them as distinct outcomes. For example: *"Triage and notify all alerts"* is slightly better because it clearly defines two outcomes, which are triaging and communicating the alert. 

You can improve further by defining boundaries: *"Notify all prioritized alerts to the optimal Incident Response Team about leaked credentials and data exposure in the dark web"* is better because it frames clearly who is going to be notified about which alerts under what conditions. We call this property the **scope** of the objective. Notice that this new objective now describes *Notification* as the *main outcome*, while *Triage* has become an explicit *scope* condition referencing the *Incident Response Team*. 

We can improve further by including explicit **dependencies**. *"Notify all prioritized alerts from an external threat-intelligence feed to the optimal Incident Response Team about leaked credentials and data exposure in the dark web by assessing the incident through alert scoring by native severity and risk entities"*. This is a great example because it clearly states the **dependencies** that are needed to achieve the objective within scope.

Here's the objective laid out in full:

> Notify all prioritized alerts from an external threat-intelligence feed to the optimal Incident Response Team about leaked credentials and data exposure in the dark web by assessing the incident through alert scoring by native severity and risk entities

You can also describe the same goal by listing its properties, as shown below:

> - **Outcome**: Notification and Triage of all prioritized threat-intel-feed alerts to the optimal IR Team
> - **Scope**: Threat-intel-feed alerts that are prioritized through the alert scoring process by assessing the original alert severity and entity risk score
> - **Dependencies**: The external threat-intelligence feed, and Alert Scoring from Alert Severity and Entity Risk Scores (Verdict and Severity)

## 3. Configuration and Deployment

> **What must be configured or deployed before this workflow can run in a tenant.**

Document everything an Integration Engineer needs to enable this workflow in a new environment. Usually, these are the third-party integrations the workflow depends on, and the parameters it exposes for an organization to tune without editing the workflow or subflow logic itself (Parameterization).

**Integrations** are all third-party tools or services this workflow connects to. You want to be explicit about the integrations that are required for the playbook to run and the integrations that are optional.

**Configurable Parameters** are the variables an integration engineer is expected to tune for their own environment, in plain language, the way you'd document a function's arguments in a comment. E.g., what the parameter controls, and why someone would change it. This is deliberately lighter than the input and output details in [Workflow and Subflows](#8-workflow-and-subflows), which very precisely describes a workflow and subflow's technical data contract for whoever is maintaining the workflow. This, in contrast, documents what a non-technical integrator is actually free to change.

> **Note**
> Every platform exposes configurability differently: dedicated playbook input parameters, environment variables, global data tables, or values hardcoded into an action that the organization is expected to edit directly. Document whichever mechanism your platform gives you; what matters is capturing every knob an organization can turn, not which primitive it uses.

You can write this section as shown below:

```md
**Integrations**
- `<integration name>`: Description
- `Example Integration Name`: (OPTIONAL) This integration adds bonus functionality. Your playbook still runs fine without it, just with less magic.

**Configurable Parameters**
- `<parameter name>` [DATA TYPE]: What it controls, and why you might change it. Default: `<value>`.
- `parameter_example_name` [INT(0,1)]: Enable or disable this magic parameter. Default: 1
```

## 4. Outcomes and Analysis

> **What this workflow produces for the analyst, and what decisions still need a human.**

Document everything a SOC Analyst needs to know when they open a case or alert this workflow has affected: what analysis has already been done on their behalf, what artifacts the workflow leaves behind, and which decisions still require a human.

**I. Automated Outcomes**

List every artifact the workflow produces without human input: case comments, tags, insights, scores written to the shared case or alert context, and any notification sent. You want to be precise about these effects so the analyst can quickly adjust their process and use them effectively. You don't want an analyst trying to decipher what your playbook did to their case or alert.

> **Tip**
> Add an automation action that includes the UCDD in the alert or case context, so the analyst can find it quickly.

**II. Human-in-the-Loop Actions**

List every decision or manual step the workflow leaves for the analyst: a pending action awaiting approval, a manual escalation step, or a judgment call the automation deliberately doesn't make because it requires that *"human touch"*. State what triggers it and what the analyst is expected to do.

**III. Platform-Native Enhancements**

Some SOAR platforms let you configure a richer, purpose-built interface for the analyst to review your alert. Document whether one exists for this use case and what it's meant to help the analyst see faster.

You can write this section as shown below:

```md
**Automated Outcomes**
- `<artifact>`: What it tells the analyst, and where they'll find it.

**Human-in-the-Loop Actions** (OPTIONAL)
- `<action>`: What triggers it, and what the analyst is expected to do.

**Platform-Native Enhancements** (OPTIONAL)
- `<enhancement name>`: What it's meant to help the analyst see faster, if configured.
```

> **Note**
> This section should read more like a briefing than a spec. An analyst opening this UCDD wants *"here's what's already been done for you, and here's what you still need to decide."*

## 5. Workflow Category

> **The category of this automation workflow.**

For consistency and sharing, I suggest categorizing playbooks as follows:

- **By structural role:** A *Workflow* is usually the entry point for an incident handling playbook or a complex automation use case, also called a *Playbook*. A *Subflow* is usually invoked by a workflow, chained or attached, optionally with its own input/output contract. It is also called a *Block*. Most SOAR platforms distinguish these two roles in some form at the object level, even where the exact terminology differs.
- **By functional goal:** Some examples include:
    - `enrichment` (attaching context before a human or another flow sees the alert)
    - `triage` (assessing and scoring severity)
    - `containment` or `response` (isolating a host, disabling an account, blocking an indicator)
    - `case-management` (creating tickets, notifying, escalating)
    - `catch-all` or `fallback` (a deliberately generic net for alert types that don't yet have a dedicated playbook, recognized as a distinct pattern in its own right rather than an afterthought).

You can use these keywords when categorizing your playbooks in the form of `{structural_role}:{main_functional_goal}`. E.g. `workflow:catch-all` or `subflow:enrichment`.

## 6. Trigger Conditions

> **The conditions that have to occur for the workflow to run.**

A workflow will run in any of the following general scenarios:

- Trigger by *ANYTHING*
- Trigger by *VENDOR*
- Trigger by *PRODUCT* from Vendor
- Trigger by *ALERT* from Product
- Trigger by *EVENT* from Alert

Each level has its own drawbacks and characteristics:

- Playbooks that trigger on *ANYTHING* **MUST NOT** depend on any original alert field. It is usually referred to as the *Fallback Playbook*.
- Playbooks that trigger on any alert of a specific *VENDOR* **MUST** include a sample of the fields shared by every product that vendor offers. This is usually too broad to be practical.
- Playbooks that trigger on any alert of a specific *PRODUCT* **MUST** include a sample of the fields shared by that specific product. This is usually broad enough to be practical for *Catch All Playbooks*.
- **ALERT** or **EVENT** specific playbooks often trigger by the identifier or name of the original alert or event. These are usually specific enough to be practical for *Incident Response Playbooks*.

Whatever the level below *ANYTHING*, check for unique identifying fields (e.g. `vendor_name`, `product_id`, `alert_id`, `event_id`) to guard against false-positive execution.

Some SOAR platforms also allow for workflow prioritization. As a rule of thumb, the more specific the playbook is, the higher its priority.

- **Priority 1** — (`EVENT`) Only for deterministic automation or event-specific incident handling. The most targeted match available for this exact condition. Anything is more specific than this playbook!
- **Priority 2** — (`ALERT`) Matches one specific alert type. Broader than event-based trigger playbooks, but still narrow enough to be deterministic.
- **Priority 3** — (`PRODUCT`) Matches every alert type a given product can raise. Suitable for Catch All logic.
- **Priority 4** — (`VENDOR`) Matches every alert any product from a given vendor can raise. Rarely practical.
- **Priority 5** — (`ANYTHING`) For Fallback logic. Fires only when nothing more specific claimed the alert.

You can write this section as shown below:

```md
**Trigger Conditions**
Explain briefly the trigger conditions here. Include a sample of the original alert or alerts that are relevant for your workflow.

**Execution Policy**
- **Priority**: State your playbook priority
- **Reason**: Why that priority?
```

## 7. Automation Strategy

> **A high-level plan for reaching the automation workflow objective.**

Write a short plan of the mechanism in free text, before it gets broken down into implementation pieces. 

You begin by describing, in plain language, how the objective is going to be automated, along with the scope and dependencies of the goal, without being rigorous about the specific technical details of your SOAR platform. This is the *"north star"* you (and others) will use when mapping the idea to an actual SOAR platform. 

You want to keep this section deliberately as vendor neutral as possible, to make it portable. The Automation Strategy **MUST** read the same regardless of which SOAR platform that eventually implements it, even if the technical details are slightly different between instances.

Here is an example:

> **Trigger** on any *{threat-intelligence feed}* (vendor) *{alert type}* (product) alert. **Score** the alert using the feed's native severity definitions. **Prioritize** the alert using the calculated scores. **Triage** the case to the correct Incident Response SOC team depending on the final chosen priority. Finally, **Notify** the SOC team leveraging *Email* and, optionally, *Telegram* integrations.

Note that mentioning the vendor and product of the alert that the playbook consumes, or the explicit Telegram integration mention, **is not** a contradiction to the portability claim. Moreover, when writing your strategy, use text formatting to your own advantage to make it clearer:

1. Use **bold text** for the **verbs** that lead to **effects** and **outcomes**. In the example above, these are: Trigger, Score, Prioritize, Triage, Notify.
2. Use *Italics* to highlight *data sources* and *SOAR components*, such as the feed vendor, the feed product, its severity definitions, and the integration names used.
3. Each sentence describes a potential subflow. *"Score the alert using the feed's native severity"* is an alert-scoring subflow; *"Notify the SOC team by email and, optionally, Telegram"* is a notification subflow that explicitly states Telegram is optional (parameterizable to be enabled or not).

### Optional Technical Strategy

After the General Strategy is set, you can write a more concrete version called *Technical Strategy*, which deliberately includes your SOAR platform's vocabulary, like the actual in-platform integration names, automated action names, and vendor-specific mechanics, while still not as detailed as the block-by-block breakdown in [Workflow and Subflows](#8-workflow-and-subflows).

> **Note**
> You can see [a worked example](/examples/dtm-catchall/dtm-catchall.md) for a Technical Strategy written out for a real SOAR platform.

> **Note**
> The benefit of dividing your strategy is that the *General Strategy* guides any implementation, while the *Technical Strategy* guides your implementation.

## 8. Workflow and Subflows

> **The skeleton of the automation workflow.**

This section breaks the final playbook into its constituent elements. This section is essential for automation engineers who need to understand the logic of your automation, in case they need to adjust it to adapt it to their environment.

**I. Subflow Breakdown** 

For each subflow, name it and state what it does. You want to include platform-specific detail here, like the integration actions and configurations each subflow calls.

The description does not have to reproduce the playbook's logic verbatim. A concise narration of what the subflow does, in what order, and what ends it is enough, as long as the other sections of this document (the strategy, the parameters, the outcomes, the error handling) carry the rest. The playbook itself remains the reference for the exact wiring; a document that mirrors every condition and branch is expensive to keep in step with it, and a concise one is easier to maintain.

**II. Input and Output Documentation** 

You must document all input and output parameters of your subflows for effective parameterization.

Most SOAR platforms support feeding and returning data in two main ways:
- **Execution output**: The subflow specifies the data model it will receive and return. You usually use this when you need to feed another block or action downstream in the workflow logic.
- **Case or Alert database**: SOAR and DFIR platforms often support adding context in case- or alert-scoped variables. In programming terms, these are like global variables. You usually use this type of output when you want to store something accessible for the analyst as well.

> **Tip**
> A consistent naming convention makes a parameter's role obvious at a glance. See [a worked example](/examples/dtm-catchall/dtm-catchall.md) for this convention applied throughout a real UCDD.

**III. Error Handling**

You **MUST** consider error handling for each action in your subflows. *What actions are subject to failure and what happens when they do?* List the actions that can realistically error, and what the playbook does about it. Accepting the risk, throwing an error, and halting the playbook are all acceptable responses, given a good reason.

For your reference, here is a list of recurring failure classes worth checking against every action: 
- `API_RATE_LIMIT`, when your integration action requires talking to a third-party tool and you reach an API rate limit or block.
- `MISSING_INTEGRATION`, when your platform lacks the integration instance needed to talk to a third-party platform.
- `MISCONFIGURED_INTEGRATION`, when your platform has the integration available but the configuration is incorrect.
- `MALFORMED_DATA`, when the data you expect from an alert or a previous integration action is not what you expected.
- `SYSTEM_ERROR`, when the SOAR platform or the third-party platform fails.

There are more things you may want to include, such as brief details about the **Integrations** used in the automation and the specific **Actions** you used and why. There's also a distinction between **Outcomes**; the planned results of your workflow, and the **Effects**, which are side-effects or secondary results of your actions and automation workflows as a whole.

> **Note**
> Usually, **Effects** are a consequence of poor vendor documentation about their integrations and actions.

## 9. Assumptions

> **The decisions, actions, and logic you took for granted when designing the workflow and subflows.**

List assumptions taken during playbook development explicitly. For example, expecting a specific data input format in your subflows, believing in the stability of third-party integrations and APIs, or trusting automated assessments and judgments made from evidence collected through enrichment.

Here are some examples of assumptions you can document:

- Assume that *all original alerts from a product have the same schema*. Even if true today, it may only hold for the current version of that third-party product. A later release may update the alert schema inadvertently.
- Assume that *adding a new firewall rule is enough to isolate a host for effective containment*. Not all hosts are weighted equally; some may run business-critical services your organization may not want isolated that way. Moreover, just adding a firewall rule may not be enough to isolate a host safely.

## 10. Improvements

> **Added logic that optimizes, improves, or enhances the workflow's outcome beyond what the objective strictly requires.**

List any added logic you included to optimize, improve, or enhance the main outcome. For example, adding extra functionality to a subflow, including error-handling for risky actions, or optimizing workflows with short-circuiting logic.

Here are some examples of improvements you can document:

- A *baseline case enrichment* subflow typically chained at the beginning of every workflow. It extends the outcome by granting the analyst information about related cases.
- A *customized case or alert view*, a feature often provided by SOAR and DFIR platforms, that allows automation engineers to build panels and dashboards from source code such as HTML or a proprietary language. It helps analysts by presenting the information they need in a clear, purpose-built interface.

## 11. Workflow Simulation and Testing

> **Tests and their results.**

Record every simulation run and every finding it produced, on whatever mechanism the platform provides for reproducible test execution. For the purposes of the UCDD, keep a record of the latest test that ran successfully end-to-end and the conditions under which it ran, though you can still keep detailed logs of every test while developing including those of individual subflows.

> **Tip**
> Save the alert or cases you used for testing for accountability and reproducibility.

> **Warning**
> If using real alerts and cases, redact first! Make sure you are not sharing sensitive information. 

## 12. Resources

> **Information used to inform the design of the use case automation workflow.**

Any reference used to design or implement this use case:

- Vendor documentation for the specific actions and platform features involved.
- Community guidance and posts.
- Related UCDDs this one depends on (most likely subflow UCDDs).

### Associated ADS

> **Optional sub-section for cross-referencing an Alerting Detection Strategy document.**

Palantir's [ADS-Framework](https://github.com/palantir/alerting-detection-strategy-framework/blob/master/ADS-Framework.md) has a **Response** section that can be linked from this file.

Together, a rule ADS and a workflow UCDD can fill the documentation gap and help you design end-to-end detection and incident response workflows!
