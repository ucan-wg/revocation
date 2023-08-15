# UCAN Revocation Specification v1.0.0-RC

## Editors

* [Brooklyn Zelenka], [Fission]

## Authors

* [Daniel Holmgren], [Bluesky]
* [Brooklyn Zelenka], [Fission]

# 0. Abstract

This specification describes how to revoke a previously issued UCAN delegation.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# 1. Introduction

## 1.1 Motivation

Even when not in error, the trust relationship between a delegator and delegatee are not immutible. While the UCAN delegation approach RECOMMENDS using the [principle of least authority][POLA], unexpected conditions can and do arise. This is an exceptional case, but is sufficiently important that a well defined method for performing revocation is needed. Revocation is the act of invalidating a UCAN after the fact, outside of the limitations placed on it by the UCAN's fields (such as its expiry). 

Expiry is preferred, but
Manual method beyond expiry

## 1.1 Approach

UCAN delegation is designed to be [local-first]. As such, [fail-stop] approach is thus not suitable for this context. 

UCAN revocation is accomplished with an eventually consistent mechanism. Note that the delivery tradeoff often associated with eventually consistent systems only apply in the worst case: it is entirely possible to communicate directly with the resource server for resources that have a single source of truth. The extra generality allows UCAN revocations to work with structures like [CRDT]s, operate over gossip channels, and at the edge.

In the case of UCAN, this MUST be done by a proof's issuer DID. For more on the exact mechanism, see the revocation validation section.

# 2 Semantics

- Invidates proofs, but can still be valid because constructive

Any issuer of a UCAN MAY later revoke that UCAN or the capabilities that have been derived from it further downstream in a proof chain.

This mechanism is eventually consistent and SHOULD be considered the last line of defense against abuse. Proactive expiry via time bounds or other constraints SHOULD be preferred, as they do not require learning more information than what would be available on an offline computer.

While some resources are centralized (e.g. access to a server), others are unbound from specific locations (e.g. a CRDT), in which case it will take longer for the revocation to propagate.

Every resource type SHOULD have a canonical location where its revocations are kept. This list is non-exclusive, and revocation messages MAY be gossiped between peers in a network to share this information more quickly.

It is RECOMMENDED that the canonical revocation store be kept as close to (or inside) the resource it is about as possible. For example, the WebNative File System maintains a Merkle tree of revoked CIDs at a well-known path. Another example is that a centralized server could have an endpoint that lists the revoked UCANs by [canonical CID].

Revocations MUST be immutible and irreversible. If the revocation was issued in error, a new unique UCAN delegation MAY be issued (e.g. by updating the nonce or changing the time bounds). This prevents confusion as the revocation moves through the network and makes revocation stores append-only and highly amenable to caching.

``` mermaid
flowchart TB
    subgraph RA[Alice can revoke]
        AB["(Root)\niss: Alice\naud: Bob\ncap:[X,Y,Z]"]

        subgraph RB[Bob can revoke]
            BC["iss: Bob\naud: Carol\ncap: [X,Y]"]
            BD["iss: Bob\naud: Dan\ncap:[Y,Z]"]

            subgraph RC[Carol can revoke]
                CD["iss: Carol\naud: Dan\ncap:[X,Y]"]

                subgraph RD[Dan can revoke]
                    DE["iss: Dan\naud: Erin\ncap:[X,Y,Z]"]
                end
            end
        end
    end

    BD -->|proof| AB
    BC -->|proof| AB
    CD -->|proof| BC
    DE -->|proof| CD
    DE -->|proof| BD
```

In this example, Alice MAY revoke any of the UCANs in the chain, Carol MAY revoke the bottom two, and so on. If the UCAN `Carol → Dan` is revoked by Alice, Bob, or Carol, then Erin will not have a valid chain for `X` since its proof is invalid. However, Erin can still prove the valid capability for `Y` and `Z` since the still-valid ("unbroken") chain `Alice → Bob → Dan → Erin` includes them. Note that despite `Y` being in the revoked `Carol → Dan` UCAN, it does not invalidate `Y` for Erin, since the unbroken chain also included a proof for `Y`. 

# 3 Format

A revocation message MUST contain the following information:

| Field | Type                     | Description                                                  | Required |
|-------|--------------------------|--------------------------------------------------------------|----------|
| `iss` | [DID]                    | The DID of the revoker                                       | Yes      |
| `rev` | [CID]                    | The [canonical CID] of the UCAN being revoked                | Yes      |
| `clg` | [base64-unpadded] string | The base64 encoded signature of `REVOKE:${canonicalUcanCid}` | Yes      |

Since revocations MAY be passed between systems, supporting the canonical JSON encoding is REQUIRED:

``` js
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "rev": "bafkreia7l6bthgtpaaw4qbfacun6p4rt5rcorsognxgojvkyvhlmo7kf4a",
  "clg": "FJi1Rnpj3/otydngacrwddFvwz/dTDsBv62uZDN2fZM"
}
```

# 4 Revocation Store

The agent that controls a resource SHOULD maintain a cache of revocations that it's seen. Other agents MAY also maintain a cache of revocations that they're aware of.

## 4.1 Eviction

Revocations MAY be deleted once the UCAN that they reference expires or otherwise becomes invalid via its proactive mechanisms.

A revocation store MOST only keep UCAN revocations for UCANs that are otherwise still valid. For example, expired UCANs are already invalid, so a revocation MUST NOT affect this invalid status. The revocation is thus redundant, and MAY be evicted from the store.



[Bluesky]: https://blueskyweb.xyz/
[Brooklyn Zelenka]: https://github.com/expede 
[CIDv1]: https://docs.ipfs.io/concepts/content-addressing/#identifier-formats
FIXME [Canonical CID]: #651-cid-canonicalization
[Capability Myths Demolished]: https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf
[Christine Lemmer-Webber]: https://github.com/cwebber
[Christopher Joel]: https://github.com/cdata
[DID fragment]: https://www.w3.org/TR/did-core/#fragment
[DID path]: https://www.w3.org/TR/did-core/#path
[DID subject]: https://www.w3.org/TR/did-core/#dfn-did-subjects
[DID]: https://www.w3.org/TR/did-core/
[Dan Finlay]: https://github.com/danfinlay
[Daniel Holmgren]: https://github.com/dholms
[ECDSA security]: https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Security
[FIDO]: https://fidoalliance.org/fido-authentication/
[Fission]: https://fission.codes
[Hugo Dias]: https://github.com/hugomrdias
[Irakli Gozalishvili]: https://github.com/Gozala
[JWT]: https://datatracker.ietf.org/doc/html/rfc7519
[Juan Caballero]: https://github.com/bumblefudge
[Local-First Auth]: https://github.com/local-first-web/auth
[Macaroon]: https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41892.pdf
[Mark Miller]: https://github.com/erights
[Mikael Rogers]: https://github.com/mikeal/
[OCAP]: http://erights.org/elib/capability/index.html
[OCapN]: https://github.com/ocapn/ocapn
[POLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[Philipp Krüger]: https://github.com/matheus23
[Protocol Labs]: https://protocol.ai/
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 8037]: https://datatracker.ietf.org/doc/html/rfc8037
[SHA2-256]: https://en.wikipedia.org/wiki/SHA-2
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Seitan token exchange]: https://book.keybase.io/docs/teams/seitan
[Steven Vandevelde]: https://github.com/icidasset
[Token Uniqueness]: #622-token-uniqueness
[URI]: https://www.rfc-editor.org/rfc/rfc3986
[Verifiable credentials]: https://www.w3.org/2017/vc/WG/
[W3C]: https://www.w3.org/
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[`did:3`]: https://github.com/ceramicnetwork/CIPs/blob/main/CIPs/cip-79.md
[`did:ion`]: https://github.com/decentralized-identity/ion
[`did:key`]: https://w3c-ccg.github.io/did-method-key/
[base32]: https://github.com/multiformats/multibase/blob/master/multibase.csv#L12
[browser api crypto key]: https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey
[canonical collections]: #71-canonical-json-collection
[capabilities]: https://en.wikipedia.org/wiki/Object-capability_model
[caps as keys]: http://www.erights.org/elib/capability/duals/myths.html#caps-as-keys
[confinement]: http://www.erights.org/elib/capability/dist-confine.html
[constructive semantics]: https://en.wikipedia.org/wiki/Intuitionistic_logic
[content addressable storage]: https://en.wikipedia.org/wiki/Content-addressable_storage
[content addressing]: https://en.wikipedia.org/wiki/Content-addressable_storage
[content identifiers]: #65-content-identifiers
[dag-json multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L104
[delegation]: #51-ucan-delegation
[disjunction]: https://en.wikipedia.org/wiki/Logical_disjunction
[invocation]: https://github.com/ucan-wg/invocation
[prf field]: #3271-prf-field
[raw data multicodec]: https://github.com/multiformats/multicodec/blob/a03169371c0a4aec0083febc996c38c3846a0914/table.csv?plain=1#L41
[replay attack prevention]: #93-replay-attack-prevention
[revocation]: #66-revocation
[rights amplification]: #64-rights-amplification
[secure hardware enclave]: https://support.apple.com/en-ca/guide/security/sec59b0b31ff
[spki rfc]: https://www.rfc-editor.org/rfc/rfc2693.html
[time definition]: https://en.wikipedia.org/wiki/Temporal_database
[token resolution]: #8-token-resolution
[top ability]: #41-top
[ucan.xyz]: https://ucan.xyz
