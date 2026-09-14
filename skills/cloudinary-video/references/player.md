# Cloudinary Video Player option shapes

Verified against `cloudinary-video-player@4.1.0`. Option names come from the
player API reference, not from memory.

## Use `cloudinary.player()`, not `cloudinary.videoPlayer()`

The 4.1.0 UMD bundle attaches five things to `window.cloudinary`:
`videoPlayer`, `videoPlayers`, `player`, `players`, `Cloudinary`. Two of those
construct a player and they are **not** the same function:

| | `cloudinary.videoPlayer()` | `cloudinary.player()` |
| --- | --- | --- |
| Status | legacy constructor | **current** — the bundle tags its own analytics `newPlayerMethod: true` |
| Returns | the player, synchronously | a **Promise** resolving to the player |
| Saved configuration | none — options only | fetches saved config before constructing |

```js
const player = await cloudinary.player('hero', {
  cloudName: 'my-cloud',
  publicId: 'my/public-id',   // also loads the source
  // …local options win over anything fetched
});
```

Because it is async, anything that touches the player (seeking from a
transcript or a timecode chip) must tolerate being called before it resolves —
queue the request and replay it on resolve.

### Saved configuration: assets and profiles

Video Player Studio saves settings **onto the video asset** by default. Passing
`publicId` makes the player retrieve them automatically (requires player
3.6.0+), so a page can inherit the styling and behaviour someone configured in
the Studio without hard-coding it.

`profile` loads a stored *profile* instead — either a built-in
(`cld-default`, `cld-looping`, `cld-adaptive-stream`, `cld-live-streaming`) or
one saved on the cloud, fetched from
`/_applet_/video_service/video_player_profiles/<name>.json`.

**Profiles are writable over HTTP — no console needed.** When the same
presentation repeats across many assets, that is the signal to create one
rather than repeat it in page code:

```sh
# create or update; also GET /player/profiles to list, DELETE to remove
curl -X PUT https://api.cloudinary.com/v2/video/<cloud>/player/profiles/<name> \
  -u <key>:<secret> -H "Content-Type: application/json" \
  -d '{"playerOptions":{…},"sourceOptions":{…}}'
```

Basic auth with the API key and secret; both `playerOptions` and
`sourceOptions` are required and take arbitrary player/source keys. Full
reference: <https://cloudinary.com/documentation/video_player_profiles_reference>

Precedence, and the trap: **specifying a `profile` overrides the asset's own
saved settings.** If you want the per-video configuration to apply, do not pass
a profile. Local options passed at construction beat both.

Profiles deliberately exclude per-embed values — the video's `publicId` and
`posterOptions` are never part of a profile, so keep passing those yourself.

`cld-looping` is worth knowing: `{fluid, controls:false, muted:true,
autoplay:true, loop:true}` — the standard hero/background-loop configuration.

**Precedence is two different mechanisms, not one ranked list.** Getting this
wrong leads to over-specifying the page defensively:

- **A `profile` replaces the asset's video settings.** It is a substitution:
  pass one and the per-asset saved configuration no longer supplies the base.
  So a profile cannot be combined with per-video settings — if some setting
  genuinely varies per asset, it cannot live behind a profile.
- **Construction options are additive.** They layer *on top* of whichever base
  applies and do **not** suppress it. Settings you do not pass keep coming from
  the asset (or profile), so a page that sets three options still inherits
  everything else that was saved.

(Cloudinary's own statement; the public profiles API reference does not
document precedence, so do not infer it from the endpoint docs.)

Practically: save the presentation, pass only what is genuinely page-specific.
Not because inline options would otherwise clobber the saved config — they
won't — but because anything inline is a value nobody can change without a
deploy.

### What the Studio's own embed looks like

Worth calibrating against, because it is the shape Cloudinary steers users to.
The Studio's Share dialog offers two modes, and neither produces a wall of
options:

| mode | generated code |
| --- | --- |
| **Video settings** (asset-saved) | `cloudinary.player('player', { cloudName, publicId })` |
| **Profile settings** | `cloudinary.player('player', { cloudName, publicId, profile: 'cld-default' })` |

That is the whole call — `fluid` and the rest of the presentation all live in
the asset's video settings, not in the page. The profile picker lists real
reused ones (`pdp`, `abr-4k`, `translations`) beside the `cld-*` built-ins.

