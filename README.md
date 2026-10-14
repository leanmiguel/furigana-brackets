# Furigana Brackets

A Japanese reading converter PoC hosted on Cloudflare Workers **Static Assets**. The responsive two-column interface converts text in the browser using Kuroshiro and Kuromoji; no Worker script, backend API, or server-side dictionary is needed.

**Live:** https://furigana-brackets.theleanmiguel.workers.dev/

**Repository:** https://github.com/leanmiguel/furigana-brackets

## Run locally

Requires Node.js 22 or newer and npm.

```sh
npm ci
npm run dev
# Equivalent: npx wrangler dev
```

Open the localhost URL printed by Wrangler (normally `http://localhost:8787`). The initial dictionary download can take a moment. Conversion runs automatically once initialization finishes; subsequent conversions use the **Convert →** button. Library and dictionary responses use normal browser/CDN caching, not an offline cache.

While initializing, the output panel and disabled button explicitly show **Loading from CDN…**, followed by **Downloading dictionary from CDN…** with the approximate 18 MB download size. Once ready, the loading notice is replaced by the conversion output. CDN failures remain visible in the output panel with instructions to check the connection and reload.

## Deploy

```sh
npx wrangler login
npm run deploy
# Equivalent: npx wrangler deploy
```

Wrangler publishes `public/` as a static-assets-only Worker named `furigana-brackets` and prints its workers.dev URL. `wrangler.toml` uses compatibility date `2026-10-01`. The pinned Wrangler version is 4.150.0; older runtime binaries may not support that date.

The public repository does not configure automatic deployment. Run the command above to publish updates.

## Conversion modes

- **Bracket {Kanji|Reading}** (default): per-kanji annotations with Hiragana, Katakana, or Hepburn Romaji readings. Original kana, English, punctuation, and line breaks remain in place.
- **Okurigana Kanji(Reading)**: Kuroshiro's parenthesized readings, e.g. `感(かん)じ`.
- **Furigana (<ruby> HTML)**: copyable HTML source, not a rendered HTML preview.
- **Normal**: Kuroshiro's continuous reading output.
- **Spaced**: Kuroshiro's token-separated reading output.

All output is assigned with `textContent`, never `innerHTML`. Pasted markup remains literal text. The Copy button uses the browser Clipboard API, which needs a secure context (HTTPS or localhost) and permission; if unavailable, select and copy the output manually. Input is limited to 5,000 UTF-16 code units, matching HTML's `maxlength` behavior.

### Exact default example

Input:

```text
感じ取れたら手を繋ごう、重なるのは人生のライン and レミリア最高！
```

Output:

```text
{感|かん}じ{取|と}れたら{手|て}を{繋|つな}ごう、{重|かさ}なるのは{人生|じんせい}のライン and レミリア{最高|さいこう}！
```

Bracket mode obtains Kuroshiro's Hiragana `furigana` output and replaces its ruby tags using:

```js
/<ruby>(.*?)<rp>\(<\/rp><rt>(.*?)<\/rt><rp>\)<\/rp><\/ruby>/g
```

For Katakana and Romaji, only the matched readings are transliterated with Kuroshiro's utilities. This avoids a Kuroshiro 1.2.0 quirk: native Romaji furigana puts the entire sentence in one ruby wrapper, which cannot be processed with the per-kanji regex. Other modes use Kuroshiro's native behavior.

## Dependencies and privacy

Only Wrangler is installed as a development dependency. The pinned Kuroshiro and analyzer libraries are loaded from jsDelivr; the Kuromoji dictionary is vendored unchanged in `public/dict/` and served from the same Cloudflare deployment:

- [Kuroshiro 1.2.0](https://github.com/hexenq/kuroshiro) — MIT
- [Kuroshiro Kuromoji analyzer 1.1.0](https://github.com/hexenq/kuroshiro-analyzer-kuromoji) — MIT
- [Kuromoji 0.1.2 dictionary](https://github.com/takuyaa/kuromoji.js) — upstream [Apache-2.0 license](public/dict/LICENSE-2.0.txt) and [IPADIC/NAIST/ICOT notices](public/dict/NOTICE.md) are included verbatim.

The dictionary download is approximately 18 MB. User text is processed locally and is not sent to a conversion service. Cloudflare and jsDelivr still receive ordinary page/asset requests. Loading requires access to jsDelivr, and third-party CDN scripts execute in the page. No analytics or text persistence is included. Dictionary readings can be ambiguous or incorrect and should be reviewed.

### HTTPS dictionary loading fix

Kuromoji's browser loader uses `path.join` on `dictPath`. An absolute URL such as `https://cdn.jsdelivr.net/npm/kuromoji@0.1.2/dict` becomes `https:/cdn.jsdelivr.net/...` (one slash). On an HTTPS page, the browser resolves this against the page's own origin, requesting nonexistent assets. On HTTP localhost, the same malformed URL resolves to the external HTTPS host, hiding the production-only bug.

The app now uses `dictPath: '/dict'`, with all 12 original `.dat.gz` files from Kuromoji 0.1.2 included as Static Assets. This removes URL-scheme normalization from dictionary loading; no library patch or request-rewriting shim is needed.

The earlier assumption that the reported stall was merely an 18 MB download/decompression delay was incorrect: the original dictionary URLs did not resolve to the dictionary on the deployed HTTPS site.

## Verification

Local Wrangler/browser verification on 2026-10-14 covered:

- Exact default Hiragana bracket output, plus exact Katakana and Romaji bracket variants.
- Okurigana, literal Ruby HTML, Normal, and Spaced output through the actual UI.
- Multiline text, empty input, and literal `<script>` text without creating HTML nodes.
- One-click copying and the actual clipboard contents.
- Disabled controls before loading; a blocked-CDN initialization error keeps conversion disabled.
- Desktop and mobile screenshots; the 390px mobile layout stacks panels without horizontal overflow.

This is a PoC, not an accuracy guarantee for arbitrary Japanese text. No permanent automated test suite is configured; verification used the running app in Chromium and Chrome.

The initial deployment succeeded on 2026-10-14 (version `974db879-3b69-4000-b9cb-c5f429ea8c7b`), and the public HTTPS HTML was fetched successfully. Those checks did not establish HTTPS dictionary initialization correctness; that production-only gap is addressed by the same-origin dictionary fix above.

The prominent CDN-loading feedback update was smoke-checked by executing the actual inline application script with controlled DOM/CDN events: library loading, dictionary loading, successful readiness, script failure, and dictionary failure all passed. Browser automation was not repeated for that update.

The same-origin dictionary fix was verified on both local HTTPS and the deployed HTTPS site. On the deployed site, all 12 `/dict/*.dat.gz` requests returned HTTP 200, automatic conversion produced the exact default output, Convert became enabled, and no browser errors were reported. A MutationObserver recorded completion at 5,825 ms after navigation in that check; this is one observed timing, not a performance guarantee. Cloudflare deployment version: `20cf8f02-ca12-4a11-9603-284747940742`.

For future initialization changes, check HTTPS explicitly rather than relying on HTTP localhost:

```sh
npm run dev -- --local-protocol https
```

Wrangler uses a local development certificate. In browser network diagnostics, all 12 dictionary URLs must resolve to the page's own origin under `/dict/`, return 200, and lead to the exact sample conversion above. Do not treat a successful HTML response as proof that the converter initialized.
