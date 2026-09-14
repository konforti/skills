# Capabilities, detection, and degradation

Every capability below is optional. A build that assumes all of them produces a
page that breaks on the first cloud without an add-on; a build that assumes none
produces a page indistinguishable from the one being replaced.

So: **detect, then flag, then degrade.** The pattern that works is a single
explicit map of what is on, resolved once at build time, with each feature
carrying a fallback rather than a blank space.

```js
const features = {
  adaptiveStreaming: true,
  aiPoster: true,
  previewLoopPoster: true,
  hoverPreviews: true,
  chapters: true,
  seekThumbnails: true,
  hotspots: true,
  machineTags: false,         // needs Google Auto Tagging
  responsivePlayer: true,
  captions: true,
  translatedSubtitles: false, // needs Google Translation
  transcript: true,
  titleAndDescription: true,
};
```

Keeping it in one object rather than scattered `if`s is what makes the
degradation auditable — you can read off what this page will and will not do,
and say so honestly at the end of a run.

## The matrix

| Capability | Needs | Absent → degrade to |
| --- | --- | --- |
| Adaptive streaming (`sp_auto`) | nothing | progressive MP4 + `f_auto/q_auto` |
| AI poster (`so_auto`) | nothing | first frame, or a chosen still |
| `e_preview` loop opening frame (long sources only) | nothing (warm it) | AI poster, then plain poster |
| Hover previews | chapter→SKU map | static image on hover |
| Chapters + rail | `auto_chaptering` (or migrated cue points) | no rail; plain scrub bar |
| Seek thumbnails (`fl_sprite`) | nothing | off — the player copes |
| Hotspots → cart | **your** coordinates + SKU map | off; rail does the linking |
| Machine tags / facets | Google Auto Tagging add-on | existing hand-authored tags |
| Captions | `auto_transcription`, or migrated captions | no caption track |
| Translated subtitles | Google Translation add-on | English captions only |
| Audio description | `ai_video_analysis` (**needs API secret**) | omit the track |
| Searchable transcript | `.transcript` JSON | off |
| Title / description / JSON-LD | `auto_video_details` | hand-authored copy |

Two rows deserve their own note, because they are the ones people assume
Cloudinary supplies and it does not:

- **Hotspot coordinates** are yours. `interactionAreas` takes coordinates you
  pass. AI Video Analysis returns a timestamped natural-language visual
  transcription and explicitly **no** spatial coordinates or bounding boxes.
- **Chapter→product bindings** are yours. Auto-chaptering finds segment
  boundaries; which SKU a segment is selling is business data. On a migration,
  check the source platform first — cue points and custom fields often already
  carry it.

## Detecting, rather than assuming

How you resolve the flags depends on the destination, and the two cases are
opposites:

**Fresh claimable cloud.** Assume no add-ons. Translated subtitles and machine
tags will be absent every time; that is correct, not a regression. Hardcoding
those two to `false` is the honest default.

**Existing customer cloud.** Do not assume either way — add-ons may well be
subscribed. Probe by attempting the operation on **one** asset and reading the
result, then set the flag. Never probe by looping the catalogue: free tiers are
small (Google Auto Tagging 50/month, Google Video Tagging 5/month) and on a
customer cloud the quota is their money.

There is no Admin API that lists entitlements — `/addons`, `/add_ons`,
`/subscriptions`, `/entitlements` all 404. Attempting the operation is the only
detection available.

## Access routing: MCP vs `CLOUDINARY_URL`

The Cloudinary MCP never exposes an API secret, so it can only do what it has a
tool for. This is not a permissions problem you can escalate — unsigned
requests to any other endpoint return
`error while authenticating: api_secret not provided`.

