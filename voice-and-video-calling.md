# Voice and video calling

This plan records the architectural decisions for Runelink voice calls, video, call signaling, and external telephony interoperability ([#23](https://github.com/runelink-org/runelink/issues/23)).

The central architecture is established: the host that owns a voice channel manages its calls, the existing federation protocol carries call signaling, and WebRTC connects each client directly to the owning host's SFU for media. This preserves the existing ownership model while avoiding the bandwidth and compute cost of routing real-time media through each participant's home host.

Shared terminology, ownership, federation, channel, and protocol concepts are defined in [Runelink architecture](architecture.md). Stored media, such as HLS video, is handled by [Integrated asset management](integrated-asset-management.md) independently of real-time calling.

## Terminology

- **Call:** a single session within a voice channel, identified by a call ID that is distinct from the voice channel ID.
- **SFU:** Selective Forwarding Unit, a media service operated by a host that forwards encoded media streams between participants without decoding them.

## Media architecture

Call media falls under the [bulk data exception](architecture.md#bulk-data-exception) to home-host routing. Signaling travels from each client through its home host and federation to the voice channel's owning host, while media travels directly between each client and the SFU operated by the owning host.

### Media transport

**Decision:** Use WebRTC for real-time audio and video.

- WebRTC provides encrypted real-time media transport, codec negotiation, and network traversal.
- Clients use browser WebRTC APIs where available.
- WebRTC does not define a call-signaling protocol; Runelink defines its own (see [Call signaling](#call-signaling)).
- Use established WebRTC libraries rather than implementing media transport.

### Selective forwarding unit

**Decision:** Use a centralized SFU model.

- The owning host of a voice channel operates the SFU for that channel's calls.
- The SFU is a [compute service](architecture.md#compute-services). From the initial implementation, it runs as a separate process from the API, managed by the host binary and communicating through the same internal API that separately hosted and multiple SFUs will use later.
- By default, the SFU runs locally and listens on a configured port: 443 when available, or any other port when 443 is taken, such as by a web client or reverse proxy on the same machine. The API listens on its own port, currently 7000 by default, so 443 is usually free.
- Participants connect directly to the owning host's SFU for WebRTC media, including when it is not their home host.
- Clients join the SFU with short-lived, single-use join tokens obtained through their home host. Access is checked when a token is issued and re-checked on every reconnect, which requires a new token.
- Once a participant is connected, the owning host's SFU session is authoritative. The owning host removes participants immediately when they lose access to the call.
- Home hosts do not relay or process their users' call media.
- The SFU forwards media streams without decoding, mixing, or re-encoding them.
- Clients receive the relevant audio streams and mix them locally.

The primary optimization target is small group calls rather than rooms with hundreds of participants.

### Deferred alternatives

- Peer-to-peer media connections may be supported later.
- Federated SFUs are rejected as the default architecture because they would impose unnecessary bandwidth and compute costs on participants' home hosts.
- Server-side audio mixing and transcoding are deferred.

## Calls

**Decision:** Server voice channels are persistent rooms, and ringing is a notification layer separate from calls.

- In server voice channels, the first participant to join starts a call, and the call ends when the last participant leaves. Server voice channels do not ring.
- Ringing is a per-user notification on top of a call, governed by each user's notification settings, rather than part of call state. DM calls ring by default.
- Every voice channel records the same durable call information: start and end times, the participant who started the call, and each participant's join and leave times as a list of intervals.
- Each participant has an explicitly attached audio stream and camera video stream, which are part of that participant rather than separate streams.
- Media shares, such as screen shares, are generic streams separate from participants, with an optional user reference, because a share may come from a source that is not a user in the call, such as an external adapter. Clients present them separately from participants.
- Calls record media shares with their start and end times, like participants joining and leaving, because each share is a separate item in the call, such as its own cell in a grid view.
- Any video stream, whether a participant's camera or a media share, may include multiple quality levels (see [Simulcast](#simulcast)).
- Presentation and retention of call records may differ by channel. DM channels integrate call events into the chat, while server voice channels may not present them or may retain them for less time.
- Participants turning their camera on or off, mute state, media content, and other detailed call events are not recorded initially.
- Media uses WebRTC's default transport encryption. End-to-end media encryption is not under consideration yet.

## Call signaling

**Decision:** Define a custom signaling protocol within the existing WebSocket protocol.

- Signaling uses the existing client and federation WebSocket connections.
- Signaling operations are new request, reply, and update types in the client and federation WebSocket protocols, grouped under new versioned capabilities.
- Clients send signaling to their home host, which forwards it over federation to the voice channel's owning host. Updates from the owning host return to clients through the same path.
- Call IDs allow multiple independent calls to share the same underlying connections.
- The protocol covers the call lifecycle, including starting, joining, leaving, and ending calls, along with ringing notifications and their acceptance or rejection.
- The protocol transports WebRTC negotiation data, including SDP offers and answers and ICE candidates.
- Signaling is ephemeral state and is modeled separately from durable content mutations.

The exact message schemas and state machines have not yet been designed.

### SIP

**Decision:** Do not use SIP as the native signaling protocol.

- The native protocol remains independent of SIP.
- Its call lifecycle should map cleanly onto SIP concepts to simplify interoperability. Gateways map ringing notifications and their acceptance or rejection onto SIP's ringing and call setup responses.
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
2. Define call signaling and the SFU join token format.
3. Implement group audio in server voice channels. This depends on the server channel architecture ([#30](https://github.com/runelink-org/runelink/issues/30)) and precedes DM calls, which follow DMs ([#24](https://github.com/runelink-org/runelink/issues/24)).
4. Add video and screen sharing using simulcast and supported temporal scalability.
5. Implement a SIP gateway, using Telnyx as the initial interoperability test without making it part of the core protocol.
6. Add advanced optimizations only when needed, including audio mixing, transcoding, spatial SVC, and peer-to-peer calls.

## Open decisions

| Area | Open question | Existing constraints |
| --- | --- | --- |
| SFU implementation | Which existing SFU library or framework should be used? | The host implementation, `runelink-server`, is written in Rust. The SFU runs as a separate process behind the internal API. |
| SFU deployment | How do SFUs on other machines register with their host, and how are calls assigned to multiple SFUs? | The initial implementation runs a single local SFU process through the same internal API. |
| SFU join tokens | What format do join tokens use, and how does the SFU verify them? | Tokens are short-lived, single-use, and obtained through the home host. Verification must also work for separately deployed SFUs. Federation authenticates hosts with short-lived JWTs verified through JWKS. |
| Signaling | What call lifecycle states and signaling messages should the protocol define? | Messages are request, reply, and update types under versioned capabilities. |
| WebRTC negotiation | How are SDP negotiation, ICE candidates, and reconnection coordinated with federation? | |
| Call membership | How are live participants and their media capabilities advertised to other participants? | Protocol capability negotiation describes connection compatibility, not per-participant media capabilities. |
| Call permissions | Deferred: which permissions govern joining, speaking, video, screen sharing, and moderation? | Calls use the role-based permission system ([#31](https://github.com/runelink-org/runelink/issues/31)), with built-in permissions for built-in operations and channel access determined by channel configuration. DM call access follows DM membership. |
| Video codecs | Which codecs are supported or preferred initially? | |
| Simulcast | What resolutions, bitrates, and frame rates are configured by default? | |
| Quality adaptation | How does the SFU decide which simulcast encoding and temporal layers to forward? | |
| SIP interoperability | What is the standard adapter interface, particularly for outbound and inbound PSTN calls? | |
| Adapter ownership | How does an external gateway join calls owned by another host without owning the channel? | |
| Reliability | What happens when a host, SFU, or network connection fails during a call? | |
| Network traversal | When is TURN needed, and how is STUN provided? | TURN is not needed initially: SFUs have public addresses, and a TCP fallback covers most networks that block UDP. TURN is a compute service paid for by the voice channel's owning host. |
