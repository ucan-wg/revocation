# UCAN Revocation Specification 1.0.0-rc.1

## Editors

* [Brooklyn Zelenka], [Fission]

## Authors

* [Brooklyn Zelenka], [Fission]
* [Daniel Holmgren], [Bluesky]
* [Irakli Gozalishvili], [Protocol Labs]
* [Philipp Krüger], [Fission]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] ([RFC2119] and [RFC8174]) when, and only when, they appear in all capitals, as shown here.

# 0 Abstract

This specification describes how to revoke a previously issued UCAN delegation.

# 1 Introduction

Using the [principle of least authprity][POLA] such as certificate expiry and reduced capabilty scope SHOULD be the preferred method for securing a UCAN, but does not cover every situation. Revocation is a manual method for reversing a delegation. It is not a perfect method, and cannot undo irreversable actions already performed with capability, but MAY limit misuse going forward.

## 1.1 Motivation

Even when not in error at time of issuance, the trust relationship between a delegator and delegatee is not immutible. An agent can go rogue, keys can be compromised, and the privacy requirements of resources changed. While the UCAN delegation approach RECOMMENDS using the [principle of least authority][POLA], such unexpected conditions can and do arise. These are exceptional cases, but are sufficiently important that a well defined method for performing revocation is nearly always desired. Revocation is the act of invalidating a UCAN after the fact, outside of the limitations placed on it by the UCAN's fields (such as its expiry). 

## 1.2 Approach

UCAN delegation is designed to be [local-first]. As such, [fail-stop] approaches are not suitable. Revocation is accomplished via an unforgable message 

# 2 Semantics

UCAN revocation is the act of invalidating a proof in a delegation chain for some specific UCAN delegation by its CID. All UCAN capabilities are either claimed by direct authority over the resource, or by delegation chain terminating in that direct ("root") authority. Each link in a delegation chain contains an explicit issuer (delegator) and audience (delegatee).

Revocations MUST be immutible and irreversible. If the revocation was issued in error, a new unique UCAN delegation MAY be issued (e.g. by updating the nonce or changing the time bounds). This prevents confusion as the revocation moves through the network and makes revocation stores append-only and highly amenable to caching.







passing and SHOULD be considered the last line of defense against abuse. Proactive expiry via time bounds or other constraints SHOULD be preferred, as they do not require learning more information than what would be available on an offline computer.





## 2.1 Scope 

An issuer of a proof in a delegation chain MAY revoke access to the capabilties that it granted. Note that this is not the same as revoking the specific delegation signed by the issuer: any UCAN that contains a proof where the revoker matches the `iss` field — even transatively in the delegation chain — MAY be revoked.

Revocation by a particular proof does not guarantee that the principle no longer has access to the capabilty in question. If a principal is able to construct a valid proof chain without relying on the revoked proof, they still have access to the capability. By real-world analogy, if Mallory has two tickets to a film, and one of them is invalidated by its serial number, she is still able to present the valid ticket to see the film.

### 2.1.1 Example

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

Here Alice is the root issuer / resource owner. Alice MAY revoke any of the UCANs in the chain, Carol MAY revoke the two innermost, and so on. If the UCAN `Carol to Dan` is revoked by Alice, Bob, or Carol, then Erin will not have a valid chain for the `X` capability, since its only proof is invalid. However, Erin can still prove the valid capability for `Y` and `Z` since the still-valid ("unbroken") chain `Alice to Bob to Dan to Erin` includes them. Note that despite `Y` being in the revoked `Carol to Dan` UCAN, it does not invalidate `Y` for Erin, since the unbroken chain also included a proof for `Y`. 

## 2.2 Consistency Models

UCAN revocation is designed to work in the broadest possible scenarios, and as such needs very weak constraints. UCAN revocation MAY operate in fully eventually consistent contexts, with single sources of truth, or among nodes participating in consensus. The format of the revocation does not change in these situations; it is entirely managed by how revocations are passed around the network.

