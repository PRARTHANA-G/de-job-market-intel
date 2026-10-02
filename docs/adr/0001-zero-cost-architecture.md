# ADR-0001: Run at zero cost on Databricks Free Edition

- Status: Accepted
- Date: 2026-10-02

## Context

The project budget is zero, and the pipeline has to run daily for months to show trends.

- Azure free account: USD 200 credit valid for 30 days. After that the subscription is
  disabled unless it is upgraded to pay-as-you-go. Not viable as the long-running platform.
- Databricks Free Edition: free, but serverless compute only, outbound internet limited to
  an unpublished set of trusted domains, no custom workspace storage locations,
  non-commercial use only, daily fair-usage quotas.

## Decision

1. Storage: Unity Catalog volume for bronze (raw API responses as files) and Delta tables
   for silver and gold, all inside Databricks Free Edition.
2. Ingestion runs outside Databricks, in GitHub Actions, and uploads the raw response bytes
   through the Databricks Files API. This avoids depending on Databricks outbound access.
3. Terraform provisions Databricks objects with the `databricks` provider and is applied
   for real. An Azure module (resource group, storage account, Key Vault) is kept in the
   repo and checked with `terraform fmt` and `terraform validate` in CI, but not applied
   by default.
4. Secrets: GitHub encrypted secrets in CI, a gitignored `.env` locally. No secret is ever
   stored in the repository.

## Consequences

Positive
- Zero cost for as long as Free Edition exists.
- Same concepts as enterprise Azure Databricks: medallion layers, Delta, Unity Catalog,
  serverless Spark.

Negative
- No live ADLS or Key Vault in the running system. The Azure module shows IaC design but
  is not exercised daily.
- No SLA and daily quotas. The pipeline must tolerate a skipped day. Bronze is
  append-only, so a missed day shows up as a visible gap rather than corrupted data.
- Inactive Free Edition accounts can be deleted. The daily run keeps the account active.

## Open questions (spike before the Terraform commit)

- Can a GitHub-hosted runner upload to a Unity Catalog volume with a token?
- Can schemas and volumes be created in the default catalog? Creating new catalogs may be
  restricted.
- Are secret scopes available in Free Edition?
