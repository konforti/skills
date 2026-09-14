---
name: cloudinary-video
description: Build websites with the Cloudinary Video Player — adaptive streaming, chapters, captions, transcripts, hotspots, AI analysis and metadata — on an existing Cloudinary cloud, or migrating the media on first. Use when building or improving a video experience on Cloudinary, moving video off Brightcove, Vimeo, YouTube or Wistia, or standing up a cloud for a demo.
license: MIT
metadata:
  author: cloudinary
  version: '2.1.0'
---

# Building video sites on Cloudinary

Covers what the published `cloudinary-transformations` and `cloudinary-docs`
skills do not: **the video player, the async AI features, provisioning, and
getting existing media onto a cloud.** Those skills handle transformation URLs.
Use them for that; use this for everything else.

## How to use this

You are helping someone reach a goal for their video experience, not applying a
checklist. So:

- **Ask when the answer changes the build.** The delivery branch, where the
  media comes from, which cloud it lands on, whether an add-on exists — a wrong
  guess here costs a whole run. One question beats an assumption. The questions
  worth asking up front are in *Start here* below.
- **Ask when you are unsure.** That is legitimate and always preferable to
  quietly picking a default and presenting it as the decision. Say what you
  would pick and why, then let them redirect.
- **Recommend, don't enumerate — in conversation.** Where this skill gives
  options, pick the one that fits and say so in a line. Spend their attention
  on the thing you built, not on a tour of the choices behind it.
- **But build generously.** The restraint is in what you *say*, never in what
  you ship. Build the full experience — chapters, transcript, captions, AI
  poster, preview loops, structured data, responsive delivery — not a minimum
  viable one. Deleting a feature they do not want takes a minute; never seeing
  it means they never knew to ask. Err toward the wow — but the wow is the
  experience working, never the page advertising how it works. Ship their site,
  not an exhibit about Cloudinary; see *Ship the site, not the demo
  scaffolding* in [references/build-patterns.md](references/build-patterns.md).
- **Say what is not there.** A missing add-on, an unreachable original, a
  feature Cloudinary does not generate — name it plainly and move on. Silence
  and approximation both cost more later.

The work has two phases, and the first one is often skipped:

- **Phase A — establish the assets.** Only needed if the media is not already
  on a Cloudinary cloud.
- **Phase B — build the site.** The substance: optimization strategy, player
  capabilities, analysis and metadata.

## Start here: get the intent, then the logistics

Do not begin until you know what they are building. Each answer below changes
the work substantially, so **ask for any you were not given** — one question
costs a message, a wrong guess costs a run.

**1. What are you building, and what should the video do there?**

The one question the rest depends on. A product-page loop, a long-form library,
a course, a news page and a landing hero want different delivery, different
player features and different page shapes. Get this and most later choices
follow; skip it and you will build a competent version of the wrong thing.

**Do not interview them about features.** Do not ask whether they want
captions, chapters, transcripts or hover previews — build them. Someone
describing what they want in chat should get the magic without being asked to
specify it, and should see everything Cloudinary can do for them before
deciding what to keep.

**2. Do you have an existing website or app — is this a migration or from
scratch?**

A migration gives you three things at once: the media, the conventions to match
(layout, naming, structure), and human-authored metadata worth more than
anything regenerated — see A2. From scratch, the builder supplies the sources;
do not go looking.

**3. Are the assets already on a Cloudinary cloud?**

Yes → **skip phase A entirely.** Go to phase B. Confirm the cloud name, how you
reach it (MCP or `CLOUDINARY_URL` — see the routing table in
[references/capabilities.md](references/capabilities.md)), and which public IDs
to build against. Do not re-upload assets that are already there.

No → migration is needed, so:

**4. Where are the resources?**

| Source | What it means |
| --- | --- |
| **An existing website** | Try to fetch it for them; otherwise ask them to provide or map it. The site is also your source of guidelines and mapping — see below. |
| **From scratch** | The builder provides the asset sources. Do not go looking; ask. |

**A named real site is never a cue to reach for Cloudinary's `demo` cloud or
any other stock/sample assets.** If the user names a real site or brand
("build a page for gap.com"), you must actually open that site and pull its
real images/video before writing any page code — do not silently substitute
placeholder assets because fetching feels slower or the site has no obvious
video element. Only fall back to demo/stock assets if, after actually trying,
the real assets are genuinely unreachable (login-gated, blocked, no usable
media) — and even then, say so explicitly and ask the user before
substituting anything. Silently swapping in demo-cloud assets for a named
brand is the single failure mode this rule exists to prevent.

