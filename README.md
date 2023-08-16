# UCAN Revocation Specification 1.0.0-rc.1

## Editors

* [Brooklyn Zelenka], [Fission]

## Authors

* [Brooklyn Zelenka], [Fission]
* [Daniel Holmgren], [Bluesky]
* [Irakli Gozalishvili], [Protocol Labs]
* [Philipp Krüger], [Fission]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# 0 Abstract

This specification describes the format and semantics for revoking a previously issued UCAN delegation.

# 1 Introduction

Using the [principle of least authority][POLA] such as certificate expiry and reduced capability scope SHOULD be the preferred method for securing a UCAN, but does not cover every situation. Revocation is a manual method for reversing a delegation. It is not a perfect method, and cannot undo irreversible actions already performed with capability, but MAY limit misuse going forward.

## 1.1 Motivation

Even when not in error at time of issuance, the trust relationship between a delegator and delegatee is not immutable. An agent can go rogue, keys can be compromised, and the privacy requirements of resources can change. While the UCAN delegation approach RECOMMENDS using the [principle of least authority][POLA], such unexpected conditions can and do arise. These are exceptional cases, but are sufficiently important that a well defined method for performing revocation is nearly always desired. Revocation is the act of invalidating a UCAN after the fact, outside of the limitations placed on it by the UCAN's fields (such as its expiry). 

## 1.2 Approach

UCAN Revocations are similar to [block lists]: they identify delegations that are retracted and no longer suitable for use. Revocation SHOULD be considered the last line of defense against abuse. Proactive expiry via time bounds or other constraints SHOULD be preferred, as they do not require learning more information than what would be available on an offline computer.

UCAN delegation is designed to be [local-first]. As such, [fail-stop] approaches are not suitable. Revocation is accomplished by delivery of an unforgeable message from a previous delegator.

# 2 Semantics

UCAN revocation is the act of invalidating a proof in a delegation chain for some specific UCAN delegation by its CID. All UCAN capabilities are either claimed by direct authority over the resource, or by delegation chain terminating in that direct ("root") authority. Each link in a delegation chain contains an explicit issuer (delegator) and audience (delegatee).

Revocations MUST be immutable and irreversible. Recipients of revocations SHOULD treat them as a grow-only set (see the [eviction] section). If a revocation was issued in error, it MUST NOT be retracted — a new, unique UCAN delegation MAY be issued (e.g. by updating the nonce or changing the time bounds). This prevents confusion as the revocation moves through the network and makes [revocation stores] append-only and highly amenable to caching.

## 2.1 Scope 

An issuer of a proof in a delegation chain MAY revoke access to the capabilities that it granted. Note that this is not the same as revoking the specific delegation signed by the issuer: any UCAN that contains a proof where the revoker matches the `iss` field — even transitively in the delegation chain — MAY be revoked.

Revocation by a particular proof does not guarantee that the principle no longer has access to the capability in question. If a principal is able to construct a valid proof chain without relying on the revoked proof, they still have access to the capability. By real-world analogy, if Mallory has two tickets to a film, and one of them is invalidated by its serial number, she is still able to present the valid ticket to see the film.

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
Weak assumptions enable UCAN to work with eventually consistent resources, such as [CRDT]s, [Git] forks, delay-tolerant replicated state machines, and so on.

These weak assumptions are often associated with being unable to guarantee delivery in a certain time bound. Weak assumptions can always be strengthened, but not vice-versa. For example, if a capability describes access to a resource with a single location or source of truth, sending a revocation to that specific agent enables confirmation in bounded time. This grants nearly identical semantics that many people turn to [ACL]s for, but with all of the benefits of capabilities during delegation and invocation.

Out of order delivery is typical of distributed systems. Further, a malicious user can otherwise delay revealing that they have a capability until the last possible moment in hopes of evading detection. Accepting revocations for resources that the agent controls prior to the delegation targeted by the revocation is received is thus RECOMMENDED. 

# 3 Store

