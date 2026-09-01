# Algolia search configuration

Search on openchoreo.dev is Algolia DocSearch (app `B8ST9KVWVJ`, index `openchoreo`),
wired up in `docusaurus.config.ts` under `themeConfig.docsearch`. The index is built by
the Algolia Crawler, and its **settings live in the Algolia dashboard, not in this repo**.

`algolia-index-settings.json` is the version-controlled record of what those settings are
supposed to be, so the config is reviewable and restorable instead of existing only in a UI.

## Applying / restoring

The dashboard keeps no settings history. To restore the intended state:

```sh
curl -X PUT "https://B8ST9KVWVJ.algolia.net/1/indexes/openchoreo/settings" \
  -H "X-Algolia-API-Key: $ALGOLIA_ADMIN_KEY" \
  -H "X-Algolia-Application-Id: B8ST9KVWVJ" \
  -H "Content-Type: application/json" \
  --data @scripts/algolia-index-settings.json
```

Or edit the same values via **Index → Configuration** in the dashboard. Settings changes
are instant and need no re-crawl.

## Keep the Crawler in sync

The Crawler re-applies its own `initialIndexSettings` on every **full reindex**. If this
file and the crawler config disagree, a reindex silently reverts the index to whatever the
crawler holds. When changing settings here, mirror them into
`initialIndexSettings["openchoreo"]` at crawler.algolia.com.

## Why the settings look like this

Two deviations from the Algolia Crawler's stock DocSearch v3 defaults:

**`searchableAttributes` puts `hierarchy.lvl0` second-to-last.** `lvl0` is the *sidebar
category label*, `lvl1` is the page's H1. The stock v3 order searches `lvl0` first, so a
page ranked above others purely because its sidebar folder matched the query — an
API-reference category named `Authorization` outranked the guide titled "Authorization in
OpenChoreo", which sat at rank #57. Demoting `lvl0` makes the page title the strongest
signal and moved that page to #2.

(The upstream fix for this class of bug is DocSearch record format `v2`, whose
`hierarchy_radio.*` attributes rank "the record's own heading" above ancestry without
inverting the ancestry order. That needs a crawler `recordVersion` change plus a full
reindex; the reorder here is the settings-only equivalent.)

**`attributeForDistinct` is `url_without_anchor` with `distinct: 2`.** The stock default
dedupes on `url`, which *includes the anchor*, so one page contributes a record per heading
and can occupy nearly every slot in the 20-hit modal. Capping at 2 keeps the page title plus
its best-matching heading.

## What actually affects search ranking

Worth knowing before trying to make a page more findable:

| Record attribute | Comes from | Ranking weight |
|---|---|---|
| `hierarchy.lvl1` | the page's **H1** (not the front-matter `title`) | highest |
| `hierarchy.lvl2`-`lvl6` | H2-H6 | high, descending |
| `hierarchy.lvl0` | the sidebar category label | low |
| `content` | body prose | lowest |

Two consequences that have already caught us out:

- **Front-matter `keywords:` does nothing for search.** It renders a `<meta name="keywords">`
  tag, but the crawler does not put it in the record and it is not in `searchableAttributes`.
  `docs/platform-engineer-guide/authorization/overview.md` and `conditions.md` still carry
  such blocks, added in `0660b29` to "improve search discoverability" — they had no effect.
  Leave them for SEO if you like, but do not expect search to read them.
- **A term that appears only in body prose is close to unfindable**, because `content` is the
  lowest-priority attribute. If users search for a word, it needs to be in a heading. This is
  how "OpenTelemetry" / "OTLP" went missing: the content documents the collector endpoints,
  but no heading contains those words.

For abbreviations users type that the docs spell out (`k8s`/`kubernetes`, `idp`/`identity
provider`, `crd`/`custom resource definition`), the fix is **Synonyms** in the dashboard
rather than editing pages.

## Guarding against regressions

`scripts/search-smoke.mjs` replays the site's real queries against the live index and
asserts both properties. Run it after any settings or crawler change:

```sh
npm run test:search
npm run test:search -- --index some_scratch_index
```

## Rollback

Pre-change settings, if the current config ever needs reverting:

```json
{
  "searchableAttributes": [
    "unordered(hierarchy.lvl0)",
    "unordered(hierarchy.lvl1)",
    "unordered(hierarchy.lvl2)",
    "unordered(hierarchy.lvl3)",
    "unordered(hierarchy.lvl4)",
    "unordered(hierarchy.lvl5)",
    "unordered(hierarchy.lvl6)",
    "content"
  ],
  "attributeForDistinct": "url",
  "distinct": true
}
```
