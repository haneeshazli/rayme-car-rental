# Rayme Car Rental — Website

Static marketing site for Rayme Car Rental (Shah Alam, Seksyen 13), deployed to
both GitHub Pages and Vercel.

## Layout

| Path                    | What it is                                                        |
| ----------------------- | ----------------------------------------------------------------- |
| `Rayme Car Rental.html` | Design source — the original self-unpacking Claude artifact export |
| `build/build.py`        | Unpacks that export into a plain static site                       |
| `public/`               | The deployed site (generated — don't hand-edit)                    |

The design source is a single 18 MB file that inlines every asset as
base64+gzip and rebuilds them as blob URLs in the browser on each load. That
works when you open the file locally, but it makes a poor website: nothing can
be cached, and the whole bundle has to download and decompress before anything
renders.

`build/build.py` unpacks it once, at build time, into real files:

Every unpacked asset carries a content hash in its filename
(`axia-white.2a6cf894.jpg`). `/assets/*` is served immutable for a year, so a
file whose contents change has to change its name too — otherwise a replaced
photo keeps its old URL and everyone who has already visited goes on seeing the
old image until the cache expires.

- `public/index.html` — 150 KB of markup instead of 18 MB
- `public/assets/images/` — the 21 car photos, named after their originals
- `public/assets/docs/` — the rental agreement PDF the page links for download
- `public/assets/fonts/` — Wix Madefor Display + Text (woff2)
- `public/assets/js/` — React 18.3.1, React DOM, the dc-runtime and the design system

Same design, same markup — just served as normal cacheable files.

## Rebuilding

After changing the design source, regenerate `public/` and commit the result:

```bash
python3 build/build.py
```

The script fails loudly if the page source has changed in a way it doesn't
recognise, rather than emitting a half-built site: every asset is decoded and
assigned its path before `public/` is touched.

Nothing in the build is pinned to an asset uuid or resource id. A fresh export
from Claude renumbers both — ids went `img7` to `r0`, and every uuid changed —
so names come from how the page itself refers to each asset: the upload paths
in its own `RID` map, the `font-family` its `@font-face` rules declare, and the
header line each generated bundle starts with.

### Patch applied during build

The authored page defines `componentDidUpdate(_, prevState)`, but `dc-runtime`
invokes it as `componentDidUpdate(prevProps)` — one argument. `prevState` is
undefined, so reading `prevState.open` threw on every render and the
`document.body` scroll-lock never applied. The build rewrites that method to
track the previous value on the instance instead.

The page has two things that cover the viewport: `open` (the car detail modal)
and `menu` (the mobile nav). The authored handler only watches `open`, which
leaves the page scrolling behind the open nav, so the rewrite widens the lock
to cover both.

## Local preview

```bash
python3 -m http.server 4173 --directory public
```

## Share card and SEO

`build/make_share_assets.py` draws `public/assets/social/share-card.jpg`
(1200x630) and the favicons from the site's own brand tokens and its own Wix
Madefor fonts, so the link preview matches the page it opens. `build.py` runs
it and injects the Open Graph, Twitter and `AutoRental` JSON-LD metadata, plus
`robots.txt` and `sitemap.xml`.

`CANONICAL` at the top of `build/build.py` is the address shared with clients.
`og:image` has to be absolute, so both hosts point their preview at that one
URL — change it there and rebuild if the site moves to a custom domain.

### The link to send clients

```
https://rayme-car-rental.vercel.app/
```

It serves the current card to anyone asking for the first time.

WhatsApp caches a preview against the URL in the message for about a week and
offers no way to purge it, and it does not refetch the page, so a URL previewed
before the card changed keeps showing the old card. Hashing the `og:image` name
does not help on its own — WhatsApp never learns the name changed.

When that happens, `SHARE-LINK.txt` also carries a refresh link,
`…/?v=<card hash>`. Because the hash tracks the card, that is a URL WhatsApp
has not cached, so it fetches the new preview; send it once and go back to the
bare address. The site ignores the parameter — the page is identical either way
— and `canonical`/`og:url` stay on the bare address so search engines
consolidate there.

Both links, and the current hash, are printed by `python3 build/build.py` and
written to `SHARE-LINK.txt`.

## Web analytics

`build.py` injects Vercel Web Analytics into the `<head>` — the plain-HTML
snippet, so there is no package to install and no build step to run:

```html
<script>window.va = window.va || function () { (window.vaq = window.vaq || []).push(arguments); };</script>
<script defer src="/_vercel/insights/script.js"></script>
```

The script is served by Vercel's Edge Network from the site's own origin, so it
is first-party and gets past the tracker blocking that stops third-party
analytics. Nothing is collected until **Web Analytics is switched on for the
project** in the Vercel dashboard (Project → Analytics → Enable); the snippet
alone does not turn it on.

On GitHub Pages and on a local preview that one request simply 404s and the
page is unaffected — those visits are not counted. Pageviews are recorded on
load, which covers the whole site: it is a single page, so the SPA route
tracking the plain-HTML snippet lacks costs nothing.

The queue shim on the first line is what buffers a `va()` call made before the
script has loaded, and it is also the hook for custom events — a WhatsApp
booking click, say — if those are wanted later.

## Deployments

Both hosts serve `public/`:

- **GitHub Pages** — `.github/workflows/deploy-pages.yml` publishes it on every push to `main`.
- **Vercel** — `vercel.json` sets `outputDirectory` to it, with immutable caching on `/assets/*`.

Asset paths are relative, so the site works from a subpath (`/<repo>/`) as well
as from a domain root.