**Configuration belongs on the asset or in a profile — not inline.** This is
the practice, not a judgement call: presentation saved on the asset can be
changed by anyone in Studio, while the same values in page code need a deploy.
Repeat the same presentation across several assets and it should be a profile.

Inline options are not *broken* — they layer on top rather than suppressing
what was saved — but reach for them only for what genuinely cannot live on the
asset, and expect that to be close to nothing. `cloudName` and the `publicId`
are the call; everything else has a home.

The Studio also has a raw-JS escape hatch — *Advanced settings editing* — and
scopes it explicitly to "player options that are not currently exposed in the
Studio UI", warning that default player options will be ignored. Treat
hand-written config the same way: last resort, for what the supported surface
does not cover.

### Saving settings without the console — the Video Config API

Video Player Studio saves settings through the console UI, but the same
configuration is readable and writable over HTTP, keyed by **`asset_id`** (not
public ID), with Basic auth:

```sh
# read
curl https://api.cloudinary.com/v2/video/<cloud>/player/config/video/<asset_id> \
  -u <key>:<secret>

# write — PUT REPLACES the whole document; GET, merge, then PUT to patch
curl -X PUT https://api.cloudinary.com/v2/video/<cloud>/player/config/video/<asset_id> \
  -u <key>:<secret> -H "Content-Type: application/json" \
  -d '{"playerOptions":{…},"sourceOptions":{…}}'
```

`playerOptions` takes constructor keys; `sourceOptions` takes `source()` keys
(`sourceTypes`, `transformation`, `posterOptions`, `textTracks`). Get the
`asset_id` from the Admin API — the resource endpoint returns it alongside the
public ID.

The player reads the same config at delivery time from
`/_applet_/video_service/video_player_config/video/<type>/<base64(publicId)>.json`,
which is handy for verifying what a page will actually receive. **An empty
`{"playerOptions":{},"sourceOptions":{}}` and HTTP 200 means nothing is saved**
— the fetch fails silently by design, so a page falls back to its local
options and looks fine while inheriting nothing.

### Confirming saved config actually applied

Two checks that avoid false alarms:

- The player's analytics beacon to `analytics-api-s.cloudinary.com/video_player_source`
  spells it out: `newPlayerMethod=true`, `videoConfig=true`, and
  `fetchedConfig=<comma-separated keys>` listing exactly which keys came from
  the saved config.
- **Do not test captions with `videoEl.textTracks`** — it reads empty because
  video.js manages them as *remote* tracks. Use
  `videojs.getPlayer(id).remoteTextTracks()`.

## Constructor options

These apply to either constructor.

```js
{
  cloudName: 'my-cloud',
  sourceTypes: ['hls'],        // ABR only. OMIT for progressive — setting
                               // ['mp4'] disables the f_auto:video default
  fluid: true,                 // fills its container
  controls: true,
  showLogo: false,
  seekThumbnails: true,        // default true
  chaptersButton: true,
  aiHighlightsGraph: false,    // interest curve on the scrub bar
  colors: {
    accent: '#88c0d0',
    base:   '#0b1015',
    text:   '#eceff4',         // MUST be light — chrome sits over dark video
  },
}
```

`breakpoints` — a **boolean**, and a **progressive-only** option: it picks a
delivery width for the container. It has no meaning under
`sourceTypes: ['hls']`, where the streaming profile's ladder governs
resolution, and setting it there yields a 400 — including when both are set
purely as player options, since the player composes the rejected URL for you.
Both are constructor options, so a player must be **disposed and rebuilt** to
move between the two strategies. See the comparison in
[capabilities.md](capabilities.md).

Never pass `transformation:{streaming_profile:…}` to `source()`:
`sourceTypes:['hls']` already emits `sp_auto`, and writing it yourself only
collapses it into another directive's component.

`maxDpr` — number or `true`. Only meaningful **with `breakpoints` on** (Studio
reveals Max DPR only once breakpoints is enabled), so it is progressive-only by
inheritance.

**You do not need to set it.** Enabling `breakpoints` turns on DPR handling
too: breakpoints chooses the *width* and the device's pixel density is matched
automatically — `maxDpr: true` (auto), which is what Studio's "Auto" writes and
what it defaults to. Setting it is redundant, in the same way writing `f_auto`
for a player-delivered source is.