**5. Where does it land — an existing cloud, or a new claimable one?**

| Destination | Consequences |
| --- | --- |
| Their **existing cloud** | No provisioning. No claim URL. Respect existing folders, naming and presets. Re-runs cost real quota and can overwrite live assets — be idempotent, and never loop the catalogue to probe. |
| A **fresh claimable cloud** | Provision it (below). Expect no add-ons. Surface the claim URL at the end. Provision a new one per test run. |

Local files into an existing cloud is a legitimate combination — source and
destination are independent choices.

**Detect what you can, ask what you cannot.** Duration, whether there is an
audio track, resolution and which add-ons a cloud has are all measurable —
`ffprobe` the source, probe one asset. Do not spend a question on something you
can check, and do not guess at something you cannot.

---

# Phase A — establish the assets

## A1. Get the originals

**Find the video in the raw HTML, not in a browser's DOM.** `curl` the page
and grep the source *first*:

```sh
curl -s -A "<a real desktop UA>" "<page url>" -o page.html
grep -oE '[A-Za-z0-9_-]+\.(mp4|m3u8|webm|mov)' page.html | sort -u
```

**Do not conclude "this site has no video" from `document.querySelectorAll`.**
An automated/headless browser's DOM is systematically *less complete* than
what a real user sees, and it fails silently — you get a clean empty array,
not an error:

- On a React/Next.js site the `<video>` element is often created by
  client-side hydration. The asset URL sits in the server-rendered JSON
  payload (`__NEXT_DATA__`, RSC flight data, inline `<script>` state) while
  the raw HTML contains **zero** `<video` tags. If hydration doesn't complete,
  the element never exists to be queried.
- Bot-detection layers (Akamai, Cloudflare, PerimeterX) serve automated
  browsers a degraded page. `document.readyState === "complete"` still
  reports complete — completeness of the *document* is not completeness of
  the *content*.
- A tell-tale sign you are looking at a hydrated-only element: its class is a
  runtime-generated hash (`sitewide-ekvlu4`, `css-1a2b3c`) that appears
  nowhere in the raw HTML.

This is a real, observed failure: on gap.com the served HTML carried six
`.mp4` URLs in its JSON payload and the driven browser reported **zero**
video elements — after waiting for idle, after a shadow-DOM-piercing search,
and at desktop viewport width. The grep above found all six instantly.

Then, to catch what the source grep cannot (assets requested only at
runtime):
- Scroll the whole page and let it idle — some clips lazy-load.
- List **every** network request, not just ones pre-filtered by extension —
  a plain `<img src="...">` can return a `video/mp4` response.
- Check more than one page type — homepage, campaign/collection, and product
  detail each carry different media.

Confirm any URL you find with `curl -sIL <url>` and read the `content-type`
before trusting it. Only if all of that comes back empty may you say the site
has no video — and then ask before falling back to placeholder assets (see
the rule above on named real sites).

**Never ingest what the website delivers.** A page serves a player embed backed
by transcoded adaptive renditions. Ingest that and Cloudinary transcodes a
transcode — quality is permanently capped below the original and every derived
variant inherits the loss. Get the master from the source platform or the
customer's own storage.

This is the rule that decides whether a migration is worth anything. Full
per-platform detail, verified, in
[references/source-platforms.md](references/source-platforms.md):

| Platform | Original retrievable? |
| --- | --- |
| Wistia | Yes — `medias#show` → `OriginalFile` asset |
| Brightcove | Yes, but **not** via `/videos/{id}/sources` (those are renditions). Needs the Social Syndication master feed, and the master may have been deleted. |
| Vimeo | Yes, but plan-gated and the field carrying the true original is undocumented — verify, do not guess |
| YouTube | **No API path.** Google Takeout or the customer's masters |

Graduated fallback when you cannot fetch: **fetch → ask them to provide → ask
them to map.** All three are legitimate outcomes. Silently substituting a
scraped rendition is not.

Prefer download-then-upload over Cloudinary remote fetch: source URLs expire and
are authenticated, and on an unclaimed cloud Cloudinary's own fetcher is blocked
by the delivery IP allow-list anyway.

`ffprobe` what you downloaded before uploading — resolution, bitrate, and
whether an audio stream exists at all. That last one decides whether
transcription and chaptering can work.

