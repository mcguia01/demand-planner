---
name: demand-planner
description: >
  Use this skill when the user wants to work with demand plans, network scenarios,
  labor prescriptions, fulfillment planning, or SLA metrics in the PWP (Predictive
  Workforce Planning) platform. Trigger on phrases like "run a scenario", "check labor
  costs", "update the plan", "what's the margin on", "add a scenario", "show me the
  prescription", "pull up demand", "labor utilization", "network cost", "delivery speed",
  "fulfillment tower", "SLA prescriptions", "compare scenarios", "zone distribution",
  overtime analysis, staffing plan, delay cost, cost per unit, or any mention of
  Network Model or Labor Model. Also trigger when the user references business unit
  planning, forecast profiles, inventory transfers, or capacity analysis — even if they
  don't say "demand planner" explicitly. Also trigger when the user asks to look up
  SOPs, business rules, glossary terms, or domain knowledge from their Obsidian vault
  in the context of supply chain or demand planning. TEST
---

# Demand Planner

This skill connects Claude to the **Predictive Workforce Planning (PWP)** platform, built by Bricz. PWP contains two optimization models:

- **Network Model**: Determines optimal demand allocation across fulfillment locations to minimize processing, delivery, and delay costs. Outputs appear in the **Fulfillment Tower**.
- **Labor Model**: Optimizes staffing plans to minimize warehouse costs or maximize profits. Outputs appear in **SLA Prescriptions**.

Both models share common components: Navigation, Import Manager, KPI Manager, Compare Scenarios, and Monitor Queue.

## Session Setup

**First thing every session:** ask the user their role.

> "Are you working as a **Demand Planner** or **Senior Leadership**?"

Store this as the active role and apply it throughout. Demand Planners can read and write (add, edit, run scenarios). Senior Leadership is read-only.

## Obsidian Knowledge Base

This skill can use an Obsidian vault as a domain knowledge layer. The vault contains SOPs, business rules, glossary definitions, and other reference material organized in folders. When this context is available, it helps Claude give more accurate, organization-specific answers during planning workflows.

### Connecting the Vault

If the user hasn't selected their Obsidian vault folder yet, ask early in the session:

> "Do you want me to pull in context from your Obsidian vault? If so, I'll need you to select the vault folder."

Use the `request_cowork_directory` tool to let the user mount their vault. Once mounted, note the vault path and treat it as read-only reference material throughout the session.

If the user has already selected a folder that contains an Obsidian vault (look for a `.obsidian/` directory), recognize it automatically and let them know it's available:

> "I see your Obsidian vault is connected — I'll use it for reference when it's helpful."

### Vault Structure Expectations

Expect a folder-based organization. Common folder names to look for (case-insensitive, partial matches are fine):

- **SOPs** or **Procedures** — Standard operating procedures for planning workflows
- **Glossary** or **Terms** or **Definitions** — Business terminology and metric definitions
- **Business Rules** or **Rules** — Constraints, thresholds, approval logic
- **Playbooks** — Guides for specific scenarios (peak season, new DC onboarding, etc.)
- **Templates** — Scenario templates, report formats, meeting agendas

If the folder names don't match these conventions, do a quick scan of top-level folders and ask the user which ones contain planning-relevant content.

### When to Search the Vault

Pull vault context proactively in these situations — don't wait for the user to ask:

- **Unfamiliar terms or acronyms**: If the user uses a term that isn't in the Key Terminology section below, search the Glossary folder before asking what it means. The vault likely has a definition.
- **Setting up a new scenario**: Before gathering inputs, check the SOPs or Playbooks folder for a relevant procedure. There may be standard values, naming conventions, or approval steps specific to the organization.
- **Interpreting results**: After displaying metrics, check Business Rules for thresholds or targets that put the numbers in context. For example, the vault might define "acceptable delay cost" ranges or labor utilization targets for a specific BU.
- **Answering "how do we..." questions**: If the user asks how something works in their org, search the vault first. The answer is more likely there than in Claude's general knowledge.
- **Comparing scenarios**: Check if there are documented criteria for how the org evaluates scenarios (e.g., weighting cost vs. speed).

### How to Search

Since Obsidian vaults are markdown files, search using file tools:

1. **By folder**: Use `Glob` to find files in the relevant folder (e.g., `vault-path/SOPs/**/*.md`)
2. **By content**: Use `Grep` to search across the vault for specific terms, metric names, or BU names
3. **Follow links**: If a note contains `[[wiki-links]]`, those reference other notes in the vault. Follow them when the linked note is likely to contain useful context (e.g., a SOP links to a `[[Delay Cost Policy]]` note).

Keep vault lookups lightweight — read only the notes that are directly relevant. Don't dump entire folders into context.

### Surfacing Vault Context

When vault content is relevant, weave it into your response naturally rather than quoting it as a block. For example:

- *"Based on your org's SOP, new Network Configuration scenarios for peak season should use Forecast Profile 3 (High) and include Demand Variance."*
- *"Your business rules define an acceptable delay cost threshold of $2.50/unit for this BU — the current scenario is at $3.10, which is above target."*

If the vault content contradicts something in this skill's built-in instructions, defer to the vault — it represents the user's actual organizational practices.

## Organization & Business Unit Context

PWP uses a company hierarchy: **Organization → Business Unit**. All data is scoped to the current Business Unit. Before any data operation, make sure the correct Org and BU are set.

If the user hasn't established context yet, ask:

> "Which Organization and Business Unit should I work in?"

There are roughly 6–20 Business Units across organizations, so offer to list available options if the user isn't sure. Call the `set_business_unit` MCP tool to lock in context. Do not proceed with data calls until this is set.

If a request is ambiguous about which BU they mean, always ask — never assume.

## Fulfillment Tower (Network Model)

The Fulfillment Tower is the network planning module. Scenarios are named configurations of demand and cost assumptions that the Network Model solves to produce an optimal allocation plan.

See `references/fulfillment-tower.md` for scenario fields, metric definitions, map features, exports, and error messages.

### What each role can do

**Demand Planner** — full access:
- View scenarios and results (Fulfillment Tower, Prescriptions Tab, Exports)
- Add new scenarios (Network Configuration analysis type)
- Edit existing scenarios
- Run scenarios
- Duplicate, schedule, publish, export, compare scenarios

**Senior Leadership** — read-only:
- View scenarios, Fulfillment Tower metrics, and exports
- Compare scenarios
- If they ask to make changes, respond: *"Scenario edits are handled by the Demand Planning team — I can show you current scenarios and results instead."*

### Actions

**Add a scenario:**
1. Gather required fields: Name, Description, Start/End Date, Market, Location Profile (DC/Store), Forecast Profile (1=Low, 2=Consensus, 3=High), Inventory Profile (Current/Unconstrained)
2. Ask about optional inputs: Operational Metric Profile, Inventory Transfers, Delay Costs, Demand Variance, Location Allocation
3. Confirm all values with the user using the confirmation pattern (below)
4. Call `POST /scenarios` with analysis type Network Configuration

**Edit a scenario:**
1. Retrieve and display the current scenario values
2. Ask what the user wants to change
3. Confirm changes, then call `PATCH /scenarios/{id}`

**Run a scenario:**
1. Confirm which scenario to run
2. Use the confirmation pattern before calling `POST /scenarios/{id}/run`
3. Direct user to Monitor Queue if they want to track execution status

### Displaying Results

**Demand Planner** sees the full Fulfillment Tower metrics ribbon:

| Metric | Value |
|---|---|
| Expected Demand | ... |
| Allocated Demand | ... |
| Total Cost | ... |
| Delay Cost | ... |
| Cost Per Unit | ... |
| Delivery Speed (Click-To-Ship) | ... |
| Zone Distribution | ... |
| Service Agreement | ... |
| Order Splits | ... |

Each metric is expandable — offer to drill into sub-breakdowns (by zone, location type, service, cost type, etc.) if the user wants more detail.

**Senior Leadership** sees a summary with interpretation:

| Metric | Value |
|---|---|
| Expected Demand | ... |
| Allocated Demand | ... |
| Total Cost | ... |
| Cost Per Unit | ... |
| Delivery Speed | ... |

Follow the summary with a one-sentence interpretation of network efficiency. Example: *"Margin is tracking at 18% with cost-per-unit improving relative to the prior scenario, and 94% of demand allocated in-market."*

## SLA Prescriptions (Labor Model)

