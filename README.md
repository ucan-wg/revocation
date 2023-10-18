# UCAN Revocation Specification v1.0.0-rc.1

## Editors

- [Brooklyn Zelenka], [Fission]

## Authors

- [Brooklyn Zelenka], [Fission]
- [Irakli Gozalishvili], [Protocol Labs]
- [Philipp Krüger], [Fission]

## Dependencies

- [DID]
- [IPLD]
- [UCAN Delegation]
- [UCAN Invocation]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# 0. Abstract

This specification defines the syntax and semantics of revoking a [UCAN Delegation], and the ability to delegate this ability to others.

# 1. Introduction

Using the [principle of least authority][POLA] such as certificate expiry and reduced capability scope SHOULD be the preferred method for securing a UCAN, but does not cover every situation. Revocation is a manual method for reversing a delegation. It cannot undo irreversible mutations (such as sending an email), but MAY limit misuse going forward. Revocation is the act of invalidating a UCAN after the fact, outside of the limitations placed on it by the UCAN's fields (such as its expiry). 

Even when not in error at time of issuance, the trust relationship between a delegator and delegatee is not immutable. An agent can go rogue, keys can be compromised, and the privacy requirements of resources can (will!) change. While the UCAN Delegation approach recommends following the [principle of least authority][POLA], unexpected conditions that require manual intervention do arise. These are exceptional cases, but are sufficiently important that a well defined method for performing revocation is nearly always desired in token and certificate systems.

# 2 Approach

UCAN delegation is designed to be [local-first], partition-tolerant, cacheable, and latency-reducing. As such, [fail-safe] approaches are not suitable. Revocation is accomplished by delivery of an unforgeable message from a previous delegator.

UCAN Revocations are similar to [block list]s: they identify delegation paths that are retracted and no longer suitable for use. Revocation SHOULD be considered the last line of defense against abuse. Proactive expiry through time bounds or other constraints SHOULD be preferred, as they do not require learning more information than what would be available on an offline computer.

UCAN Revocation is a mechanism for invalidating a particular Delegation when used in conjunction with another Delegation in an Invocation proof chain. This is conceptually recursive, and more easily described in pictures:

``` mermaid
flowchart RL
    invoker((&nbsp&nbsp&nbsp&nbspDan&nbsp&nbsp&nbsp&nbsp))
    revoker((&nbsp&nbsp&nbsp&nbspBob&nbsp&nbsp&nbsp&nbsp))
    subject((&nbsp&nbsp&nbsp&nbspAlice&nbsp&nbsp&nbsp&nbsp))

    subgraph Delegations
        subgraph root [Root UCAN]
            subgraph rooting [Root Issuer]
                rootIss(iss: Alice)
                rootSub(sub: Alice)
            end

            rootAud(aud: Bob)
        end

        subgraph del1 [Delegated UCAN]
            del1Iss(iss: Bob) --> rootAud
            del1Aud(aud: Carol)
            del1Sub(sub: Alice)

            del1Sub --> rootSub
        end

        subgraph del2 [INVALIDATED Delegation]
            del2Iss(iss: Carol) --> del1Aud
            del2Aud(aud: Dan)
            del2Sub(sub: Alice)

            del2Sub --> del1Sub
        end
    end

    subgraph rev [Revocation]
        revArg("arg: {revoke: cid(carol_to_dan)}")
        revCmd("cmd: ucan/revoke")
        revIss(iss: Bob)
        revPrf("proofs")
    end

     subgraph inv [INVALIDATED Invocation]
        invSub(sub: Alice)
        invIss(iss: Dan)
        invPrf("proofs")
    end

    revoker --> revIss
    revArg:::revoked -.-> del2:::revoked
    revIss -.-> del1Iss
    revPrf:::revocation -.-> del1:::revocation

    inv:::revoked
    invPrf:::revoked

    invIss --> del2Aud
    invoker --> invIss
    invSub --> del2Sub
    rootIss --> subject
    rootSub --> subject
    invPrf --> Delegations

    classDef revocation stroke:blue,fill:#76b0ff
    classDef revoked stroke:red,fill:#ff7676,color:red
```

# 3 Semantics

Revocation is the act of invalidating a proof in a delegation chain for some specific UCAN delegation by its CID. All UCAN capabilities are either claimed by direct authority over the Subject, or by delegation chain terminating in that direct ("root") authority. Each link in a delegation chain contains an explicit issuer (delegator) and audience (delegatee).