## A2. Mine the existing site for guidelines and mapping

When migrating from an existing website, the site is an **input to the build**,
not just a media source. Three things to take from it:

- **Structure and conventions** — layout, look, naming, how pages are
  organized. You are improving their page, not replacing it with yours.
- **Human-authored metadata** — captions, chapter lists, titles, descriptions,
  tags. **Prefer these over regenerating.** A caption a human wrote and paid for
  beats `auto_transcription` output. Source platforms often carry cue points and
  custom fields that already hold the chapter→product bindings Cloudinary cannot
  infer.
- **The old→new mapping** — old asset/embed reference to new public ID. This is
  what makes rewiring mechanical and verifiable rather than a hunt.

Say what had no Cloudinary equivalent rather than approximating it.

## A3. Provision a cloud (fresh-cloud destinations only)

```sh
npx @cloudinary/cloud --goal "<what you are building>" --model <your-model-id>
```

No signup, no credentials. Writes `CLOUDINARY_URL` to `./.env` and prints a
**claim URL** a human can open later to keep the account.

- `--ip <address>` — public IP allowed to view delivered media, repeatable, max
  3. **Leave it off unless you know better**: the default locks delivery to the
  calling address, which is almost always right. Getting this wrong is the main
  way a migration appears to succeed while every asset 404s — uploads are never
  IP-restricted, only CDN delivery.
- `--json` — machine-readable, including `claim_url` and `expires_at`
- `--no-env` — print credentials instead of writing `.env`

**The cloud expires 24 hours after provisioning** unless claimed. Surface the
claim URL at the end; it is what makes the work permanent. Never commit `.env`.

**Provision a new cloud for every test run.** Reusing one skips exactly the work
most likely to break, because it only breaks the first time: assets are already
uploaded, async analysis already reports `complete` rather than `pending`, and
derivations are warm so nothing returns 423. The clouds are free and expire on
their own.

If provisioning fails with `delivery_ips_not_public`, the connection is behind a
VPN or secure gateway. That is the user's to resolve — see
[references/gotchas.md](references/gotchas.md).

## A4. Upload, and trigger analysis on the way in

Ask for the AI features **on the upload call**. Re-running them afterwards costs
a second operation against a quota.

```
upload video:                    # free
  auto_transcription: true       # transcript + captions
  auto_chaptering: true          # chapter VTT
  auto_video_details: true       # title + description

then, separately, and tolerate failure:
  auto_transcription: { translate: ['es','fr'] }   # needs Google Translation
  categorization: 'google_video_tagging'           # needs that add-on
```

`translate` is the add-on part, not transcription itself. Requesting
`auto_transcription` **with** `translate` on a cloud without Google Translation
fails the **whole** transcription — no transcript, no captions — where plain
`auto_transcription` would have succeeded.

`auto_video_details` is worth requesting here rather than leaving to the player.
The player's `title: true` / `description: true` read the asset's context
metadata and only trigger generation if no value is found — so an
un-pre-generated title is a request happening for the first time mid-demo.

```
upload image:                    # free
  (no analysis parameters)

then, separately, and tolerate failure:
  categorization: 'google_tagging'
  auto_tagging: 0.65             # needs Google Auto Tagging
```

Know what image tagging returns before building on it: it describes **what is in
the frame**. A hotel room yields `bed`, `bedding`, `ceiling`, `lamp` — accurate,
and not what a guest filters by. Commercial attributes like "ocean view" are not
visible to an image model. Use it for library search and coverage, not as a
replacement for curated facets.

### Upload rules that have bitten real runs

1. **Upload with only the free features first.** Requesting a subscribed-only
   add-on rejects the whole upload — the asset is never stored — which looks
   like a broken upload rather than a missing entitlement.
2. **Then try add-ons separately**, and treat failure as expected.
3. **Add-ons cannot be enabled by an agent.** It is a console action, which on a
   claimable cloud means claiming it first. No Admin API exists: `/addons`,
   `/add_ons`, `/subscriptions`, `/entitlements` all 404.
4. **An account-level upload preset can rewrite every asset on ingest** — adding
   a watermark, downsizing — injected into signed uploads whether you ask or
   not. Pass an explicit clean `upload_preset`. See
   [references/gotchas.md](references/gotchas.md).
5. **Overwriting a public ID can silently keep the old asset.** If a re-upload
   reports new dimensions but delivery serves the old file, change the path.

