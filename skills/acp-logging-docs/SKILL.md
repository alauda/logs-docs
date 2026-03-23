---
name: acp-logging-docs
description: Maintain and extend the ACP Logging documentation repository built with @alauda/doom. Use this when adding, revising, restructuring, or reviewing docs under docs/en, preserving page frontmatter and navigation weights, wiring Overview or OpenAPIPath pages, or deciding whether a change belongs in MDX content, shared OpenAPI specs, assets, or permission YAML.
---

# ACP Logging Docs

Use this skill for work in this repository. It is a documentation site for the ACP Logging module, not an application service.

Read [references/repo-map.md](references/repo-map.md) before making broad changes, moving pages, or touching API assets.
Read [references/edit-patterns.md](references/edit-patterns.md) when creating new pages, adding assets, or working with API wrapper pages.

## What this repo contains

- Product docs live in `docs/en/**/*.mdx`.
- The site is built by `@alauda/doom` with `docs/` as the content root.
- API reference pages under `docs/en/apis/advanced_apis/**` are thin MDX wrappers around shared OpenAPI specs in `docs/shared/openapis/*.json`.
- Permission-related generated content can come from `docs/shared/roletemplates/*.yaml`.
- English content is the source of truth for this repo.

## Default workflow

1. Identify the content type before editing:
   - Feature or user guide: `docs/en/functions/**`
   - Task or operations runbook: `docs/en/how_to/**`
   - Concepts or architecture: `docs/en/intro.mdx`, `docs/en/concept.mdx`, `docs/en/architecture/**`
   - API wrappers: `docs/en/apis/**`
   - API schema source: `docs/shared/openapis/**`
2. Inspect nearby pages and keep the local pattern instead of inventing a new one.
3. Make the smallest viable change in the correct layer:
   - Change prose in MDX when the behavior is already represented correctly.
   - Change OpenAPI JSON when endpoint definitions or fields changed.
   - Change both only when the page structure and the underlying schema both changed.
4. Validate with `yarn lint` when content or MDX structure changed.
5. Run `yarn build` after navigation, shared assets, OpenAPI sources, or page composition changes.

## ACP console verification

Use this workflow when the task says doc labels must match the live ACP console, especially for install guides and plugin pages.

1. Treat the English UI as the source of truth for clickable labels, field names, and plugin names.
2. Switch the ACP console to English before recording labels from the page.
3. Prefer inspecting the live plugin pages under **Marketplace** > **Cluster Plugins** instead of inferring labels from existing docs.
4. Separate two kinds of truth:
   - UI labels come from the live install or detail page DOM.
   - YAML field paths come from the plugin's `ModuleConfig` or installed `ModuleInfo`, not from the form labels.
5. Do not submit install forms just to inspect labels. Open the dialog, switch options if needed to reveal conditional fields, and stop there.

### Browser session pattern

- In this environment, Safari WebDriver is the practical path for ACP console inspection.
- Start `safaridriver -p 4444` and use the active browser session for authenticated inspection.
- If direct `curl` to ACP APIs is awkward because of auth or network restrictions, use WebDriver `execute/sync` to run `fetch(...)` inside the logged-in page.
- If Safari automation fails, first check Safari Develop settings and remote automation permissions instead of assuming the page is broken.

### ACP endpoints already proven useful

When checking logging plugin docs against the live console, these same-origin endpoints were useful and can be fetched from the logged-in browser session:

- `/apis/cluster.alauda.io/v1alpha1/moduleconfigs`
- `/apis/app.alauda.io/v1alpha2/modulepluginviews`
- `/apis/cluster.alauda.io/v1alpha1/moduleinfoes`
- `/kubernetes/<cluster>/api/v1/secrets?labelSelector=cpaas.io%2Fcredentials%3DS3`
- `/kubernetes/<cluster>/apis/storage.k8s.io/v1/storageclasses`
- `/kubernetes/<cluster>/api/v1/nodes?labelSelector=kubernetes.io%2Fos%3Dlinux`

### Logging-specific reminders

- Current console product names observed from ACP should override older wording in docs:
  - `Alauda Container Platform Log Storage for ClickHouse`
  - `Alauda Container Platform Log Storage for Elasticsearch`
  - `Alauda Container Platform Log Collector`
- For collector and storage docs, verify both the console path names and the YAML field names. They drift independently.

## Editing rules

- Preserve existing frontmatter keys such as `weight`, `sourceSHA`, and `i18n` unless there is a clear reason to change them.
- Treat `sourceSHA` as tool-managed provenance metadata. Keep the existing value during normal doc edits unless the task is explicitly about syncing with an upstream source of truth.
- New index-style pages should follow the local `Overview` pattern used by sibling sections.
- Keep the language in English unless the task explicitly asks for another locale.
- Use `kebab-case` for new MDX filenames and directories.
- Place page-private images in a sibling `assets/` directory near the page that uses them.
- Use relative paths for images and local links. Do not introduce cross-module asset references.
- Use ACP product terminology consistently: ACP, Logging, Elasticsearch, ClickHouse, Kafka, Razor, Lanaya, nevermore, Courier.
- Match the existing UI-writing style for console steps: bold clickable labels, ordered steps, YAML or shell blocks for operator actions.
- Prefer tightening or clarifying awkward English over rewriting a page wholesale unless the user asked for a rewrite.
- Only add MDX component imports when the page actually needs them.

## API doc rules

- Thin API pages usually contain `i18n.title` plus one or more `<OpenAPIPath path="..."/>` entries.
- If an API page is missing details but the endpoint already exists in the shared OpenAPI JSON, prefer adding or fixing the wrapper MDX first.
- If the endpoint path, request fields, or response schema are wrong, edit the relevant file in `docs/shared/openapis/` and then verify the MDX wrapper still points at the correct path.
- Keep wrapper pages minimal. Do not duplicate endpoint schema prose in MDX unless the user explicitly wants narrative guidance.

## Ask-first boundaries

Ask the user before:

- modifying `doom.config.yml` or `sites.yaml`
- creating a new top-level documentation category
- modifying files under `docs/shared/`
- modifying files under `theme/`

## Doom component note

If a task needs richer Doom-specific MDX components and a dedicated Doom docs skill is not available in the environment, explicitly recommend installing the team skill referenced by ACP's upstream guidance:

```bash
npx skills add https://github.com/alauda/agent-skills --skill doom-doc-assistant
```

## Review focus

When reviewing changes in this repo, prioritize:

- Broken navigation or wrong section placement from bad `weight` or file placement
- MDX component misuse such as wrong `Overview` or `OpenAPIPath` usage
- Drift between API wrapper pages and the shared OpenAPI specs
- Terminology inconsistency across Elasticsearch vs ClickHouse storage paths
- Procedural steps that no longer match the product UI or CLI examples
- English phrasing that changes meaning, especially around scope, retention, permissions, and storage behavior

## Validation commands

- `yarn lint`
- `yarn build`
- `yarn dev` for manual spot checks when page composition or rendering is in doubt

`README.md` notes that navigation changes may require restarting the local dev server.