| Step | Over MCP | With `CLOUDINARY_URL` |
| --- | --- | --- |
| Upload, with free analysis on the call | yes | yes |
| Poll asset for async status | yes | yes |
| Asset update / tags / context | yes | yes |
| Delivery URLs, warming derivations | yes (plain HTTP) | yes |
| Add-on categorization / auto-tagging | yes | yes |
| `ai_video_analysis` (visual transcription) | **no tool — unreachable** | yes |
| Any v2 or beta route | **no** | yes |

Consequence worth stating to the user rather than discovering late: on an
MCP-only build the **audio description track is simply unavailable**, because
`POST /v2/video/<cloud>/ai_video_analysis` cannot be signed. Everything else in
the matrix is reachable either way.

Before concluding a feature does not exist, check whether it is merely
unreachable by the route in use. The two are indistinguishable from inside an
MCP session.

## Optimization strategy: `sp_auto` (ABR) vs `breakpoints` (progressive)

Two mechanisms for the same goal — stop sending every viewer the same bytes —
working in opposite ways. Pick one **per source**; the choice is not reversible
without rewiring the player.

| | `sp_auto` — adaptive bitrate | `breakpoints` — progressive |
| --- | --- | --- |
| `sourceTypes` | `['hls']` (or `['dash']`) | **leave unset** — the default. `['mp4']` pins the format and disables `f_auto:video` |
| What it delivers | one manifest listing a rendition ladder | one file at one chosen width |
| Who decides | the **player**, continuously, per segment | the **player**, once, at load |
| Adapts to | bandwidth *and* changes mid-playback | viewport/DPR at startup only |
| Recovers from a stall | yes — drops to a lower rung | no — it is one file |
| Startup cost | manifest + first segments | one request |
| Overhead | segmenting, manifest, more derivations to warm | none beyond the resize |

### Duration decides this, not "main player vs decoration"

**Check the asset's duration before choosing.** Cloudinary's own field guidance
is duration-based, and it is the opposite of "ABR is the default for anything
important":

| Duration | Approach |
| --- | --- |
| **Up to 1–2 minutes** | **Progressive** — the player's default. `f_auto` + `q_auto`, plus responsive breakpoints. Product videos, campaign films, promos, social-style clips, anything inline or autoplaying. |
| **More than 1–2 minutes**, especially viewer-selected and not autoplaying | **ABR** (`sp_auto` via `sourceTypes:['hls']`). Long-form, where a network can realistically degrade mid-watch. |

`sp_auto` is the only mechanism that responds to bandwidth dropping *while
someone is watching* — but that advantage needs a watch long enough for it to
act. On a 14-second campaign film the ladder never meaningfully switches, and
you have bought manifest round-trips, slower startup, and a pile of renditions
to generate and warm in exchange for nothing. **Progressive is the default for
short web video; reach for ABR when length justifies it.**

`breakpoints` is right for short video generally — including silent decorative
loops, hover previews and background clips, where a manifest is pure overhead.

**Do not reach for `e_preview` to make a short clip shorter.** `e_preview`
extracts a teaser from a *long* source. Generating a 6-second preview of a
14-second video produces something barely shorter than the asset itself, costs
extra derivations to warm, and is strictly worse than looping the real file.

### The player already applies `f_auto:video` — do not restate it

Measured against `cloudinary-video-player@4.1.0`, with the same asset:

| Options | URL the player generates |
| --- | --- |
| none | `f_auto:video/v1/<id>` |
| `breakpoints` | `c_limit,w_640/f_auto:video/v1/<id>` |
| `sourceTypes:['mp4']` + `breakpoints` | `c_limit,w_640/<id>.mp4` |
| `sourceTypes:['mp4']` + `breakpoints` + `transformation:{fetch_format:'auto',quality:'auto'}` | `c_limit,f_auto,q_auto,w_640/<id>.mp4` |

The third row is the trap: asking for MP4 **removes** `f_auto:video`, pinning
one format for every viewer. The fourth row is people noticing the loss and
manually re-adding what the default gave them for free — while still pinned to
the `.mp4` container.

