# Microsoft Fabric Data Agent — Oracle Orders POC

A proof-of-concept deployment of a Microsoft Fabric Data Agent enabling 
natural language querying of Oracle ERP sales order data — allowing VMI 
analysts and inventory planners to query order status, holds, quantities, 
and fulfillment data without writing SQL.

Built on the `scut` schema shortcut views established during the 
[VMI Analytics Fabric Lakehouse Migration](https://github.com/Cruth98/VMI_Analytics_FabricMigration).

**AI-augmented development:** Agent instructions, data source descriptions, 
and example query library were developed using Claude as a productivity 
multiplier — accelerating configuration and documentation without 
sacrificing precision or data governance rigor.

---

## What This Is

Microsoft Fabric Data Agents are conversational AI interfaces that sit on 
top of structured data sources. Users ask questions in plain English — the 
agent interprets intent, generates SQL, queries the underlying views, and 
returns a formatted answer. No SQL knowledge required for end users.

This POC connects the agent to four Oracle order views via the VMI 
Analytics Lakehouse `scut` schema, with governed agent instructions 
controlling which views are used for which query types and how results 
are presented.

---

## Data Sources Connected

| View | Purpose | Safe for Aggregation |
|------|---------|---------------------|
| `vw_Orders_Base` | Primary source — one deduplicated row per order line, open and shipped | ✅ Yes |
| `vw_OpenOrdersWithNPHold` | Open orders on active Non-Pricing holds only | ❌ List/identify only |
| `vw_OpenOrdersWithHold` | Open orders with any active hold | ❌ List/identify only |
| `vw_Orders` | Adds City, State, pricing strategy fields — not deduplicated | ❌ Display only |

**Why `vw_Orders_Base` is the only aggregation-safe view:** A `ROW_NUMBER()` 
CTE deduplicates fulfill lines before any quantity columns are exposed — 
preventing inflated totals on rescheduled or amended orders. `vw_Orders` 
uses a raw LEFT JOIN on fulfill lines and will overcount quantities when 
multiple fulfill line records exist per order line.

---

## Agent Configuration

### Agent Instructions

The agent is governed by a custom instruction set written in Markdown 
per Microsoft Fabric Data Agent specification. Key rules encoded:

- `vw_Orders_Base` is the default source for all order, quantity, and 
  status queries
- `vw_OpenOrdersWithNPHold` is used exclusively for NP hold triage
- `vw_OpenOrdersWithHold` is used for broad hold triage when type is 
  unspecified
- `vw_Orders` may only be referenced for City, State, or pricing 
  strategy display — never for aggregation
- Charge lines (`SKU = 'PALLETSLIPCHG'`) are excluded from all quantity 
  analysis by default unless explicitly requested
- Open orders filter: `ShipDate IS NULL`
- Results returned as tables by default; narrative responses on request
- Agent will not speculate about data outside these views — no inventory 
  levels, forecasts, or replenishment recommendations
- Bailment / Control State queries redirected with explanation that 
  Oracle agent covers Open State only

### Data Source Descriptions

Each connected view has a structured description that tells the agent 
what the view contains, its grain, and when it should or should not 
be used — preventing the agent from selecting the wrong source for 
a given query type.

### Example Query Library (`examples.json`)

Eight pre-built example queries seeded into the agent covering the 
most common planning use cases:

| Query | Pattern |
|-------|---------|
| Open orders for a specific customer | `ShipDate IS NULL + CustomerName LIKE` |
| Past-due open orders | `ShipDate IS NULL + RequestedShipDate < today` |
| NP hold queue | `vw_OpenOrdersWithNPHold` |
| Ordered vs. shipped quantity by SKU | `SUM(OrderQty)` vs `SUM(FulfillLineShippedQTY)` |
| Orders due to ship this week | `RequestedShipDate BETWEEN Monday AND Friday` |
| Holds for a specific customer | `vw_OpenOrdersWithHold + CustomerName LIKE` |
| Open orders for a specific SKU | `SKU = exact + ShipDate IS NULL` |
| Open orders for a specific plant | `Plant = code + ShipDate IS NULL` |

---

## Key Terminology Defined in Agent

| Term | Definition |
|------|-----------|
| RSD | Requested Ship Date — when the customer asked for the order to ship |
| Open Order | Order line where `ShipDate IS NULL` — not yet shipped |
| OTIF | On Time In Full — shipped on time and in full quantity |
| NP Hold | Non-Pricing hold — logistics/fulfillment issue blocking shipment; highest-priority exception for planners |
| Short Ship | `FulfillLineShippedQTY` < `OrderQty` — customer received less than ordered |
| Charge Line | Non-product row (pallet/slip charges); `SKU = 'PALLETSLIPCHG'` — exclude from quantity analysis |
| Open State | Distributor markets managed through Oracle ERP |
| OrdersBaseKey | Composite join key: SKU + AccountNumber + Company + DeliveryAddressCode + OrganizationCode |

---

## Documentation

### End User Guide (`Sazerac_vw_Orders_Base_EndUser_Guide.pdf`)

Plain-English reference for non-technical planners. Covers what the 
dataset is, what questions it can answer, a field-by-field guide in 
plain English, common filters, and key terms. Designed to be distributed 
to planners who interact with the agent but have no SQL background.

### Technical Reference (`Sazerac_vw_Orders_Base_Technical_Reference.pdf`)

Full technical documentation for the underlying order views. Covers:

- `vw_Orders_Base` design decisions and underlying table list (12 Oracle 
  DOO extract tables)
- Complete column reference with types, nullability, and validation notes
- Related views comparison matrix
- Hold code logic — two-level capture at fulfill line and order line via 
  `STRING_AGG` CTE
- Recommended query patterns for common use cases
- Known risks and open items

---

## Repository Contents

```
DataAgent_POC/
├── agent_instructions.md          # Governed agent instruction set
├── data_source_descriptions.md    # Per-view descriptions for agent config
├── examples.json                  # 8 example queries seeded into agent
└── docs/
    ├── Sazerac_vw_Orders_Base_EndUser_Guide.pdf
    └── Sazerac_vw_Orders_Base_Technical_Reference.pdf
```

---

## Roadmap

This POC is the first phase of a broader agent strategy for the VMI 
and inventory planning team:

**Phase 1 — POC (Current)**
Build, configure, and internally validate the Orders agent against 
real Oracle data with a governed instruction set and example query library.

**Phase 2 — Testing & Publication**
Structured UAT with VMI analysts and planners. Validate response accuracy, 
edge case handling, and instruction set gaps before publishing to the 
broader team.

**Phase 3 — Microsoft Teams Integration**
Deploy the published agent into Teams so planners can query order data 
directly from their existing workflow without navigating to Fabric.

**Phase 4 — End User Adoption & Measurement**
Monitor adoption, query volume, and planner feedback. Measure reduction 
in ad hoc SQL requests and analyst time spent answering routine order 
status questions.

**Phase 5 — Domain Expansion**
Develop additional agents as new governed data sources are onboarded 
to the Lakehouse — inventory position, Atlas planning data, OOS analysis, 
and Bailment/Control State orders.

**Phase 6 — Agent Consolidation**
Build a unified multi-domain agent that routes questions across all 
domain-specific agents, giving planners a single conversational interface 
for the full breadth of VMI analytics data.

## Relationship to Fabric Migration

This POC is built on top of the shortcut views and Lakehouse architecture 
established in the VMI Analytics Fabric Lakehouse Migration. The 
`scut.vw_Orders_Base` view that powers this agent is the same validated 
view (2,632,879 rows) built and documented during that project.

The agent demonstrates the end-user analytics enablement layer that 
the Lakehouse migration makes possible — clean, governed data surfaces 
in a natural language interface without any additional ETL or modeling.
