# Field guide: delivery best practices

Condensed from Cloudinary's internal CSM video onboarding checks & best
practices guide. This is the guidance to apply when advising on, reviewing or
building a customer's video implementation — it is about *delivering video
well*, not about the player's API surface (see [player.md](player.md)).

## Understand the use case before recommending anything

Recommendations follow from the use case, so establish it first:

- **What types of video?** In e-commerce, typically short product videos on
  PDPs, plus campaign/content videos on homepages and landing pages.
- **How long are they?** Under a minute, or longer? This drives the delivery
  method more than anything else.
- **Where are they delivered?** Web (desktop/mobile), native mobile apps, or
  other channels. Native apps change the rules — see below.
- **Do they need analytics?** Views, engagement, watch time.
- **Do they release videos into traffic spikes?** Catalogue launches, major
  campaigns — this is what makes eager transformations worth it.
- **Do they need subtitles/captions?** Already authored, or generated and
  translated?
- **Any other accessibility or navigation needs?** Audio descriptions, chapters.

## Delivery cheat sheet

| Scenario | Approach | Notes |
| --- | --- | --- |
| Short, inline, autoplay, PDP, campaign or social-style web video | **Cloudinary Video Player, progressive** | Optimizes by default (the URL it builds carries `f_auto:video`) — write no format/quality transformation, and do not set `sourceTypes`. Enable responsive breakpoints for per-screen dimensions |
| Short web video, player unavailable | **Native `<video>` with multiple `<source>`** | `c_limit,w_[width]` + `f_auto:video` + `q_auto` |
| Longer-form / viewer-selected video | **ABR, preferably via the player** | Set `hls` (or `dash`) in `sourceTypes`; the player adds `sp_auto` for you |
| ABR, player unavailable | **HLS via a compatible third-party player** | Generate the manifest with `sp_auto`; verify browser/device support |
| Major launch, big catalogue drop, high-traffic event | **Eager transformations, on top of the chosen approach** | Pre-generate derivatives at upload; do not let first requests build them |
| Native iOS/Android app | **Mobile-specific strategy** | App picks codec/format and dimensions; URL params or `Accept` header |
| Analytics, captions, accessibility, chapters, branded UI needed | **Use the Cloudinary Video Player** | All of it works out of the box |

## Typical issues to avoid

- **Delivering the original**, untransformed. Default video quality only kicks
  in when *some* transformation is present in the URL — a bare original URL
  gets no optimization at all.
- **No maximum dimensions.** Shipping 4K/8K where 1080p is indistinguishable
  burns bandwidth and units for no visible gain. Cap with `c_limit,w_…` for
  progressive, and cap the top rung for ABR too.
- **Partial optimization** — applying some of `f_auto` / `q_auto` / resizing but
  not the rest, and ending up with files larger than necessary.
- **Assuming `f_auto` works in native mobile apps.** It is web delivery
  behaviour.
- **Preloading everything.** Preloading improves perceived start time but
  consumes units for videos nobody watches. Prefer lazy loading, and make
  autoplay/preload a deliberate decision rather than a default.

## `f_auto` causes a transformation-usage spike — say so in advance

Introducing `f_auto`, or adding format/codec combinations, forces new derived
versions to be generated. Expect a visible **spike in transformation usage**,
especially across a large existing library. It settles once the derivatives
exist and get reused. Warn the customer *before* the rollout so the bill does
not surprise them.

## Account-level optimization settings

- **Default automatic format** applies `f_auto`/`q_auto` to web-delivered video
  without touching delivery URLs. Being rolled out to plans on the *video
  seconds* metric; not available on video-bandwidth plans. Web only.
- **Default video quality** lets Cloudinary pick quality and codec (`vc_auto`)
  automatically. Available on all plans — but **only applies when another
  transformation is present in the URL**.

## Eager vs on-the-fly

On-the-fly is the right default: derivatives build on first request, no
preparation needed. Switch to **eager transformations at upload** when a launch
or campaign will drive a traffic surge, when videos are long or high-resolution
(2K/4K), or when transformations are complex enough to be slow to build. This
is the production-grade version of "warm the derivations": generate them at
upload rather than discovering the cost on the first viewer's request.

## Mobile apps are a different problem

The player requires a mobile SDK (iOS, Android, Flutter, React Native).

**`f_auto` does not work the same way.** The app must tell Cloudinary what the
device supports, either by naming it in the URL:

- Android: `f_mp4,vc_av1` → `f_webm,vc_vp9` → `f_mp4,vc_h264` (fallback)
- iOS: `f_mp4,vc_av1` → `f_mp4,vc_h265` → `f_mp4,vc_h264` (fallback)

…or by sending an `Accept` header (e.g. `video/webm; codecs="vp9"`,
`video/mp4; codecs="hvc1"`, `video/mp4; codecs="avc1"`). AV1 via `Accept` is
not supported yet.

**Responsive breakpoints are not supported in the mobile SDKs.** Request
explicit widths per device instead (`c_limit,w_640` / `w_1280` / `w_1920`), and
pick the smallest that suits the screen.

## What the player gives you for free

- **Analytics** — collected by default for anything delivered through the
  player; visible in Console under Video → Analytics, and available via API.
- **Accessibility** — WCAG 2.1 AA, keyboard navigation, captions/subtitles
  (existing VTT/SRT or generated transcripts), audio descriptions via text
  tracks or alternative audio tracks on ABR, and chapters (VTT, manual, or
  AI-generated).
- **Customization** — skins, colour schemes, fonts, titles/subtitles display.
- **Video Player Studio** — a visual configurator that emits the player config,
  rather than hand-writing it. Also handles transcripts and chapters.

## Validate before launch

Confirm, on representative videos and real devices:

- Delivery URLs are **not** serving originals, and have size limits.
- The intended optimization approach is actually enabled (check the URL that
  ships, not the one you wrote).
- Autoplay/preload behaviour is deliberate.
- Playback works on target browsers, devices and networks.
- Captions/subtitles and analytics work where required.
- The expected initial transformation/delivery usage has been explained.

Then review 2–4 weeks after launch: confirm delivery, check usage and
transformation patterns, and look for optimization opportunities.