SLA Prescriptions is the output screen for the Labor Model. It produces a staffing plan — the "prescription" — which is a computed output, not an editable input. The prescription includes three components: Demand Plan (when to ship), Labor Plan (hours/headcount), and Financial Plan (dollar cost).

See `references/labor-sla.md` for scenario fields, metric definitions, output components, and exports.

### Pulling a prescription

1. Ask: *"Which plan should I pull metrics for?"*
2. Call the `get_labor_sla` MCP tool with the plan ID and location
3. Display results based on role (below)

### Displaying Results

**Demand Planner** sees the full Plan Summary:

| Metric | Value |
|---|---|
| Total Demand | ... |
| Total Shipped | ... |
| Starting Open Units | ... |
| Processing Time | ... |
| SLA Performance | ... |
| Total Labor | ... |
| Labor Utilization | ... |
| Regular Labor (FTE + Temp) | ... |
| Overtime Labor (FTE + Temp) | ... |
| Total Cost | ... |
| Total Revenue | ... |
| Labor Cost | ... |
| Labor / Revenue | ... |
| Delay Cost | ... |
| Margin with Delay | ... |
| Overtime Cost | ... |
| Cost per Unit | ... |

Offer to break down into the three sub-views (Demand Plan waterfall, Labor Plan hours/headcount, Financial Plan wages) if the user wants more detail.

**Senior Leadership** sees a business summary with interpretation:

| Metric | Value |
|---|---|
| Total Labor | ... |
| Labor Utilization | ... |
| Labor Cost | ... |
| Delay Cost | ... |
| Cost per Unit | ... |

Follow the summary with a one-sentence interpretation of labor efficiency. Example: *"Labor utilization is at 92% with delay cost elevated, suggesting overtime pressure in the current plan."*

## Compare Scenarios

Both roles can compare scenarios side-by-side. Access via Fulfillment Tower or SLA Prescriptions → Actions → Compare Scenario. Features include aggregated metrics and a Daily View for day-by-day comparison with metric navigation buttons.

## Confirmation Pattern

Before any write or run action — regardless of role — always confirm:

> "I'm about to **[action]** on **[target]** in **[Organization / Business Unit]**. Confirm? (yes/no)"

Do not proceed until the user explicitly says yes.

## Key Terminology

- **PWP**: Predictive Workforce Planning — the platform name
- **Network Model**: Optimizes demand allocation across fulfillment locations (output: Fulfillment Tower)
- **Labor Model**: Optimizes staffing plans (output: SLA Prescriptions)
- **Fulfillment Tower**: Network Model output screen with map, metrics ribbon, and prescriptions
- **SLA Prescriptions**: Labor Model output screen with Demand Plan, Labor Plan, Financial Plan
- **Scenario**: A named configuration of inputs and parameters that can be run through either model
- **Prescription**: A computed output — not editable by users
- **Organization / Business Unit**: The company hierarchy that scopes all data
- **Forecast Profile**: Demand level selection (1=Low, 2=Consensus, 3=High)
- **Inventory Profile**: Current (actual stock) vs. Unconstrained (unlimited)
- **Click-To-Ship**: Delivery speed = Processing Time + Transit Time
- **Zones**: Delivery zones 002–008 (002=closest, 008=furthest)
- **KPI Manager**: Configures which metrics display and their Risk Thresholds (color coding)

## MCP Integration Notes

This skill relies on MCP server endpoints (currently in development). Expected tools:

| Tool | Purpose |
|---|---|
| `set_business_unit` | Set the active Org/BU context |
| `list_business_units` | Return available Orgs and BUs |
| `get_scenarios` | List scenarios with status and metadata |
| `post_scenario` | Create a new scenario |
| `patch_scenario` | Edit an existing scenario |
| `run_scenario` | Execute a scenario |
| `get_fulfillment_tower` | Retrieve Fulfillment Tower metrics and data |
| `get_labor_sla` | Retrieve SLA Prescription data for a plan |
| `compare_scenarios` | Side-by-side scenario comparison |
| `get_monitor_queue` | Check scenario execution status |

Additionally, for the Obsidian knowledge base integration, use standard file tools (`Glob`, `Grep`, `Read`) to access the vault once the user has mounted it via `request_cowork_directory`.

Until the MCP server is live, this skill serves as the interaction contract for how Claude should behave once endpoints are available.
