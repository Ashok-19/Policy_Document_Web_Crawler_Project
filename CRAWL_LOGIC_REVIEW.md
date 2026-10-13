# Crawl Logic Review: Exploration, HTML Policies, CDN, Related Domains

Review only, no code changed. Verdict: **the engine is sound, but link gating is tuned for PDFs. HTML-native policies, paginated hubs and sitemap-only content are the biggest recall leaks.**

## 1. HTML-native policies (highest risk)

**What works:** `_maybe_html_document` + `html_policy_evidence` do test fetched pages as policy documents, and render snapshots are covered too.

**Gaps**
- **Pages never inherit hub evidence.** `hub_score` bonus is applied only to `doc`/`maybe_doc` links. A hub listing 400 HTML policies with plain titles ("Abdominal Aortic Aneurysm") scores below `PAGE_FOLLOW_MIN_SCORE` (2.0), so those pages are never fetched.
- **Skipped low-score *pages* are not recorded.** Only documents reach `skipped_links`, so lost HTML policies are invisible.
- **Strict acceptance test:** content score ≥ 5 *and* ≥ 2 markers *and* `doc_links < 8` *and* `link_density ≤ 0.35`. A real policy that lists 8+ related PDFs or references is rejected as an "index".
- **`<header>` is stripped** before text extraction, which can drop in-header titles, dates and policy numbers on CMS-built pages.
- **Rendered pages:** only the *last* snapshot is evaluated, so content shown in earlier tabs/accordions can be lost.
- Depth-0 pages are never evaluated as documents.

**Fix**
- Apply hub inheritance to page links too, when the parent is a hub (≥ 5.0) and the link is a sibling in a list.
- Treat dense lists of same-pattern URLs under a hub as one family, and follow all of them.
- Record skipped pages in `skipped_links`, with `kind=page`.
- Relax the index test: count only links *outside* the main content, or use `doc_links ≥ 8 and body < 1500 chars`.
- Keep `<header>` content (strip only `nav` and cookie banners), and evaluate every snapshot.

## 2. Link exploration

**Gaps**
- **Single hard gate:** in-scope pages need `score ≥ 2.0` from anchor/URL text only. Intermediate navigation ("Resources" → "Tools") scores near 0 and the path dies.
- **Pagination and A–Z filters** ("Next", "2", "A") score ~0, so static crawling stops at page 1. Only the headless path clicks pagination, and it triggers only when `needs_js` or `link_score ≥ 5 and doc_links < 3`.
- **Sitemap gating by URL words only:** `sc ≥ 3` on the URL. Opaque slugs/IDs (`/policy/12345`) score 0, so a complete sitemap inventory is thrown away. `sitemap_lastmod` is stored but never used.
- **JSON links dropped:** `json` is in `JUNK_EXTS`, so static JSON listing endpoints are skipped.
- **No trap protection** (faceted filters, calendars, session-like params) beyond the page cap. `MAX_PAGES_PER_PAYER` exhaustion silently skips the rest.
- **No recall signal:** nothing compares what was found against what a hub claims to hold ("523 policies").
- **My earlier keyword change:** "policy updates/notifications" now score −4, which can stop the crawler following *hub pages* that list real policies. The penalty should apply to document typing, not page following.

**Fix**
- Add a "pagination follow" rule: follow next/numbered/letter links on any page already scored as a hub.
- Use sitemap entries from a hub-like path (same prefix as known good policies) even with a low URL score.
- Allow `json` URLs under `/api/` or `/search` paths.
- Cap per-pattern URL families (e.g. 200 URLs per path template) to avoid traps; log when a cap or budget is hit.
- Compare discovered counts against hub-declared counts and warn on a shortfall.
- Do not apply the bulletin/manual penalty to page-follow scoring.

## 3. CDN list and logic

- Current list: azureedge, azurefd, cloudfront, blob.core.windows, akamaized, amazonaws, ctfassets, widen, bynder, brightspot, prismic, googleusercontent.
- **Contradiction:** `googleapis.com` is in `THIRD_PARTY_BLOCKLIST` and is dropped before the CDN logic runs, so `storage.googleapis.com` documents never count.
- **Likely missing:** fastly.net, edgekey/edgesuite/akamaihd.net, scene7.com (Adobe), sharepoint.com, box.com, cloudinary.com, r2.dev/pages.dev, sitecorecloud.io. Verify against hosts that actually appear in `crawl.jsonl` and the skipped CSV before adding.
- **Shared-tenant risk:** any PDF on a CDN gets a free baseline score (`3.0`) and **skips the brand check** in `_finalize`, even if it belongs to another tenant or a third party.
- Extension-less CDN links get no baseline (they need 5.0).

**Fix**
- Learn CDN hosts per payer: a host becomes trusted after N accepted documents linked from in-scope pages.
- Keep brand/ownership checks for CDN documents, but accept a weaker proof (brand in text *or* trusted host).
- Pin tenant paths (e.g. the Azure container name) once learned.
- Make the blocklist and CDN list consistent (a host can't be in both).

## 4. Related-domain logic

- An off-domain *page* scoring ≥ 8.5 (`in_scope_candidate`: not blocklisted, not `.gov`/`.edu`) becomes a related domain and is then fully in scope (depth budget 3), persisted across runs.
- **Risks:** one strong link to another Blue plan, a hospital system or an aggregator opens a whole domain. The brand check protects the *documents*, but crawl budget is wasted.
- **Brand check is a substring test** on the first 6,000 chars and the last 3,000 chars. Short or generic terms ("UHC", "Health Net", "Anthem") can match by accident.
- `.gov` hosts are never related domains, so state-Medicaid-hosted payer documents are only reachable as external documents (threshold 5.0).
- `register_brands` is global, so brand terms leak between payers in parallel runs.

**Fix**
- Require ≥ 2 accepted documents from a related domain within a small probe budget, else drop it (domain trial period).
- Ask for brand evidence in the *page* (title/footer) before the domain opens, not only for documents.
- Add a manual allow/deny list per payer in the registry; log every domain added and why.
- Make the brand set per payer instead of global.

## 5. Other weaknesses

- Per-payer scope is registrable-domain wide (all of `anthem.com`, including member sites), so budget goes to non-provider areas. Prefer path/host hints from the registry.
- Docs are always processed before pages, so the page budget is only spent after every doc is done. Fine, but make the doc budget visible per hub.
- No coverage report per hub (found vs. expected), so tuning is blind.

## 6. Priority order

1. Hub inheritance for HTML pages and recording of skipped pages.
2. Pagination / A–Z follow on hubs.
3. Sitemap use without URL-word gating (hub-prefix based).
4. Learned, tenant-pinned CDN trust; fix the googleapis conflict.
5. Related-domain trial period + per-payer brands.
6. Hub coverage report and URL-family caps.
