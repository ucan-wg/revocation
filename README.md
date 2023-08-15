# UCAN Revocation Specification v1.0.0-RC

## Editors

* [Brooklyn Zelenka], [Fission]

## Authors

* [Daniel Holmgren], [Bluesky]
* [Brooklyn Zelenka], [Fission]

# 0. Abstract

This document describes how to revoke a previously issued delegation.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# 1. Introduction

## 1.1 Motivation

## 2.9 Revocation

Revocation is the act of invalidating a UCAN after the fact, outside of the limitations placed on it by the UCAN's fields (such as its expiry). 

In the case of UCAN, this MUST be done by a proof's issuer DID. For more on the exact mechanism, see the revocation validation section.

## 6.6 Revocation

Any issuer of a UCAN MAY later revoke that UCAN or the capabilities that have been derived from it further downstream in a proof chain.

This mechanism is eventually consistent and SHOULD be considered the last line of defense against abuse. Proactive expiry via time bounds or other constraints SHOULD be preferred, as they do not require learning more information than what would be available on an offline computer.

While some resources are centralized (e.g. access to a server), others are unbound from specific locations (e.g. a CRDT), in which case it will take longer for the revocation to propagate.

Every resource type SHOULD have a canonical location where its revocations are kept. This list is non-exclusive, and revocation messages MAY be gossiped between peers in a network to share this information more quickly.

It is RECOMMENDED that the canonical revocation store be kept as close to (or inside) the resource it is about as possible. For example, the WebNative File System maintains a Merkle tree of revoked CIDs at a well-known path. Another example is that a centralized server could have an endpoint that lists the revoked UCANs by [canonical CID].

Revocations MUST be irreversible. If the revocation was issued in error, a unique UCAN MAY be issued (e.g. by updating the nonce or changing the time bounds). This prevents confusion as the revocation moves through the network and makes revocation stores append-only and highly amenable to caching.

# Format

A revocation message MUST conform to the following JSON format:

``` js
{
  "iss": did,
  "revoke": canonicalUcanCid,
  "challenge": base64Unpadded(sign(did.privateKey, `REVOKE:${canonicalUcanCid}`))
}
```

This format makes it easy to select the relevant UCAN, confirm that the issuer is in the proof chain for some or all capabilities, and validate that the revocation was signed by the same issuer.

Any other proofs in the selected UCAN not issued by the same DID as the revocation issuer MUST be treated as valid.

Revocations MAY be deleted once the UCAN that they reference expires or otherwise becomes invalid via its proactive mechanisms.

## 6.6.1 Example

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



[Alan Karp]: https://github.com/alanhkarp
[Benjamin Goering]: https://github.com/gobengo
[Biscuit]: https://github.com/biscuit-auth/biscuit/
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brendan O'Brien]: https://github.com/b5
[Brian Ginsburg]: https://github.com/bgins
[Brooklyn Zelenka]: https://github.com/expede 
[CACAO]: https://blog.ceramic.network/capability-based-data-security-on-ceramic/
[CIDv1]: https://docs.ipfs.io/concepts/content-addressing/#identifier-formats
[Canonical CID]: #651-cid-canonicalization
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
