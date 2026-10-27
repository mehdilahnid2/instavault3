# InstaVault

Archive Instagram media you **own or have permission to download**, through official APIs or a lawful source you
configure. Next.js 16 · React 19 · TypeScript · Tailwind 4 · Archiver 8.

Flow: paste URL → server validates and normalizes → first available source resolves metadata → gallery with
thumbnails, type, caption and status → select items → download files individually or as one ZIP, with progress,
cancel and retry. Nothing is fabricated: if no source can serve a URL, the app says why.

## What the official APIs can and cannot do (checked October 2026)

| Source | Reaches | Downloadable files? |
|---|---|---|
| **Instagram API with Facebook Login**, your own token | All media of *your own* Business/Creator account | Yes (`media_url`; carousel files via `children`) |
| Same API, **Business Discovery** | Public media of *other* Business/Creator accounts, by username | Yes, but `media_url` is omitted for videos with copyrighted/licensed audio and for Reels whose owner disabled downloads. A single post of another account can't be looked up by URL. |
| **oEmbed** | Public single posts/Reels | **No.** Thumbnail, author, caption only. Used as a preview-only fallback. |
| Instagram API with Instagram Login | Your own professional account | Own media only (set `IG_GRAPH_HOST=graph.instagram.com`) |
| Instagram Basic Display API | Personal accounts | **Discontinued Dec 2024** |

**Not reachable by any official API:** arbitrary personal profiles, private accounts, Stories/Highlights, and other
people's individual posts. InstaVault does not bypass logins, private-account rules, rate limits or CAPTCHAs, and never
asks for or stores Instagram passwords or session cookies. Details change; re-check Meta's docs
(<https://developers.facebook.com/docs/instagram-platform/>) before relying on a capability.

## Setup

```bash
npm install
cp .env.example .env.local   # fill in values
npm run dev                  # http://localhost:3000
npm test && npm run typecheck && npm run build && npm start
```

**Credentials (official):** create a Meta app in the [App Dashboard](https://developers.facebook.com/apps), add the
Instagram product, connect an Instagram Business/Creator account to a Facebook Page, and generate a user access token
with the Instagram permissions you need (e.g. `instagram_basic`). Put the token in `IG_ACCESS_TOKEN` and the Instagram
professional account ID in `IG_USER_ID`. Business Discovery and some permissions need Meta App Review for production
use. Generate `APP_SECRET` with `openssl rand -hex 32`.

**Try it without Instagram credentials** (exercises the real pipeline against a local fixture provider):

```bash
node tests/fixture-server.mjs &
EXTERNAL_SOURCE_URL=http://127.0.0.1:4010 INSTAVAULT_ALLOW_PRIVATE_HOSTS=1 npm run dev
# then paste https://www.instagram.com/fixture_user/
```

`INSTAVAULT_ALLOW_PRIVATE_HOSTS` is ignored in production builds.

## Architecture

```
src/app/page.tsx            dashboard (client)
src/app/api/resolve         POST url → items with signed file tokens; GET → source status
src/app/api/thumb|download  token-gated streaming proxies (MIME + size checked)
src/app/api/jobs/**         ZIP jobs: create, poll, cancel (DELETE), retry, download
src/lib/url.ts              parse/normalize/reject URLs
src/lib/sources/*           MediaSource interface + graph-api, oembed, external, registry
src/lib/safe-fetch.ts       manual redirects, timeouts, size cap, MIME allow-list
src/lib/net-guard.ts        https-only, host allow-list, blocks private/loopback/link-local IPs
src/lib/tokens.ts           HMAC-signed, expiring tokens (media URLs never reach the browser)
src/lib/jobs.ts             temp-dir jobs, Archiver ZIP, manifest.txt, TTL sweeper
```

## Security notes

- Download and thumbnail endpoints accept only tokens minted by `/api/resolve`, so they are not an open proxy
  (SSRF guard, host allow-list and public-IP check on every redirect hop).
- Images/videos only (jpeg, png, webp, gif, heic, mp4, mov, webm); default cap 200 MB per file; 30 s API and 120 s download timeouts.
- Strict CSP; thumbnails are proxied so visitors never contact Instagram CDNs directly.
- Temp files live in `os.tmpdir()/instavault/<job>` and are deleted after `JOB_TTL_MINUTES` (default 30).
- Job state and rate limits are in memory: run **one instance**, or move them to Redis/object storage to scale out.
- Add authentication in front of this app before exposing it publicly.

## Responsible use

You must have the right to download the selected content (your own, licensed, or permitted by the owner). Respect
Instagram's Terms, Meta's Platform Terms and applicable copyright and privacy law.
