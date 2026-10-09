# Integrated asset management

This plan records the direction for integrated asset storage, delivery, and media processing ([#9](https://github.com/runelink-org/runelink/issues/9)). Shared terminology and architecture are defined in [Runelink architecture](architecture.md).

## Direction and confirmed decisions

- Asset management is integrated into the core host and client experience, closer to Matrix's media repository than optional XMPP extensions. Basic attachments must work on a fresh host without configuring an external service.
- Storage backends implement a common server-side Rust trait, so new backends are easy to add. Local filesystem storage is the default backend and works without manual configuration. A host may instead configure another supported backend, initially S3-compatible object storage. Configuring a backend should be simple and must not change asset IDs or client behavior.
- Every asset belongs to an [asset store](architecture.md#terminology). Initially, only server text channels and DM channels own asset stores. **The asset store's owning host owns and pays for its assets**, including uploads by remote users.
- Access to an asset follows the permissions of its asset store. Asset store permissions are similar to message permissions but separate from them, so a user may, for example, be allowed to send messages without being allowed to upload assets.
- Asset store permissions are evaluated in the context of the owning channel or DM, just as channels inherit their category's permissions, even though the data model keeps asset stores separate.
- Referencing an asset never grants access to it. Users who cannot read the asset store see a referenced asset as private or unauthorized, just as they would a referenced message from a channel they cannot read.
- Asset bytes fall under the [bulk data exception](architecture.md#bulk-data-exception). Signed URLs are issued through the home host, and clients upload and download bytes directly from the endpoints the owning host designates.
- Assets have permanent IDs that map to short-lived signed URLs, valid for about five minutes. Each user receives their own unique signed URL for rate limiting and auditing.
- Home hosts request signed URLs automatically when fetching content that references assets. Clients can also request and refresh URLs through their home host, potentially in bulk.
- A permanent asset ID identifies the entire logical upload, including its normalized original and every generated representation. At the bottom are immutable blobs of bytes. Intermediate structure and storage granularity depend on media type; their names, hierarchy, and database mapping are not fixed.
- For supported media, the default **Download** action returns a standardized, broadly compatible version. **Download original** returns the metadata-normalized source without media re-encoding. Generic files without a normalized download representation use the original for Download.
- Normalize source metadata, including stripping sensitive location and capture-time fields, on the client and/or owning host. The original is the result of that normalization, not necessarily the exact uploaded bytes. Preserve it for the asset's lifetime; retaining a pre-normalization upload is not required.
- Home hosts request URLs from owning hosts on behalf of their users and cache them per user. One user's signed URL is never served to another user.
- Every backend, including local storage, returns compatible signed URLs for uploads and downloads. The storage abstraction exists only on the server; clients use signed URLs the same way regardless of which backend, host endpoint, or CDN they point to.
- Assets are first-class resources and may exist independently of messages. Their metadata is **global to the asset**, never per-message or per-use.
- Uploading an asset is separate from referencing it. An asset uploaded as an attachment records its uploading message in a host-managed field, and the uploading message references it the same way any other message could. The field may be backfilled after the message is published; the exact order depends on upload implementation details. Clients derive presentation differences between the uploading message and other references from that metadata, not from how the reference is made.
- Apart from the uploading message, assets do not record which messages reference them; references are tracked on the messages' side.
- Assets are not tied to their asset store, and asset stores are not tied to their owning resource, so either can be moved later without changing IDs. Moving an asset to another host copies it into a new asset with a new ID.
- Standard metadata includes MIME type, description, and alt text, plus a generic JSON object for arbitrary custom metadata. Prefer the name `custom`; the exact name is not finalized.
- Host-managed fields, such as the asset store and uploading message, are separate from editable metadata and require different permissions to change.
- Anyone permitted to edit an asset's `custom` metadata can view and edit all of it. The core does not reserve extension namespaces or require registered custom schemas. Extensions may namespace their own keys voluntarily.
- Store metadata in the database to support efficient queries such as filtering by MIME type. Add specialized tables where media-specific relationships and query patterns justify them.
- The first implementation treats all assets as generic files with generic metadata. The design must accommodate image resolutions, thumbnails for multiple media types, and HLS video variants with optional multilingual audio and subtitles.
- Media processing should work locally with FFmpeg by default. HLS processing must eventually support optionally configured compute infrastructure separate from the host's API process.
- Video support must accommodate full-length movies, including resumable large transfers, durable background processing, seeking, and uninterrupted authorization renewal during long playback.

## Proposed implementation shape

The following records the implementation approach discussed. Exact schemas, endpoint names, and delivery details remain subject to implementation review.

### Identity, metadata, and storage

- Separate logical asset identity from physical bytes, generated representations, authorization, and upload state. These responsibilities do not prescribe a fixed hierarchy or set of tables. Keep physical storage keys and credentials out of messages and public asset metadata.
- Use permanent, owner-resolvable asset IDs. Never reuse an ID after deletion; a permanent identifier does not promise permanent byte retention. The URI syntax is still open.
- Publish the normalized original as immutable bytes while allowing global descriptive metadata to change. Replacing original content creates a new asset; generating or rebuilding derivatives preserves the existing asset ID and creates new immutable outputs.
- Store common searchable fields in typed columns. Use PostgreSQL `jsonb` for `custom`, bounded by size and nesting limits. Arbitrary custom keys need not be indexed or queryable through a general-purpose public query API.
- Distinguish editable descriptive metadata from host-verified technical facts. Preserve declared and detected MIME types separately; use verified properties for serving and processing decisions. Keep canonical MIME types normalized and codec details separate.
- Record technical facts for the actual representation being served. An asset can encompass several MIME types, dimensions, and codecs; the searchable asset-level MIME type should describe its normalized original rather than imply that all representations share a format.
- Track metadata versions and use optimistic concurrency for updates. Remote metadata caches should refresh independently of URL caches.
- The storage backend trait produces signed upload and download URLs with explicit expiry that behave compatibly across backends, including resumable uploads and ranged downloads. Backend-specific mechanisms, such as presigned or multipart object-storage operations, stay behind the trait and must not change client behavior. The exact trait methods remain open.
- Record the storage backend and private key per blob. Changing the configured backend affects new uploads; existing blobs remain readable. Background migration can move bytes without changing asset IDs.
- Internal blob deduplication and reuse across representations must not merge asset permissions or metadata.

The model must support the following responsibilities without committing to intermediate entity names or one database row per encoded object:

- Permanent asset identity, asset store, owning host, creator, global metadata, lifecycle, and metadata version.
- Immutable stored bytes with backend locations, verified sizes, digests, and serving information.
- Media-specific relationships, representation selection, readiness, technical properties, and processing versions.
- Asset store placement and authorization, independent of the messages that reference assets.
- Temporary upload allocation, transfer constraints, completion state, and expiry.
- Durable asynchronous work with attempts, leases, progress, and validated outputs.
- Enough accounting and inventory information to authorize delivery, verify completeness, migrate storage, and reliably clean up all owned bytes.
- Database records, manifests, inventories, or storage conventions as appropriate to each media type. Choose indexing and storage granularity using required queries and measured processing, delivery, and cleanup performance. Internal layout changes must preserve asset IDs and client behavior.

### Original normalization and download behavior

- Normalize sensitive embedded metadata before publishing the original. The owning host is responsible for ensuring its normalization policy has been applied, whether normalization happens on the client, owning host, or both.
- Metadata normalization must not re-encode image, video, or audio content. Container or metadata rewriting may change file bytes and size without changing encoded media content.
- Distinguish sensitive capture dates and location timestamps from timing required for playback, synchronization, and subtitles. Preserve interpretation information such as orientation, color profiles, and HDR signaling when needed to display unchanged media correctly.
- Embedded file metadata is separate from editable asset descriptions, alt text, and `custom`. Removing embedded fields does not remove those database fields.
- Verify upload integrity separately from the digest of the retained normalized original. Preserve the source filename for original downloads and use an appropriate extension for generated downloads.
- Clients resolve the permanent asset ID for a purpose such as original access, default download, preview, or playback. Available purposes and their internal structure depend on media type; clients do not need to understand storage layout.
- Durable sharing links may select original access or another supported purpose without introducing a new top-level asset ID. Resolve them through normal authorization rather than persisting expiring byte URLs.
- Keep default representation selection under owning-host control, outside user-editable metadata. Defaults may move to new immutable outputs without changing asset identity.
- Publish availability independently. Once validated and normalized, the original can be attached to a message and downloaded while derivatives are processing. Show processing or failure for an unavailable default download rather than silently changing its format to the original.
- Define media-specific normalization profiles for default downloads and previews, including quality, resolution, frame rate, color, HDR, transparency, animation, embedded metadata, and audio/subtitle handling. Do not upscale or discard visible content characteristics merely to force every image into one format.
- H.264/AAC MP4 is a candidate default video download format, not a finalized profile. Image downloads may need different formats for photos, lossless images, transparency, and animation. Exact format choices, resolution caps, HDR policy, and track selection remain open.
- A compatible download is a derivative, not a replacement archival master. Reuse compatible input or encoded streams where practical; default downloads need not incur a second lossy encode after streaming outputs are generated. Whether to generate downloads eagerly or on demand remains open.
- The UI should explain that Download original preserves source media quality with sensitive metadata removed; it must not promise an untouched copy of the pre-normalization file.

### Uploads and federation

1. Allocate an asset and upload session in an asset store through the user's home host, which forwards the request to the asset store's owning host when needed.
2. The owning host checks asset store permissions and quotas and returns transfer instructions with expiry and required headers.
3. Transfer directly using the signed URLs returned by the owning host's configured storage backend, such as presigned object-storage operations or the owning host's upload endpoint for local storage. This applies to remote users as well.
4. Complete through the same application request path. Verify object existence, size, digest, session ownership, and actual stored content. Apply or verify required metadata normalization before publishing the immutable original.
5. Publish the asset in its asset store once its original is ready. Messages, including the one it was uploaded with, may then reference it. Derivative generation can continue after publication.

- Make allocation, completion, and message publication idempotent. Use unique object keys and prevent a reusable upload capability from modifying published bytes, including after completion.
- Support resumable uploads with every backend, using multipart or equivalent operations where external storage provides them. Transfer state must survive interrupted connections without requiring movie-sized files to restart from zero.
- Keep the owning host authoritative for asset authorization, which it evaluates against the asset store. Peer hosts only represent their users and gain no access of their own.
- Asset store operations form their own versioned capabilities.
- Bulk resolution groups requests by owning host and returns per-asset results. Scope authorization caches correctly; a cached URL must not authorize a different user implicitly.

### Delivery and caching

- Return explicit URL expiry timestamps. Refresh lazily for needed assets, renew before expiry during active use, allow a clock-skew margin, coalesce concurrent refreshes, and use bounded retry after expiry failures.
- Never persist expiring delivery URLs in messages or permanent asset records. Keep signatures out of logs and analytics.
- Separate metadata, authorization, delivery-URL, and immutable-byte caches. URL expiry neither deletes cached bytes nor revokes copies already obtained.
- Key byte caches by asset and representation rather than by signed URL, so per-user URLs do not defeat byte caching. CDNs in front of external storage should exclude signatures from their cache keys.
- Home hosts may serve cached remote content, such as messages from the content cache ([#26](https://github.com/runelink-org/runelink/issues/26)), before they have the requesting user's signed URLs. Send the content immediately, then push the asset-to-URL mapping to the client as an update once the owning host returns it.
- Owning hosts may rate limit signed URL issuance per user. Rate limits apply per host and user, and possibly per server later, rather than per asset store. Local storage endpoints may also rate limit downloads; issuance is the main control for external backends that cannot limit per URL.
- A signed delivery URL may point to the owning host's external storage, CDN, or media endpoint, and clients treat all of them the same way.
- Uploads, downloads, and HLS all transfer bytes directly between clients and those endpoints, including for remote users. Only URL issuance and authorization pass through the home host.
- Support HTTP ranges for large files and playback. Use bounded streaming and backpressure rather than buffering whole files in the API process.
- Renew delivery authorization during long playback and resumed downloads without restarting the media. Account for expiry of child requests as well as the initial manifest or file URL.
- URL expiry applies when a request starts; transfers already in progress may continue. New requests, such as resumed ranges and subsequent HLS segments, need a current URL.
- Explicit downloads to disk should survive interruptions such as connection blips. Desktop, mobile, and CLI/TUI clients are planned surfaces for resumable downloads. Pin the specific representation with its size and digest, keep the partially downloaded bytes, and resume with a range request and a freshly issued URL. Verify the size and digest on completion. Resuming must not switch to a newer default representation.
- Small in-memory views can simply restart, and long media playback uses HLS.
- Resumable web downloads are deferred until after the initial implementation. Until then, web clients use regular browser downloads and can offer a copyable CLI download command for large files.

### Media processing and HLS

- Use asynchronous jobs for inspection and derivative generation. Local FFmpeg runs in a worker outside request handling; external workers use the same job contract.
- Processing must survive API and worker restarts with durable progress and safe retries. Bound worker concurrency, memory, and scratch-space use for movie-length inputs.
- Dispatch processing using verified media properties. Keep jobs, media relationships, representation selection, and processing state out of user-editable `custom` metadata.
- Version processing profiles and associate outputs with the profile/version that produced them. Configuration changes should not silently alter existing immutable outputs.
- External workers lease jobs, report progress, retry safely, and receive scoped input/output access rather than unrestricted storage credentials. Validate outputs before marking them ready.
- Generic files may need only original/download access. Images may need sizes, formats, previews, and animation support. Videos may need posters, downloadable files, alternative codecs, adaptive quality levels, multilingual audio, and subtitles. The relationships and storage layout may differ across these types.
- Adaptive playback must allow the player to select available qualities using measured network throughput, buffer health, device capabilities, and user preferences without changing asset identity. Align encoded variants as required for seamless switching.
- Audio and subtitle support needs language, label, codec, default/forced flags, and accessibility information where applicable. Choose specialized tables only where relationships and measured query needs justify them.
- HLS delivery must describe manifests, quality variants, audio, subtitles, and segments, with enough inventory information for completeness checks and cleanup. Individual database rows for segments or other streaming components are optional. Choose representation boundaries, packaging, and inventory mechanisms based on media-specific needs and performance measurements.
- New codecs, qualities, or track selections may produce new immutable playback outputs that reuse existing bytes. Updating the asset's playback default should allow active clients to finish using the prior output under a defined retirement policy.
- HLS needs package/session delivery authorization. Signing only the master playlist does not authorize its children, and query signatures are not automatically inherited by relative segment URLs.
- Keep generated manifests free of permanently embedded expiring URLs. Evaluate a media gateway with path-scoped capabilities, CDN authorization, or manifest rewriting. One candidate: the home host delivers a short-lived token scoped to a playback output, the owning host's media endpoint propagates it to child URLs when serving playlists, and the client renews it through the home host. Final HLS delivery mechanism remains open.

### Lifecycle and operations

- Ship sensible upload limits, quotas, safe download headers, temporary-file handling, and cleanup with local storage. Expose simple configuration for each supported external backend, such as bucket, region, endpoint, credentials, and compatible addressing for S3.
- Account for original storage, generated downloads, streaming outputs, and processing costs on the owning host. Include duration, dimensions, frame rate, and track-count limits alongside upload-byte limits; bound temporary processing storage separately.
- Keep incomplete-upload expiry separate from asset retention. **An asset without a message reference is not an abandoned asset.**
- What happens to an asset when the message it was uploaded with is deleted is configurable. The exact options remain open.
- Deleting the channel or DM that owns an asset store deletes the store and its assets by default. Other configurable options may be considered later.
- Define retention, deletion, quarantine, reference removal, and tombstones explicitly. Garbage collection must respect asset store retention and must not infer abandonment from message references.
- Original retention follows the asset lifecycle, not any single message reference. Do not discard the normalized original merely because derivatives are ready. Pre-normalization uploads and obsolete generated outputs have separate cleanup policies.
- Copying an asset into another asset store creates a new asset, which may reuse verified bytes internally when both stores are on the same host. Exact copying, moving, and retention semantics remain open.
- Isolate untrusted content delivery, enforce safe MIME/disposition handling, and constrain media processors. Signing and request authentication do not make uploaded content safe to execute.
- If end-to-end encrypted attachments are added, clients encrypt bytes and sensitive metadata and create encrypted thumbnails. Server-side plaintext transcoding cannot operate on ciphertext. Encryption support was discussed but is not a confirmed requirement here.

## Initial scope and follow-on work

1. Implement generic assets, database metadata, asset stores with store-based authorization, immutable original storage, required source-metadata normalization, metadata versioning, and upload lifecycle.
2. Provide integrated local storage, federated uploads/downloads, and bulk signed-URL resolution with client and peer host caches.
3. Add S3-compatible storage as the first supported external backend, with simple configuration and backend migration support that preserve asset identity and client behavior.
4. Add durable derivative-processing jobs and thumbnails, followed by image-size variants and standardized media downloads with a separate Download original action.
5. Add adaptive video playback, HLS delivery authorization, multilingual audio/subtitles, movie-length processing and transfer support, and optional external compute.

The generic-file release may serve originals as its only download. Its access contract must accommodate purpose-based resolution and independent processing states so later media support can provide the normalized default-download experience.

### Open decisions

- Before the generic-file implementation, settle asset store permissions, metadata edit/federation semantics, retention defaults, and the configurable behavior when an uploading message is deleted.
- Define which embedded metadata to remove, which formats can be sanitized, and what happens when sanitization cannot meet the no-re-encoding requirement.
- Decide whether original access can be restricted separately from preview or default-download access.
- Decide whether thumbnails ship in the first release or the next increment.
- Before shipping normalized media downloads, settle processing profiles, eager versus on-demand generation, and retirement of obsolete outputs.
- Choose intermediate media structures and database granularity during media-specific implementation, informed by performance measurements.

### Deferred

- Asset stores for server channel kinds other than text.
- Moving assets between asset stores.
- Profile pictures, server icons, emoji, and similar assets. Hosts or servers will likely have separate stores for these, possibly with different permission or signing systems.

## Prior art

- [Matrix content repository](https://spec.matrix.org/v1.19/client-server-api/#content-repository): integrated media IDs and authenticated downloads; useful separation between logical identifiers and delivery.
- [Matrix federation media](https://spec.matrix.org/v1.19/server-server-api/#content-repository): media bytes or a possibly expiring storage location. Matrix advises against caching returned locations; Runelink intends explicit expiry-aware URL caching.
- [MSC3911](https://github.com/matrix-org/matrix-spec-proposals/pull/3911): proposed media/event authorization, not stable Matrix behavior. Runelink authorizes assets through their asset stores rather than through message links.
- [XMPP HTTP Upload](https://xmpp.org/extensions/xep-0363.html) and [stateless file sharing](https://xmpp.org/extensions/xep-0447.html): useful transfer allocation and file/source separation; avoid permanent delivery URLs in archived messages and optional-extension friction for basic media.
