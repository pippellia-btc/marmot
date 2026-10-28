# marmot.transport.nostr.routing.v2

Status: adopted.

## Registry

- Component id: `0x800e`
- Name: `marmot.transport.nostr.routing.v2`
- Location: GroupContext `app_data_dictionary`
- Default requirement: required only for Nostr-routed Marmot groups that use the v2 routing profile

## State

```text
struct {
  opaque url<1..512>;
} MarmotNostrRelayV2;

struct {
  opaque nostr_routing_key[32];
  MarmotNostrRelayV2 relays<V>;
} MarmotNostrRoutingV2;
```

`nostr_routing_key` is a BIP-340 secret key encoded as exactly 32 bytes. It is interpreted as a big-endian unsigned
integer `s` and is valid only when `1 <= s < n`, where `n` is the secp256k1 group order. The group's routing public key
is `routing_pubkey = xonly_secp256k1(nostr_routing_key)`, the BIP-340 x-only public key for `s`. It is 32 bytes and is
not stored in the component.

`routing_pubkey` is the group's Nostr routing address. The Nostr transport uses it as the kind `445` event author and
as the `authors` filter value ([../transports/nostr.md](../transports/nostr.md), "Group message delivery").

The Nostr routing key is used only to sign kind `445` events. It MUST NOT be used as an MLS signature key, an account
identity, or a signing key for any other Nostr event kind. It is generated with a cryptographically secure random
generator at group creation and at every routing rotation. It MUST NOT be derived from an account id, member id, public
key, MLS group id, KeyPackage id, message id, or relay URL.

`relays` is the canonical sorted list of unique relay URL byte strings for a Nostr-routed group. The list is signed
group state. The relay URL profile is defined in [../transports/nostr.md](../transports/nostr.md).

The component carries secret group state. Clients MUST keep `nostr_routing_key` confidential from everyone except
current members. It MUST NOT be published outside encrypted MLS group state. Every current member can sign kind `445`
events under `routing_pubkey`; a former member that retained the component can do so for the epochs it retained.

## Update

The update payload is a full replacement state:

```text
MarmotNostrRoutingV2 MarmotNostrRoutingUpdateV2;
```

An update MAY change the relay list, `nostr_routing_key`, or both. An update that changes `nostr_routing_key` is called a
routing rotation.

## Routing rotation

- The commit that carries a routing change MUST be published at the prior epoch's routing address, that is, signed with
  the prior `nostr_routing_key` and addressed to members subscribed to the prior `routing_pubkey`.
- After applying the commit, members sign subsequent kind `445` events with the new `nostr_routing_key`.
- Members MUST continue to accept and fetch traffic at a prior routing address while any epoch that used it remains
  inside either retained-history window, and MUST map more than one routing public key to the same group until then.
- A group MAY be reachable at more than one Nostr routing address over its lifetime. v2 state carries exactly one
  current `nostr_routing_key`.

A member catching up across a rotation uses its recorded routing-state history to fetch older epochs at their then-active
addresses. A new joiner receives the current routing state in its Welcome and needs older addresses only within the
retained-history rules.

Routing rotation after a member removal is optional. A removed member knows the prior `nostr_routing_key` and can sign
kind `445` events at the prior address. When a group chooses to hide the replacement routing address from that member,
it uses two accepted commits:

1. a membership commit removes the member without changing `nostr_routing_key`;
2. after that removal is canonical, a later commit replaces `nostr_routing_key` from the post-removal epoch.

Changing membership and `nostr_routing_key` in the same commit does not provide this property: the removed member can
decrypt the removal commit under the source epoch and read the new Nostr routing key. The later rotation commit is
published at the prior routing address, but its kind `445` envelope uses the post-removal epoch key, so the removed
member cannot decrypt it. The removed member can still sign at the prior address; members keep that address only for
the retained-history window. This sequence does not require automatic rotation after every removal.

## Validation

A routing v2 state is valid if:

- `nostr_routing_key` is exactly 32 bytes and decodes to a scalar `s` with `1 <= s < n`;
- the relay list is not empty;
- the relay list contains at most 16 entries;
- every relay URL satisfies the Nostr relay URL profile in [../transports/nostr.md](../transports/nostr.md);
- relay URLs are sorted lexicographically over the URL content bytes, per the default sort order in
  [../foundation/canonical-encoding.md](../foundation/canonical-encoding.md) ("Sorting and duplicates");
- relay URLs have no duplicates.

Clients compare relay URL bytes exactly after validation. Producers SHOULD normalize before proposing a group-state
update, but a client MUST NOT rewrite relay URL bytes while applying signed group state.

A client MUST derive `routing_pubkey` from `nostr_routing_key` for every event author and filter comparison. The component
has no public key field.

## Authorization

Only an active admin MAY send a standalone routing update proposal.

Only an active admin MAY commit a routing update. This includes routing rotations: a `nostr_routing_key` change is
admin-gated like every other routing update.

Commit authorization, including removal authorization, follows the shared candidate-parent rule in
[README.md](./README.md) ("Authorization Evaluation").

## Removal

Only an active admin MAY commit removal of this component.

This component MUST NOT be removed while the group is Nostr-routed and lists it as required in GroupContext
`app_components`. It MAY be removed only from a group that no longer routes over Nostr. If it is removed from a group
that still exists over another transport, the Nostr transport can no longer route group messages from signed group
state.

## Negotiation and presence

A group that uses the v2 routing profile MUST require component id `0x800e` in its GroupContext `app_components`
required list. Every KeyPackage and every current member LeafNode MUST advertise `0x800e` in its `app_components`
support list. A group MUST NOT require both `0x8004` and `0x800e`, and a group that requires `0x800e` MUST NOT carry a
`0x8004` GroupContext entry.

A client that does not support `0x800e` MUST NOT join a group that requires it.

## Migration from v1

A v1 group MAY migrate to v2 with an admin commit that adds `0x800e` and removes `0x8004`, after every current member
advertises `0x800e`. The commit MUST include a fresh `nostr_routing_key` and the current relay list, and is published at
the old v1 routing address. After it is canonical, members use the v2 envelope profile. There is no fallback: a group
requires exactly one routing component version. A v1 group that does not migrate remains on v1.

## Security considerations

- Admission depends on the relay verifying NIP-01 signatures and serving the `authors` filter. The client still verifies
  the signature before trial decryption.
- The Nostr routing key is shared by all current members. It stops non-members, not a malicious member. A leaked key
  allows signing at the current address until rotation, but reveals no content or MLS state.