The agent that controls a resource SHOULD maintain a cache of revocations that it has seen. Agents are not limited to only storing revocations for resources that they control.

During validation of a UCAN delegation chain, the [canonical CID] of each UCAN delegation MUST be checked against the cache. If there's a match, the relevant delegation MUST be ignored. Note that this MAY NOT invalidate the entire UCAN chain.

## 3.1 Locality

Revocation caches SHOULD be kept as close to the resource they describe as possible.

Resources with a single source of truth SHOULD follow the typical approach of maintaining a revocation store at the same physical location as the resource. For example, a centralized server MAY have an endpoint that lists the revoked UCANs by [canonical CID].

For eventually consistent data structures, this MAY be achieved by including the store directly inside the resource itself. For example, a CRDT-based file system SHOULD maintain the revocation store directly at a well-known path.

## 3.2 Eviction

Revocations MAY be deleted once the UCAN that they reference expires or otherwise becomes invalid through its proactive mechanisms.

A revocation store MOST only keep UCAN revocations for UCANs that are otherwise still valid. For example, expired UCANs are already invalid, so a revocation MUST NOT affect this invalid status. The revocation is thus redundant, and MAY be evicted from the store.

# 4 Format

A revocation message MUST contain the following information:

| Field | Type                               | Description                                                | Required |
|-------|------------------------------------|------------------------------------------------------------|----------|
| `urv` | [Semver] string                    | Version of UCAN Revocation (`1.0.0-rc.1`)                  | Yes      |
| `iss` | [DID]                              | Revoker DID                                                | Yes      |
| `rvk` | [CIDv1]                            | The [canonical CID] of the UCAN being revoked              | Yes      |
| `sig` | [base64-unpadded][RFC 4648] string | The base64 encoded signature of `` `REVOKE-UCAN:${rvk}` `` | Yes      |

Revocations MAY be gossiped between systems. As such, they need to be parsable by a wide number of languages and contexts. To accommodate this, compliant UCAN revocations MUST be JSON-encoded.

## 4.1 `urv` UCAN Revocation Version

The UCAN Revocation Version, i.e. the version of this document: `1.0.0-rc.1`.

## 4.2 `iss` Revocation Issuer

The issuer DID of this revocation. This DID MUST match one or more `iss` fields in the proof chain of the UCAN listed in the `rvk` field. This DID MUST also validate against the signature in the `sig` field.

## 4.3 `rvk` Revoked UCAN

The `rvk` field MUST contain the [canonical CID] for the UCAN delegation being revoked. The target UCAN MUST contain a (potentially nested) UCAN proof where the `iss` field matches the `iss` field of this revocation. 

Note that this revocation MUST only revoke the particular capabilities from the relevant proof(s). The resultant proof chain MUST be treated as if that proof were not present, with no other changes.

## 4.4 `sig` Prefixed Signature

The `sig` field MUST contain a signature that validates against the revoker's DID. The format to sign over MUST be prefixed with the UTF-8 string `REVOKE-UCAN:`, followed by the CID from the `rvk` field. Note that the prefix contains no spaces.

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

[Revocation lists][Cert Revocation Wikipedia] are a fairly widely used concept.

[SPKI/SDSI] is closely related to UCAN. A different format is used, and some details vary (such as a delegation-locking bit), but the core idea and general usage pattern are very close. UCAN can be seen as making these ideas more palatable to a modern audience and adding a few features such as content IDs that were less widespread at the time SPKI/SDSI were written.

[X.509 Certificate Revocation Lists][RFC 5280] defines two kinds of certificate invalidation: temporary ("hold") and permanent ("revocation"). This RFC also includes a field for indicating a reason for revocation. UCAN Revocation has no concept of a temporary hold on a capability, but this behavior MAY be emulated by revoking a credential and issuing a new UCAN with a `nbf` field set to a time in the future.

[ZCAP-LD] is closely related to UCAN, but situated in the W3C-style linked data world (the "LD" in ZCAP-LD). Revocation in ZCAP-LD is only granted to those who have a special caveat on a capability. In contrast, UCAN capabilities MAY be revoked by anyone in the relevant delegation path.

