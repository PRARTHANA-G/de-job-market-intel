# ADR-0001: Run at zero cost on Databricks Free Edition

- Status: Accepted
- Date: 2026-10-02 (updated 2026-10-03 with spike results)

## Context

The project budget is zero, and the pipeline has to run daily for months to show trends.

- Azure free account: USD 200 credit valid for 30 days. After that the subscription is
  disabled unless it is upgraded to pay-as-you-go. Not viable as the long-running platform.
- Databricks Free Edition: free, but serverless compute only, outbound internet limited to
  an unpublished set of trusted domains, no custom workspace storage locations,
  non-commercial use only, daily fair-usage quotas.

## Decision

1. Storage: everything lives in the `workspace` catalog, split by schema:
   `workspace.bronze` (a managed volume holding raw API responses as files),
   `workspace.silver` and `workspace.gold` (Delta tables).
2. Ingestion runs outside Databricks, in GitHub Actions, and uploads the raw response bytes
   through the Databricks Files API. This avoids depending on Databricks outbound access.
3. Terraform provisions the schemas and volume with the `databricks` provider and is
   applied for real. An Azure module (resource group, storage account, Key Vault) is kept
   in the repo and checked with `terraform fmt` and `terraform validate` in CI, but not
   applied by default.
4. Secrets: a Databricks personal access token stored as a GitHub encrypted secret in CI,
   OAuth login (`databricks auth login`) locally. No secret is ever stored in the repo.

## Spike results (2026-10-03)

| Test | Result |
|---|---|
| Create a catalog via API | Fails: needs a storage location; Free Edition only allows it in the UI |
| Create schema and managed volume via API | Works |
| Upload a file to a volume from outside Databricks | Works (`databricks fs cp`) |
| Secret scopes | Available |
| Personal access tokens | Allowed |
| Region | AWS us-east-2, chosen by Free Edition |

Still to confirm: upload from a GitHub-hosted runner (same public API, verified when CI
is added).

## Consequences

Positive
- Zero cost for as long as Free Edition exists.
- Same concepts as enterprise Azure Databricks: medallion layers, Delta, Unity Catalog,
  serverless Spark.
- Everything is created by Terraform, nothing by hand.

Negative
- No live ADLS or Key Vault in the running system. The Azure module shows IaC design but
  is not exercised daily.
- Data is stored in the US region Free Edition assigns. Acceptable for public job
  postings; a production setup would use an EU region for GDPR.
- No SLA and daily quotas. The pipeline must tolerate a skipped day. Bronze is
  append-only, so a missed day shows up as a visible gap rather than corrupted data.
- Inactive Free Edition accounts can be deleted. The daily run keeps the account active.
