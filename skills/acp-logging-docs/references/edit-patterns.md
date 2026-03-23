# Edit Patterns

## Use This Reference For

- adding a new page in an existing section
- creating or updating an `index.mdx`
- adding page-local images
- editing API wrapper pages
- making final checks before handing work back

## New Narrative Page Pattern

Prefer matching nearby pages, but this is the default shape for a new narrative page:

```mdx
---
weight: 20
---

# Page Title

Short intro paragraph.

## Section

Content.
```

Rules:

- Reuse the frontmatter keys already used by sibling pages.
- Add `sourceSHA` only when the page already participates in a source-sync workflow.
- Keep one H1 per page.
- Choose `weight` by checking sibling pages first.

## Section Index Pattern

Directory-level `index.mdx` pages should stay lightweight and use the local `Overview` pattern:

```mdx
---
weight: 50
---

# Guides

<Overview />
```

In some sections the repo uses:

```mdx
<Overview overviewHeaders={[]} />
```

Mirror the local section instead of normalizing unrelated pages.

## Asset Placement Pattern

When a page needs a private diagram or screenshot:

1. Put the file in a sibling `assets/` directory near that page or section.
2. Reference it with a relative path.
3. Do not link to assets from another module's directory.

Examples:

- `docs/en/assets/log.png`
- `![](../assets/log.png)`

If a page lives deeper in the tree, compute the relative path from that page rather than moving the image to a shared location casually.

## API Wrapper Page Pattern

Advanced API pages are intentionally thin:

```mdx
---
weight: 10
i18n:
  title:
    en: Search
---

# Search

<OpenAPIPath path="/platform/logging.alauda.io/v2/logs/search"/>
<OpenAPIPath path="/platform/logging.alauda.io/v2/clusters/{cluster}/logs/search"/>
```

Use this pattern when:

- the endpoint already exists in `docs/shared/openapis/*.json`
- the page only needs to expose one or more paths

Do not add long prose to these pages unless the user explicitly asks for narrative API guidance.

## Shared OpenAPI Change Heuristic

Before editing `docs/shared/openapis/**`, confirm the task truly requires schema-level changes.

Good reasons:

- endpoint path changed
- request or response fields changed
- operation summaries or descriptions are wrong in generated API output

Weak reasons:

- only the section title on the MDX wrapper is awkward
- only page placement or ordering needs adjustment

`docs/shared/**` is an ask-first boundary in this repo.

## Writing Style Pattern

Prefer the style already used in this repository:

- concise overview paragraph before steps when useful
- ordered steps for UI and operational procedures
- bold UI labels for clickable console items
- fenced `yaml` or `bash` blocks for operator actions
- stable product terminology over stylistic variation

When editing English:

- fix awkward phrasing, but do not silently change product meaning
- preserve scope words such as cluster, project, namespace, retention, archive, audit, and event
- keep commands and YAML copyable

## Done Checklist

- correct section and file placement
- `weight` checked against siblings
- relative links and asset paths verified
- `sourceSHA` preserved unless explicitly resyncing
- `yarn lint` run for doc or MDX changes
- `yarn build` run for navigation, assets, API schema, or page composition changes
- mention that `yarn dev` may need restart if sidebar or navigation changed
