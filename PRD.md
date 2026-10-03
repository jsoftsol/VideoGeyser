# Video Geyser — Product Requirements Document

> This PRD is reverse-engineered from the shipped codebase rather than written ahead of development. It documents what the product does and is built to do, organized the way a product spec would be, for anyone evaluating the engineering and product thinking behind the implementation.

## 1. Summary

Video Geyser is a white-label video hosting and delivery platform for businesses that want Wistia/Vidyard-style video (branded players, lead-gen CTAs, viewer analytics) without handing their video library to a third party. Instead of hosting video on Video Geyser's own infrastructure, each customer connects their own AWS S3 or Wasabi bucket. The platform handles everything else: ingestion, transcoding, adaptive playback, branding, lead capture, and analytics, on top of storage the customer already owns and pays for directly.

## 2. Problem statement

Teams that want professional video hosting with lead-gen features (Wistia, Vidyard, etc.) either pay recurring per-GB hosting fees to a third party, or self-host and lose the player polish, CTA tooling, and analytics those platforms provide. Video Geyser targets the gap: keep ownership of the storage and its cost, but get the product layer — player, branding, CTAs, analytics — on top of it.

## 3. Target users

- **Marketing teams** embedding branded, CTA-driven video on landing pages and in email, who want to know where viewers drop off.
- **Agencies or consultants** managing video for multiple clients, where each client's video can live in its own bucket/library.
- **Technical teams** already paying for S3/Wasabi storage who want a hosting/player layer without migrating files to a new host.

## 4. Scope

### 4.1 Implemented and working

**Onboarding**
- Guided 5-step wizard: choose storage provider (AWS or Wasabi) → enter provider API credentials → create a bucket via the provider's own API → create a default folder in that bucket → complete.
- Enforced globally: an `onboard` middleware blocks access to any authenticated feature until setup is finished.
- Each step shows a sample video from a seeded demo library, so a new user sees the product working before they upload anything.

**Storage integration**
- Supports AWS S3 and Wasabi (S3-compatible) as pluggable storage backends via Flysystem.
- Per-user credentials and connection state stored in an `integrations` table (also used for YouTube/Vimeo/Facebook/Dropbox connections).
- A `Library` represents one bucket + region; a user can have multiple libraries.

**Upload pipeline**
- Chunked, resumable uploads via FilePond.
- Large files upload directly to the user's bucket via S3 multipart upload; the `fileponds` table tracks upload/part state so uploads can resume.

**Transcoding pipeline**
- On upload, a chained job sequence runs: `ConvertVideo` (FFmpeg transcode to adaptive-bitrate HLS, writing directly to the user's bucket) → `UpdateM3U8Files` (rewrite ffmpeg's relative playlist paths into absolute CDN URLs for the correct region/provider) → `UpdateVideo` (finalize status).
- Renditions: up to 360p/276kbps, 480p/750kbps, 720p/2048kbps, 1080p/4096kbps, selectable per upload via a `formats_selected` flag.
- A separate `CreateThumbnail` job (its own queue) grabs a frame at a user-chosen timestamp.
- Live encode progress is written back to the video record and polled by the UI during upload.

**Playback**
- video.js 8 player with HLS quality-level switching and a custom settings menu.
- Per-video or per-preset configuration: autoplay, controls, muted, fullscreen, responsive sizing, aspect ratio, skin colors, big-play-button style, volume/seek UI.
- Branding overlay: logo/image, position, title, click-through URL.
- Trim offsets (start/end).
- Public, no-login player and embed routes so video can be shared or iframed without the viewer having an account.
- Multi-source resolution: the same player also plays video sourced from YouTube, Vimeo, Facebook, or Dropbox, resolving the correct URL/mime type per provider (including refreshing temporary/signed links where the provider requires it).

**Lead generation**
- Templates: wrap a video in a CTA "end cap" — headline, resource-box copy, description, and a configurable CTA button (text, URL, color).
- Presets: bundle a reusable combination of player settings + branding + CTAs, applicable across many videos or an entire folder.