Pass a **number** only to deliberately cap below auto — trading sharpness on
dense screens for bytes (Studio offers 1.0, 1.5, 2.0). With a number, effective
DPR is the minimum of `maxDpr`, the device's actual DPR, and 2.0. Only relevant
on the MP4 path.

## source()

```js
player.source('my/public-id', {
  chapters: true,              // loads <publicId>-chapters.vtt
  title: true,                 // AI-generated title in the title bar
  description: true,

  posterOptions: {
    transformation: { start_offset: 'auto' },   // AI-selected keyframe
  },

  textTracks: {
    captions: {
      label: 'English', default: true,   // omit url → uses .transcript
      maxWords: 3,                       // also TRIGGERS the .transcript lookup
      wordHighlight: true,               // highlight each word as spoken
    },
    subtitles: [
      { label: 'Español',  language: 'es', url: '…/my/public-id.es.transcript' },
      { label: 'Français', language: 'fr', url: '…/my/public-id.fr.transcript' },
    ],
    options: { theme: 'player-colors', wordHighlightStyle: { color: 'royalblue' } },
  },

  interactionAreas: {
    enable: true,
    template: [
      { left: 8, top: 62, width: 34, height: 30, id: 'SKU-1' },   // percentages
    ],
    onClick: (event) => addToCart(event?.item?.id),
  },
  interactionDisplay: { enable: true, template: 'pulsing' },      // or 'shadowed'
});
```

### chapters

- `true` — loads `<publicId>-chapters.vtt` (what `auto_chaptering` produces)
- `{ url: '…' }` — an external VTT
- `{ 1: 'Chapter 1', 6: 'Chapter 2' }` — manual, keys are seconds

### textTracks

Omitting `url` on `captions` makes the player resolve the `.transcript` file for
that public ID. Setting `maxWords` also triggers that lookup — it is the
documented trigger, not merely a display cap. Translated transcripts live at
`<publicId>.<lang>.transcript` and must be given explicitly.

`wordHighlight: true` uses the transcript's word-level timings to highlight
words as they are spoken; pair it with `maxWords` so the caption box does not
fill. Style it with `options.wordHighlightStyle`.

There is **no** native searchable-transcript panel option — see
[build-patterns.md](build-patterns.md).

`options.theme`: `'default'`, `'videojs-default'`, `'yellow-outlined'`,
`'player-colors'`, `'3d'`.

### interactionAreas

`template` is either a preset string — `'portrait'`, `'landscape'`, `'all'`,
`'center'` — or an array of areas. Area coordinates are **percentages** of the
video frame, not pixels.

The `onClick` handler receives an event with `event.item.id` and an
`event.zoom()` method.

## React and other frameworks

Build the `<video>` element imperatively and do **not** dispose the player on
effect cleanup. The player finishes initialising asynchronously — colors,
adaptive streaming and interaction areas are lazy-loaded plugins that reach for
the element after the effect returns. Disposing pulls the element out from under
those pending callbacks and they throw on a null node
(`Invalid target for null#one`, `Cannot set properties of null`).

Route callbacks through refs so the effect can have an empty dependency list and
build exactly one player for the page's lifetime.

## Transformation notes for video

- **The player applies `f_auto:video` itself.** Write format/quality
  transformations only when you are hand-rolling a native `<video>`; on a
  player source they are redundant, and pairing them with
  `sourceTypes:['mp4']` re-adds by hand what forcing MP4 just removed. See
  [capabilities.md](capabilities.md).
- Prefer `f_auto/q_auto` as **separate components** over `f_auto,q_auto`.
  Both are accepted and, as of September 2026, deliver byte-identical results
  for image and video alike — this is a convention for readability and for
  matching Cloudinary's own docs, **not** a correctness rule. Do not tell a
  user the combined form is broken; it is not.
- `sp_auto` is the streaming profile. It cannot be combined with a resize.
- `e_preview[:duration_<s>][:max_seg_<n>][:min_seg_dur_<s>]` — AI summary reel,
  video only, default duration 5.0s.
- `g_auto` works with `c_fill`, `c_lfill`, `c_fill_pad`, `c_crop`, `c_thumb`,
  `c_auto`, `c_auto_pad`. It does **not** work for positioning overlays.