**Say which features are missing and why.** A run that silently drops translated
subtitles has misled the user; one that says "translated subtitles need the
Google Translation add-on, which needs the cloud claimed first" has not.

## A5. Wait for the async features

`auto_transcription`, `auto_chaptering` and title/description return
`status: "pending"` and finish later — **or return `status: "failed"`**. Poll
each until `complete`; never assume success.

```sh
curl -sf -o /dev/null https://res.cloudinary.com/<cloud>/raw/upload/<public_id>.transcript
```

They need **real speech**. Music-only or silent video produces no transcript, no
chapters and no generated title. These features fail, they do not degrade.

## A6. Warm the derivations

The **first** request for an un-derived variant returns **HTTP 423** while
Cloudinary builds it. `e_preview` can take minutes. An un-derived variant
requested for the first time in front of an audience is a visible stall.

```sh
until [ "$(curl -s -o /dev/null -w '%{http_code}' "$URL")" = "200" ]; do sleep 5; done
```

Warm every URL the page will use, including hover previews and sprite sheets.

**A 200 is not enough for video.** A derivative still being built can return
200 with a *truncated* file, which the browser then caches — a 14-second source
delivered as 2.6 seconds that refuses to play, with nothing in the status code
to say so. Check the duration, not the status:

```sh
curl -s "$URL" -o /tmp/w.mp4
ffprobe -v error -show_entries format=duration -of default=nk=1:nw=1 /tmp/w.mp4
```

**Re-warm whenever the delivery URLs change.** Each distinct transformation is
its own derivative, so warming `sp_auto` does nothing for the progressive MP4s
you switch to later, and responsive breakpoints mean warming a set of widths.
Hard-reload afterwards to evict any truncated response already cached.

For a real launch or a traffic spike, use **eager transformations at upload**
instead of warming after the fact — see
[references/field-guide.md](references/field-guide.md).

---

# Phase B — build the site

**Read [references/field-guide.md](references/field-guide.md) alongside this.**
It carries Cloudinary's own delivery best practices — the duration-based
delivery cheat sheet, the optimizations customers most often miss, the
`f_auto` transformation-usage spike to warn about, eager transformations,
mobile-app delivery, and the pre-launch validation checklist. Phase B below is
how to build; the field guide is what "built well" means.

## B1. Resolve what this cloud can actually do

Before writing page code, settle the capability set — one explicit flag map,
with a fallback per feature. This is what keeps a page honest on a cloud without
add-ons instead of rendering empty rails and dead caption buttons.

**On a fresh cloud**, assume no add-ons: translated subtitles and machine tags
will be absent every time, and that is correct rather than a regression.

**On a customer cloud**, probe — attempt the operation on **one** asset and read
the result. Never loop the catalogue: free tiers are small (Google Auto Tagging
50/month, Video Tagging 5/month) and on a customer cloud quota is their money.

The full matrix, the detection rules, and the **MCP vs `CLOUDINARY_URL` routing
table** are in [references/capabilities.md](references/capabilities.md). The
routing matters concretely: `ai_video_analysis` has no MCP tool and cannot be
signed without an API secret, so on an MCP-only build the audio description
track is unavailable. Everything else is reachable either way.

## B2. Choose the optimization strategy

An explicit decision, not a default — retrofitting the other one means rewiring
the player.

**This is a fork, not a comparison.** Video Player Studio encodes it as a
segmented `Progressive | Adaptive` control — one source, one branch, and the
two sets of controls are disjoint. Reproduce that shape in code: decide the
branch first, then set only that branch's options.

| Studio branch | what it offers |
| --- | --- |
| **Progressive** | Format (Auto / WebM-VP9 / MP4-H.265 / MP4-H.264), Enable breakpoints (→ Max DPR), Auto HDR, Aspect Ratio (→ Resize Mode) |
| **Adaptive** | Format: **HLS or MPEG-DASH. Nothing else.** |

Everything that shapes delivery — breakpoints, DPR, HDR, aspect ratio, resize
mode — is absent from the Adaptive branch, because the streaming profile owns
all of it. That is the same rule the server enforces with *resize is only
supported for overlay*; the Studio simply never lets you reach the error.

**The real question is: is this decoration, or is this content someone
watches?** ABR only pays off when a viewer is watching long enough for the
ladder to react to their network. Decoration never gets that far.

