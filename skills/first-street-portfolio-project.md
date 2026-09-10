---
name: Run a portfolio through the Enterprise API
description: Upload a portfolio of assets, validate the staged rows, commit them to a Project, refresh the analysis modules and export the result — the Enterprise API's full write flow, including which steps cannot be taken back.
api: First Street Enterprise API
endpoint: https://api.firststreet.org/enterprise/graphql
contract: graphql/first-street-enterprise-api.graphql
operations: [createProjectJobUploadLink, setUploadJobPendingImport, projectJob, createProject, importProjectAssets, addProjectAssetByPlaceID, refreshProjectModule, createProjectExport, generateProjectExportDownloadLink, cancelProjectExport, setProjectStatus, deleteProjectAsync]
generated: '2026-09-10'
method: generated
source: https://docs.firststreet.org/api/enterprise-api/workflow
---

# Run a portfolio through the Enterprise API

Everything that writes lives here. There is **no idempotency key** on this API — no
`Idempotency-Key` header, no request key, no documented replay window. A retried mutation
is a second mutation. Treat every call in this skill as at-most-once and confirm before
retrying.

## The shape of the flow

```
upload file → Job → STAGED assets → validate → commit to a Project → module data → export
```

## Step 1 — get an upload link

```graphql
mutation ($input: CreateProjectJobUploadLinkInput!) {
  createProjectJobUploadLink(input: $input) { projectUploadLink ... }
}
```

`createProjectJobUploadLink` returns a signed link and creates a **Job**. PUT your CSV or
XLS to that link. Headers must match First Street's expected schema exactly.

Data handling, from the docs: uploaded files are encrypted at rest and **automatically
deleted after 7 days**; assets are visible to your Organization and your Account Executive.

## Step 2 — poll the job

Query `projectJob` until `JobStatus` reaches a terminal state. This is the same
poll-until-terminal discipline as the Climate Risk API; there is no webhook and no
subscription (`APIs provided by First Street do not support subscriptions`).

## Step 3 — inspect the STAGED assets before committing

This is the only rehearsal step the API gives you, and it is the one place a mistake is
free. Read the staged rows, check the match rate and the building metadata, and only then
proceed. Nothing after this point has a dry-run.

## Step 4 — create the Project and commit

`createProject` first (name + description), then commit the `ProjectJobID` to it
(`importProjectAssets`). Committing kicks off `ProjectModuleData` generation.

To add or remove individual locations afterwards use `addProjectAssetByPlaceID` and
`deleteProjectAssetByPlaceID` — the Place ID is the join to the global climate model.

## Step 5 — refresh and read the modules

`refreshProjectModule` re-runs an analysis; poll until it is ready. The modules are
Overview, Climate Exposure, Scenario Analysis and Company Overview. Some carry `AIInsights`
with its own status enum (`GENERATING`, `GENERATED`, `FAILED`, `DIRTY`) — a `DIRTY`
insight is stale relative to the underlying data, not an error.

## Step 6 — export

`createProjectExport` → poll → `generateProjectExportDownloadLink`.
`cancelProjectExport(projectJobId)` cancels a running export; the docs state no cutoff, so
do not assume a cancel after completion is safe.

## Reversibility — read this before you delete anything

| Write | Can it be undone? |
|---|---|
| `setProjectStatus(ARCHIVED)` | Yes — set it back to `ACTIVE`. No time limit stated. |
| `createProjectExport` | `cancelProjectExport` while it runs. No window stated. |
| `importProjectAssets` | Per-asset only, via `deleteProjectAssetByPlaceID`. No bulk unwind. |
| move/copy portfolio + asset mutations | Structurally, by moving back. Not a declared reversal. |
| **`deleteProjectAsync`** | **No.** "Delete a project along with all of its associated resources." No restore exists. |
| **`deleteUserData`** | **No.** Irreversible by design. |

Prefer `setProjectStatus(ARCHIVED)` over deleting. Use `deleteProjectAsync`, not the
deprecated `deleteProject`.

## Deprecations to avoid on the way in

The peril `*Factor` / `factorScale` scalars are deprecated across every hazard in favour of
`factorScore.score` / `factorScore.scale`, and the matching `CLIMATE_EXPOSURE_*_FACTOR_*`
sort enums are deprecated with them. Write new code against `factorScore`. There is no
published sunset date — the `@deprecated` directive in the SDL is the only notice you get.

## Errors and limits

Same envelope as the rest of the surface: HTTP 200 with `errors[]` for schema-valid
requests, 422 for malformed queries, 429 on rate limit (150 req/min default,
`x-ratelimit-reset` in seconds, no `Retry-After`), and a 700-point query-complexity budget
that can reject a single wide selection.