**The rule this table is an instance of:** express delivery through player
options and let it build the URL. Anything written into `transformation:` that
an option already covers gets merged into the player's own component. Save
hand-written transformations for assets no player manages — posters, hover
loops, `e_preview` teasers, the native `<video>` fallback.

So for progressive: set `breakpoints`, leave `sourceTypes` alone, and write no
format/quality transformation. The extensionless `f_auto:video` URL is
content-negotiated per request — the same URL returned WebM to one request and
MP4 to another in testing.

### If the Cloudinary player cannot be used

Deliver progressive with a native `<video>` and several `<source>` elements, so
the browser picks by viewport — the same optimizations, without the player:

```html
<video controls autoplay muted playsinline preload="metadata">
  <source media="(max-width: 640px)"
    src=".../video/upload/c_limit,w_640/f_auto:video/q_auto/v1/<id>.mp4" type="video/mp4">
  <source media="(max-width: 1280px)"
    src=".../video/upload/c_limit,w_1280/f_auto:video/q_auto/v1/<id>.mp4" type="video/mp4">
  <source
    src=".../video/upload/c_limit,w_1920/f_auto:video/q_auto/v1/<id>.mp4" type="video/mp4">
</video>
```

Note `f_auto:video` (not bare `f_auto`) for the format/codec pick, `q_auto` for
quality, and `c_limit,w_…` to cap dimensions. For ABR without the player,
generate the manifest with `sp_auto` and hand the `.m3u8` to a compatible
third-party player.

### `breakpoints` is progressive-only

`breakpoints` exists to pick a delivery **width** for the container on the
progressive path. ABR does not need one — the streaming profile defines the
rendition ladder and the player moves between rungs as bandwidth changes. So
the two are not alternatives to weigh against each other on one source:
`breakpoints` has no meaning once `sourceTypes:['hls']` is set, exactly as
`maxDpr` has none.

Set it anyway and the player **prepends a resize** to the source URL. With
`sp_auto` in the chain that produces:

```
c_limit,w_2560/sp_auto/….m3u8   →   HTTP 400
```

Cloudinary rejects a resize combined with a streaming profile ("sp_auto
transformation is not allowed (resize is only supported for overlay)"), because
the streaming profile *already defines* the rendition ladder — asking for one
width contradicts the whole mechanism. Verified at every width tried, 640
through 2560: always 400, and the video simply never loads.

So `breakpoints` stays off under `sourceTypes:['hls']` — not as a workaround,
but because there is no progressive path here to size. Using player options
rather than hand-written URLs does not avoid it; the player composes the
invalid URL for you.

Don't hand-write the profile either: `sourceTypes:['hls']` already emits
`sp_auto`, and adding `transformation:{streaming_profile:'auto'}` merges it
into the resize's component, which hides the real cause behind a second error.

Both are constructor options, so dispose and rebuild the player when a page
moves a viewer between the two strategies.

`maxDpr` is narrower than "MP4-only": Video Player Studio reveals Max DPR only
*after* breakpoints is enabled, so it is a child of breakpoints rather than an
independent option. With ABR neither exists — the ladder governs resolution.

### The normal shape is both, on different sources

A page usually wants ABR for the main tour and progressive for the loops:

```js
// main player, LONG source — adapts while watching
sourceTypes: ['hls'],  breakpoints: false

// preview loop cut from that long source — one small file, no manifest
<video src=".../e_preview:duration_10.0/f_auto/q_auto/hero.mp4" muted loop>
```

That is the common configuration, not a compromise. What is *not* available is
both mechanisms on the same source.

If the main video is itself short, this shape does not apply: everything is
progressive, and the "loop" is simply the real file
(`c_limit,w_…/f_auto/q_auto/hero.mp4`) rather than an `e_preview` cut of it.

### Naming collision worth flagging

The player option `breakpoints` (a boolean, video) is unrelated to the image URL
parameter `w_auto:breakpoints` (responsive image widths). Same word, different
subsystems — do not let one's documentation answer a question about the other.