These weak assumptions are often associated with being unable to guarantee delivery in a certain time bound. Weak assumptions can always be strengthened, but not vice-versa. For example, if a capability describes access to a resource with a single location or souce of truth, sending a revocation to that specific agent enables confirmatation in bounded time. This grants nearly idential semantics that many people turn to ACLs for, but with all of the benefits of capabilties.

# 3 Cache

The agent that controls a resource SHOULD maintain a cache of revocations that it has seen. Other agents MAY also maintain a cache of revocations that they're aware of. On recept of a UCAN delegation or invocation, the 


kepp them close to eth reouce

MAY hold on to extra ones for othrs


It is RECOMMENDED that the canonical revocation store be kept as close to (or inside) the resource it is about as possible. For example, the WebNative File System maintains a Merkle tree of revoked CIDs at a well-known path. Another example is that a centralized server could have an endpoint that lists the revoked UCANs by [canonical CID].


## 3.1 Eviction

Revocations MAY be deleted once the UCAN that they reference expires or otherwise becomes invalid via its proactive mechanisms.

A revocation store MOST only keep UCAN revocations for UCANs that are otherwise still valid. For example, expired UCANs are already invalid, so a revocation MUST NOT affect this invalid status. The revocation is thus redundant, and MAY be evicted from the store.

# 4 Format

A revocation message MUST contain the following fields:

| Field | Type                     | Description                                                | Required |
|-------|--------------------------|------------------------------------------------------------|----------|
| `urv` | [Semver] string          | Version of UCAN Revocation (`1.0.0-rc.1`)                  | Yes      |
| `iss` | [DID]                    | Revoker DID                                                | Yes      |
| `rvk` | [CID]                    | The [canonical CID] of the UCAN being revoked              | Yes      |
| `sig` | [base64-unpadded] string | The base64 encoded signature of `` `REVOKE-UCAN:${rvk}` `` | Yes      |

Revocations MAY be gossiped between systems. As such, they need to be parsable by a wide number of lanaguges and contexts. To accomodate this, compliant UCAN revocations MUST be JSON-encoded.

## 4.1 `urv` UCAN Revocation Version

The UCAN Revocation Version, i.e. the version of this document: `1.0.0-rc.1`.

## 4.2 `iss` Revocation Issuer

The issuer DID of this revocation. This DID MUST match one or more `iss` fields in the proof chain of the UCAN listed in the `rvk` field. This DID MUST also validate against the signature in the `sig` field.

## 4.3 `rvk` Revoked UCAN

The `rvk` field MUST contain the [canonical CID] for the UCAN delegation being revoked. The target UCAN MUST contain a (potentially nested) UCAN proof where the `iss` field matches the `iss` field of this revocation. 

Note that this revocation MUST only revoke the particular capabilities from the relevant proof(s). The resultant proof chain MUST be treated as if that proof were not present, with no other changes.

## 4.4 `sig` Prefixed Signature

The `sig` field MUST contain a signature that validates against the revoker's DID. The format to sign over MUST be prefixed with the UTF-8 string `REVOKE-UCAN:`, followed by the CID from the `rvk` field.

## 4.5 Example

``` json
{
  "urv": "1.0.0-rc.1",
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "rvk": "bafkreia7l6bthgtpaaw4qbfacun6p4rt5rcorsognxgojvkyvhlmo7kf4a",
  "sig": "FJi1Rnpj3/otydngacrwddFvwz/dTDsBv62uZDN2fZM"
}
```

# 5 Prior Art

SPKI/SDSI recovcation lists

# 6 Acknowledgements








<!-- External Links -->

[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Bluesky]: https://blueskyweb.xyz/
[Brooklyn Zelenka]: https://github.com/expede 
[CIDv1]: https://docs.ipfs.io/concepts/content-addressing/#identifier-formats

<!-- Internal Links -->