**`ffprobe` answers it — read the source before choosing.** The signals are in
the file:

| Signal | Reading | Delivery |
| --- | --- | --- |
| **Loops** | decoration by definition — seen in passing, never watched through | **progressive**, always |
| **No audio track** | background/ambient, and almost always short and autoplaying | **progressive** |
| **Audio but no speech** (music-only) | montage or mood piece, not something followed | **progressive** |
| **Portrait 9:16 or square** | social/short-form | **progressive** |
| **Under ~20s** | over before a ladder could switch | **progressive** |
| **Speech, over ~1–2 min, viewer-pressed play** | content — a talk, explainer, interview, course | **ABR** |

Then:

Each branch is one option. Nothing else:

| Approach | Set |
| --- | --- |
| **Progressive** | `breakpoints: true` |
| **ABR** | `sourceTypes: ['hls']` |

That is the whole configuration. No `sourceTypes` on the progressive branch, no
`breakpoints` on the adaptive one, and on neither a format, quality, width or
`maxDpr` — the player's default is already `f_auto:video`, `breakpoints` brings
DPR handling with it, and `sourceTypes:['hls']` emits `sp_auto` itself. Reach
for `maxDpr` only to deliberately cap DPR below auto.

Duration is the proxy, not the rule — a three-minute silent background loop is
still progressive. Where signals disagree, **behaviour wins over length**.

The same probe decides more than delivery: no audio track, or audio without
speech, also means no transcript, no chapters and no generated title (A5). One
`ffprobe` per source, read once.

**Neither is free to get wrong, and the costs are asymmetric.** ABR on a short
clip buys a manifest round-trip before the first frame — slower start on the
content most sensitive to it — and a ladder to warm that never switches.
Progressive on a long watch fixes one rendition at load, so a network that
degrades at minute eight stalls with no recovery. That is worse, so a borderline
video someone genuinely watches goes ABR; a borderline video that autoplays
stays progressive whatever its length.

`sp_auto` is the only mechanism that recovers from a network degrading
*mid-playback* — but that needs a watch long enough for it to act. **Putting
ABR on a short clip is a real and easy mistake**: on a 14-second film the
ladder never meaningfully switches, and you have paid manifest round-trips,
slower startup and a pile of renditions to warm for nothing. "It's the main
player" is not a reason; length is.

**Never use `e_preview` to shorten an already-short video.** It exists to cut a
teaser from a *long* source. A 6-second preview of a 14-second film is barely
shorter than the asset, costs extra derivations, and is worse than just
looping the real file.

**Do not hand-write `f_auto`/`q_auto` for player-delivered video, and do not
force `sourceTypes:['mp4']`.** The player's default already is `f_auto:video`:

```
default                      → c_limit,w_640/f_auto:video/v1/<id>
sourceTypes:['mp4']          → c_limit,w_640/<id>.mp4        ← format pinned
```

Setting `sourceTypes:['mp4']` **replaces** `f_auto:video` with a hard-coded
extension, so every viewer gets MP4 and automatic format/codec selection is
lost — verified against 4.1.0. Leave `sourceTypes` unset for progressive and
just enable `breakpoints`; set it only to `['hls']`/`['dash']` for ABR. Written
transformations like `c_limit,w_…/f_auto:video/q_auto` belong to the **native
`<video>` fallback**, where no player is applying them for you.

If the player cannot be used at all, deliver progressive from a native
`<video>` with media-queried `<source>`s using
`c_limit,w_…/f_auto:video/q_auto` — see
[references/capabilities.md](references/capabilities.md).

### Configure the player; do not hand-write its delivery transformation

The player composes the delivery transformation from its options. Anything you
write into `transformation:` that an option already covers lands in the *same*
component and breaks the URL rather than adding to it.

```js
// WRONG — sourceTypes already produces sp_auto
player.source(id, { transformation: { streaming_profile: 'auto' } });
// RIGHT
cloudinary.player(el, { sourceTypes: ['hls'] });   // → sp_auto/<id>.m3u8
```

Same for `f_auto`/`q_auto` (the default is already `f_auto:video`) and widths
(that is `breakpoints`). Hand-written transformations are for assets no player
manages — posters, hover loops, `e_preview` teasers, the native `<video>`
fallback.

`breakpoints` belongs to the progressive branch only; under ABR the ladder
governs resolution. Set it there anyway and delivery 400s
(`c_limit,w_…/sp_auto/…` — *resize is only supported for overlay*), whether you
set it through player options or by hand.

