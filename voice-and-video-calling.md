# Voice and video calling

This plan records the architectural decisions for Runelink voice calls, video, call signaling, and external telephony interoperability ([#23](https://github.com/runelink-org/runelink/issues/23)).

The central architecture is established: the host that owns a voice channel manages its calls, the existing federation protocol carries call signaling, and WebRTC connects each client directly to the owning host's SFU for media. This preserves the existing ownership model while avoiding the bandwidth and compute cost of routing real-time media through each participant's home host.

## Existing model

### Terminology

- **Host:** a Runelink deployment, identified by its host name and port, that operates any number of servers. A single `runelink-server` process may run several hosts in cluster mode.
- **Home host:** the single host on which a user's account exists. Users are identified by a `UserRef` of name and host.
- **Server:** a community that lives on exactly one host.
- **Owning host:** the host that is authoritative for a resource. A server channel's owning host is the host of its server. A DM channel is owned by a single host ([#24](https://github.com/runelink-org/runelink/issues/24)).
- **Underlying channel:** a minimal protocol resource with a stable ID and no presentation metadata. `text_channels` contain messages; `voice_channels` contain calls and call events.
- **Presentation channel:** a resource that gives underlying channels ownership and presentation. `server_channels` form a server's category tree, and `dm_channels` represent one-to-one and group DMs.
- **Call:** a single session within a voice channel, identified by a call ID that is distinct from the voice channel ID.
- **SFU:** Selective Forwarding Unit, a media server that forwards encoded media streams between participants without decoding them.

### Ownership and federation

- Every resource has exactly one owning host, which is authoritative for that resource.
- Federation enables communication between hosts without distributing ownership of individual resources.
- Clients authenticate only with their home host. End-user credentials are never shared with other hosts.
- Requests for resources on another host are sent to the home host with a target host. The home host forwards them over federation using server-to-server requests authenticated by short-lived JWTs, which remote hosts verify against the keys published at `/.well-known/jwks.json`.

### Channels

As decided in [#37](https://github.com/runelink-org/runelink/issues/37) and being implemented in [#30](https://github.com/runelink-org/runelink/issues/30):

- Text channels and voice channels are separate underlying resources with separate IDs.
- Server channels have a `kind`. The initial kinds are `text` and `category`; server voice channels will be added as a `voice` kind that owns an underlying voice channel.
- Each DM channel owns both an underlying text channel and an underlying voice channel. One-to-one and group calls therefore take place in the DM channel's voice channel.
- Server channel IDs remain stable regardless of their position in the category tree.
- Categories may be nested to arbitrary depth. Presentation limits are the responsibility of clients, not the protocol.

### Messaging and assets

- Clients and hosts, and federated hosts, communicate over persistent WebSocket connections. Traffic consists of requests, replies, and push-only updates.
- Operations are grouped into independently versioned capabilities. Each WebSocket connection negotiates them through `hello` and `welcome` envelopes, and operations whose capabilities were not negotiated remain unavailable.
- Capabilities describe protocol compatibility, not authorization. Permissions are a separate, many-to-many policy system.
- Message bodies are plain text by default, with optional metadata for richer content.
- Durable content mutations and ephemeral state, such as presence ([#28](https://github.com/runelink-org/runelink/issues/28)), are modeled separately.
- Media assets are owned by channels and stored through external storage/CDN infrastructure.
- Federation exchanges asset metadata rather than transferring large files between hosts.
- Stored media may use formats such as HLS, independently of the real-time calling system.

## Media architecture

Call media follows a different path from all other federated traffic. Signaling travels from each client through its home host and federation to the voice channel's owning host, while media travels directly between each client and the SFU operated by the owning host.

### Media transport

**Decision:** Use WebRTC for real-time audio and video.

- WebRTC provides encrypted real-time media transport, codec negotiation, and network traversal.
- Clients use browser WebRTC APIs where available.
- WebRTC does not define a call-signaling protocol; Runelink defines its own (see [Call signaling](#call-signaling)).
- Use established WebRTC libraries rather than implementing media transport.

### Media server

**Decision:** Use a centralized SFU model.

- The owning host of a voice channel operates the SFU for that channel's calls.
- Participants connect directly to the owning host's SFU for WebRTC media, including when it is not their home host.
- This is an explicit exception to the rule that clients communicate only with their home host. It applies to call media only.
- Home hosts do not relay or process their users' call media.
- The SFU forwards media streams without decoding, mixing, or re-encoding them.
- Clients receive the relevant audio streams and mix them locally.

The primary optimization target is small group calls rather than rooms with hundreds of participants.

### Deferred alternatives

- Peer-to-peer media connections may be supported later.
- Federated SFUs are rejected as the default architecture because they would impose unnecessary bandwidth and compute costs on participants' home hosts.
- Server-side audio mixing and transcoding are deferred.

## Call signaling

**Decision:** Define a custom signaling protocol within the existing WebSocket protocol.

- Signaling uses the existing client and federation WebSocket connections.
- Signaling operations are new request, reply, and update types in the client and federation WebSocket protocols, grouped under new versioned capabilities.
- Clients send signaling to their home host, which forwards it over federation to the voice channel's owning host. Updates from the owning host return to clients through the same path.
- Call IDs allow multiple independent calls to share the same underlying connections.
- The protocol covers the call lifecycle, including initiation, ringing, acceptance, rejection, and termination.
- The protocol transports WebRTC negotiation data, including SDP offers and answers and ICE candidates.
- Signaling is ephemeral state and is modeled separately from durable content mutations.

The exact message schemas and state machines have not yet been designed.

### SIP

**Decision:** Do not use SIP as the native signaling protocol.

- The native protocol remains independent of SIP.
- Its call lifecycle should map cleanly onto SIP concepts to simplify interoperability.
- SIP compatibility is provided by adapters or gateways. Native clients are not required to support SIP.

## External interoperability

**Decision:** Provide interoperability with external calling services through adapters.

Hosts must be able to bridge users and calls into external communication systems. The primary example is Telnyx for conventional telephone numbers and PSTN calling.

- Third parties must be able to implement adapters independently.
- The core protocol must not contain Telnyx-specific functionality.
- SIP gateways translate between the native call protocol and external SIP systems.
- An adapter may run on the host that owns a voice channel, or may participate in calls in voice channels owned by other hosts.
- Participants and endpoints may expose different media capabilities.
- External telephone participants are not required to support native functionality such as video, screen sharing, or rich messaging.

Resource ownership and external gateway functionality must remain distinct concepts.

## Video architecture

### Simulcast

**Decision:** Use simulcast for adaptive video quality.

- Video senders produce multiple independent encodings at different resolutions, initially about three.
- The SFU selects which encoding to forward to each recipient.
- Different recipients may receive the same stream at different qualities.
- The SFU does not decode or re-encode video for each recipient.

This applies to both camera video and screen sharing. Their encoding configurations have not been decided.

### Temporal scalability

**Decision:** Include temporal scalability in the initial implementation where codecs support it.

- Video streams should have multiple temporal layers, allowing receivers to receive different effective frame rates.
- The SFU selectively forwards temporal layers based on network conditions and recipient needs.
- Use browser and WebRTC codec support rather than custom encoding.

### Scalable Video Coding

**Decision:** Prefer simulcast over spatial SVC initially.

- Simulcast was selected for its relatively simple implementation and broad practical support.
- Spatial SVC may be explored later.
- Temporal scalability may be used with simulcast without adopting spatial SVC.

## Audio architecture

**Decision:** Use a single adaptive audio stream per sender.

- Each sender encodes audio once, without simulcast.
- An adaptive codec such as Opus adjusts its bitrate to network conditions.
- The SFU forwards the encoded audio to recipients.
- Receiving clients mix audio.

### Deferred optimizations

The following are not priorities for the initial implementation:

- Per-recipient audio transcoding.
- Server-side audio mixing.
- Lower-bitrate audio produced specifically for bandwidth-constrained recipients.
- Mixing multiple speakers into a single stream for particular clients.

## Implementation priorities

These are recommendations rather than decisions.

1. Choose the SFU implementation. This choice is likely to shape WebRTC negotiation, participant management, and simulcast handling.
2. Define call signaling and SFU authentication, in particular how federated signaling authorizes a client to connect directly to a remote SFU.
3. Implement group audio in server voice channels. This depends on the server channel architecture ([#30](https://github.com/runelink-org/runelink/issues/30)) and precedes DM calls, which follow DMs ([#24](https://github.com/runelink-org/runelink/issues/24)).
4. Add video and screen sharing using simulcast and supported temporal scalability.
5. Implement a SIP gateway, using Telnyx as the initial interoperability test without making it part of the core protocol.
6. Add advanced optimizations only when needed, including audio mixing, transcoding, spatial SVC, and peer-to-peer calls.

## Open decisions

| Area | Open question | Existing constraints |
| --- | --- | --- |
| SFU implementation | Which existing SFU library or framework should be used? | The server is written in Rust. |
| SFU deployment | Should the SFU run inside the host process or as a separate service? | |
| SFU authentication | How do clients authenticate directly with a remote owning host's SFU? | Clients authenticate only with their home host and must not present their credentials to other hosts. Federation authenticates hosts with short-lived JWTs verified through JWKS. |
| Signaling | What call lifecycle states and signaling messages should the protocol define? | Messages are request, reply, and update types under versioned capabilities. |
| WebRTC negotiation | How are SDP negotiation, ICE candidates, and reconnection coordinated with federation? | |
| Call membership | How are call participants tracked, and how are their media capabilities advertised? | Protocol capability negotiation describes connection compatibility, not per-participant media capabilities. |
| Call permissions | Which permissions govern joining, speaking, video, screen sharing, and moderation? | Calls use the role-based permission system ([#31](https://github.com/runelink-org/runelink/issues/31)), with built-in permissions for built-in operations and channel access determined by channel configuration. DM call access follows DM membership. |
| Video codecs | Which codecs are supported or preferred initially? | |
| Simulcast | What resolutions, bitrates, and frame rates are configured by default? | |
| Quality adaptation | How does the SFU decide which simulcast encoding and temporal layers to forward? | |
| SIP interoperability | What is the standard adapter interface, particularly for outbound and inbound PSTN calls? | |
| Adapter ownership | How does an external gateway join calls owned by another host without owning the channel? | |
| Reliability | What happens when a host, SFU, or network connection fails during a call? | |
| Network traversal | How are STUN and TURN services deployed, and who pays for TURN bandwidth? | |
| Encryption | Is WebRTC transport encryption sufficient initially, or should end-to-end media encryption be supported? | |
| Call behavior | Are server voice channels persistent rooms that users join and leave, and do ringing semantics apply only to DM calls? | Server voice channels and DM channels each own an underlying voice channel. |
