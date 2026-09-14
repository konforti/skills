# Gotchas

Each of these cost real debugging time on a real migration. They all look like
code bugs and none of them are.

## Upload presets can rewrite your assets on ingest

An account can have an upload preset carrying an **incoming transformation**,
which permanently rewrites every uploaded asset — resizing it, watermarking it,
or both. The signing endpoint may inject that preset into signed uploads whether
you asked for it or not.

Symptom: uploads report success, but stored dimensions are smaller than the file
you sent, and delivered images carry a watermark.

What does **not** work:

- `upload_preset: ""` — ignored, the preset is still applied
- an explicit `transformation:` parameter — the upload *response* reports your
  dimensions while the *stored* asset keeps the preset's
- `invalidate: true` — this is not a cache problem

What does work: pass an explicit clean preset by name (often `default` — check
what exists on the account), or clear the offending preset's incoming
transformation in Console → Settings → Upload → Upload presets.

Note the watermark also **degrades Cloudinary's own AI**. Google Auto Tagging on
a watermarked image returned 3 tags where the clean original returned 27.

## A public ID path can become unusable

On one account, uploads to a particular folder path kept storing the old
788px watermarked asset no matter what — including after deleting the asset and
uploading fresh. An identical upload to a *different* path stored correctly
first time.

If a path misbehaves twice, stop debugging it and change the path. Verify by
fetching the delivery URL with a cache-busting query and checking the actual
JPEG dimensions, not the upload response.

## Add-on quotas are small and easy to burn

Free-plan monthly limits worth knowing before you loop over an asset list:

| Add-on | Limit |
| --- | --- |
| Google Auto Tagging | 50 |
| Google Video Tagging | 5 |
| Google Speech | 120 |
| Image generation | 50 (a single request can consume 3) |
| Image-to-video | 160 |

Repeated re-uploads to fix something else will exhaust these. Get the upload
parameters right before running the batch.

## Async AI features fail rather than degrade

`auto_chaptering`, `auto_transcription` and the generated title/description all
return `status: "pending"` and complete later — **or return `status: "failed"`**.

They need substantial real speech. On a 5-second silent clip:

```json
"auto_chaptering":    { "status": "failed", "data": "Failed to process request" },
"auto_video_details": { "status": "failed", "data": "Failed to process request" }
```

Always poll for `complete`. Never assume.

## HTTP 423 on first request

An un-derived variant returns **423 Locked** while Cloudinary builds it
asynchronously. `e_preview` on a 3-minute video took minutes. Poll until 200.

For anything going into a launch or a traffic spike, the real fix is **eager
transformations at upload** rather than warming by polling — see
[field-guide.md](field-guide.md).

### A 200 is not proof the derivative is complete

Worse than a 423: a video derivative that is still building can return **200
with a truncated file**, and the browser will cache that. Observed on a
14.1-second source — the page reported `duration: 2.625`, buffering stopped at
2.24s, and playback simply refused to start. Nothing in the HTTP status said
anything was wrong.

So verify the *content*, not the status code:

```sh
curl -s "$URL" -o /tmp/w.mp4
ffprobe -v error -show_entries format=duration -of default=nk=1:nw=1 /tmp/w.mp4
# compare against the known source duration; retry until it matches
```

And **re-warm whenever the delivery URLs change.** Warming `sp_auto` does
nothing for the `c_limit,f_auto,q_auto,w_…` MP4s you switch to later; each
distinct transformation is its own derivative. With responsive breakpoints the
player may request several widths, so warm the whole set. After re-warming,
hard-reload — the browser is still holding the truncated response.

## Video.js rearranges your DOM, and two bugs follow

The Cloudinary player is video.js underneath, which performs DOM surgery on
init. Two failures come from this and both are silent:

**The original `id` moves to a wrapper div.** After init,
`document.getElementById('hero')` returns a `<div class="video-js">`, not the
`<video>`. Attaching media listeners to it does nothing:

```js
document.getElementById('hero').addEventListener('timeupdate', …) // never fires
player.on('timeupdate', …)                                        // correct
```

**An unclosed `<video>` tag swallows everything after it.** HTML parsing puts
subsequent siblings *inside* the video element, where they never render:

```html
<video id="hero" class="cld-video-player" playsinline>   <!-- no </video> -->
<button class="soundbtn">…</button>   <!-- becomes a child of <video> -->
<div class="overlay">…</div>          <!-- gone: 0×0, never painted -->
```

The tell is an element measuring 0×0 whose `parentElement` is `VIDEO.vjs-tech`.
Always close the tag.

## The Cloudinary player proxies only some videojs methods

