# Repo Map

## Build and toolchain

- Package manager: `yarn@4.9.2`
- Site builder: `@alauda/doom`
- Main scripts:
  - `yarn dev`
  - `yarn lint`
  - `yarn build`
  - `yarn serve`
  - `yarn translate`
- Site config: `doom.config.yml`
  - content root: `docs`
  - default lang: `en`
  - shared API assets are loaded from `docs/shared/openapis/**/*`

## Content layout

- `docs/en/index.mdx`
  - top-level navigation overview page
- `docs/en/intro.mdx`
  - module introduction
- `docs/en/install_log.mdx`
  - installation planning and installation procedures
- `docs/en/concept.mdx`
  - concepts, terminology, and flow diagrams
- `docs/en/architecture/*.mdx`
  - architecture, component selection, capacity planning
- `docs/en/functions/*.mdx`
  - product-facing guides such as logs, audit, events
- `docs/en/how_to/*.mdx`
  - operational how-to pages and runbooks
- `docs/en/apis/**/*.mdx`
  - API reference landing pages and wrapper pages
- `docs/en/assets/*`
  - page-local images
- `docs/public/logo.svg`
  - public site asset

## Shared source assets

- `docs/shared/openapis/logging.alauda.io.v2.json`
  - shared schema for log search, aggregation, context, and archive endpoints
- `docs/shared/openapis/events.alauda.io.v1.json`
  - shared schema for event query endpoints
- `docs/shared/roletemplates/roletemplates.yaml`
  - permission data consumed by the site

## Common page patterns

### Frontmatter

Common keys already in use:

- `weight`
- `sourceSHA`
- `i18n.title`

Use the minimum set already established by neighboring pages.

### Index pages

Patterns in use:

- `<Overview />`
- `<Overview overviewHeaders={[]} />`

Mirror sibling pages instead of standardizing globally during unrelated work.

### API wrapper pages

Typical shape:

```mdx
---
weight: 10
i18n:
  title:
    en: Search
---

# Search

<OpenAPIPath path="/platform/logging.alauda.io/v2/logs/search"/>
```

These pages are usually thin wrappers over shared OpenAPI specs.

## Content conventions inferred from the repo

- The repo is English-first today: `docs/i18n.json` is empty and there is only `docs/en`.
- Many narrative pages include `sourceSHA`, suggesting sync from another source. Preserve that metadata unless the task is explicit about resyncing.
- `sourceSHA` should be treated as tool-managed metadata, not hand-edited content.
- Console procedures use bold UI labels and numbered steps.
- Operational docs often include full YAML examples and should keep commands copyable.
- New doc filenames and directories should stay in `kebab-case`.
- Private page assets should live in a nearby `assets/` directory and be linked with relative paths.
- Terminology is product-specific and should stay stable across pages:
  - ACP Logging
  - Elasticsearch / ClickHouse storage backends
  - Filebeat, Kafka, Razor, Lanaya, Vector, nevermore, Courier

## Practical editing heuristics

- For prose-only fixes, edit the MDX page and avoid touching generated-looking shared assets.
- For endpoint coverage changes, inspect both the wrapper page and the backing OpenAPI JSON before editing.
- For new docs, place them in the existing section whose neighboring pages match the user intent instead of creating a new top-level bucket casually.
- If changing page order within a section, adjust `weight` conservatively and check siblings first.
- Directory-level `index.mdx` pages should follow the local `Overview` pattern.
- Ask before changing `doom.config.yml`, `sites.yaml`, `docs/shared/**`, or `theme/**`.
