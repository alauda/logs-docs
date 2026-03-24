# Documentation Cleanup Context

Last updated: 2026-03-24

## Purpose

This note records the documentation cleanup and review work completed on 2026-03-24 after syncing the local repository to the latest remote `main`.

## Work Completed

The work in this round focused on reviewing the documentation changes made on 2026-03-23, fixing the remaining issues found during review, and normalizing product naming across related pages.

Completed items:

- Reviewed the documentation updates from 2026-03-23 and checked for omissions, naming drift, and contradictory instructions.
- Fixed leftover product-name drift in logging guides, including replacing old short names such as `ACP Log Collector`.
- Fixed the circular prerequisite in the project cold-data retention guide.
- Cleaned up old spellings such as `ElasticSearch` and `Clickhouse` in user-facing documentation where they were not intended as literal YAML values.
- Updated older console-navigation wording to match the currently documented ACP English console path: `Marketplace` > `Cluster Plugins`.
- Updated one historical context note so its terminology matches the current doc set.

## Files Updated In This Round

User-facing docs:

- [docs/en/intro.mdx](/Users/mac/product-doc/logs-docs/docs/en/intro.mdx)
- [docs/en/architecture/architecture.mdx](/Users/mac/product-doc/logs-docs/docs/en/architecture/architecture.mdx)
- [docs/en/architecture/capacity_planning.mdx](/Users/mac/product-doc/logs-docs/docs/en/architecture/capacity_planning.mdx)
- [docs/en/architecture/component_suggestion.mdx](/Users/mac/product-doc/logs-docs/docs/en/architecture/component_suggestion.mdx)
- [docs/en/concept.mdx](/Users/mac/product-doc/logs-docs/docs/en/concept.mdx)
- [docs/en/functions/log.mdx](/Users/mac/product-doc/logs-docs/docs/en/functions/log.mdx)
- [docs/en/how_to/log_integration.mdx](/Users/mac/product-doc/logs-docs/docs/en/how_to/log_integration.mdx)
- [docs/en/how_to/project-cold-data-retention.mdx](/Users/mac/product-doc/logs-docs/docs/en/how_to/project-cold-data-retention.mdx)
- [docs/en/install_log.mdx](/Users/mac/product-doc/logs-docs/docs/en/install_log.mdx)

Context note:

- [notes/clickhouse-s3-doc-update-context-2026-03-23.md](/Users/mac/product-doc/logs-docs/notes/clickhouse-s3-doc-update-context-2026-03-23.md)

## Key Decisions

- Keep literal YAML enum values unchanged when they reflect the actual API shape. For example, `ElasticSearch` remains valid in YAML examples and field-reference tables where it is the documented enum value.
- Normalize user-facing prose to `Elasticsearch` and `ClickHouse`.
- Treat ACP English console labels as the source of truth for current UI wording when the task is about label alignment.

## Validation

- `yarn lint` completed successfully with 0 errors and 0 warnings.
- `git diff --check` completed successfully.
- `yarn build` did not complete because the local build crashed inside `sass-embedded-darwin-x64` / Dart Sass runtime with an `unreachable code` failure followed by `EPIPE`. This looked like a local runtime issue rather than an MDX syntax error introduced by this round.

## Residual Notes

- Historical context notes may intentionally retain examples of old wording when they describe what was searched for or cleaned up.
- YAML examples that document real field values may still use `ElasticSearch` if that matches the current API enum.