`player.currentTime()` and `player.play()` are proxied through the Cloudinary
wrapper. `player.muted()` and `player.controls()` are **not** — calling them
throws `TypeError: player.muted is not a function`, which aborts the rest of
your click handler and looks like "the button does nothing".

Reach the underlying videojs player for those:

```js
var vjs = window.videojs && window.videojs.getPlayer('hero');
if (vjs) { try { vjs.muted(false); vjs.controls(true); } catch (e) {} }
```

Setting `muted`/`volume` directly on the underlying `<video>` element also
works and is the most robust option.

## Artifacts cannot host a Cloudinary video demo

The Claude Artifact viewer's CSP admits only the artifact's own files, Google
Fonts and a few script CDNs. Every `res.cloudinary.com` image, video and
`fetch()` is blocked, so the page publishes "successfully" and renders as an
empty shell. Inlining media as data URIs would defeat the point (no adaptive
delivery, no transformations). Serve these pages over HTTP instead.

## Verifying what was actually stored

The upload response can disagree with what is stored and delivered. To check
real delivered dimensions:

```sh
curl -s "https://res.cloudinary.com/<cloud>/image/upload/<public_id>.jpg?bust=$RANDOM" \
| python3 -c "
import sys,struct
d=sys.stdin.buffer.read(); i=2
while i<len(d)-9:
    if d[i]==0xFF and d[i+1] in (0xC0,0xC1,0xC2):
        h,w=struct.unpack('>HH',d[i+5:i+9]); print(f'{w}x{h} {len(d)}B'); break
    i+=1
"
```

For video, `ffprobe` on a downloaded file is the honest check — including
whether an audio stream exists at all, which decides whether transcription and
chaptering can work.

## The MCP has no API secret

The Cloudinary MCP holds its credentials internally and never exposes the API
secret, so an agent working through it can only do what the MCP has a tool for.
Any other endpoint — a v2 route, a beta API — cannot be signed and returns:

```
{"error":{"message":"error while authenticating: api_secret not provided"}}
```

`POST /v2/video/<cloud>/ai_video_analysis` is the common case: no MCP tool, so
the visual transcription cannot be produced this way at all.

An agent that provisions its own cloud has no such limit. `npx
@cloudinary/cloud` writes `CLOUDINARY_URL=cloudinary://<key>:<secret>@<cloud>`,
and every endpoint is reachable with it.

Before concluding a feature is unavailable, check whether it is merely
unreachable — the two look identical from inside an MCP session.

## Provisioning fails behind a VPN

`npx @cloudinary/cloud` derives the media-delivery allow-list from the source
IP, and refuses to provision from a private address:

```
delivery_ips_not_public — delivery_ips must contain at least one public IP address
```

A corporate VPN or secure gateway (Cloudflare WARP and similar) triggers this.
The tool's own guidance says an agent should **report it and let the user
decide** rather than changing network settings. Pausing the VPN for that one
command is the fix.

Related, and easy to misdiagnose: **read `delivery_ips` back from the response**
rather than trusting what you sent. Behind a proxy your own address is silently
dropped, and you end up with a cloud whose media nothing can view.

### A VPN toggling *after* provisioning silently kills a working cloud

Worse than the provisioning failure, because it fails late and quietly. The
cloud is pinned to the IP it was provisioned from, so if the network changes —
a VPN reconnecting, a gateway rotating its egress, moving to another network —
every delivery URL starts returning **401**, on a cloud that worked minutes ago.

The tell is *which* requests fail: uploads and Admin API calls keep succeeding,
because only CDN delivery is IP-restricted. So an agent reports a successful
migration while every image and video 401s.

```sh
# 401 on delivery but uploads fine? compare the current egress with the
# address the cloud was provisioned from
curl -s https://api.ipify.org
```

The 401 body says the OAuth token expired, which is misleading — it is an
allow-list rejection, not an auth problem.

For a demo, keep the VPN off for the **whole session**, not just the
provisioning command.

## The delivery IP restriction blocks Cloudinary's own fetches

On an unclaimed cloud, `POST /upload` with `file=<a delivery URL on that same
cloud>` fails with `401 Unauthorized`. Cloudinary's fetcher is not one of the
allowed delivery IPs. Upload from a local file instead.

Claiming the cloud lifts the restriction entirely.

## Claimable clouds

- Expire **24 hours** after provisioning unless claimed
- Media delivery is restricted to the IPs in `delivery_ips`; **uploads and API
  calls are not restricted**, so a wrong IP looks like a working migration with
  broken media
- The CLI's default (no `--ip`) locks delivery to the calling address, which is
  usually correct — prefer it over guessing
- Surface the `claim_url` to the user; it is what makes the cloud permanent
