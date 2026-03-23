# ClickHouse S3 Doc Update Context

Last updated: 2026-03-23

## Purpose

This note captures the working context for the recent ACP Logging documentation update around ClickHouse S3 support, so future sessions can resume without re-discovering the source material and decisions.

## Source Material

Primary upstream source pages used in this round:

- Confluence PRD page: `https://confluence.alauda.cn/pages/viewpage.action?pageId=383681161`
- Confluence design page: `https://confluence.alauda.cn/pages/viewpage.action?pageId=383681183`

Do not store or repeat account credentials in this file.

## Product Scope Confirmed From Source

The source pages describe three related capability areas:

1. ClickHouse supports storing log table data in S3.
2. ClickHouse supports cold and hot separation, with hot data kept in active storage and cold data moved to S3.
3. Project-level retention policy evolves for ClickHouse, including separate hot-data and cold-data retention labels.

Important source constraints:

- S3 connection info must be provided by a Secret in `cpaas-system`.
- The Secret must include label `cpaas.io/credentials: S3`.
- The install flow currently supports one S3 target for log data and one S3 target for cold data.
- `Storage Policy` cannot be changed after deployment.
- `Cold and Hot Separation` cannot be toggled after deployment.
- Cold-data retention settings can be adjusted after deployment.
- This capability requires compatible versions of `Alauda Container Platform Log Essentials` and `Alauda Container Platform Log Storage with Clickhouse`.
- In disaster recovery scenarios, primary and standby ClickHouse deployments must not use the same S3 bucket.
- S3 secrets and access keys must not be exposed in plaintext documentation records.

## Local Repository Changes Made In This Round

New or updated pages tied to this work:

- [docs/en/install_log.mdx](/Users/mac/product-doc/logs-docs/docs/en/install_log.mdx)
- [docs/en/how_to/clickhouse-s3-storage.mdx](/Users/mac/product-doc/logs-docs/docs/en/how_to/clickhouse-s3-storage.mdx)
- [docs/en/how_to/project-cold-data-retention.mdx](/Users/mac/product-doc/logs-docs/docs/en/how_to/project-cold-data-retention.mdx)
- [docs/en/how_to/log_archive.mdx](/Users/mac/product-doc/logs-docs/docs/en/how_to/log_archive.mdx)
- [docs/en/architecture/component_suggestion.mdx](/Users/mac/product-doc/logs-docs/docs/en/architecture/component_suggestion.mdx)
- [docs/en/functions/log.mdx](/Users/mac/product-doc/logs-docs/docs/en/functions/log.mdx)
- [docs/en/concept.mdx](/Users/mac/product-doc/logs-docs/docs/en/concept.mdx)

Main content changes already completed:

- Added a dedicated S3 how-to page for ClickHouse.
- Added a dedicated how-to page for project-level cold-data retention.
- Expanded ClickHouse install docs to cover S3 Secret preparation, storage policy, cold/hot separation, and ModuleInfo YAML.
- Added an archive-page note to distinguish external log archive/export from ClickHouse native S3 storage.
- Updated component selection guidance to mention S3-backed and tiered ClickHouse storage scenarios.
- Updated the existing log retention guide to point to the new ClickHouse retention model and use the new project labels.
- Added concept-level explanations for `Storage Policy`, `Cold and Hot Separation`, and updated ClickHouse TTL wording.

## Important Documentation Decisions

### 1. Where S3 metadata-storage guidance should live

The metadata-storage recommendation should live mainly in the S3 how-to page, not in the concept page and not heavily embedded in field descriptions.

Final placement chosen:

- Main warning block in [docs/en/how_to/clickhouse-s3-storage.mdx](/Users/mac/product-doc/logs-docs/docs/en/how_to/clickhouse-s3-storage.mdx)
- Keep [docs/en/install_log.mdx](/Users/mac/product-doc/logs-docs/docs/en/install_log.mdx) field descriptions lighter

Reason:

- This guidance is deployment advice, not a core product concept.
- Putting it directly into field descriptions makes the install table too heavy.
- A standalone warning in the how-to page is easier to notice and easier to maintain.

### 2. How to phrase the metadata-storage performance requirement

Desired message from product side:

- When ClickHouse uses S3 for log data, metadata storage should preferably use local storage.
- If local metadata storage cannot be used and `StorageClass` must be used, the storage backend should have read/write performance greater than `200 MiB/s`.

Final wording choice:

- Keep `LocalVolume` as the recommended option.
- If `StorageClass` must be used, call `>200 MiB/s` an "engineering recommendation".
- Do not claim that `>200 MiB/s` is a ClickHouse official hard requirement.

Reason:

- ClickHouse official material supports the general direction that object storage introduces latency and that local SSD / local cache remain important.
- No official ClickHouse source was found that defines `>200 MiB/s` as a universal requirement for metadata storage in this scenario.

## External Reference Notes Used For The Wording

Relevant official references checked during this round:

- ClickHouse blog: `https://clickhouse.com/blog/building-a-distributed-cache-for-s3`
- ClickHouse release notes / blog for 24.10: `https://clickhouse.com/blog/clickhouse-release-24-10`

What these references support:

- Remote object storage has higher latency characteristics.
- Local filesystem cache and local storage remain important in object-storage-backed deployments.
- Table metadata and related local-access patterns still matter for performance.

What they do not explicitly provide:

- A ClickHouse-official numeric threshold such as `200 MiB/s` for metadata storage.

## Current Recommended Warning Text Direction

Current guidance in the S3 how-to is intentionally framed like this:

- Metadata performance still matters even when log data is stored in S3.
- Prefer `LocalVolume` for ClickHouse metadata storage.
- If metadata must use a `StorageClass`, choose a high-performance backend.
- `>200 MiB/s` is an engineering recommendation to reduce metadata-related bottlenecks.

## Current Project Label Guidance

For new ClickHouse retention-related configuration, prefer:

- `cpaas.io/project.hotDataRetentionDays`
- `cpaas.io/project.coldDataRetentionDays`
- `cpaas.io/project.logPolicyEnabled`

Legacy labels may still appear in ACP 4.2 compatibility scenarios:

- `cpaas.io/project.esIndicesKeepDays`
- `cpaas.io/project.esPolicyEnabled`

Do not use the legacy labels as the recommended path in new examples.

## Validation Notes

`yarn lint` was run after the documentation updates and passed.

Important environment note:

- In a restricted sandbox, `yarn lint` may falsely fail because `@alauda/doom/cspell` fetches remote terms and the fetch can be blocked.
- Outside the sandbox, the same `yarn lint` command completed successfully.

## Suggested Resume Checklist For Future Sessions

If continuing this topic later, start by checking:

1. Whether the upstream Confluence pages changed again.
2. Whether `docs/en/functions/log.mdx` still matches the current product UI behavior for project retention policy management.
3. Whether additional pages need cross-links to the ClickHouse S3 how-to.
4. Whether product wants the `>200 MiB/s` recommendation refined, softened, or moved.
5. Whether the Chinese source-of-truth docs also need parallel updates outside this repo.