[OAuth 2.0 Revocation][RFC 7009] is very similar to UCAN revocation. It is largely concerned with the HTTP interactions to make OAuth revocation work. OAuth doesn't have a concept of sub-delegation, so only the user that has been granted the token can revoke it. However, this may cascade to revocation of other tokens, but the exact mechanism is left to the implementer.

While strictly speaking being about assertions rather than capabilities, [Verfiable Credential Revocation][VC Revocation] spec follows a similar pattern to those listed above.

[E][E-lang]-style [object capabilities][Robust Composition] use active network connections with [proxy agents][Robust Composition] to revoke delegations. Revocation is achieved by shutting down that proxy to break the authorizing reference. In many ways, UCAN Revocation attempts to emulate this behavior. Unlike UCAN Revocations, E-style object capabilities are [fail-stop] and thus by definition not partition tolerant.

# 6 Acknowledgements

Thank you [Blaine Cook] for the real-world feedback, ideas on future features, and lessons from other auth standards.

Thanks to [Juan Caballero] for the numerous questions, clarifications, and general advice on putting together a comprehensible spec.

Many thanks to [Alan Karp] for sharing his vast experience with capability-based authorization, patterns, and many right words for us to search for.

Thanks to [Benjamin Goering] for the many community threads and connections to [W3C] standards.

Many thanks to [Christine Lemmer-Webber] for her handwritten(!) feedback on the design of UCAN, spearheading the [OCapN] initiative, and her related work on [ZCAP-LD].

Thanks to the entire [SPKI WG][SPKI/SDSI] for their closely related pioneering work.

We want to especially recognize [Mark Miller] for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

<!-- Internal Links -->

[eviction]: #32-eviction
[revocation stores]: #3-store

<!-- External Links -->

[ACL]: https://en.wikipedia.org/wiki/Access-control_list
[Alan Karp]: https://github.com/alanhkarp
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Benjamin Goering]: https://github.com/gobengo
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brooklyn Zelenka]: https://github.com/expede 
[CIDv1]: https://docs.ipfs.io/concepts/content-addressing/#identifier-formats
[CRDT]: https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type
[Cert Revocation Wikipedia]: https://en.wikipedia.org/wiki/Certificate_revocation_list
[Christine Lemmer-Webber]: https://github.com/cwebber
[DID]: https://www.w3.org/TR/did-core/
[Daniel Holmgren]: https://github.com/dholms
[E-lang]: http://erights.org/elang/
[Fission]: https://fission.codes
[Git]: https://git-scm.com/
[Irakli Gozalishvili]: https://github.com/Gozala
[Juan Caballero]: https://github.com/bumblefudge
[Mark Miller]: https://github.com/erights
[OCAP]: http://erights.org/elib/capability/index.html
[OCapN]: https://github.com/ocapn/ocapn
[POLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[Philipp Krüger]: https://github.com/matheus23
[Protocol Labs]: https://protocol.ai/
[RFC 4648]: https://www.rfc-editor.org/rfc/rfc4648.html
[RFC 5280]: https://www.rfc-editor.org/rfc/rfc5280
[RFC 7009]: https://datatracker.ietf.org/doc/html/rfc7009
[Robust Composition]: https://jscholarship.library.jhu.edu/bitstream/handle/1774.2/873/markm-thesis.pdf?page=100
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[Semver]: https://semver.org/
[VC Revocation]: https://w3c-ccg.github.io/vc-status-rl-2020/
[W3C]: https://www.w3.org/
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[block lists]: https://en.wikipedia.org/w/index.php?title=Block_list&redirect=no
[canonical CID]: https://github.com/ucan-wg/spec/blob/d5a844cceff569838881c7fa30ff4bfad338e771/README.md?plain=1#L800-L807
[fail-stop]: https://en.wikipedia.org/wiki/Fail-stop
[local-first]: https://www.inkandswitch.com/local-first/
