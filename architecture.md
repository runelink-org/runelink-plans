# Runelink architecture

This document records the shared terminology and architecture that Runelink plans build on. Plans link here and record only the decisions specific to their area.

- [Integrated asset management](integrated-asset-management.md)
- [Voice and video calling](voice-and-video-calling.md)

## Direction

Runelink is an extensible, encompassing federated network. Aim for XMPP's technical merit and performance with Discord's ease of use and simplicity.

## Terminology

- **Host:** a Runelink deployment, which would conventionally be called a server, identified by its host name and port. A host operates any number of servers. Machines, processes, and deployments are always hosts; a single `runelink-server` process may run several hosts in cluster mode.
- **Server:** a conceptual server, meaning a community that lives on exactly one host. "Server" never refers to a machine or process, except in the common phrase "server-side" for compute as opposed to client-side.
- **Home host:** the single host on which a user's account exists. Users are identified by a `UserRef` of name and host.
- **Owning host:** the host that is authoritative for a resource. It may also be named for the resource, such as the **channel host** or **voice channel host**. A server channel's owning host is the host of its server. A DM channel is owned by a single host ([#24](https://github.com/runelink-org/runelink/issues/24)).
- **Peer host:** another host participating in federation.
- **Underlying channel:** a minimal protocol resource with a stable ID and no presentation metadata. `text_channels` contain messages; `voice_channels` contain calls and call events.
- **Presentation channel:** a resource that gives underlying channels ownership and presentation. `server_channels` form a server's category tree, and `dm_channels` represent one-to-one and group DMs.

## Ownership and federation

- Every resource has exactly one owning host, which is authoritative for that resource.
- Federation enables communication between hosts without distributing ownership of individual resources.
- Clients authenticate only with their home host. End-user credentials are never shared with other hosts.
- Requests for resources on another host are sent to the home host with a target host. The home host forwards them over federation using host-to-host requests authenticated by short-lived JWTs, which remote hosts verify against the keys published at `/.well-known/jwks.json`.
- Owning hosts verify that a peer host may represent the user it claims to act for. A valid peer host identity alone grants no access to resources.

## Bulk data exception

- Bulk data, meaning call media and asset bytes, is an explicit exception to home-host routing. Clients transfer it directly with endpoints designated by the owning host. Home hosts do not relay it.
- Home hosts handle signaling, authorization, and credential or signed URL issuance up front. Owning hosts never send credentials or signed URLs to clients directly; the home host delivers them over the client's persistent WebSocket connection, or over its regular HTTP API.
- Credentials and signed URLs are issued to a single user and never serve another user.
- Access is checked when credentials or signed URLs are issued, when they are renewed, and whenever a change makes re-checking feasible, such as reconnecting to a call. Short lifetimes bound how long revoked access remains usable.
- Each plan defines its own credentials: signed URLs for assets and join tokens for the SFU.

## Channels

As decided in [#37](https://github.com/runelink-org/runelink/issues/37) and being implemented in [#30](https://github.com/runelink-org/runelink/issues/30):

- Text channels and voice channels are separate underlying resources with separate IDs.
- Server channels have a `kind`. The initial kinds are `text` and `category`; server voice channels will be added as a `voice` kind that owns an underlying voice channel.
- Each DM channel owns both an underlying text channel and an underlying voice channel. One-to-one and group calls therefore take place in the DM channel's voice channel.
- Server channel IDs remain stable regardless of their position in the category tree.
- Categories may be nested to arbitrary depth. Presentation limits are the responsibility of clients, not the protocol.

## Protocol

- Clients and hosts, and federated hosts, communicate over persistent WebSocket connections. Traffic consists of requests, replies, and push-only updates. Clients can also use a regular HTTP API with their home host.
- Operations are grouped into independently versioned capabilities. Each WebSocket connection negotiates them through `hello` and `welcome` envelopes, and operations whose capabilities were not negotiated remain unavailable.
- Capabilities describe protocol compatibility, not authorization. Permissions are a separate, many-to-many policy system.
- Message bodies are plain text by default, with optional metadata for richer content.
- Durable content mutations and ephemeral state, such as presence ([#28](https://github.com/runelink-org/runelink/issues/28)), are modeled separately.