Both live on the **constructor**, so a page moving a viewer between a
progressive and an ABR asset must **dispose and rebuild the player**. Calling
`source()` on a player built for the other strategy carries the wrong option
across.

Most pages want both, on **different** sources: ABR for the tour, progressive
for the loops. Full comparison table in
[references/capabilities.md](references/capabilities.md).

## B3. Load and configure the player

Install from npm and use the **ES import** — the package ships an `exports`
map with an `import` condition, bundles its own video.js, and loads the player
core behind a dynamic `import()`, so a bundler code-splits it for you:

```sh
npm install cloudinary-video-player
```

```js
import 'cloudinary-video-player/cld-video-player.min.css';
import cloudinary from 'cloudinary-video-player';
```

For a single-file page with no build step, the UMD build off a CDN works the
same way and puts `cloudinary` on `window`:

```html
<link rel="stylesheet" href="https://unpkg.com/cloudinary-video-player/dist/cld-video-player.min.css">
<script src="https://unpkg.com/cloudinary-video-player/dist/cld-video-player.min.js"></script>
```

**Construct with `cloudinary.player()`, not `cloudinary.videoPlayer()`.**
`videoPlayer()` is the legacy synchronous constructor. `player()` is the
current one: it returns a **Promise**, and it resolves *saved configuration*
first — pass `publicId` and it picks up the settings saved onto that asset in
Video Player Studio; pass `profile` for a stored profile instead.

```js
const player = await cloudinary.player('hero', {
  cloudName, publicId: 'my/video',   // asset's saved settings apply
  /* local options win over saved config */
});
```

Precedence is **construction options > profile > asset-saved > defaults**, so
repeating settings locally makes the saved config dead weight — save the
presentation, pass only what is page-specific.

Settings are normally saved through Video Player Studio in the console, but
the **Video Config API** writes the same thing without console access — useful
on a claimable cloud, and the way to make a build reproducible:

```sh
curl -X PUT https://api.cloudinary.com/v2/video/<cloud>/player/config/video/<asset_id> \
  -u <key>:<secret> -H "Content-Type: application/json" \
  -d '{"playerOptions":{…},"sourceOptions":{…}}'
```

Keyed by `asset_id`, not public ID, and PUT replaces the whole document.
Passing `publicId` with no saved config is silent — you get HTTP 200 and an
empty `{}`, and the page quietly falls back to its local options.

Because construction is async, guard anything that seeks the player — a
transcript line clicked before it resolves must queue, not throw. Details, the
built-in profiles and how to verify what actually applied:
[references/player.md](references/player.md).

Constructor options vs `source()` options — the split is about **how many
assets one player shows**. Constructor options configure the *player*;
`source()` options describe the *asset currently in it*.

**One player, one video:** set everything on the constructor and never call
`source()` — pass `publicId` and you are done.

**One player, several videos** — a playlist, a rail, a gallery where clicking a
thumbnail changes the video — build the player **once** and call `source()` per
asset. Constructor options persist across the swap; the per-asset metadata
(`title`, `chapters`, `textTracks`) rides along with each `source()` call.
Building a new player per video is the common mistake: it costs a fresh
video.js instance each time and leaks the old ones unless you `dispose()`.

The exception is the delivery strategy above: because it lives on the
constructor, swapping between an ABR and a progressive asset is the one case
that *does* need a dispose-and-rebuild rather than a `source()` call.

Getting these the wrong way round silently does nothing:

| Constructor | `source()` |
| --- | --- |
| `cloudName`, `sourceTypes`, `fluid`, `controls`, `showLogo`, `colors` | `chapters`, `title`, `description`, `textTracks`, `posterOptions`, `interactionAreas` |
| `seekThumbnails`, `chaptersButton`, `aiHighlightsGraph` | |

**`colors.text` must be a LIGHT colour.** Player chrome sits over dark video; a
dark text token makes the controls invisible.

Full option shapes: [references/player.md](references/player.md).

## B4. Wire the experience

The capabilities are only worth having if the page uses them. Patterns that
worked, with the reasoning, in
[references/build-patterns.md](references/build-patterns.md):

- **For a long source: open on an `e_preview` loop and build the player lazily**
  on first interaction — motion instead of a black frame, and no stream fetched
  until wanted. **For a short clip, just loop the real video muted** and build
  eagerly; there is no stream worth deferring and no teaser worth cutting.
