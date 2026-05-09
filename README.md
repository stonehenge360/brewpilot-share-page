# Brew Pilot share page

Static site that backs Brew Pilot's recipe-share Universal Links.
Lives at **`share.brewpilotapp.com`** — a dedicated subdomain so the apex
`brewpilotapp.com` stays free for the marketing site (Google Sites or
otherwise).

```
share-page/
├── _headers                                # Cloudflare Pages: forces application/json on the AASA file
├── .well-known/
│   └── apple-app-site-association          # Universal Links handshake (no extension)
├── index.html                              # https://share.brewpilotapp.com/
├── r.html                                  # https://share.brewpilotapp.com/r?d=<recipe>
└── README.md
```

## What this does

When a Brew Pilot user shares a recipe (via QR code or link), the iOS app
encodes the recipe as a `https://share.brewpilotapp.com/r?d=<base64>` URL.
iOS opens that URL one of two ways:

| Recipient | Behavior |
| --- | --- |
| Has Brew Pilot installed | iOS routes the URL into the app via the AASA file's `applinks` rule for `/r`. App decodes `?d=…` and imports the recipe. |
| No Brew Pilot | URL opens in Safari → `r.html` decodes `?d=…` in JS, shows a recipe preview, offers an "Open in Brew Pilot" deep link and an App Store badge. |

The decode is fully client-side — the server never sees recipe contents,
which means zero PII, zero moderation surface, and the obligations stay at
"Pastebin tier" rather than "social platform tier."

## Deployment (Cloudflare Pages)

1. **Push this folder to a new GitHub repo** (just the contents of `share-page/`,
   not the iOS project).
2. **Create a Cloudflare Pages project**, connect the repo. Build command:
   *(none — it's a static site)*. Output directory: `/`.
3. **Add the custom domain** `share.brewpilotapp.com` in Pages → Custom
   domains. Cloudflare auto-provisions TLS. (Apex `brewpilotapp.com` stays
   pointed at wherever the marketing site is hosted.)
4. **DNS:** add a `CNAME` record on `brewpilotapp.com`'s DNS:
   `share` → `<your-project>.pages.dev`. If the domain is in Cloudflare's
   own DNS, Pages does this automatically when you add the custom domain.
5. **Verify** by visiting `https://share.brewpilotapp.com/.well-known/apple-app-site-association`.
   It MUST return JSON with `Content-Type: application/json` and HTTP 200
   (no redirects). Cloudflare Pages picks up the content type from `_headers`.

## After the iOS app ships with the matching entitlement

1. Build & install the iOS app on a real device (Universal Links don't
   resolve in the simulator).
2. AirDrop or iMessage yourself a `https://share.brewpilotapp.com/r?d=<test>`
   URL — tapping it should open Brew Pilot directly, NOT Safari.
3. If it still goes to Safari: long-press the link → "Open in Brew Pilot"
   should appear. If it doesn't, AASA isn't being fetched correctly —
   check `https://app-site-association.cdn-apple.com/a/v1/share.brewpilotapp.com`
   to see what Apple's CDN has cached.

## TODO before going live

- [ ] Replace `app-id=0000000000` in the `apple-itunes-app` meta tags and
      every `apps.apple.com/app/id0000000000` link with the real App Store ID
      once the app is published. Search both `index.html` and `r.html`.
- [ ] Optional: add a Cloudflare Worker that decodes `?d=…` server-side
      and rewrites OG/Twitter meta tags with the actual recipe name. Without
      this, link previews in iMessage/Slack/Discord show a generic "Brew
      Pilot Recipe" card instead of the recipe-specific one.
- [ ] Optional: add a favicon and `apple-touch-icon`.
- [ ] Optional: a privacy policy at `/privacy` (simple statement: recipe
      data lives entirely inside the URL, the server never stores it).

## Browser support note

The decoder uses `DecompressionStream('deflate-raw')`, which requires:

- iOS Safari 16.4+
- Chrome 113+
- Firefox 120+

This covers ~90% of modern devices. Older browsers see a friendly error
asking them to update; recipes are never silently lost.

## Cross-reference with the iOS app

If you ever change the share-link URL pattern (subdomain, path, query
param name, compression algorithm), the matching iOS code lives in:

- `Brew_Day_Assistant/Export/RecipeURLCodec.swift` — encoder/decoder
- `Brew Day Assistant.entitlements` — `applinks:share.brewpilotapp.com`
- `Brew_Day_Assistant/ContentView.swift` — `onOpenURL` routing
- `Brew_Day_Assistant/Views/Recipes/RecipeListView.swift` — `handleSharedURL`

The compression on the iOS side is `COMPRESSION_ZLIB` (which Apple
confusingly names — it's actually raw deflate, RFC 1951). The browser
matching mode is `'deflate-raw'`. Don't switch one without switching the
other.
