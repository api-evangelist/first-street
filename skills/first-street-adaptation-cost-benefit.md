---
name: Price climate adaptations for a property
description: Get First Street's recommended mitigation measures for a Place — flood barriers, floodproofing, defensible space, roof and wind upgrades — with setup cost, annual cost, loss reduction and payback period under a chosen emissions scenario.
api: First Street Climate Risk API
endpoint: https://api.firststreet.org/v3/graphql
contract: graphql/first-street-climate-risk-api.graphql
mcp: https://mcp.firststreet.org/mcp
operations: [adaptations, place]
mcp_tools: [get_adaptations, get_place_by_id]
generated: '2026-09-10'
method: generated
source: https://docs.firststreet.org/api/mcp/getting-started
---

# Price climate adaptations for a property

This is the flow that turns a risk score into a decision: what can be done about it, what
does it cost, and how long until it pays back.

## Inputs you must decide first

| Input | Why it matters |
|---|---|
| `placeId` | The property. Get it from the property-lookup skill. |
| `holdPeriod` (years) | Payback is computed against how long you will hold the asset. |
| `ssp` | The emissions scenario. Changing it changes the loss avoided, and therefore the payback. |

`ssp` is one of:

- `SSP_1_26` — optimistic; low emissions, sustainable development
- `SSP_2_45` — moderate; middle-of-the-road
- `SSP_5_85` — pessimistic; high emissions, fossil-fuel driven

**Never pick the scenario for the user.** Run at least `SSP_2_45` and `SSP_5_85` and
present the spread — a measure that pays back under one and not the other is the finding.

## Call it

GraphQL:

```graphql
query Adaptations($placeId: String!, $holdPeriod: Int!, $ssp: SSP!) {
  adaptations(placeId: $placeId, input: { holdPeriod: $holdPeriod, ssp: $ssp }) {
    possibleAdaptations { type name peril annualCost setupCost paybackYears { ... } aal { ... } }
    recommendedAdaptations { ... }
    averageAnnualCashLossCumulative { ... }
  }
}
```

MCP: `get_adaptations` with `{ placeId, holdPeriod, ssp }` — same three inputs.

You can also reach the same data through `place(placeId).adaptations(input:
{ holdPeriod, ssp, relativeYear })` when you already have the Place selection open.

## Read the result honestly

- `possibleAdaptations` is the menu; `recommendedAdaptations` is First Street's shortlist.
  Report both — the menu shows what was considered and rejected.
- `setupCost` is one-off, `annualCost` is recurring. A measure with a low setup cost and a
  high annual cost can lose to the opposite over a long hold period.
- `aal` is average annual loss **with the measure applied** — the saving is the delta
  against the unmitigated AAL, not the number itself.
- `paybackYears` is scenario-dependent and hold-period-dependent. Quote the scenario every
  time you quote a payback.
- Measure families you will see: flood barriers (1–8 ft variants), dry and wet
  floodproofing, raised foundation, retaining walls, drainage improvements, elevation of
  critical equipment, wind design standard upgrades, fire-proofing and defensible space.

## What not to do

Do not present a payback period as a guarantee, and do not aggregate adaptation costs
across a portfolio from this endpoint — portfolio-level aggregation is the Enterprise API's
job (see the portfolio skill).
