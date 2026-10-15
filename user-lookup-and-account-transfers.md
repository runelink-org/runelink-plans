# User lookup and account transfers

This plan records the direction for locating users across hosts and for moving accounts between hosts. Shared terminology and architecture are defined in [Runelink architecture](architecture.md).

The motivating case is a large operator, such as `runelink.chat`, running separate regional hosts like `eu.runelink.chat` and `us.runelink.chat` through the existing federation model. Users should be able to sign up and find each other without dealing with regions, while the protocol keeps a single owning host for every resource.

## Direction and confirmed decisions

- A `UserRef` always names the user's real home host, such as `alice@eu.runelink.chat`. Nothing that hides the home host enters the protocol.
- Regional hosts are ordinary hosts that federate with each other. Regions do not introduce distributed ownership.
- User lookup is a small client layer on top of the protocol, not part of it. Hosts are not required to provide it.
- Account transfers will eventually be added to the spec, both between hosts run by the same operator and to unrelated hosts, such as moving from `alice@eu.runelink.chat` to `alice@alicedomain.com`.

## User lookup

- A host may offer a lookup endpoint that maps users to their real home host. For `runelink.chat`, the root host serves this mapping through a CDN so lookups are fast.
- Clients check whether a host provides the lookup endpoint. If it does not, they use the given host directly.
- Using the real regional host directly, such as signing in at `eu.runelink.chat`, works the same as going through the lookup layer.
- Lookup applies to every client input that identifies a user, including sign-in, user search, mentions, and invites. It works for users on any host, so a client on another host can resolve `bob@runelink.chat` the same way.
- Clients resolve users before anything is stored or sent. Messages and other protocol data only ever contain real `UserRef`s.
- Stale lookup results correct themselves: after an account transfer, the old home host redirects to the new one. Lookup mappings therefore do not need very short cache lifetimes.
- The mapping reveals which usernames exist and on which host. That is largely inherent to unique usernames, but it should contain nothing else.

### Operator infrastructure

The following is operator infrastructure for `runelink.chat` and is outside the protocol spec:

- Sign-up at the root `runelink.chat` suggests a region, for example based on the user's location, and asks the user to confirm it before creating the account on that regional host.
- Regional hosts coordinate internally to keep usernames unique across regions, so transfers between regions can keep the same name.
- New servers default to their creator's region. Communities spanning regions federate as usual.

## Account transfers

Account transfers are deferred. They follow a forwarding-address model, similar to mail forwarding when someone moves.

- The old home host keeps a forwarding record pointing to the user's new `UserRef`.
- Peer hosts that contact the old address are redirected and update their records, so traffic eventually goes directly to the new home host. The old host may also push the change to known peers.
- Both hosts must agree to a transfer. The new host confirms the destination account, and the old host authorizes the move with the user's credentials, so accounts cannot be redirected by anyone else.
- Memberships, roles, and contacts follow the account. Resources owned by hosts, such as servers, stay with their owning host. Existing messages keep their original author `UserRef`, which resolves to the new account through the forwarding record.
- Mastodon's account `Move` is useful prior art.

### Open decisions

- How long must an old host keep forwarding records?
- What happens when the old host disappears before peers have updated, such as a self-hosted host shutting down? The new host may need to present proof of the move from the old host.
- What happens to DM channels owned by the old host ([#24](https://github.com/runelink-org/runelink/issues/24))?
- How are name conflicts handled when moving to a host where the username is taken?
- Which data, such as profile and settings, is copied to the new host, and how?
