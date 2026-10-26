---
name: mux-migration
description: Plans and executes migrations of video libraries to Mux from other providers (Cloudflare Stream, JW Player, api.video, Vimeo, Brightcove) or self-managed stacks (S3, CloudFront, FFmpeg, MediaConvert). Use for migration assessments, bulk asset import, feature parity mapping, playback URL cutover, and rollback planning.
---

# Mux migration

Migrate a video library to Mux without losing content, breaking playback, or surprising anyone on cost. This skill encodes the workflow and the judgment calls; for exact API request/response shapes, always fetch current Mux docs (see the `mux-video` skill) — never generate Mux API calls from memory.

Two official resources to start from, in this order:

1. **The migration guide** — https://www.mux.com/docs/guides/migrate-from-self-managed.md — Mux's own phased migration walkthrough (assess, set up, test batch, bulk migrate, cut over). Fetch it first; it is the source of truth if anything here drifts.
2. **Truckload** — https://migrate.mux.dev ([source](https://github.com/muxinc/truckload)) — Mux's open-source bulk migration tool. Check its README for currently supported sources (Amazon S3, api.video, Cloudflare Stream, Vimeo at last check). Prefer it over hand-rolled scripts when the source is supported; self-host and extend it when close-but-not-quite.

## Workflow

1. **Inventory the source library** before writing any migration code. Enumerate assets via the source platform's API — never from memory or a dashboard screenshot. For each asset capture: source ID, title, duration, best-available source/master URL, captions/subtitle tracks, poster/thumbnail choice, playback policy (public / signed / DRM), and any app-side metadata or taxonomy.
2. **Choose tooling**: Truckload if the source is supported and the library fits its flow; otherwise a custom script using URL-based asset creation (`POST /video/v1/assets` with `inputs[].url` — fetch the Video API reference for the current shape).
3. **Map features, not just files** (see Feature mapping below). Every source feature must map to a Mux equivalent or an explicit "not migrated" row with a reason and an owner.
4. **Cost gate.** Estimate before executing: total minutes to encode, storage, and expected delivery, against current pricing (https://www.mux.com/docs/pricing.txt). Present the estimate and get explicit user approval before any bulk run.
5. **Pilot batch.** Migrate a small, deliberately varied batch first: the longest video, one with caption tracks, one with signed/restricted playback, one with an unusual codec or container. Verify each end to end before scaling up.
6. **Bulk migrate.** Batch with bounded concurrency and backoff (check current rate limits in the docs). Rely on `video.asset.ready` / `video.asset.errored` webhooks rather than polling. Record a mapping row per asset: `source_id → mux_asset_id, playback_id, status`. Put the source ID in `passthrough` so webhooks and future API responses correlate back.
7. **Verify** against the Validation gates below. Reconcile counts before touching playback.
8. **Cut over playback** behind a flag or config switch: HLS via `https://stream.mux.com/{PLAYBACK_ID}.m3u8` (or Mux Player), thumbnails/posters via `image.mux.com`, signed-playback token minting if the source used signed URLs. Rollback = flip the switch back; the source is still live.
9. **Decommission the source only with explicit user approval**, after an agreed retention window. Never delete source content as part of the migration itself.

## Feature mapping

Concept-level map; fetch the relevant docs page for current specifics before implementing any row.

| Source concept | Mux equivalent | Watch for |
|---|---|---|
| VOD asset | Asset created from `inputs[].url` | Ingest quality = quality of the URL you feed it; find the best master |
| Signed / tokenized playback | Signed playback IDs + JWT signing keys | Different token model than most sources; re-implement minting, don't port |
| DRM | Mux DRM (Widevine/FairPlay) | Plan/feature availability — confirm in docs before promising parity |
| Captions / subtitles | Text-track inputs at asset creation, or add tracks later | Fetch tracks from the source API; they rarely travel with the video URL |
| Thumbnails / posters | Generated on the fly via `image.mux.com` | Hand-picked poster frames need their timestamp or image carried over app-side |
| Manual ABR ladder / rendition config | Automatic — no ladder to configure | Rendition-specific hacks in the player usually just get deleted |
| Analytics | Mux Data | History does not migrate; export source analytics before decommission |
| Live channels | Live streams | New stream keys — coordinate broadcasters; source DVR recordings migrate as VOD |
| Folders / tags / collections | Your own DB + `passthrough` correlation | Mux metadata is minimal by design; keep taxonomy app-side |
| Server-side ad insertion | No direct equivalent | Explicit decision point — flag it, don't silently drop it |

## Source-specific traps

### Cloudflare Stream
- Originals are not downloadable by default. MP4 downloads must be enabled per video via the downloads API, then polled until ready — budget this as its own pipeline stage.
- Those MP4s are encoded renditions, not the uploaded original: expect one generation of quality loss. Tell the user; it's their call.
- Videos with `requireSignedURLs` need signed tokens on the download URLs so Mux can fetch them.

### JW Player
- Pick the highest-resolution source (the passthrough/original where available) from the media sources list, not the first MP4.
- Captions are separate track objects fetched per media item from the Management API.
- If URL token signing is enabled on the property, delivery URLs need valid tokens at ingest time.

### api.video
- Source MP4s exist on video objects; private videos need per-video tokenized URLs minted at ingest time.
- Supported by Truckload — prefer it.

### Vimeo
- Download/source links require a paid tier; they come from the API's file/download fields.
- Links expire. Generate them per batch immediately before ingest — an inventory taken last week has dead URLs today.

### Brightcove
- Check whether Digital Masters were archived. If yes, ingest those; if no, the best source is the highest rendition from the CMS API's sources — another generation-loss decision for the user.
- API access is OAuth client-credentials; sort auth before promising an inventory.

### Self-managed (S3 + CloudFront + FFmpeg/MediaConvert)
- This is the exact case the official migration guide covers — follow it.
- Presigned URL expiry vs. bulk queue time: URLs must still be valid when Mux dequeues the job, not just when you submit it. Generate per batch with generous expiry.
- If only HLS renditions survive (no mezzanine files), verify Mux's currently supported input formats in the docs before assuming `.m3u8` ingest works; otherwise remux to a single file first.

## Gotchas

- Mux fetches input URLs server-side. Sources must be publicly reachable or presigned/tokenized — do not design around IP allowlisting Mux's fetchers.
- `passthrough` is size-limited; store the source ID there, keep everything else in your own mapping table.
- A "successful run" with silent failures is the worst outcome: every source asset must end as ready, errored-with-reason, or skipped-with-reason. No fourth state.
- Watch the first webhook deliveries during the pilot — a misconfigured webhook endpoint discovered during the bulk run means re-reconciling everything by polling.
- Historical view data does not migrate. If the user cares about analytics continuity, export from the source before decommission; Mux Data starts counting at cutover.

## Validation gates

- **Count reconciliation**: source inventory total == ready + errored + skipped, with a reason on every non-ready row. Stop on mismatch.
- **Spot-check a sample** (include every pilot asset): duration matches source within ~1s, expected caption tracks present, HLS manifest returns 200, thumbnail URL renders.
- **Signed playback**: mint a real token and play a signed asset end to end before cutover — not after.
- **Cutover safety**: playback switch is reversible, source is still serving, and a rollback owner is named before the flip.
- Before declaring done, produce the asset accounting table (migrated / errored + reason / skipped + reason) and the feature mapping table with every "not migrated" row acknowledged by the user.

## Assessment template

```markdown
## Mux Migration Assessment

Source platform:
Library size: <count> assets, <total> minutes
Source of masters: <originals | encoded renditions (generation loss) | mixed>
Tooling: <Truckload | custom script> — why:
Feature parity: <n> mapped, <n> decision points (list)
Playback policy: <public | signed | DRM> — token strategy:
Cost estimate: encoding <x>, storage <y>/mo, delivery <z>/mo (pricing docs fetched <date>)
Pilot batch: <asset list and what each exercises>
Cutover: <flag/config>, rollback owner: <name>, source retention: <window>
Not migrating: <explicit list with reasons>
```