_Revocations MUST be immutable and irreversible._ Recipients of revocations SHOULD treat them as a monotonically-growing set. If a Revocation was issued in error, it MUST NOT be retracted — a new, unique UCAN delegation MAY be issued (e.g. by updating the nonce or changing the time bounds). This prevents confusion as the revocation moves through the network and makes [revocation store]s append-only and highly amenable to caching and gossip.

## 3.1 Scope 

An Issuer of a particular Delegation in a proof chain MAY revoke that Delegation. Note that this is not always the same as revoking the Delegation they they Issued; any UCAN that contains a proof where the revoker matches the `iss` field — even transitively in the delegation chain — MAY be revoked.

Revocation of a particular proof does not guarantee that the Agent can no longer access to the capability in question. If an Agent is able to construct a valid proof chain without relying on the revoked proof, they still have access to the capability. By real-world analogy, if Mallory has two tickets to a film, and one of them is invalidated by its serial number, she is still able to present the valid ticket to see the film.

``` mermaid
flowchart TB
    subgraph RA[Alice can revoke]
        direction RL
        
        AB["(Root)\niss: Alice\naud: Bob\niff: [X,Y,Z]"]

        subgraph RB[Bob can revoke]
            BC["iss: Bob\naud: Carol\niff: [X,Y]"]
            BD["iss: Bob\naud: Dan\niff: [Y,Z]"]

            subgraph RC[Carol can revoke]
                CD["iss: Carol\naud: Dan\niff: [X,Y]"]

                subgraph RD[Dan can revoke]
                    DE["iss: Dan\naud: Erin\niff: [X,Y,Z]"]
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

Here Alice is the root Issuer. Alice MAY revoke any of the UCANs in the chain, Carol MAY revoke the two innermost, and so on. If the UCAN `Carol -> Dan` is revoked by Alice, Bob, or Carol, then Erin will not have a valid chain for the `X` capability, since its only proof is invalid. However, Erin can still prove the valid capability for `Y` and `Z` since the still-valid ("unbroken") chain `Alice to Bob to Dan to Erin` includes them. Note that despite `Y` being in the revoked `Carol -> Dan` UCAN, it does not invalidate `Y` for Erin, since the unbroken chain also included a proof for `Y`. 

## 3.2 Consistency Model

UCAN revocation is designed to work in the broadest possible scenarios, and as such needs very weak constraints. UCAN revocation MAY operate in fully eventually consistent contexts, with single sources of truth, or among nodes participating in consensus. The format of the revocation does not change in these situations; it is entirely managed by how revocations are passed around the network. 
Weak assumptions enable UCAN to work with eventually consistent resources, such as [CRDT]s, [Git] forks, delay-tolerant replicated state machines, and so on.

These weak assumptions are often associated with being unable to guarantee delivery in a certain time bound. Weak assumptions can always be strengthened, but not vice-versa. For example, if a capability describes access to a resource with a single location or source of truth, sending a revocation to that specific agent enables confirmation in bounded time. This grants nearly identical semantics that many people turn to [ACL]s for, but with all of the benefits of capabilities during delegation and invocation.

Out of order delivery is typical of distributed systems. Further, a malicious user can otherwise delay revealing that they have a capability until the last possible moment in hopes of evading detection. Accepting revocations for resources that the agent controls prior to the delegation targeted by the revocation is received is thus RECOMMENDED. 

# 4 Store

The Agent that controls a resource MUST maintain a cache of Revocations for which it is the Subject. The Agent MAY additionally cache gossiped Revocations about other Subjects as part of a [store and forward] mechanism.

During validation of a UCAN delegation chain, the [canonical CID] of each UCAN delegation MUST be checked against the cache. If there's a match, the relevant Delegation MUST be ignored. Note that this MAY NOT invalidate the entire UCAN chain.

``` js
// Pseudocode
const delegators = invocation.prf.map(proof => proof.iss)

