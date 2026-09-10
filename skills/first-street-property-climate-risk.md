---
name: Look up property-level climate risk
description: Resolve a street address or coordinate to a First Street Place, then read flood, wildfire, wind, heat, cold and drought risk for it — polling correctly, because the model runs on demand.
api: First Street Climate Risk API
endpoint: https://api.firststreet.org/v3/graphql
contract: graphql/first-street-climate-risk-api.graphql
mcp: https://mcp.firststreet.org/mcp
operations: [placeByAddress, placeByCoordinate, place, geospatial]
mcp_tools: [get_place_by_address, get_place_by_coordinate, get_place_by_id, find_buildings]
generated: '2026-09-10'
method: generated
source: https://docs.firststreet.org/api/climate-risk-api/getting-started
---

# Look up property-level climate risk

## Before you start

- You need an API key. There is no free tier, no test mode and no sandbox — every call
  below, including anything you run in the Playground, bills against your contract.
- Send the key as `Authorization: Bearer <key>` (preferred) or `?key=<key>`.
- **Access is per schema node.** Your contract may not include every peril. An unentitled
  peril returns HTTP 200 with that branch `null` and an `Error 15` entry in `errors[]`.
  That is an entitlement message, not a bug — do not retry it.

## Step 1 — resolve to a Place

Use whichever entry point matches your input:

| Input | GraphQL field | MCP tool |
|---|---|---|
| Street address | `placeByAddress(address, lat?, lng?)` | `get_place_by_address` |
| Coordinate | `placeByCoordinate(lat, lng)` | `get_place_by_coordinate` |
| Known Place ID | `place(placeId)` | `get_place_by_id` |
| GeoJSON polygon (≤ 9 km²) | `geospatial` | `find_buildings` |

Pass `lat`/`lng` alongside an ambiguous address to improve matching.

## Step 2 — ask for status AND data in the same query

This is the step people get wrong. First Street models each peril **on demand**, so the
first response is often a `PENDING`/`RUNNING` placeholder with a null `data` branch and an
HTTP 200. Always select `status { name }` next to `data`:

```graphql
query Place($placeId: String!) {
  place(placeId: $placeId) {
    placeId
    vintage { vintageId releaseDate }
    flood { status { name } data { floodFactor } }
    wind  { status { name } data { windFactor } }
  }
}
```

## Step 3 — poll until every peril is terminal

`status.name` is `FSModelResponseStatus`:

- `PENDING`, `RUNNING` → keep polling. The docs' own example uses a 2-second interval.
- `SUCCESS` → data is available. Note the peril may still be *Excluded* for that place.
- `FAILED`, `TIMEOUT`, `ERROR` → terminal. Stop. First Street is notified automatically;
  do not hammer the endpoint.

Gate every read on `status.name == SUCCESS` before you consume `data`. This is a change
from the old v2 US-domestic API, which was synchronous.

## Step 4 — pin the vintage if the answer has to be reproducible

An unpinned query always resolves to the **latest** vintage, so the same query returns
different numbers after a model release (Vintage 4 has been the default since 2026-07-01).
Only two vintages are retained.

```graphql
place(placeId: "dqbutwn70", options: { vintageId: 1 }) { vintage { vintageId releaseDate } }
```

Read back `place.vintage.vintageId` and store it with the result.

## Step 5 — override building characteristics when you know better

Damage modelling is driven by building attributes. Supply `BuildingInput` (as `building`
on any of the lookup fields, or the `building` object on the MCP tools) to override
defaults: `assetTypeId`, `buildingSf`, `stories`, `yearBuilt`, `foundationType`,
`foundationHeight`, `basement`, `constructionType`, `roofType`, `fireProofing`,
`defensibleSpace`, `windDesignStandard`, `rebuildCostPerSf`.

## Errors

- HTTP 422 with `extensions.code = GRAPHQL_VALIDATION_FAILED` — your selection does not
  match the schema and `data` is null. Validate against the SDL in `graphql/` first.
- HTTP 429 — read `x-ratelimit-reset` (seconds) and wait. There is no `Retry-After`.
  Default limit is 150 req/min.
- A wide selection can be rejected on **query complexity** (budget: 700) even when you are
  well inside the rate limit. Narrow the selection set, do not slow down.
- Every GraphQL error message carries `RequestID: <hex>` — quote it to api@firststreet.org.

## Rate and cost discipline

150 requests/minute by default, adjusted per contract. Cache by `placeId` + `vintageId`:
the same place at the same vintage does not change.
