<a href="https://crawlcove.com/download?utm_source=github&utm_medium=readme&utm_campaign=schema-markup-validator"><img src="https://raw.githubusercontent.com/CrawlCove/.github/main/banners/schema-markup-validator.svg" width="1200" alt="schema-markup-validator: Check every schema.org node on a page against the required and recommended properties for rich results." /></a>

# schema-markup-validator

A **schema markup validator** CLI for Schema.org JSON-LD: extract every `<script type="application/ld+json">` block from a page (or a file, or stdin), parse it and say exactly why it doesn't parse when it doesn't, flatten `@graph`, and check each node against the properties Google's rich-result documentation **requires** and **recommends** for Article, Product, Offer, FAQPage, BreadcrumbList, Organization, LocalBusiness, Event, Recipe, HowTo, JobPosting, VideoObject, SoftwareApplication and more. Also catches the mistakes that are provably mistakes: wrong casing of a type, a missing `@context`, a relative URL, a date that isn't ISO 8601, a breadcrumb with gaps in its positions. Non-zero exit code for CI.

Need to write the markup instead? [crawlcove.com/tools/schema-markup-generator](https://crawlcove.com/tools/schema-markup-generator?utm_source=github&utm_medium=schema-markup-validator) builds it; paste the result back through this tool to check it.

<img src="docs/demo.svg" width="980" alt="Terminal recording: validating a product page on the SEO test site finds invalid JSON in one block and a Product with no name in the other">

*Recorded from a real run against [seo-test-site](https://github.com/CrawlCove/seo-test-site), a public site with 54 planted SEO problems that you can test against too.*

## Quick start

```sh
npx github:CrawlCove/schema-markup-validator https://crawlcove.github.io/seo-test-site/products/ember.html
```

```
https://crawlcove.github.io/seo-test-site/products/ember.html
  HTTP 200, 2 JSON-LD block(s)
  ✗ block 1: invalid JSON (Expected double-quoted property name in JSON at position 194 (line 6 column 1))
  ✓ block 2: 1 node(s) — Product

  ✗ invalid-json (block 1): Block 1 is not valid JSON: Expected double-quoted property name in JSON at position 194 (line 6 column 1).
  ✗ missing-required (block 2): (top level): Product is missing "name", which Google requires before it shows a rich result.
  ! missing-recommended (block 2): (top level): Product has no "aggregateRating" (recommended by Google; more properties, more rich-result features).
  ! missing-recommended (block 2): (top level): Product has no "review" (recommended by Google; more properties, more rich-result features).
  ! missing-recommended (block 2): (top level): Product has no "brand" (recommended by Google; more properties, more rich-result features).
  ! missing-recommended (block 2): (top level): Product has no "sku" (recommended by Google; more properties, more rich-result features).

2 error(s), 4 warning(s).
```

## Install

```sh
# one-off, nothing installed (Node 18+):
npx github:CrawlCove/schema-markup-validator https://www.example.com/product/widget

# global command, from the release tarball:
npm install -g https://github.com/CrawlCove/schema-markup-validator/archive/refs/tags/v1.0.3.tar.gz
schema-validator --version
```

(The tarball form is deliberate: a global `github:` install on npm 10 leaves a dangling symlink. The npm package is coming.)

## Usage

```sh
schema-validator <url> [options]
schema-validator --file page.html
cat markup.json | schema-validator --file -

  -f, --file <path>       validate a local HTML file or a bare JSON-LD document ("-" for stdin)
  --timeout <ms>          request timeout (default 10000)
  --user-agent <ua>       User-Agent header to send
  --json                  JSON output
  --fail-on <level>       error (default), warning, none
```

Example: a product page whose offer forgot the currency and whose type was typed in lower case:

```
$ schema-validator https://www.example.com/product/widget
https://www.example.com/product/widget
  HTTP 200, 2 JSON-LD block(s)
  ✓ block 1: 1 node(s) — BreadcrumbList
  ✓ block 2: 1 node(s) — product

  ✗ type-case (block 2): (top level): @type "product" is not a schema.org type; types are case-sensitive and you probably mean "Product".
  ✗ missing-required (block 2): offers: Offer is missing "priceCurrency", which Google requires before it shows a rich result.

2 error(s), 0 warning(s).
```

Exit codes: `0` clean (or only warnings with the default `--fail-on error`), `1` a failing finding, `2` usage error. `--json` prints the full report (every block with its parse state, `@context`, node count and types, plus the microdata types seen) for scripting.

Recommended-property warnings are raised only for top-level nodes, the entities Google would build a rich result for. An author's nested `Person` is not nagged about its `jobTitle`. Types the tool has never heard of are left alone: schema.org has about 800 and a rare one is more likely real than wrong.

## What each finding means, and the fix

| Finding | Severity | Why it matters | Fix |
|---|---|---|---|
| `fetch-error` | error | The page could not be read (DNS, TLS, timeout, 4xx/5xx). | Make the URL reachable. |
| `no-structured-data` | warning | No JSON-LD and no microdata on the page: no rich results, and less for AI answer engines to cite. | Add the JSON-LD your page type calls for. |
| `microdata-only` | warning | Only microdata found. Google accepts it but recommends JSON-LD, and this tool validates only JSON-LD. | Migrate to a JSON-LD block. |
| `invalid-json` | error | The block does not parse, so search engines ignore all of it. The message is the parser's own reason. The usual cause is a raw line break inside a string, which is invisible in the source. | Fix the character the message points at; escape newlines as `\n`. |
| `missing-context` | error | Without `"@context": "https://schema.org"` the vocabulary is undefined. | Add it at the top level. |
| `wrong-context` | warning | The context is not schema.org. | Use `https://schema.org`. |
| `missing-type` | error | A node with no `@type` describes nothing. | Add the `@type`. |
| `type-case` | error | `product`, `FAQpage`, `Localbusiness`: types are case-sensitive and a wrong case is a different, non-existent type. | Use the exact schema.org spelling the message suggests. |
| `malformed-type` | error | `@type` should be a bare type name, not a URL or a phrase. | `"@type": "Product"` |
| `missing-required` | error | A property Google's docs list as required for that type's rich result (`a or b` means either satisfies it). | Add it. Subtypes such as `BlogPosting` or `Restaurant` are checked against their parent's rules. |
| `missing-recommended` | warning | A property Google lists as recommended; without it the rich result has fewer features or may not be eligible for some placements. | Add it when you have the data. |
| `wrong-nested-type` | error | An FAQPage's `mainEntity` items must be `Question`s with an `acceptedAnswer`. | Fix the nested type. |
| `breadcrumb-positions` | error | ListItem positions must run 1, 2, 3 with no gaps. | Renumber. |
| `thin-address` | warning | A `PostalAddress` outside a `JobPosting` with neither `streetAddress` nor `addressLocality`. Google marks no address sub-property required for LocalBusiness or Event, but asks for as full an address as possible. Inside a `JobPosting`, a missing `addressCountry` is a `missing-required` error instead. | Fill in the address. |
| `invalid-date` | error | Dates must be ISO 8601 (`2026-09-29` or `2026-09-29T14:30:00+01:00`). `29/09/2026` is ignored. | Reformat. |
| `invalid-duration` | error | Durations must be ISO 8601 (`PT30M`, `PT1H15M`), not "45 minutes". | Reformat. |
| `relative-url` | error / warning | URLs in `url`, `image`, `logo`, breadcrumb `item`, `sameAs` and friends must be absolute; a relative `@id` is a warning. | Include scheme and host. |
| `empty-value` | warning | An empty string is worse than an absent property. | Drop it or fill it. |
| `empty-block` | warning | The block parses but contains no nodes. | Remove it. |
| `duplicate-type` | warning | `BreadcrumbList`, `FAQPage`, `WebSite` or `Organization` declared more than once on one page; search engines may pick either. | Keep one. |

## Works with CrawlCove

This validates one page. [Crawl Cove](https://crawlcove.com/?utm_source=github&utm_medium=schema-markup-validator), the desktop SEO crawler, runs the same schema-validity check across a whole-site crawl, reports every page that should carry structured data and doesn't, and tracks both over time.

This repo has its own page on crawlcove.com: [Crawl Cove schema validator CLI](https://crawlcove.com/open-source/crawlcove-schema-validator?utm_source=github&utm_medium=schema-markup-validator).

## Related tools

<!-- related:start (generated by CrawlCove/.github scripts/related.mjs) -->
| Repo | What it does |
|---|---|
| [seo-crawler-cli](https://github.com/CrawlCove/seo-crawler-cli) | Crawl a site from a script or CI. Checks status codes, redirect chains, titles, meta descriptions, H1s, canonicals, noindex and broken links, and fails the build on regressions. |
| [seo-audit-action](https://github.com/CrawlCove/seo-audit-action) | Crawl a staging or preview URL on every pull request. Fails the check on broken links, missing titles, noindex and redirect chains, with the details as a PR comment. |
| [seo-mcp-server](https://github.com/CrawlCove/seo-mcp-server) | Crawl a site, list SEO issues, inspect pages and find broken links from inside Claude, Cursor and other AI assistants. |
| [claude-seo-plugin](https://github.com/CrawlCove/claude-seo-plugin) | Seven SEO skills for Claude Code backed by a real site crawl: audits, broken links, redirects, titles and fixes, from a chat. |
| [seo-chrome-extension](https://github.com/CrawlCove/seo-chrome-extension) | On-page SEO checker with an AI visibility report: which AI crawlers can read the page, snippet blocks, llms.txt and extractability. Nothing leaves the browser. |
| [xml-sitemap-validator](https://github.com/CrawlCove/xml-sitemap-validator) | Check a sitemap or sitemap index against the sitemaps.org protocol and search engine limits. Flags bad URLs, lastmod dates and cross-host entries. |
| [robots-txt-tester](https://github.com/CrawlCove/robots-txt-tester) | Fetch and lint a robots.txt, test which URLs each crawler may fetch and see the deciding line. Fails deploys that block the site. |
| [redirect-chain-checker](https://github.com/CrawlCove/redirect-chain-checker) | Follow every hop of a URL's redirects and flag chains, loops, HTTPS downgrades, meta refreshes and redirects to error pages. |
| [hreflang-checker](https://github.com/CrawlCove/hreflang-checker) | Validate the hreflang tags on a page or across a whole XML sitemap, including self-reference, x-default and return tags. |
| [seo-test-site](https://github.com/CrawlCove/seo-test-site) | A deliberately broken SEO test site with 54 planted problems and the expected findings, for testing any SEO crawler. |
| [seo-crawl-export-spec](https://github.com/CrawlCove/seo-crawl-export-spec) | An open format for site crawl data: JSON Schema and CSV column reference, with a real sample export and a validator script. |
| [seo-crawl-export-js](https://github.com/CrawlCove/seo-crawl-export-js) | Load, query and convert site crawl data in JavaScript and TypeScript, with typed pages, issues by type and status filters. |
| [screaming-frog-export-converter](https://github.com/CrawlCove/screaming-frog-export-converter) | Turn an SEO Spider internal_all.csv into the open JSON crawl export format, with a report of which columns carried over. |
| [seo-audit-google-sheets](https://github.com/CrawlCove/seo-audit-google-sheets) | An Apps Script add-on that turns a site crawl export into an audit workbook: pages, issues by type, pages by status and title and meta flags. |
| [wordpress-seo-connector](https://github.com/CrawlCove/wordpress-seo-connector) | Bulk-apply fixes from the Crawl Cove desktop crawler to Yoast SEO, Rank Math, SEOPress or AIOSEO, with a change log and one-click revert. |
<!-- related:end -->

## License

MIT, see [LICENSE](LICENSE).