invocation.prf.forEach(delegation => {
  // Is the proof in the revocation store?
  store.lookup(delegation).then(revocation => {
    // Is the revocation issuer in this proof chain?
    if (delegators.includes(revocation.iss)) {
      throw new Error("Invalidated via revocation by delegation issuer")
    }
    
    // Is the revocation based on a delegated revocation?
    const cids = revocation.iff.filter(cav => !!cav.rev)
    if (cids.length === 1 && invocation.prf.includes(cids[0])) {
      throw new Error("Invalidated by delegated revocation")
    }
  })
})
```

## 4.1 Locality

Resources with a single source of truth SHOULD follow the typical approach of maintaining a revocation store at the same physical location as the resource. For example, a centralized server MAY have an endpoint that lists the revoked UCANs by [canonical CID].

For eventually consistent data structures, this MAY be achieved by including the store directly inside the resource itself. For example, a CRDT-based file system SHOULD maintain the revocation store directly at a well-known path.

## 4.2 Monotonicity

Since Revocations MUST NOT be reversible, a new Delegation SHOULD be issued if a Revocation was issued in error.

``` mermaid
flowchart LR
    Alice((&nbsp&nbsp&nbspAlice&nbsp&nbsp&nbsp))
    Bob((&nbsp&nbsp&nbspBob&nbsp&nbsp&nbsp))
    Carol((&nbsp&nbsp&nbspCarol&nbsp&nbsp&nbsp))
    Dan((&nbsp&nbsp&nbspDan&nbsp&nbsp&nbsp))

    del1{{Delegate\ncan: crud/read Alice's DB}}
    del2{{Delegate\ncan: crud/read Alice's DB}}
    del3{{Delegate\ncan: crud/read Alice's DB}}
    newDel{{"Delegate\ncan: crud/read Alice's DB\n(Resissued) "}}

    Alice === del1 ==> Bob === del2:::Revoked ===x Carol === del3 ==> Dan
    Alice === newDel:::Reissued ===> Carol

    rev>Revoke!]
    Alice === rev:::Invocation
    rev -.->|rev| del2

    classDef Invocation stroke:#F00,fill:#F00,color:#000;
    classDef Revoked stroke:#F00;
    classDef Reissued stroke:green;

    linkStyle 2 stroke:red
    linkStyle 3 stroke:red

    linkStyle 8 stroke:red
    linkStyle 9 stroke:red

    linkStyle 6 stroke:green
    linkStyle 7 stroke:green
```

## 4.3 Eviction

Revocations MAY be evicted once the UCAN that they reference expires or otherwise becomes invalid through its proactive mechanisms, such as expiry (`exp`) plus some clock-skew buffer.

# 5 Delegating Revocation

The authority to revoke some Delegation MAY be itself delegated to a Principal not in the delegation chain. The revoked Delegation SHOULD be referenced by its [canonical CID].

| Field | Value                    |
|-------|--------------------------|
| `can` | `"ucan/revoke"`          |
| `iff` | `[{"rev": &Delegation}]` |

This is a Delegation of the ability to Revoke:

``` js
{
  "iss": "did:web:alice.example.com",
  "aud": "did:web:zelda.example.com",
  "sub": "did:web:alice.example.com",
  "can": "ucan/revoke",
  "iff": [
    {"rev": {"/": "bafkreiem4on23qnu2nn2jg7vwzxkns6sxi5faysq7ekwtjhugqga3vbhim"}}
  ],
  // ...
}
```

``` mermaid
flowchart LR
    Alice((&nbsp&nbsp&nbspAlice&nbsp&nbsp&nbsp))
    Bob((&nbsp&nbsp&nbspBob&nbsp&nbsp&nbsp))
    Carol((&nbsp&nbsp&nbspCarol&nbsp&nbsp&nbsp))
    Dan((&nbsp&nbsp&nbspDan&nbsp&nbsp&nbsp))
    Zelda((&nbsp&nbsp&nbspZelda&nbsp&nbsp&nbsp))

    del1{{Delegate\ncan: crud/read Alice's DB}}
    del2{{Delegate\ncan: crud/read Alice's DB}}
    del3{{Delegate\ncan: crud/read Alice's DB}}:::Revoked

    delRev{{Delegate\ncan: ucan/revoke}}

    Alice === del1 ==> Bob === del2 ===> Carol === del3 ===x Dan
    Alice === delRev ===> Zelda
    delRev -.->|cid| del2

    rev>Revoke]
    Zelda === rev:::Invocation ===> Alice
    rev:::Invocation -.->|rev| del3

    classDef Revoked stroke:#F00;
    classDef Invocation stroke:#F00,fill:#F00,color:#000;

    linkStyle 4 stroke:red
    linkStyle 5 stroke:red
    linkStyle 9 stroke:red
    linkStyle 10 stroke:red
```

# 6 Invoking Revocation

A revocation Action MUST take the following shape:

| Field | Value           |
|-------|-----------------|
| `cmd` | `"ucan/revoke"` |
| `arg` | See [Arguments] |
| `nnc` | `""`            |

Note that per [UCAN Invocation], the `nnc` field SHOULD is set to `""` since revocation is idempotent.

## 6.1 Arguments

Being expressed as an Invocation means that Revocations MUST define an Action type for the command `ucan/revoke`.

| Field | Type            | Required | Description                                                              |
|-------|-----------------|----------|--------------------------------------------------------------------------|
| `rev` | `&Delegation`   | Yes      | The CID of the [UCAN Delegation] that is being revoked                   |
| `pth` | `[&Delegation]` | No       | A [delegation path] that includes the Revoker and the revoked Delegation |

### 6.1.1 Path Witness

Since all delegation chains MUST be rooted in a Delegation where the `iss` and `sub` fields are equal, the root Issuer is a priori in every delegation chain. This is not the case for sub-delegation. There are many paths through the authority network. For example, take the following delegation network:

``` mermaid
flowchart LR
    Alice -->|delegates| Bob -->|delegates| Dan -->|delegates| Erin
    Bob -->|delegates| Carol -->|delegates| Erin
    Alice -->|delegates| Mallory
```

Mallory is not in the delegation chain of Erin. This is fine, since the semantics of revocation merely state that she would assert that no delegation of hers may be used in the `prf` field of an Invocation if it also includes the `rev` Delegation. However, issuing spurious Revocations and requiring them to be stored is a potential DoS vector. Executors MAY require a delegation path witness be included to avoid this situation.
 
Unlike Mallory, Bob, Carol, and Dan can both provide valid delegation paths that include Delegations that they have issued. Bob has two paths (`Alice -> Bob -> Dan -> Erin` or `Alice -> Bob -> Carol -> Erin`), and either will suffice.

# 7 Prior Art

[Revocation lists][Cert Revocation Wikipedia] are a fairly widely used concept.

[SPKI/SDSI] is closely related to UCAN. A different format is used, and some details vary (such as a delegation-locking bit), but the core idea and general usage pattern are very close. UCAN can be seen as making these ideas more palatable to a modern audience and adding a few features such as content IDs that were less widespread at the time SPKI/SDSI were written.

[X.509 Certificate Revocation Lists][RFC 5280] defines two kinds of certificate invalidation: temporary ("hold") and permanent ("revocation"). This RFC also includes a field for indicating a reason for revocation. UCAN Revocation has no concept of a temporary hold on a capability, but this behavior MAY be emulated by revoking a credential and issuing a new UCAN with a `nbf` field set to a time in the future.

[ZCAP-LD] is closely related to UCAN, but situated in the W3C-style linked data world (the "LD" in ZCAP-LD). Revocation in ZCAP-LD is only granted to those who have a special caveat on a capability. In contrast, UCAN capabilities MAY be revoked by anyone in the relevant delegation path.

[OAuth 2.0 Revocation][RFC 7009] is very similar to UCAN revocation. It is largely concerned with the HTTP interactions to make OAuth revocation work. OAuth doesn't have a concept of sub-delegation, so only the user that has been granted the token can revoke it. However, this may cascade to revocation of other tokens, but the exact mechanism is left to the implementer.

While strictly speaking being about assertions rather than capabilities, [Verfiable Credential Revocation][VC Revocation] spec follows a similar pattern to those listed above.

[E][E-lang]-style [object capabilities] use active network connections with [proxy agents][Robust Composition] to revoke delegations. Revocation is achieved by shutting down that proxy to break the authorizing reference. In many ways, UCAN Revocation attempts to emulate this behavior. Unlike UCAN Revocations, E-style object capabilities are [fail-safe] and thus by definition not partition tolerant.

# 8 Acknowledgements

Thank you [Blaine Cook] for the real-world feedback, ideas on future features, and lessons from other auth standards.

Thanks to [Juan Caballero] for the numerous questions, clarifications, and general advice on putting together a comprehensible spec.

Many thanks to [Alan Karp] for sharing his vast experience with capability-based authorization, patterns, and many right words for us to search for.

Thanks to [Benjamin Goering] for the many community threads and connections to [W3C] standards.

Many thanks to [Christine Lemmer-Webber] for her handwritten(!) feedback on the design of UCAN, spearheading the [OCapN] initiative, and her related work on [ZCAP-LD].

Thanks to the entire [SPKI WG][SPKI/SDSI] for their closely related pioneering work.

We want to especially recognize [Mark Miller] for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

# 9. Acknowledgments

Thanks to the entire [SPKI WG][SPKI/SDSI] for their closely related pioneering work.

Many thanks to [Alan Karp] for sharing his vast experience with capability-based authorization, patterns, and many right words for us to search for.

We want to especially recognize [Mark Miller] for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

<!-- Footnotes -->

<!-- Internal Links -->

[Arguments]: #61-arguments
[delegation path]: #611-path-witness
[revocation store]: #4-store

<!-- External Links -->

[ACL]: https://en.wikipedia.org/wiki/Access-control_list
[Alan Karp]: https://github.com/alanhkarp
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Benjamin Goering]: https://github.com/gobengo
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brendan O'Brien]: https://github.com/b5
[Brian Ginsburg]: https://github.com/bgins
[Brooklyn Zelenka]: https://github.com/expede
[CIDv1]: https://github.com/multiformats/cid?tab=readme-ov-file#cidv1
[CRDT]: https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type
[Cert Revocation Wikipedia]: https://en.wikipedia.org/wiki/Certificate_revocation
[Christine Lemmer-Webber]: https://github.com/cwebber
[Christopher Joel]: https://github.com/cdata
[Command]: https://github.com/ucan-wg/spec#33-command
[DAG-CBOR]: https://ipld.io/specs/codecs/dag-cbor/spec/
[DID fragment]: https://www.w3.org/TR/did-core/#terminology
[DID]: https://www.w3.org/TR/did-core/
[Dan Finlay]: https://github.com/danfinlay
[Daniel Holmgren]: https://github.com/dholms
[E-lang]: http://www.erights.org/
[ES256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.4
[EdDSA]: https://en.wikipedia.org/wiki/EdDSA
[Executor]: https://github.com/ucan-wg/spec#31-roles
[Fission]: https://fission.codes
[Git]: https://git-scm.com/
[Hugo Dias]: https://github.com/hugomrdias
[IEEE-754]: https://ieeexplore.ieee.org/document/8766229
[IPLD]: https://ipld.io/
[Ink & Switch]: https://www.inkandswitch.com/
[Invocation]: https://github.com/ucan-wg/invocation
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[Juan Caballero]: https://github.com/bumblefudge
[Mark Miller]: https://github.com/erights
[Martin Kleppmann]: https://martin.kleppmann.com/
[Mikael Rogers]: https://github.com/mikeal/
[OCAP]: https://en.wikipedia.org/wiki/Object-capability_model
[OCapN]: https://github.com/ocapn/
[Philipp Krüger]: https://github.com/matheus23
[PoLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[Protocol Labs]: https://protocol.ai/
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 5280]: https://www.rfc-editor.org/rfc/rfc5280
[RFC 7009]: https://www.rfc-editor.org/rfc/rfc7009
[RFC 8037]: https://www.rfc-editor.org/rfc/rfc8037
[RS256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.3
[Raw data multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L41
[Robust Composition]: http://www.erights.org/talks/thesis/markm-thesis.pdf
[SHA2-256]: https://github.com/multiformats/multicodec/blob/master/table.csv#L9
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Steven Vandevelde]: https://github.com/icidasset
[UCAN Delegation]: https://github.com/ucan-wg/delegation
[UCAN Invocation]: https://github.com/ucan-wg/invocation
[UCAN]: https://github.com/ucan-wg/spec
[VC Revocation]: https://learn.microsoft.com/en-us/azure/active-directory/verifiable-credentials/how-to-issuer-revoke
[W3C]: https://www.w3.org/
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[`did:key`]: https://w3c-ccg.github.io/did-method-key/
[`did:plc`]: https://github.com/did-method-plc/did-method-plc
[`did:web`]: https://w3c-ccg.github.io/did-method-web/
[base32]: https://github.com/multiformats/multibase/blob/master/multibase.csv#L13
[block list]: https://en.wikipedia.org/wiki/Blacklist_(computing)
[canonical CID]: https://github.com/ucan-wg/spec#41-content-identifiers
[dag-json multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L112
[did:key ECDSA]: https://w3c-ccg.github.io/did-method-key/#p-256
[did:key EdDSA]: https://w3c-ccg.github.io/did-method-key/#ed25519-x25519
[did:key RSA]: https://w3c-ccg.github.io/did-method-key/#rsa
[external resource]: https://github.com/ucan-wg/spec#55-wrapping-existing-systems
[fail-safe]: https://en.wikipedia.org/wiki/Fail-safe
[local-first]: https://www.inkandswitch.com/local-first/
[object capabilities]: https://en.wikipedia.org/wiki/Object-capability_model
[revocation]: https://github.com/ucan-wg/revocation
[store and forward]: https://en.wikipedia.org/wiki/Store_and_forward
[ucan.xyz]: https://ucan.xyz