- **Guard against the async constructor** — `cloudinary.player()` returns a
  Promise, so queue seeks that arrive before it resolves.
- **Compose `source()` from the flags** by conditional spread. Skip absent
  features; never stub them with a URL that 404s.
- **Bind playback both directions** — rail and transcript seek the player;
  `timeupdate` highlights the current segment. Render the rail before the player
  exists, since clicking it is what builds the player.
- **The transcript is JSON, not VTT** — hand it to the player for captions,
  fetch and render it yourself for on-page search with click-to-seek.
- **Emit JSON-LD** from the generated title, description and chapters.

## B4a. Generated, supplied, or hand-authored — in that order

Video Player Studio offers the same three-tier ladder on every piece of
content, and it is a good default for deciding where metadata comes from:

| panel | tier 1 — generate | tier 2 — supply | tier 3 — author |
| --- | --- | --- | --- |
| Chapters | Automatically generate chapters | Import a VTT file | Type `00:00` + name |
| Transcript | Generate | Upload | — |
| Poster | Suggested posters (AI frames) | Media Library / current frame | Upload |

Two things follow. First, **every generated artefact has a supply path beside
it** — so when a human-authored caption, chapter list or poster already exists
(see A2), use it; the ladder exists precisely so generation is not the only
route. Second, the tiers are per-asset, not per-site: generating chapters for
a long talk and hand-authoring three for a short explainer is normal.

## B5. Be straight about what Cloudinary does not generate

- **Chapter→product bindings.** Auto-chaptering finds the segments; which
  product a segment sells is your data. Check the source platform's cue points
  and custom fields first — often it is already there.
- **Hotspot positions.** `interactionAreas` takes coordinates you supply.
  Cloudinary does not track objects and hand you positions. The AI Video
  Analysis API returns a timestamped natural-language visual transcription and
  explicitly **no** spatial coordinates or bounding boxes.
- **Video background removal, virtual try-on, shade matching.** These do not
  exist. Do not approximate them with something adjacent.

## Close out

- Which capabilities are on, which are off, **and why** — name the add-on and
  what enabling it needs.
- What was migrated, and anything that had no Cloudinary equivalent.
- On a claimable cloud: **the claim URL**, and that it expires in 24 hours.

## How much of this is verified

Calibrate accordingly — the guidance is not uniformly tested:

- **Cloudinary behaviour is machine-verified.** The delivery, transformation and
  async-output claims are asserted against a live cloud by a test suite
  (`sp_auto` rejecting a resize at every width, the transcript being JSON with
  word-level timings, the chapters VTT path, the pinned player URLs). If
  Cloudinary changes, those assertions fail rather than quietly misleading you.
- **Source-platform APIs are documentation research, not tested calls.** The
  routes in [references/source-platforms.md](references/source-platforms.md)
  were read from each vendor's docs (September 2026) without an account to
  exercise them. Treat the *shape* as reliable and confirm the specifics for the
  account in front of you.
- **One item is explicitly unresolved:** which Vimeo response field carries the
  untranscoded original. Vimeo's own docs do not say. Verify before trusting a
  field, or ask for the customer's master.
- **Nothing here has been run end to end against a real existing-customer
  migration.** The from-scratch path has a worked example behind it; the
  existing-cloud path does not.

Say which of these applies when you rely on one, rather than presenting all of
it with equal confidence.

## Reference

- [references/field-guide.md](references/field-guide.md) — delivery best
  practices: use-case discovery, the delivery cheat sheet, issues to avoid,
  `f_auto` usage spikes, eager transformations, mobile apps, analytics,
  accessibility, pre-launch validation
- [references/capabilities.md](references/capabilities.md) — capability matrix,
  detection, degradation, MCP vs `CLOUDINARY_URL` routing, ABR vs progressive
- [references/build-patterns.md](references/build-patterns.md) — lazy player,
  flag composition, playback binding, transcript handling, shipping the site
  rather than the demo
- [references/source-platforms.md](references/source-platforms.md) — getting
  originals off Brightcove, Vimeo, YouTube, Wistia
- [references/player.md](references/player.md) — full player option shapes
- [references/gotchas.md](references/gotchas.md) — upload presets, poisoned
  paths, quotas, async failure modes, truncated derivatives, video.js DOM
  surgery, player API gaps, VPN and delivery-IP traps
