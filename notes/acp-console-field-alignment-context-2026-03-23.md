# ACP Console Field Alignment Context

Last updated: 2026-03-23

## Purpose

This note captures the working context for the ACP Logging documentation update that aligned console labels and YAML field names with the live ACP console, so future sessions can resume without repeating the same environment setup and verification steps.

Do not store or repeat account credentials in this file.

## Scope Of This Round

The user asked for three things:

1. Compare the live ACP console page with the repository docs.
2. Switch the console to English first, then use English UI labels as the source of truth.
3. Focus on whether field names are consistent, not on environment-specific values such as cluster names, versions, or currently installed settings.

Live ACP console area inspected in this round:

- `Marketplace` > `Cluster Plugins`
- ClickHouse storage plugin detail and install page
- Elasticsearch storage plugin detail and install page

## Environment Notes That Worked

The practical path in this environment was Safari WebDriver.

Working pattern:

1. Start `safaridriver -p 4444`.
2. Reuse the authenticated browser session.
3. Switch the ACP console language to English before collecting labels.
4. Use WebDriver `execute/sync` to run DOM queries and same-origin `fetch(...)` calls from inside the logged-in page.

This avoided separate kubeconfig or direct API-auth setup.

## Source-Of-Truth Rules Confirmed

Two different sources of truth were required:

- UI labels, menu names, button names, and install-form field names came from the live English ACP console pages.
- YAML field paths came from ACP API objects such as `ModuleConfig` and `ModuleInfo`, not from the visible form labels.

Important distinction:

- Console labels and YAML keys drift independently.
- A form field name matching the UI does not mean the YAML path in docs is correct.

## ACP Endpoints Proven Useful

These same-origin endpoints were successfully queried from the logged-in ACP page:

- `/apis/cluster.alauda.io/v1alpha1/moduleconfigs`
- `/apis/app.alauda.io/v1alpha2/modulepluginviews`
- `/apis/cluster.alauda.io/v1alpha1/moduleinfoes`
- `/kubernetes/<cluster>/api/v1/secrets?labelSelector=cpaas.io%2Fcredentials%3DS3`
- `/kubernetes/<cluster>/apis/storage.k8s.io/v1/storageclasses`
- `/kubernetes/<cluster>/api/v1/nodes?labelSelector=kubernetes.io%2Fos%3Dlinux`

These were enough to verify:

- plugin display names
- install-page labels
- `ModuleConfig.spec.config` field paths
- installed `ModuleInfo.spec.config` field paths

## Console Labels Confirmed In This Round

Observed English console path names:

- `Marketplace`
- `Cluster Plugins`

Observed plugin names:

- `Alauda Container Platform Log Collector`
- `Alauda Container Platform Log Storage for ClickHouse`
- `Alauda Container Platform Log Storage for Elasticsearch`

Observed ClickHouse install-page labels that mattered:

- `Size`
- `Storage Policy`
- `Storage Method`
- `Metadata Storage Type`
- `Log Node`
- `Storage Path on Host`
- `Metadata Storage Path`
- `Log Replicas`
- `StorageClass`
- `Metadata StorageClass`
- `Capacity`
- `Metadata Capacity`
- `S3 for Log Data`
- `Cold and Hot Separate`
- `S3 for Log Cold Data`
- `Log Platform`
- `Log Workload`
- `Log System`
- `Log Kubernetes`
- `Kubernetes Event`
- `Audit`

Observed Elasticsearch install-page labels that mattered:

- `External Elasticsearch`
- `size`
- `Storage Method`
- `Log Node`
- `Storage path on host`
- `Capacity`
- `Log data nodes resources`
- `Kafka Nodes`
- `LogPlatform`
- `LogWorkload`
- `LogSystem`
- `LogKubernetes`
- `Kubernetes Event`
- `Audit`

## YAML Field-Name Findings

### Elasticsearch storage plugin

The docs previously used `spec.config.clusterView.isPrivate` in the YAML example, but the live `ModuleConfig` structure used:

- `spec.config.self.storage.disabled`

Important field groups confirmed from `ModuleConfig.spec.config`:

- `spec.config.components.elasticsearch.*`
- `spec.config.components.kafka.*`
- `spec.config.components.kibana.install`
- `spec.config.components.storageClassConfig.*`
- `spec.config.components.zookeeper.storageSize`
- `spec.config.self.storage.disabled`
- `spec.config.ttl.*`

### ClickHouse storage plugin

Important field groups confirmed from `ModuleConfig.spec.config` and installed `ModuleInfo.spec.config`:

- `spec.config.components.clickhouse.storagePolicy.type`
- `spec.config.components.clickhouse.storagePolicy.defaultSpec.dataStorage.*`
- `spec.config.components.clickhouse.storagePolicy.defaultSpec.dataStorage.size`
- `spec.config.components.clickhouse.storagePolicy.sscSpec.metaStorage.*`
- `spec.config.components.clickhouse.tieredPolicy.*`
- `spec.config.components.clickhouse.ttl.*`
- `spec.config.components.clickhouse.resources.*`
- `spec.config.components.razor.resources.*`
- `spec.config.components.vector.resources.*`
- `spec.config.components.vector.storage.path`
- `spec.config.components.vector.storage.size`

### Log Collector plugin

Important field groups confirmed from `ModuleConfig.spec.config` and installed `ModuleInfo.spec.config`:

- `spec.config.crossClusterDependency.logcenter`
- `spec.config.crossClusterDependency.logclickhouse`
- `spec.config.dataSource.*`
- `spec.config.storage.type`

Important label keys confirmed:

- `metadata.labels.logcenter.plugins.cpaas.io/cluster`
- `metadata.labels.logclickhouse.plugins.cpaas.io/cluster`

Important enum spelling confirmed:

- `spec.config.storage.type: ElasticSearch`
- `spec.config.storage.type: ClickHouse`

## Repository Files Updated In This Round

Docs updated:

- [docs/en/install_log.mdx](/Users/mac/leizhu/Document/alauda_new/logs-docs/docs/en/install_log.mdx)
- [docs/en/how_to/clickhouse-s3-storage.mdx](/Users/mac/leizhu/Document/alauda_new/logs-docs/docs/en/how_to/clickhouse-s3-storage.mdx)
- [docs/en/how_to/project-cold-data-retention.mdx](/Users/mac/leizhu/Document/alauda_new/logs-docs/docs/en/how_to/project-cold-data-retention.mdx)
- [docs/en/functions/log.mdx](/Users/mac/leizhu/Document/alauda_new/logs-docs/docs/en/functions/log.mdx)

Skill updated:

- [skills/acp-logging-docs/SKILL.md](/Users/mac/leizhu/Document/alauda_new/logs-docs/skills/acp-logging-docs/SKILL.md)

## Skill Update Added

The project skill was updated with a dedicated ACP console verification section covering:

- English UI as source of truth
- Safari WebDriver workflow
- same-origin ACP API fetch pattern
- separation of UI-label verification and YAML-field verification
- logging-specific product naming reminders

## Temporary Files Created And Cleaned Up

Temporary inspection files created during this round were removed before closing the task:

- `.codex-acp-plugin-remoteEntry.mjs`
- `acp_plugin_main.js`
- `acp_plugin_remoteEntry.mjs`
- `codex_fs_probe`

## Validation Notes

- `git diff --check` passed for the documentation changes.
- No site build was run in this round.
- The final pass prioritized field-name alignment only, not environment-specific values.

## Suggested Resume Checklist

If continuing this topic later, start with:

1. Reopen the ACP console in English.
2. Re-check whether install-page labels changed again.
3. Re-check whether `ModuleConfig` field paths changed for `logcenter`, `logclickhouse`, or `logagent`.
4. Verify whether any other docs still use old wording such as `with Clickhouse`, `with ElasticSearch`, `Cluster Plugin`, or `App Store Management`.
5. Keep the distinction clear between live UI labels and YAML field paths.
