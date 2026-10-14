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

Startup uses a dedicated, centered loading screen with a spinner and two real initialization stages: loading converter libraries and preparing the Japanese dictionary. The entire converter stays hidden until initialization and the automatic example conversion finish. No fabricated percentage or countdown is shown. The spinner respects reduced-motion preferences. On loading failure, the screen stops spinning, explains the error, and offers **Reload and try again**.

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

Only Wrangler is installed as a development dependency. All runtime libraries and the pinned Kuromoji dictionary are fetched from jsDelivr; no dictionary files are stored in the current source tree or shipped as Static Assets:

- [Kuroshiro 1.2.0](https://github.com/hexenq/kuroshiro) — MIT
- [Kuroshiro Kuromoji analyzer 1.1.0](https://github.com/hexenq/kuroshiro-analyzer-kuromoji) — MIT
- [Kuromoji 0.1.2 dictionary](https://github.com/takuyaa/kuromoji.js) — upstream [Apache-2.0 license](https://cdn.jsdelivr.net/npm/kuromoji@0.1.2/LICENSE-2.0.txt) and [IPADIC/NAIST/ICOT notices](https://cdn.jsdelivr.net/npm/kuromoji@0.1.2/NOTICE.md).

The dictionary download is approximately 18 MB. User text is processed locally and is not sent to a conversion service. Cloudflare and jsDelivr still receive ordinary page/asset requests. Loading requires access to jsDelivr, and third-party CDN scripts execute in the page. No analytics or text persistence is included. Dictionary readings can be ambiguous or incorrect and should be reviewed.

### Dictionary delivery

Kuromoji's loader uses filesystem-style `path.join`, which damages absolute HTTPS dictionary URLs. A relative path avoids that bug.

The app uses `dictPath: '/dict'`. `public/_redirects` sends each dictionary request to the original pinned jsDelivr file with an HTTP 302 redirect:

```text
/dict/* https://cdn.jsdelivr.net/npm/kuromoji@0.1.2/dict/:splat 302
```

The browser follows the redirect and downloads directly from jsDelivr, whose dictionary responses allow cross-origin access. Kuromoji only joins the safe relative path; it never joins an absolute CDN URL. No library patch, Worker backend, or dictionary download/build step is needed. Cloudflare's [Static Assets redirect rules](https://developers.cloudflare.com/workers/static-assets/redirects/) support external destinations and splats.