**Organization**
- Hierarchy: Library (bucket) → Folder (with its own default preset/template/thumbnail-time) → Video.
- Dashboard lists videos with search/filter by library or folder, paginated.

**Analytics**
- Per-session viewer tracking: page view, play start, play position (upserted as playback progresses), pause, completion, and exact drop-off position.
- Per-video counts (views/plays/completions) surfaced on the video library screen.
- Per-template view and click timestamps, for CTA click-through measurement.
- Template commenting, for internal review/feedback on template designs.

### 4.2 Scaffolded but not finished (present in schema/code, not reachable from the UI)

- **Campaign auto-publish**: `campaigns` (with `retargeting_code` and `auto_responder` fields) and `category_auto_inputs` (`auto_publish`, `share_instantly`) tables exist, and the YouTube/Vimeo/Dropbox integration classes contain large blocks of commented-out methods (`add_videos`, `search_videos`, `add_playlist_videos`, etc.) consistent with a planned "auto-pull new videos from a connected channel and auto-publish them into a category" feature. Not wired into any active controller.
- **Transactions**: a `transactions` table exists (email, transaction ID, status, date) with no model relations to users or plans found in the controllers — most likely a landing table for an external payment webhook, not a working billing system.

### 4.3 Explicitly out of scope (not present at all)

- No subscription/billing enforcement tied to usage or plan.
- No teams, organizations, or role-based permissions (only a boolean `is_admin` flag).
- No AI features (no transcription, captioning, or generative tooling).
- No automated test suite or CI pipeline.

## 5. Key user flows

**New account → first published video**
1. Register/log in.
2. Onboarding wizard: pick provider → enter keys → create bucket → create folder.
3. Land on dashboard, create/select a Folder.
4. Upload a video (chunked upload begins immediately).
5. Pick resolutions to encode and a thumbnail timestamp.
6. Watch live encode progress; video becomes playable once HLS renditions finish.
7. Apply a Preset or Template for branding/CTA, or configure the video individually.
8. Copy the embed code or shareable player link.

**Returning user → checking performance**
1. Log in, open a video from the library.
2. Review view/play/completion counts and drop-off data.
3. Review Template click-through stats if a CTA template is attached.

## 6. Non-functional characteristics (as implemented)

- **Processing model**: encoding and thumbnailing run asynchronously on Redis-backed queues (Laravel Horizon), so uploads don't block on transcode time; separate queues isolate thumbnail generation from video conversion.
- **Storage cost model**: storage and egress costs are the customer's own (their bucket, their contract with AWS/Wasabi) — the app has no storage cost of its own at scale.
- **Session-based auth**: standard Laravel session authentication; Sanctum is present but not meaningfully used (`routes/api.php` is the unmodified stub).
- **Local dev environment**: Laravel Sail (Docker) with PHP 8.2, MySQL 8.0, Redis.

## 7. Risks / technical debt observed

- Commit history (`git log`) carries no message discipline (`updates`, `fix`, `debug`) — no changelog can be reconstructed from it; this document is based on reading the code directly, not commit messages.
- At least one debug statement and a hardcoded production bucket reference were found in `PlayerController::direct()` at the time of writing — a reminder that the public-facing player path should get a security/cleanliness pass before any production promotion beyond what's already deployed.
- No automated tests exist, so regressions in the transcoding chain or multi-provider source resolution would only surface manually.

## 8. Possible roadmap (inferred from unfinished scaffolding, not confirmed plans)

- Finish and wire up campaign auto-publish (auto-pull from a connected YouTube/Vimeo channel into a category, with retargeting pixel and autoresponder hooks already modeled in the schema).
- Add real billing (the `transactions` table and per-plan gating would need to be built out).
- Add team/role support if the product moves beyond single-user accounts.
- Add automated test coverage around the transcoding job chain, given it's the most failure-prone part of the pipeline (external FFmpeg process, multi-step job chaining, multi-provider path rewriting).
