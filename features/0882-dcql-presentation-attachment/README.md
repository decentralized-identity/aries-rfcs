# Aries RFC 0882: DCQL Attachment format for requesting and presenting proofs

- Authors: [Tarun Vadde](mailto:vaddeofficial@gmail.com) (2060.io)
- Status: [PROPOSED](/README.md#proposed)
- Since: 2026-10-28
- Status Note: Initial draft. See [Implementations](#implementations).
- Supersedes:
- Start Date: 2026-10-28
- Tags: [feature](/tags.md#feature), [protocol](/tags.md#protocol), [credentials](/tags.md#credentials)

## Summary

This RFC registers attachment formats for the [Present Proof Protocol 2.0 (RFC 0454)](../0454-present-proof-v2/README.md) that use the Digital Credentials Query Language (DCQL) from [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) (OpenID4VP). A verifier sends a DCQL query with a nonce and an audience. The prover answers with a `vp_token` object in the shape OpenID4VP 1.0 defines. The formats work over DIDComm v1 and DIDComm v2 and cover three credential formats:

- IETF SD-JWT VC (`dc+sd-jwt`)
- W3C Verifiable Credentials Data Model 2.0 secured with SD-JWT (`vc+sd-jwt`), the format issued by [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md)
- ISO mdoc (`mso_mdoc`)

For `mso_mdoc` this RFC defines a DIDComm specific `Handover` for the ISO 18013-5 `SessionTranscript`, the DIDComm counterpart of `OpenID4VPHandover`.

## Motivation

- The present-proof v2 registry has request and presentation formats for Hyperledger Indy, DIF Presentation Exchange and AnonCreds only. There is no way to request or present an SD-JWT VC or an mdoc over DIDComm.
- RFC 0881 issues `vc+sd-jwt` credentials over issue-credential v2. Its Drawbacks section says a separate attachment format is needed to present them with RFC 0454.
- Agents that hold `dc+sd-jwt` or `mso_mdoc` credentials issued with [OpenID for Verifiable Credential Issuance](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) have no DIDComm path to present them to verifiers they already have a connection with.
- OpenID4VP 1.0 already defines a query language (DCQL), a response shape (`vp_token`) and the binding rules for `dc+sd-jwt` and `mso_mdoc`. This RFC defines the identifier and binding rules for `vc+sd-jwt`. Reusing DCQL, `vp_token` and the OpenID4VP binding rules avoids a new Aries query language and lets one implementation serve both OpenID4VP and DIDComm.
- DCQL names the credential format per query. One attachment format can therefore cover `dc+sd-jwt`, `vc+sd-jwt` and `mso_mdoc`, including requests that mix them.

## Tutorial

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY" and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

The roles are the `verifier` and the `prover` of [RFC 0454](../0454-present-proof-v2/README.md). The prover is the holder of the credentials.

### Overview

1. The verifier sends a `request-presentation` with one attachment in the `didcomm/dcql-request@v1.0` format. It holds a DCQL query, a fresh `nonce` and an `audience` that names the verifier.
2. The prover checks that the `audience` names the party the DIDComm layer will deliver the presentation to, finds credentials that match the query and gets the user's consent, or applies its policy when it presents without a user.
3. The prover sends a `presentation` with one attachment in the `didcomm/dcql-presentation@v1.0` format. It holds a `vp_token` with one or more presentations. Presentations with holder binding are bound to the `nonce` and the `audience`.
4. The verifier checks the presentations and answers with an `ack` (when `will_confirm` is `true`) or a `problem-report`.

Holder binding ties a presentation to the key the credential is bound to: the `cnf` claim for `dc+sd-jwt` and `vc+sd-jwt`, and the device key in the MSO for `mso_mdoc`. It does not use the DIDComm connection keys or the prover's DID for the connection, which only show who sent the message (step 2 of [Verifier processing rules](#verifier-processing-rules)). A credential received over OpenID4VCI and bound to a `did:key`, a `did:jwk` or a plain JWK can therefore be presented over a `did:peer` connection that has nothing to do with that key. Whether holder binding is required comes from `require_cryptographic_holder_binding` in the request, never from the presentation.

A single query can ask for credentials in different formats. For example, this query accepts either an SD-JWT VC or an mDL:

```json
{
  "credentials": [
    {
      "id": "pid",
      "format": "dc+sd-jwt",
      "meta": { "vct_values": ["https://credentials.example.com/identity_credential"] },
      "claims": [{ "path": ["family_name"] }, { "path": ["given_name"] }]
    },
    {
      "id": "mdl",
      "format": "mso_mdoc",
      "meta": { "doctype_value": "org.iso.18013.5.1.mDL" },
      "claims": [
        { "path": ["org.iso.18013.5.1", "family_name"] },
        { "path": ["org.iso.18013.5.1", "given_name"] }
      ]
    }
  ],
  "credential_sets": [{ "options": [["pid"], ["mdl"]] }]
}
```

Complete examples of messages are provided in the [reference section](#reference).

## Reference

### Attachment format identifiers

| Message | Format identifier | Content |
| --- | --- | --- |
| `propose-presentation` | `didcomm/dcql-request@v1.0` | `dcql_query` only |
| `request-presentation` | `didcomm/dcql-request@v1.0` | `dcql_query`, `nonce`, `audience` |
| `presentation` | `didcomm/dcql-presentation@v1.0` | `vp_token` |

Proposals and requests share one identifier, as in [RFC 0510](../0510-dif-pres-exch-attach/README.md) and [RFC 0771](../0771-anoncreds-attachments/README.md). Read the message `@type` (or `type` in DIDComm v2) together with the `format` to know which one an attachment is.

All attachments of these formats are JSON objects and use the `application/json` mime type. Implementations MUST ignore unknown top-level properties of these attachments other than `transaction_data`, and unknown properties inside `dcql_query` as DCQL requires ([OpenID4VP section 6, Digital Credentials Query Language (DCQL)](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6)).

[RFC 0454](../0454-present-proof-v2/README.md) says multiple attachments in one message are alternatives for the same presentation. A message therefore carries at most one attachment in these formats, and that attachment holds the whole query or the whole `vp_token`.

### `propose-presentation` attachment format

Format identifier: `didcomm/dcql-request@v1.0`

A prover can propose what it is willing to present. The attachment is a JSON object with:

- `dcql_query` - Required. A DCQL query as defined in [OpenID4VP section 6](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6), with the format rules in [Credential formats](#credential-formats).

`nonce` and `audience` MUST NOT be present in a proposal. A verifier that accepts a proposal sends a `request-presentation` with a query it is willing to verify (which can be the proposed one), a fresh `nonce` and its `audience`.

A `propose-presentation` with a `formats` entry for `didcomm/dcql-request@v1.0` and no attachment tells the verifier that the prover supports this format, as described in [RFC 0454](../0454-present-proof-v2/README.md#propose-presentation).

A complete message is shown in the [`propose-presentation` example](#propose-presentation-example).

### `request-presentation` attachment format

Format identifier: `didcomm/dcql-request@v1.0`

The attachment is a JSON object with:

- `dcql_query` - Required. A DCQL query as defined in [OpenID4VP section 6](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6), with the format rules in [Credential formats](#credential-formats).
- `nonce` - Required. A string the verifier creates for this request. It MUST be fresh for every `request-presentation`, MUST come from a cryptographically secure random source with at least 120 bits of entropy, and MUST only contain ASCII URL safe characters (letters, digits, `-`, `.`, `_` and `~`). A version 4 UUID made with such a source ([RFC 9562 section 5.4, UUID Version 4](https://www.rfc-editor.org/rfc/rfc9562.html#section-5.4), 122 random bits) meets this, and so does the base64url encoding of 16 or more random bytes. These are the rules of [OpenID4VP section 5.2, Existing Parameters](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5.2) plus a minimum entropy. The verifier stores it with the protocol thread.
- `audience` - Required. The string `didcomm:` followed by a DID of the verifier, without a path, query or fragment. The verifier chooses which of its DIDs to name, for example its public DID, within the rules of [Audience](#audience).

```json
{
  "dcql_query": {
    "credentials": [
      {
        "id": "pid",
        "format": "dc+sd-jwt",
        "meta": { "vct_values": ["https://credentials.example.com/identity_credential"] },
        "claims": [{ "path": ["given_name"] }, { "path": ["family_name"] }]
      }
    ]
  },
  "nonce": "rRgk_0pAsc6WId7Z-xqdq4GFBe6Mz8yWnilxR52-eAs",
  "audience": "didcomm:did:example:verifier"
}
```

When a verifier sends more than one `request-presentation` in a thread (for example after a counter proposal), each one gets a new `nonce`. Only the last request the verifier sent in the thread is valid.

These formats use only the parts of OpenID4VP 1.0 that this RFC references. Other OpenID4VP parameters and features are not supported. The exception is `transaction_data`, which a Wallet that does not support it must reject instead of ignore. A prover MUST reject a request whose attachment has a `transaction_data` property ([OpenID4VP section 5, Authorization Request](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5), [section 8.4, Transaction Data](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-8.4)).

#### Audience

The `audience` plays the role of the OpenID4VP Client Identifier. The prover copies it unchanged into every presentation that has holder binding: the `aud` of a KB-JWT, the `aud` of a `vp+sd-jwt` and the first element of the [DIDComm Handover](#didcomm-handover) info. The verifier compares it as an exact string with the value it sent.

The `didcomm:` prefix keeps these presentations apart from presentations to the same DID in protocols that use the bare DID as the audience, such as OpenID4VP drafts before 1.0 with the `did` Client Identifier Scheme, or an OpenID4VP 1.0 pre-registered client whose Client Identifier is the DID ([OpenID4VP section 5.9.2, Fallback](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5.9.2)). OpenID4VP 1.0 itself writes a DID Client Identifier with the `decentralized_identifier:` prefix ([OpenID4VP section 5.9.3, Defined Client Identifier Prefixes](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5.9.3)). OpenID4VP uses `origin:` the same way for the Digital Credentials API ([OpenID4VP Appendix A.4, Response](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-A.4)). OpenID4VP lets a wallet treat a Client Identifier with an unknown prefix as a pre-registered client ([OpenID4VP section 5.9.2, Fallback](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5.9.2)). A wallet that also implements OpenID4VP MUST therefore reject an OpenID4VP request whose Client Identifier starts with `didcomm:`, including as a pre-registered client.

The value only protects against replay to another verifier if the prover checks it. Before it creates any presentation, the prover MUST check that the DID in `audience` identifies the party that will receive the presentation. The check uses the DIDComm layer (what it will encrypt to, or the invitation DID the connection was verified against), never the attachment alone.

Over DIDComm v1, messages have no `from`, so the check uses the `recipientKeys` of the service the prover sends the `presentation` to. With a connection, that is the verifier's DIDComm service for the connection. Without a connection, it is the service in the [`~service` decorator](../0056-service-decorator/README.md) of the request when it has one, and otherwise the service in the `services` of the [out-of-band invitation](../0434-outofband/README.md) that carried the request (for a DID in `services`, the DIDComm service in that DID's document). When a request in an out-of-band invitation also has a `~service` decorator, the prover checks the `audience` against both. Recipient key rule: the DID MUST resolve to a DID document that has every one of these recipient keys. Each of them MUST be referenced from the `keyAgreement` or `authentication` relationship of the DID document or listed in `recipientKeys` of a DIDComm service in it. The X25519 key converted from an Ed25519 recipient key does not count as that recipient key. Routing keys are not part of this check. The verifier's DID for the connection meets this rule, and so does a [`did:key`](../0360-use-did-key/README.md) of the recipient key when the service has only one. The verifier MAY also name any other DID it controls whose DID document meets this rule, such as its public DID.

Invitation DID rule: over a connection where the verifier was the inviter, the prover MAY also accept as `audience` the DID of the invitation that the connection was made from, when one of these holds:

- The connection was made with [DID Exchange (RFC 0023)](../0023-did-exchange/README.md#2-exchange-response) and the `did_rotate~attach` or `did_doc~attach` of the `response` is signed with a key of the invitation DID, or with the [Connection Protocol (RFC 0160)](../0160-connection-protocol/README.md#2-connection-response) and the `connection~sig` of the `response` is signed with such a key. The key MUST be one the prover encrypted its `request` to. The DID that is signed (the `did_rotate~attach` value, the `id` of the DID document in `did_doc~attach`, or the DID in the signed `connection`) MUST be the verifier's DID for the connection. The prover MUST have verified that signature itself when it accepted the connection. Both attachments are optional and the requester validates them only when they are present, so a `response` with neither of them signed does not meet this rule.
- The verifier's DID for the connection is the invitation DID.

The invitation DID MUST be a resolvable DID: the DID in the `services` entry of the [out-of-band invitation](../0434-outofband/README.md) that the prover sent its `request` to, the `did` of an RFC 0160 invitation, or the public DID of an [implicit invitation](../0023-did-exchange/README.md#implicit-and-explicit-invitations). Another DID listed in the invitation does not count. An invitation whose `services` are only inline service blocks with keys has no invitation DID for this rule. When the verifier was the invitee there is no such link, and only the recipient key rule above applies. A connection reused with the [RFC 0434 reuse messages](../0434-outofband/README.md#reuse-messages) keeps the invitation DID it was made from, and so does a connection after a later DID rotation such as [RFC 0794](../0794-did-rotate/README.md). This adds no new trust assumption. When it accepted the connection, the prover already tied the invitation DID to it with that signature, or the two DIDs are the same.

A request that arrives over a connection is checked against that connection, also when its thread matches a request the prover saw in an out-of-band invitation. A verifier that wants the presentation over a connection sends the `request-presentation` after the connection exists, not in `requests~attach` of the invitation that sets it up, because it does not know its keys for that connection when it creates the invitation. A prover that answers an attached request over a connection made from or reused for the invitation checks the `audience` against that connection's keys as above.

When a DID method defines more than one form of the same DID, such as the short and long forms of `did:peer:4`, the prover treats them as the same DID in these checks. Over a connection that the DIDComm layer matched by the sender of the request, the DID the prover holds for the verifier on that connection passes these checks.

If the check fails, the prover MUST NOT send a presentation (see [Prover processing rules](#prover-processing-rules)). Over DIDComm v2, the check is the one in [DIDComm v2](#didcomm-v2).

### `presentation` attachment format

Format identifier: `didcomm/dcql-presentation@v1.0`

The attachment is a JSON object with:

- `vp_token` - Required. An object as defined for the `vp_token` response parameter in [OpenID4VP section 8.1, Response Parameters](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-8.1). Each key is the `id` of a Credential Query in the request. Each value is a non-empty array of presentations that match that query. The array MUST hold exactly one presentation unless the query has `multiple` set to `true`. An optional query that the prover cannot match has no entry. For the three formats in this RFC every presentation is a string, and every SD-JWT in it, including one in a `data:` URL, uses the compact serialization of [RFC 9901 section 4](https://www.rfc-editor.org/rfc/rfc9901.html#section-4).

```json
{
  "vp_token": {
    "pid": [
      "eyJhbGciOiJFUzI1NiIsInR5cCI6ImRjK3NkLWp3dCIsImtpZCI6ImRpZDpleGFtcGxlOmlzc3VlciNrZXktMSJ9...(clipped)...~eyJhbGciOiJFUzI1NiIsInR5cCI6ImtiK2p3dCJ9...(clipped)"
    ]
  }
}
```

### Credential formats

DCQL Credential Queries in these attachments can use the Credential Format Identifiers below. Credential Queries with any other `format` are out of scope for this version. A prover treats them as queries it cannot satisfy. An option in `credential_sets` that names one of them is not met, but another option of the same set can be.

| Credential Format Identifier | Credential | Required `meta` | Presentation |
| --- | --- | --- | --- |
| `dc+sd-jwt` | IETF SD-JWT VC | `vct_values` | SD-JWT with a KB-JWT |
| `vc+sd-jwt` | W3C VCDM 2.0 secured with SD-JWT | `type_values` | `vp+sd-jwt` signed by the holder |
| `mso_mdoc` | ISO mdoc | `doctype_value` | `DeviceResponse` with a `deviceSignature` over `DeviceAuthenticationBytes`, whose `SessionTranscript` uses the DIDComm Handover |

The `meta` object is REQUIRED in every Credential Query ([OpenID4VP section 6.1, Credential Query](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.1)). Claims path pointers follow [OpenID4VP section 7, Claims Path Pointer](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-7).

#### `dc+sd-jwt`

The credential is an IETF SD-JWT VC ([SD-JWT VC draft -22](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22)). The identifier, `meta` and presentation rules are those of [OpenID4VP Appendix B.3, IETF SD-JWT VC](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.3), with the `audience` of the request in place of the Client Identifier.

- `meta.vct_values` is REQUIRED ([OpenID4VP Appendix B.3.5, Parameter in the meta parameter in Credential Query](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.3.5)). For matching, the prover and the verifier SHOULD treat a credential as being of a type if that type is its `vct` or one of its `aka_vcts` values ([SD-JWT VC draft -22 section 2.2.2.2, Other Credential Types - aka_vcts Claim](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.2.2.2)). A prover that processes Type Metadata MAY also return a credential whose `vct` extends a listed type ([SD-JWT VC draft -22 section 5.4, Extending Type Metadata](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-5.4)), but a verifier that does not process Type Metadata rejects it.
- Claims path pointers start at the root of the SD-JWT VC payload.
- When holder binding is required (`require_cryptographic_holder_binding` is absent or `true`), the presentation is an SD-JWT with a Key Binding JWT (SD-JWT+KB). A credential without a `cnf` claim cannot be returned. The KB-JWT follows [RFC 9901 section 4.3, Key Binding JWT](https://www.rfc-editor.org/rfc/rfc9901.html#section-4.3):
  - header `typ` is `kb+jwt` and `alg` is a digital signature algorithm, never `none`
  - `nonce` is the `nonce` of the request
  - `aud` is the `audience` of the request
  - `iat` is the time the KB-JWT was created
  - `sd_hash` is computed as in [RFC 9901 section 4.3.1, Binding to an SD-JWT](https://www.rfc-editor.org/rfc/rfc9901.html#section-4.3.1)
  - it is signed with the holder key named by the credential's `cnf` claim (`cnf.jwk`, or `cnf.kid` as an absolute DID URL) ([SD-JWT VC draft -22 section 2.4, Verification and Processing](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.4)). With `cnf.kid`, the verification method it dereferences to MUST be in the `authentication` relationship of the resolved DID document ([DID Core section 5.3.1, Authentication](https://www.w3.org/TR/did-1.0/#authentication)).
- When `require_cryptographic_holder_binding` is `false`, the prover MAY return the SD-JWT without a KB-JWT. No nonce comes back in that case. The DIDComm thread links the response to the request, the job OpenID4VP gives to `state` ([OpenID4VP section 5.3, Requesting Presentations without Holder Binding Proofs](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5.3)). Such a presentation has no replay protection ([OpenID4VP section 14.1.1, Presentations without Holder Binding Proofs](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-14.1.1)).
- The verifier gets the issuer key with a key discovery method its policy permits for that issuer ([SD-JWT VC draft -22 section 2.5, Issuer Verification Key Discovery and Validation](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.5)). Besides the two mechanisms defined there, a verifier MAY permit DID-based key discovery. `iss` is a DID and the header `kid` MUST be a DID URL of that DID: either relative, such as `#key-1`, and resolved against `iss`, or absolute with the same DID as `iss`. The key is the verification method that DID URL dereferences to, and it MUST be in the `assertionMethod` relationship of the resolved DID document. A verifier that permits DID-based key discovery MUST use only that mechanism when `iss` is a DID, so that an attacker cannot change which mechanism is used, for example by adding an `x5c` header ([SD-JWT VC draft -22 section 7.3, Ecosystem-specific Public Key Verification Methods](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-7.3)). The `dc+sd-jwt` examples in this RFC use DID-based key discovery.

#### `vc+sd-jwt`

OpenID4VP 1.0 defines no Credential Format Identifier for W3C Verifiable Credentials Data Model 2.0. Its W3C appendix covers VCDM 1.1 only ([OpenID4VP Appendix B.1, W3C Verifiable Credentials](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.1)). Appendix B lets other specifications define the parameters and rules for Credential Formats it does not mention, and this RFC does so for VCDM 2.0 secured with SD-JWT under the identifier `vc+sd-jwt`:

- The credential is a [VCDM 2.0](https://www.w3.org/TR/vc-data-model-2.0/) credential secured with SD-JWT as in [VC-JOSE-COSE section 3.2.1, Securing JSON-LD Verifiable Credentials with SD-JWT](https://www.w3.org/TR/vc-jose-cose/#securing-with-sd-jwt), with media type `application/vc+sd-jwt` ([VC-JOSE-COSE section 6.1.3, application/vc+sd-jwt](https://www.w3.org/TR/vc-jose-cose/#vc-json-sd-jwt)). Its header `typ`, when present, MUST be `vc+sd-jwt`. This is the credential [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md) issues.
- `@context`, `type`, `issuer`, `validFrom`, `validUntil`, `credentialStatus`, `credentialSchema`, `relatedResource` and `cnf`, and any property inside them, MUST NOT be selectively disclosable. [RFC 9901 section 9.7, Selectively Disclosable Validity Claims](https://www.rfc-editor.org/rfc/rfc9901.html#section-9.7) says an Issuer MUST NOT make content that is critical for the authenticity or validity of an SD-JWT selectively disclosable and leaves the list of that content to profiles. VC-JOSE-COSE section 3.2.1 names some of these properties, but only as a SHOULD NOT. Without this rule a holder could withhold `validUntil` or `credentialStatus` and hide that the credential expired or was revoked. This is a rule for issuers. A verifier cannot tell which property a withheld top-level disclosure was for, so for that case it relies on trusting the issuer.
- `meta.type_values` is REQUIRED. It is a non-empty array of non-empty string arrays, as in [OpenID4VP Appendix B.1.1, Parameters in the meta parameter in Credential Query](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.1.1). Its values are compared as exact strings with the `type` values as they appear in the credential, with no JSON-LD processing. A `type` that is a single string counts as an array of one. Each inner array is one alternative. A credential matches an inner array when its `type` includes every value in that array, in any order and with other types allowed. The credential matches `type_values` when it matches at least one inner array. This differs from Appendix B.1.1, which compares fully expanded type IRIs. OpenID4VP 1.0 has no rules for VCDM 2.0 credentials, and expanded matching would need the context documents of each credential, which VC-JOSE-COSE processing does not otherwise need. The same `type` value can mean different things under different contexts, so a verifier that needs a type to have a set meaning also checks the credential `@context`, as in [VCDM 2.0 section 6.3, Type-Specific Credential Processing](https://www.w3.org/TR/vc-data-model-2.0/#type-specific-credential-processing).
- OpenID4VP drafts up to -22 used `vc+sd-jwt` as the identifier for IETF SD-JWT VC (renamed to `dc+sd-jwt` in draft -23). A query written for those drafts has no `type_values`, so it is invalid here.
- Claims path pointers start at the root of the credential, not the presentation ([OpenID4VP Appendix B.1.2, Claims Matching](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.1.2)). For example `["credentialSubject", "degree", "name"]`.

When holder binding is required (`require_cryptographic_holder_binding` is absent or `true`), the credential MUST have a `cnf` claim and the presentation is a compact `vp+sd-jwt` as in [VC-JOSE-COSE section 3.2.2, Securing JSON-LD Verifiable Presentations with SD-JWT](https://www.w3.org/TR/vc-jose-cose/#securing-vps-sd-jwt):

- Header `typ` MUST be `vp+sd-jwt`. This tightens the SHOULD in VC-JOSE-COSE section 3.2.2. When present, `cty` SHOULD be `vp`. `alg` is a digital signature algorithm, never `none`.
- The payload is a VCDM 2.0 `VerifiablePresentation`. `verifiableCredential` holds exactly one `EnvelopedVerifiableCredential` ([VCDM 2.0, Enveloped Verifiable Credentials](https://www.w3.org/TR/vc-data-model-2.0/#enveloped-verifiable-credentials)). Its `id` is `data:application/vc+sd-jwt,` followed by the credential SD-JWT with the disclosures the prover selected. That SD-JWT MUST NOT have a KB-JWT.
- The payload MUST have `nonce` (the `nonce` of the request) and `aud` (the `audience` of the request, as a single string, not an array) as plain JWT claims. The base context defines `aud` and `iat` but defines `nonce` only inside Data Integrity proofs, so unless another context in the payload `@context` defines it, `https://www.w3.org/ns/credentials/undefined-terms/v2` MUST be its last value ([VCDM 2.0 section 5.2, Extensibility (Semantic Interoperability)](https://www.w3.org/TR/vc-data-model-2.0/#semantic-interoperability)). `iat` is RECOMMENDED. `nbf` is NOT RECOMMENDED ([VC-JOSE-COSE section 3.1.3, JOSE Header Parameters and JWT Claims](https://www.w3.org/TR/vc-jose-cose/#jose-header-parameters-jwt-claims)) and the claim names `vc` and `vp` MUST NOT be used ([VC-JOSE-COSE section 1.1.2.1, JWT Format and Requirements](https://www.w3.org/TR/vc-jose-cose/#jwt-format-and-requirements)). `nonce`, `aud` and `iat` MUST NOT be selectively disclosable. The `vp+sd-jwt` SHOULD have no disclosures.
- The `vp+sd-jwt` MUST be signed with the holder key named by the `cnf` claim of the enveloped credential (`cnf.jwk`, or `cnf.kid` as an absolute DID URL whose verification method MUST be in the `authentication` relationship of the resolved DID document). With `cnf.kid`, the header `kid` MUST be present and MUST be the same string as `cnf.kid` ([VC-JOSE-COSE section 4.1.1, kid](https://www.w3.org/TR/vc-jose-cose/#kid)). With `cnf.jwk`, the verifier ignores any `kid` and uses `cnf.jwk`. If `holder` is present and the credential uses `cnf.kid`, `holder` (or `holder.id` when `holder` is an object) MUST be the DID of the `cnf.kid` DID URL.

The signature on the `vp+sd-jwt` is the proof of possession of the `cnf` key. A KB-JWT is not used for `vc+sd-jwt`. Holder binding uses only `cnf`, not `credentialSubject.id`.

When `require_cryptographic_holder_binding` is `false`, the presentation is the credential SD-JWT itself with the selected disclosures and no KB-JWT, following the rule in OpenID4VP Appendix B.1.

#### `mso_mdoc`

The credential is an ISO mdoc. The identifier, `meta` and claims rules are those of [OpenID4VP Appendix B.2, Mobile Documents or mdocs (ISO/IEC 18013 and ISO/IEC 23220 series)](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.2):

- `meta.doctype_value` is REQUIRED ([OpenID4VP Appendix B.2.3, Parameter in the meta parameter in Credential Query](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.2.3)).
- A claims path pointer is exactly two strings, the namespace and the data element identifier ([OpenID4VP section 7.2, Semantics for ISO mdoc-based credentials](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-7.2)). A Claims Query MAY have `intent_to_retain` ([OpenID4VP Appendix B.2.4, Parameter in the Claims Query](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.2.4)).
- The presentation is the base64url encoded (without padding) `DeviceResponse` CBOR structure ([OpenID4VP Appendix B.2.5, Presentation Response](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.2.5)). Each `DeviceResponse` MUST hold exactly one document, so that each presentation in the `vp_token` is one credential. Its `status` MUST be 0.
- The document has no device-signed data elements: `deviceSigned.nameSpaces` MUST encode an empty map.
- The document MUST use `deviceSignature` for mdoc authentication. `deviceMac` cannot be used because its key is derived from a reader key ([ISO/IEC 18013-5:2021](https://www.iso.org/standard/69084.html) section 9.1.3.5, mdoc MAC Authentication), and no reader key is exchanged in this protocol. `require_cryptographic_holder_binding` has no effect on `mso_mdoc`.
- The `deviceSignature` is computed over `DeviceAuthenticationBytes` (ISO/IEC 18013-5:2021 section 9.1.3.4, Mechanism, under 9.1.3 mdoc authentication), whose `SessionTranscript` is the one of section 9.1.5.1, Session transcript, with these values:
  - `DeviceEngagementBytes` is `null`
  - `EReaderKeyBytes` is `null`
  - `Handover` is the `DIDCommHandover` defined below

##### DIDComm Handover

The DIDComm Handover follows the shape of `OpenID4VPHandover` ([OpenID4VP Appendix B.2.6.1, Invocation via Redirects](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.2.6.1)):

```cddl
DIDCommHandover = [
  "DIDCommHandover",
  DIDCommHandoverInfoHash
]

DIDCommHandoverInfoHash = bstr

DIDCommHandoverInfoBytes = bstr .cbor DIDCommHandoverInfo

DIDCommHandoverInfo = [
  audience,
  nonce
]

audience = tstr

nonce = tstr
```

- The first element of `DIDCommHandover` MUST be the string `DIDCommHandover`. It tells this handover apart from every other handover type.
- `DIDCommHandoverInfoHash` MUST be the SHA-256 hash of `DIDCommHandoverInfoBytes`, the CBOR encoding of `DIDCommHandoverInfo`. The hash is over the plain CBOR array, without a tag 24 wrapper, the same as for `OpenID4VPHandoverInfo`.
- `audience` MUST be the `audience` of the request, including the `didcomm:` prefix.
- `nonce` MUST be the `nonce` of the request.
- `DIDCommHandoverInfo` and `DIDCommHandover` MUST be encoded with the core deterministic encoding requirements of [RFC 8949 section 4.2.1, Core Deterministic Encoding Requirements](https://www.rfc-editor.org/rfc/rfc8949.html#section-4.2.1), so that the prover and the verifier compute the same bytes.

There is no counterpart of the `jwkThumbprint` and `responseUri` elements of `OpenID4VPHandoverInfo`. DIDComm encrypts every message to keys in the verifier's DID document and delivers it to the verifier's DIDComm service, and the [Audience](#audience) check ties the `audience` DID to those keys. The DIDComm thread id is not included either, since the `nonce` is already unique per request.

The structure of `DIDCommHandoverInfo` MUST NOT change without a new first element string. The verifier always builds the `SessionTranscript` itself from its own request and never takes it from the presentation.

Non-normative test vector, computed with two independent CBOR encoders. Inputs:

- `audience`: `didcomm:did:example:verifier`
- `nonce`: `MTnN_oAE76NqlCOOIEfyBh3u2uZQFItlTPnNmXXXkpQ`

`DIDCommHandoverInfo`:

```text
Hex:

82781c646964636f6d6d3a6469643a6578616d706c653a7665726966696572782b4d54
6e4e5f6f414537364e716c434f4f494566794268337532755a514649746c54506e4e6d
5858586b7051

CBOR diagnostic:

["didcomm:did:example:verifier", "MTnN_oAE76NqlCOOIEfyBh3u2uZQFItlTPnNmXXXkpQ"]
```

`DIDCommHandoverInfoHash`:

```text
9781703613531ed5d4a91b1d7ec9aecd46e783f0364f6c3345f60644b1cf413b
```

`DIDCommHandover`:

```text
Hex:

826f444944436f6d6d48616e646f76657258209781703613531ed5d4a91b1d7ec9aecd
46e783f0364f6c3345f60644b1cf413b

CBOR diagnostic:

["DIDCommHandover", h'9781703613531ed5d4a91b1d7ec9aecd46e783f0364f6c3345f60644b1cf413b']
```

`SessionTranscript`:

```text
Hex:

83f6f6826f444944436f6d6d48616e646f76657258209781703613531ed5d4a91b1d7e
c9aecd46e783f0364f6c3345f60644b1cf413b

CBOR diagnostic:

[null, null, ["DIDCommHandover", h'9781703613531ed5d4a91b1d7ec9aecd46e783f0364f6c3345f60644b1cf413b']]
```

### Credential status

If a credential carries status information, the verifier MUST check it as its credential format defines: the `status` claim of a `dc+sd-jwt` ([SD-JWT VC draft -22 section 2.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.4), [Token Status List draft -21 section 8.3, Validation Rules](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21#section-8.3)), the `credentialStatus` of a `vc+sd-jwt` ([VC-JOSE-COSE section 5.4, Validation](https://www.w3.org/TR/vc-jose-cose/#validation-algorithm)) and the status in the MSO of an `mso_mdoc` when it has one. The verifier MUST discard a presentation whose credential is revoked (for a Token Status List, the status `0x01` INVALID). Other status values, such as `0x02` SUSPENDED, are left to verifier policy. If the verifier cannot determine the status, also when it does not support the status mechanism, it MUST discard the presentation. This is stricter than SD-JWT VC, which says status SHOULD be checked, and than VC-JOSE-COSE, which ignores a `credentialStatus` type the verifier does not understand.

The verifier fetches a status list only after it has decided to trust the issuer (see [Verifier processing rules](#verifier-processing-rules)), only from an `https` URI, and accepts it only when it is signed by a key the verifier trusts for that issuer. Token Status List leaves key resolution and trust to the verifier ([section 11.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21#section-11.3)).

### Prover processing rules

On a `request-presentation` with a `didcomm/dcql-request@v1.0` attachment, the prover:

1. Checks that the attachment has a `dcql_query`, a `nonce` and an `audience` in the forms defined above and no `transaction_data`, and that the DCQL query is valid ([OpenID4VP section 6](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6)) with the `meta` rules of [Credential formats](#credential-formats). The prover can check the characters of the `nonce` but not its entropy.
2. Checks the `audience` as described in [Audience](#audience).
3. Evaluates the query with the selection rules of [OpenID4VP section 6.4, Selecting Claims and Credentials](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.4). A `trusted_authorities` entry that the prover cannot evaluate for a credential, such as one with a type it does not know, does not match that credential. It MUST NOT disclose selectively disclosable claims that were not selected. If it cannot deliver every credential that is not optional, it MUST NOT return any credential.
4. Gets the consent of the user when there is one, showing which claims will be shared and who the verifier is ([OpenID4VP section 15.1, User Consent](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-15.1)). A prover that presents without a user, such as an organization or service agent, applies its policy for automated presentation instead. The `label` of an out-of-band invitation or of a connection ([RFC 0023](../0023-did-exchange/README.md)), the `goal` of an out-of-band invitation ([RFC 0434](../0434-outofband/README.md)) and the `comment` of the request are chosen by the sender. The prover SHOULD NOT show them as a verified identity of the verifier or use them to identify the verifier in its policy. Only what the prover has tied to the `audience` DID by means it trusts, such as the domain of a `did:web`, [DID Configuration](https://identity.foundation/.well-known/resources/did-configuration/) linked domains or a trust registry, can be shown or used as the verifier's verified identity. An invitation DID that the prover accepted as `audience` under the invitation DID rule in [Audience](#audience) counts as tied to the connection, so a wallet can show, for example, the domain of a `did:web` invitation DID.
5. Creates each presentation as defined for its format in [Credential formats](#credential-formats).
6. Sends a `presentation` in the same thread with exactly one `didcomm/dcql-presentation@v1.0` attachment. When the request came over a connection, the prover MUST authcrypt the presentation over that connection. Over DIDComm v2, see [DIDComm v2](#didcomm-v2).

If step 1, 2 or 3 fails, or the user or the prover's policy declines, the prover sends a `problem-report` and does not present. It MAY instead send a `propose-presentation` to negotiate. After a step 3 failure the prover SHOULD answer only after the user, when there is one, has seen the request, and a `problem-report` it sends then SHOULD be the same as the one it sends on a decline, with no hint of which check failed, so that the verifier cannot tell a failed match from a refusal ([OpenID4VP section 15.1, User Consent](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-15.1), [section 15.4.1, DCQL Value Matching](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-15.4.1)).

### Verifier processing rules

On a `presentation` with a `didcomm/dcql-presentation@v1.0` attachment, the verifier:

1. Finds the last `request-presentation` it sent in the thread and uses its `dcql_query`, `nonce` and `audience`. A presentation that does not belong to that thread is rejected. The verifier accepts at most one `presentation` per `nonce`.
2. Checks that the presentation was sent by the party the request was sent to:
   - With a connection: the presentation MUST be authcrypted and its sender key MUST be the prover's key for that connection. An anoncrypted message never belongs to a connection.
   - A connectionless request has no expected sender (see [Security considerations](#security-considerations)).
   - DIDComm v2 when the request had a `to`: see [DIDComm v2](#didcomm-v2).
3. Checks the `vp_token` shape against [OpenID4VP section 8.1](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-8.1). Every key is the `id` of a Credential Query in the request, and every array holds exactly one presentation unless that query has `multiple` set to `true`.
4. Selects the verification rules for each presentation from the `format` of the Credential Query whose `id` is its `vp_token` key, never from the presentation itself, and discards a presentation that is not of that format.
5. Decides with its own policy whether it trusts each issuer for the type of its credential. For that decision it MAY resolve the issuer DID and, for `dc+sd-jwt`, MAY retrieve the Type Metadata named by `vct` and the types it `extends` ([SD-JWT VC draft -22 section 5.3, Retrieving Type Metadata](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-5.3)). It SHOULD retrieve Type Metadata only after it has verified the Issuer-signed JWT, which it can do for this ahead of step 6, so that the `vct` it follows is the one the issuer signed ([section 2.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.4) of the same draft). These retrievals follow [section 3, Retrieving Documents and Resources via HTTP](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-3) and [section 7.1, Server-Side Request Forgery](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-7.1) of the same draft. The verifier retrieves nothing else a credential names, such as issuer metadata or a status list, before it trusts the issuer. A presentation from an issuer it does not trust is discarded. `trusted_authorities` in the query is only a hint to the prover ([OpenID4VP section 6.1](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.1)). The `trusted_authorities` types of OpenID4VP 1.0 do not identify DID issuers, so a verifier that wants credentials from given DID issuers can match the issuer DID in `iss`, `issuer` or `issuer.id` with `values` in a Claims Query instead ([OpenID4VP section 6.1.1, Trusted Authorities Query](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.1.1)).
6. Verifies each remaining presentation as described in [Verifying each format](#verifying-each-format). A presentation that fails a check is discarded ([OpenID4VP section 8.6, VP Token Validation](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-8.6)). If any presentation that this step verifies carries a wrong `nonce` or `audience`, the whole response is rejected. This is the `nonce` rule of [OpenID4VP section 14.1.2, Verifiable Presentations](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-14.1.2), limited to the presentations this step verifies and applied to the `audience` as well. A presentation that needs holder binding, or a `dc+sd-jwt` presentation that has a KB-JWT, carries a wrong value when its KB-JWT or `vp+sd-jwt` does not have a plain `nonce` and `aud` claim equal to the request. For `mso_mdoc` the values are only in the `SessionTranscript`, so a `deviceSignature` that does not verify counts as a wrong value.
7. Checks, with the verified data only, that the remaining presentations satisfy the query: the format and `meta` of each Credential Query, the requested claims and their `values`, `claim_sets` and `credential_sets` ([OpenID4VP section 6.4](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.4)). It MUST NOT rely on the prover having applied the query ([OpenID4VP section 14.9, Security Checks on the Returned Credentials and Presentations](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-14.9)).

If steps 1 to 3 pass, no presentation that step 6 verifies carries a wrong `nonce` or `audience`, and the presentations left after steps 4 to 6 satisfy the query in step 7, the verifier moves to `done` and sends an `ack` when `will_confirm` was `true`. Otherwise it sends a `problem-report` ([RFC 0035](../0035-report-problem/README.md)) and the protocol is `abandoned`, as in [RFC 0454](../0454-present-proof-v2/README.md#problem-report).

#### Verifying each format

`dc+sd-jwt`:

- Verify the SD-JWT as in [RFC 9901 section 7.1, Verification of the SD-JWT](https://www.rfc-editor.org/rfc/rfc9901.html#section-7.1) and [SD-JWT VC draft -22 section 2.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.4). Check that the header `typ` is `dc+sd-jwt`. The issuer key comes from a key discovery method the verifier's policy permits: one of the mechanisms in section 2.5 of the same draft, or the DID-based key discovery in [Credential formats](#credential-formats).
- Check that `vct`, or one of the `aka_vcts` values when the verifier applies the `aka_vcts` rule in [Credential formats](#credential-formats), is in `meta.vct_values`. A verifier that processes Type Metadata MAY also accept a `vct` that extends a listed type ([SD-JWT VC draft -22 section 5.4, Extending Type Metadata](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-5.4)). Whichever way the type matched (`vct`, `aka_vcts` or `extends`), the verifier's policy MUST trust the issuer for that type, not only the issuer in general ([section 7.7, Credential Type Extension and Issuer Authorization](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-7.7) of the same draft).
- Unless the query set `require_cryptographic_holder_binding` to `false`, require a KB-JWT and verify it as in [RFC 9901 section 7.3, Verification by the Verifier](https://www.rfc-editor.org/rfc/rfc9901.html#section-7.3) with the key from `cnf` (for `cnf.kid`, a verification method in the `authentication` relationship of the resolved DID document): `nonce` equals the request `nonce`, `aud` equals the request `audience`, `iat` is within the window the verifier accepts, and `sd_hash` matches. Whether binding is required comes from the request, never from whether a KB-JWT is present. When it is not required and the presentation has a KB-JWT, verify the KB-JWT the same way, as [OpenID4VP section 14.1.2](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-14.1.2) requires for every presentation with a holder binding proof.
- Check the status as in [Credential status](#credential-status).

`vc+sd-jwt`:

- Whether holder binding is required comes from `require_cryptographic_holder_binding` in the request, never from the `typ` of the presentation. Otherwise anyone who received a `vp+sd-jwt` could take the credential out of it and present it bare.
- When it is absent or `true`, the presentation MUST be a `vp+sd-jwt` and a bare credential SD-JWT MUST be rejected. Check that the header `typ` is `vp+sd-jwt`, take the one `EnvelopedVerifiableCredential` and read the credential SD-JWT from its `data:` URL. Verify that SD-JWT as in [VC-JOSE-COSE section 5.2, Verifying a Credential or Presentation Secured with SD-JWT](https://www.w3.org/TR/vc-jose-cose/#alg-sd-jwt), with the issuer key found as in [section 4, Key Discovery](https://www.w3.org/TR/vc-jose-cose/#key-discovery) of the same specification. When that key is a verification method of the `issuer` DID, it MUST be in the `assertionMethod` relationship of the resolved DID document. The credential SD-JWT's header `typ`, when present, MUST be `vc+sd-jwt`, and it MUST NOT have a KB-JWT. Then verify the `vp+sd-jwt` signature with the key from the credential's `cnf` (for `cnf.kid`, a verification method in the `authentication` relationship of the resolved DID document, named by a header `kid` that is the same string as `cnf.kid`), and check that `nonce` equals the request `nonce`, `aud` is a string equal to the request `audience` and `iat`, when present, is within the window the verifier accepts. If the payload has a `holder` and the credential uses `cnf.kid`, check that `holder` (or `holder.id`) is the DID of the `cnf.kid` DID URL.
- When it is `false`, the presentation is the credential SD-JWT. Verify it the same way. A `vp+sd-jwt` MUST be rejected ([OpenID4VP Appendix B.1](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#appendix-B.1)).
- Reject the presentation if it has a disclosure for a property that MUST NOT be selectively disclosable, or if such a property holds an `_sd` or `...` digest at any depth (see [Credential formats](#credential-formats)).
- Check the credential `type` against `meta.type_values` with the exact string matching in [Credential formats](#credential-formats).
- Reject the credential if the current time is before its `validFrom` or after its `validUntil` ([VCDM 2.0 section 4.9, Validity Period](https://www.w3.org/TR/vc-data-model-2.0/#validity-period)).
- Check `credentialStatus` as in [Credential status](#credential-status).

`mso_mdoc`:

- Decode the base64url `DeviceResponse`. Check that its `status` is 0 and that it holds exactly one document whose `docType` equals `meta.doctype_value`.
- Verify the issuer data as in [ISO/IEC 18013-5:2021](https://www.iso.org/standard/69084.html) section 9.3.1, Inspection procedure for issuer data authentication: the issuer certificate chain under the verifier's policy, the issuer signature over the MSO, the digests of the disclosed data elements, that the `docType` in the MSO equals the `docType` of the document, and the validity period.
- Build the `SessionTranscript` with the [DIDComm Handover](#didcomm-handover) from the request `audience` and `nonce`, and verify `deviceSignature` over `DeviceAuthenticationBytes` with the device key in the MSO. Reject `deviceMac`.
- Only issuer-signed data elements count when checking Claims Queries. Device-signed data elements MUST be ignored.

### DIDComm v2

Present-proof 2.0 messages with these formats can also be sent over [DIDComm v2.1](https://identity.foundation/didcomm-messaging/spec/v2.1/). The attachment content and the format identifiers do not change. These rules replace the DIDComm v1 parts (`recipientKeys`, `~service`, connections) of [Audience](#audience), [Prover processing rules](#prover-processing-rules) and [Verifier processing rules](#verifier-processing-rules):

- Audience: the prover sends the `presentation` to the DID of the `from` header (its value without any path or query) of the message that carried the request, or to the DID the verifier has since rotated to ([DID Rotation](https://identity.foundation/didcomm-messaging/spec/v2.1/#did-rotation)). That message is the `request-presentation` itself, or the out-of-band invitation for a connectionless request ([Message Headers](https://identity.foundation/didcomm-messaging/spec/v2.1/#message-headers), [Invitation](https://identity.foundation/didcomm-messaging/spec/v2.1/#invitation)). The DID in `audience` MUST be that `from` DID, another DID of the verifier that meets the recipient key rule in [Audience](#audience) for the `keyAgreement` keys of the DID the prover sends the `presentation` to, or another form of one of these defined by its DID method. Over a connection, the `from` DID of the verifier's out-of-band invitation also counts when the `from` of the request is that DID or a DID the verifier rotated to from it with `from_prior`. A verifier that used its public DID as the `from` of its invitation can therefore name that DID. The `audience` stays the value in the request.
- Sender: a `request-presentation` MUST have a `from` and a `to` with exactly one DID unless it is carried in an out-of-band invitation. The prover MUST reject a request that breaks this rule. When the request has a `to`, the prover MUST authcrypt the `presentation` from that DID, or from a DID it rotated to with `from_prior`. In that case the verifier MUST check that the `presentation` is authcrypted and that its `from` matches the sender key ([Message Layer Addressing Consistency](https://identity.foundation/didcomm-messaging/spec/v2.1/#message-layer-addressing-consistency)). It MUST also check that the DID of that `from`, without any path or query, is the DID in the `to` of the request, another form of that DID defined by its DID method (such as the short and long forms of `did:peer:4`), or a DID that DID rotated to with a valid `from_prior`.
- Layout: this layout is used until a specification defines present-proof 2.0 over DIDComm v2. Each attachment that would be in `proposals~attach`, `request_presentations~attach` or `presentations~attach` goes in the top-level `attachments` ([Attachments](https://identity.foundation/didcomm-messaging/spec/v2.1/#attachments)). It has an `id`, a `media_type` of `application/json`, the format identifier in `format`, and the same `data` object as in DIDComm v1, with the content in `data.json` or `data.base64`. `body` has no `formats` and none of these `~attach` arrays, except that a `propose-presentation` `formats` entry with no attachment stays in `body.formats`. Other fields such as `comment`, `goal_code` and `will_confirm` stay in `body`.

A `request-presentation` over DIDComm v2, as a plaintext message:

```json
{
  "id": "d13452a5-a831-4a53-be7e-e637e13edf9f",
  "type": "https://didcomm.org/present-proof/2.0/request-presentation",
  "from": "did:example:verifier",
  "to": ["did:example:prover"],
  "body": {
    "comment": "Please share your name",
    "will_confirm": true
  },
  "attachments": [
    {
      "id": "2407340e-0cd3-4e26-98b5-e12837077951",
      "media_type": "application/json",
      "format": "didcomm/dcql-request@v1.0",
      "data": {
        "json": {
          "dcql_query": {
            "credentials": [
              {
                "id": "pid",
                "format": "dc+sd-jwt",
                "meta": { "vct_values": ["https://credentials.example.com/identity_credential"] },
                "claims": [{ "path": ["given_name"] }, { "path": ["family_name"] }]
              }
            ]
          },
          "nonce": "Gn4uqRQaXKFeCOoWt1gSQ-pgzEEwfftyWDH1dCyg654",
          "audience": "didcomm:did:example:verifier"
        }
      }
    }
  ]
}
```

[Present Proof 3.0](https://github.com/decentralized-identity/waci-didcomm/blob/main/present_proof/present-proof-v3.md) is a separate protocol with the same attachment layout. This RFC registers its formats for RFC 0454. The formats do not depend on present-proof 2.0. A later present-proof version can register the same identifiers and only has to restate the protocol rules: one attachment per message, the propose semantics, the state transitions and the DIDComm v2 layout.

### Examples

The examples use DIDComm v1 messages, `did:example` DIDs and clipped tokens. Decoded contents are shown below each message.

#### `propose-presentation` example

A complete [`propose-presentation`](../0454-present-proof-v2/README.md#propose-presentation) message. It has no `nonce` and no `audience`:

```json
{
  "@id": "7eb2ac27-727f-4be9-9105-f849127bc6e1",
  "@type": "https://didcomm.org/present-proof/2.0/propose-presentation",
  "comment": "I can share my name",
  "formats": [
    {
      "attach_id": "7343e800-b245-485e-a93f-361148e9a633",
      "format": "didcomm/dcql-request@v1.0"
    }
  ],
  "proposals~attach": [
    {
      "@id": "7343e800-b245-485e-a93f-361148e9a633",
      "mime-type": "application/json",
      "data": {
        "json": {
          "dcql_query": {
            "credentials": [
              {
                "id": "pid",
                "format": "dc+sd-jwt",
                "meta": { "vct_values": ["https://credentials.example.com/identity_credential"] },
                "claims": [{ "path": ["given_name"] }, { "path": ["family_name"] }]
              }
            ]
          }
        }
      }
    }
  ]
}
```

#### `dc+sd-jwt` example

A complete [`request-presentation`](../0454-present-proof-v2/README.md#request-presentation) message:

```json
{
  "@id": "a3ab6216-ed5a-4f31-9958-7a90609b70be",
  "@type": "https://didcomm.org/present-proof/2.0/request-presentation",
  "comment": "Please share your name and city",
  "will_confirm": true,
  "formats": [
    {
      "attach_id": "5b7bff9e-73a7-454d-9dfa-13bcf46f8827",
      "format": "didcomm/dcql-request@v1.0"
    }
  ],
  "request_presentations~attach": [
    {
      "@id": "5b7bff9e-73a7-454d-9dfa-13bcf46f8827",
      "mime-type": "application/json",
      "data": {
        "json": {
          "dcql_query": {
            "credentials": [
              {
                "id": "pid",
                "format": "dc+sd-jwt",
                "meta": { "vct_values": ["https://credentials.example.com/identity_credential"] },
                "claims": [
                  { "path": ["given_name"] },
                  { "path": ["family_name"] },
                  { "path": ["address", "locality"] }
                ]
              }
            ]
          },
          "nonce": "rRgk_0pAsc6WId7Z-xqdq4GFBe6Mz8yWnilxR52-eAs",
          "audience": "didcomm:did:example:verifier"
        }
      }
    }
  ]
}
```

A complete [`presentation`](../0454-present-proof-v2/README.md#presentation) message:

```json
{
  "@id": "55eaf2b0-a5e3-4eea-b521-c3d5ec3c2610",
  "@type": "https://didcomm.org/present-proof/2.0/presentation",
  "~thread": { "thid": "a3ab6216-ed5a-4f31-9958-7a90609b70be" },
  "formats": [
    {
      "attach_id": "b29a8f59-3bb7-45b7-a985-973c10319aa7",
      "format": "didcomm/dcql-presentation@v1.0"
    }
  ],
  "presentations~attach": [
    {
      "@id": "b29a8f59-3bb7-45b7-a985-973c10319aa7",
      "mime-type": "application/json",
      "data": {
        "json": {
          "vp_token": {
            "pid": [
              "eyJhbGciOiJFUzI1NiIsInR5cCI6ImRjK3NkLWp3dCIsImtpZCI6ImRpZDpleGFtcGxlOmlzc3VlciNrZXktMSJ9...(clipped)~WyIyR0xDNDJzS1F2ZUNmR2ZyeU5STjl3IiwiZ2l2ZW5fbmFtZSIsIkVyaWthIl0~WyJlbHVWNU9nM2dTTklJOEVZbnN4QV9BIiwiZmFtaWx5X25hbWUiLCJNdXN0ZXJtYW5uIl0~WyI2SWo3dE0tYTVpVlBHYm9TNXRtdlZBIiwibG9jYWxpdHkiLCJCZXJsaW4iXQ~eyJhbGciOiJFUzI1NiIsInR5cCI6ImtiK2p3dCJ9...(clipped)"
            ]
          }
        }
      }
    }
  ]
}
```

The issuer-signed JWT payload:

```json
{
  "iss": "did:example:issuer",
  "vct": "https://credentials.example.com/identity_credential",
  "_sd": [
    "0md4eSFDWig7Gad1CZiRy1I4gcG4fKOmzGNEmOdABiI",
    "ZmgxyNRwqRHzYCllHNESq_NuKQJ7GvmDtXJA9OhKJbA"
  ],
  "address": {
    "_sd": ["YGz3c8c7IBtD8uofNuzW3FgUygjvTJEufJjsFhh-Q4U"]
  },
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "TCAER19Zvu3OHF4j4W4vfSVoHIP1ILilDls7vCeGemc",
      "y": "ZxjiWWbZMQGHVWKVQ4hbSIirsVfuecCE6t4jT9F2HZQ"
    }
  },
  "_sd_alg": "sha-256"
}
```

The disclosures in the presentation decode to:

```text
["2GLC42sKQveCfGfryNRN9w", "given_name", "Erika"]
["eluV5Og3gSNII8EYnsxA_A", "family_name", "Mustermann"]
["6Ij7tM-a5iVPGboS5tmvVA", "locality", "Berlin"]
```

The KB-JWT header and payload:

```json
{
  "alg": "ES256",
  "typ": "kb+jwt"
}
```

```json
{
  "iat": 1793179800,
  "aud": "didcomm:did:example:verifier",
  "nonce": "rRgk_0pAsc6WId7Z-xqdq4GFBe6Mz8yWnilxR52-eAs",
  "sd_hash": "x8Wh5moL37iiH7sjt6f3sOkGZU2MJDoBZmb3ZdXZwBo"
}
```

#### `vc+sd-jwt` example

The credential is a VCDM 2.0 credential issued with [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md). The query matches the `type` value `UniversityDegreeCredential` as it appears in the credential.

A complete `request-presentation` message:

```json
{
  "@id": "d34561e8-f3da-403e-8fef-8857009cca5c",
  "@type": "https://didcomm.org/present-proof/2.0/request-presentation",
  "comment": "Please share your degree",
  "formats": [
    {
      "attach_id": "a623bf54-6e28-4031-99cc-f13e0e1b0e7c",
      "format": "didcomm/dcql-request@v1.0"
    }
  ],
  "request_presentations~attach": [
    {
      "@id": "a623bf54-6e28-4031-99cc-f13e0e1b0e7c",
      "mime-type": "application/json",
      "data": {
        "json": {
          "dcql_query": {
            "credentials": [
              {
                "id": "degree",
                "format": "vc+sd-jwt",
                "meta": {
                  "type_values": [["UniversityDegreeCredential"]]
                },
                "claims": [{ "path": ["credentialSubject", "degree", "name"] }]
              }
            ]
          },
          "nonce": "GcdTZqx-xRhqZnTJ1pKjkL_n_aXc5SraXJOPZsaaft0",
          "audience": "didcomm:did:example:verifier"
        }
      }
    }
  ]
}
```

A complete `presentation` message:

```json
{
  "@id": "9dc55ab3-c971-4b50-97f0-c6220684a2b0",
  "@type": "https://didcomm.org/present-proof/2.0/presentation",
  "~thread": { "thid": "d34561e8-f3da-403e-8fef-8857009cca5c" },
  "formats": [
    {
      "attach_id": "201ac237-5619-4d48-9142-bf1f8ef47ad5",
      "format": "didcomm/dcql-presentation@v1.0"
    }
  ],
  "presentations~attach": [
    {
      "@id": "201ac237-5619-4d48-9142-bf1f8ef47ad5",
      "mime-type": "application/json",
      "data": {
        "json": {
          "vp_token": {
            "degree": [
              "eyJhbGciOiJFUzI1NiIsInR5cCI6InZwK3NkLWp3dCIsImN0eSI6InZwIiwia2lkIjoiZGlkOmV4YW1wbGU6aG9sZGVyI2tleS0xIn0...(clipped)~"
            ]
          }
        }
      }
    }
  ]
}
```

The `vp+sd-jwt` header:

```json
{
  "alg": "ES256",
  "typ": "vp+sd-jwt",
  "cty": "vp",
  "kid": "did:example:holder#key-1"
}
```

The `vp+sd-jwt` payload:

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2", "https://www.w3.org/ns/credentials/undefined-terms/v2"],
  "type": ["VerifiablePresentation"],
  "holder": "did:example:holder",
  "verifiableCredential": [
    {
      "@context": "https://www.w3.org/ns/credentials/v2",
      "type": "EnvelopedVerifiableCredential",
      "id": "data:application/vc+sd-jwt,eyJhbGciOiJFUzI1NiIsInR5cCI6InZjK3NkLWp3dCIsImN0eSI6InZjIiwia2lkIjoiZGlkOmV4YW1wbGU6dW5pdmVyc2l0eSNrZXktMSJ9...(clipped)~WyJRZzhJeFVWT2I3dFZaUEtJYTdCYUF3IiwibmFtZSIsIkJhY2hlbG9yIG9mIFNjaWVuY2UgYW5kIEFydHMiXQ~"
    }
  ],
  "nonce": "GcdTZqx-xRhqZnTJ1pKjkL_n_aXc5SraXJOPZsaaft0",
  "aud": "didcomm:did:example:verifier",
  "iat": 1793179800
}
```

The enveloped credential's issuer-signed payload (with its `_sd` digests clipped) and the one disclosure:

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://example.org/contexts/degree/v1"
  ],
  "type": ["VerifiableCredential", "UniversityDegreeCredential"],
  "issuer": "did:example:university",
  "validFrom": "2026-01-01T00:00:00Z",
  "credentialSubject": {
    "id": "did:example:holder",
    "degree": { "_sd": ["...(clipped)"] }
  },
  "cnf": { "kid": "did:example:holder#key-1" },
  "_sd_alg": "sha-256"
}
```

```json
["Qg8IxUVOb7tVZPKIa7BaAw", "name", "Bachelor of Science and Arts"]
```

#### `mso_mdoc` example

A complete `request-presentation` message:

```json
{
  "@id": "d2d986da-3fac-41e6-91cd-8d8e74b334d6",
  "@type": "https://didcomm.org/present-proof/2.0/request-presentation",
  "comment": "Please prove you are over 21",
  "formats": [
    {
      "attach_id": "14f3c8cb-d381-43a8-9b82-e3e1d481c7b8",
      "format": "didcomm/dcql-request@v1.0"
    }
  ],
  "request_presentations~attach": [
    {
      "@id": "14f3c8cb-d381-43a8-9b82-e3e1d481c7b8",
      "mime-type": "application/json",
      "data": {
        "json": {
          "dcql_query": {
            "credentials": [
              {
                "id": "mdl",
                "format": "mso_mdoc",
                "meta": { "doctype_value": "org.iso.18013.5.1.mDL" },
                "claims": [
                  { "path": ["org.iso.18013.5.1", "family_name"], "intent_to_retain": false },
                  { "path": ["org.iso.18013.5.1", "age_over_21"], "intent_to_retain": false }
                ]
              }
            ]
          },
          "nonce": "MTnN_oAE76NqlCOOIEfyBh3u2uZQFItlTPnNmXXXkpQ",
          "audience": "didcomm:did:example:verifier"
        }
      }
    }
  ]
}
```

A complete `presentation` message:

```json
{
  "@id": "b1d3a084-5465-4cdc-96cc-899db7ffccb9",
  "@type": "https://didcomm.org/present-proof/2.0/presentation",
  "~thread": { "thid": "d2d986da-3fac-41e6-91cd-8d8e74b334d6" },
  "formats": [
    {
      "attach_id": "66b611ea-a0f8-44f6-9da4-45f458c2a25f",
      "format": "didcomm/dcql-presentation@v1.0"
    }
  ],
  "presentations~attach": [
    {
      "@id": "66b611ea-a0f8-44f6-9da4-45f458c2a25f",
      "mime-type": "application/json",
      "data": {
        "json": {
          "vp_token": {
            "mdl": ["o2d2ZXJzaW9uYzEuMGlkb2N1bWVudHOBo2dkb2NUeXBl...(clipped)"]
          }
        }
      }
    }
  ]
}
```

The `DeviceResponse` in CBOR diagnostic notation (abbreviated):

```text
{
  "version": "1.0",
  "documents": [
    {
      "docType": "org.iso.18013.5.1.mDL",
      "issuerSigned": {
        "nameSpaces": {
          "org.iso.18013.5.1": [
            24(<< {"digestID": 3, "random": h'...', "elementIdentifier": "family_name", "elementValue": "Mustermann"} >>),
            24(<< {"digestID": 9, "random": h'...', "elementIdentifier": "age_over_21", "elementValue": true} >>)
          ]
        },
        "issuerAuth": [h'...', {...}, h'...', h'...']
      },
      "deviceSigned": {
        "nameSpaces": 24(<< {} >>),
        "deviceAuth": {
          "deviceSignature": [h'a10126', {}, null, h'...']
        }
      }
    }
  ],
  "status": 0
}
```

The `deviceSignature` is computed over `DeviceAuthenticationBytes` built with the `SessionTranscript` from the [DIDComm Handover](#didcomm-handover) test vector, since this example uses the same `audience` and `nonce`.

### Specifications

This RFC uses these versions:

- [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html), the latest errata version as [section 1.2, Errata revisions](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-1.2) asks other documents to reference. The as-approved text is at <https://openid.net/specs/openid-4-verifiable-presentations-1_0-final.html>.
- [RFC 9901: Selective Disclosure for JSON Web Tokens](https://www.rfc-editor.org/rfc/rfc9901.html)
- [SD-JWT-based Verifiable Digital Credentials (SD-JWT VC), draft -22](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22)
- [Token Status List, draft -21](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21)
- [Securing Verifiable Credentials using JOSE and COSE](https://www.w3.org/TR/vc-jose-cose/), W3C Recommendation ([13 October 2026](https://www.w3.org/TR/2026/REC-vc-jose-cose-20261013/))
- [Verifiable Credentials Data Model v2.0](https://www.w3.org/TR/vc-data-model-2.0/), W3C Recommendation
- [Decentralized Identifiers (DIDs) v1.0](https://www.w3.org/TR/did-1.0/), W3C Recommendation
- [DIDComm Messaging v2.1](https://identity.foundation/didcomm-messaging/spec/v2.1/)
- [ISO/IEC 18013-5:2021](https://www.iso.org/standard/69084.html)
- [RFC 8949: Concise Binary Object Representation (CBOR)](https://www.rfc-editor.org/rfc/rfc8949.html)

[OpenID4VP section 13.4, Pre-Final Specifications](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-13.4) lists SD-JWT draft -22 and SD-JWT VC draft -09. Its reference list in [section 16, Normative References](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-16) cites SD-JWT VC draft -10. This RFC references the published RFC 9901 and SD-JWT VC draft -22 instead. `aka_vcts` was added in SD-JWT VC draft -18 and is not in the draft OpenID4VP pins.

The media types `application/vc+sd-jwt` and `application/vp+sd-jwt` are registered by the W3C ([IANA Media Types](https://www.iana.org/assignments/media-types/media-types.xhtml)). They are the names to use for `vc+sd-jwt` credentials and their presentations.

### Security considerations

- The `nonce` and `audience` bind each presentation that has holder binding to one request and one verifier. They only work if the verifier creates a fresh `nonce` per request, checks both in every such presentation it verifies and accepts one `presentation` message per `nonce`, and the prover checks the `audience` against the DIDComm channel as described in [Audience](#audience). For a request sent over a connection, the prover's check stops a malicious verifier from forwarding another verifier's request and relaying the answer.
- An out-of-band invitation or a request with a `~service` decorator can be copied, also when the invitation only sets up the connection that the request is sent over later. Someone who starts a session at a verifier can put the QR code or link that verifier shows in front of another holder, who then presents to that verifier for the attacker's session. The `audience` check does not stop this, because the holder really is presenting to that verifier. The presentation does not prove that the presenter is the user of the session that showed the QR code or link ([OpenID4VP section 14.2, Session Fixation](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-14.2), [RFC 10027, Best Current Practice for Security of Cross-Device Flows](https://www.rfc-editor.org/rfc/rfc10027.html)).
- Presentations without holder binding can be replayed by anyone who has seen them ([OpenID4VP section 14.1.1](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-14.1.1)). Verifiers should only request them when that risk is acceptable. For a connectionless request the thread id is then the only link between such a presentation and the request, so the verifier MUST choose that thread id the same way as the `nonce`: from a cryptographically secure random source with at least 120 bits of entropy (a version 4 UUID made with such a source meets this). This is lower than the 128 bits [OpenID4VP section 5.3](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-5.3) requires for `state`, so that the usual DIDComm message ids qualify.
- DCQL constraints are applied by the prover, and `values` only as a best effort ([OpenID4VP section 6.4.1, Selecting Claims](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.4.1)). The verifier checks the verified data itself, as step 7 of [Verifier processing rules](#verifier-processing-rules) requires.
- The DIDComm Handover identifier keeps mdoc device signatures made for DIDComm from being accepted in OpenID4VP or proximity flows, and the other way around.

### Privacy considerations

- The prover discloses only the selectively disclosable claims the query selected ([OpenID4VP section 6.4](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.4)) and presents only with the user's consent or under the prover's policy.
- Fetching a status list tells the status list host that some verifier is checking a credential from that list. A verifier SHOULD NOT keep `uri` and `idx` to poll the status later without the holder knowing ([Token Status List draft -21 section 12.3, Observability of Relying Parties](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21#section-12.3)).

## Drawbacks

- Verifiers and provers need SD-JWT, SD-JWT VC, Token Status List and mdoc support. That is a large stack compared to the existing JSON-LD and AnonCreds formats.
- The DIDComm Handover is new. mdoc libraries often only decode the handover types they know. For example `@owf/mdoc` rejects `SessionTranscript` bytes with an unknown handover, both when signing and when verifying. Until libraries support it, provers and verifiers have to build the `SessionTranscript` structure themselves.
- `vc+sd-jwt` had a different meaning in OpenID4VP drafts up to -22, where it named IETF SD-JWT VC. Wallets and DCQL libraries that still support those drafts read it that way, and an agent that supports them next to this RFC sees both meanings. Implementations that still use the old meaning will not interoperate.
- For `vc+sd-jwt`, `type_values` holds the `type` values as they appear in the credential, not the expanded IRIs that OpenID4VP Appendix B.1.1 uses for its W3C formats. An agent that uses one DCQL engine for OpenID4VP and for this RFC passes the credential `type` values unchanged for `vc+sd-jwt` and expanded types for the OpenID4VP formats. A verifier that sends expanded IRIs for `vc+sd-jwt` gets no match.
- The prover's `audience` check needs data from the DIDComm layer (the sender DID, the keys it will encrypt to or the invitation DID of the connection) inside the attachment format handler. Deriving the `audience` instead of sending it would need the same data.
- The format depends on DCQL as defined in OpenID4VP 1.0. Later DCQL changes need a new version of these identifiers.
- There is no `transaction_data` support.
- DCQL picks the credential formats and can return several credentials in one `vp_token`. Both are protocol-level concerns in RFC 0454, and here they sit inside one attachment. The RFC 0454 `formats` array names only these identifiers, not the credential formats inside. Neither discover features RFC has a feature type for attachment formats, so a verifier only learns that a prover supports these formats from a `propose-presentation`.

## Rationale and alternatives

- [RFC 0510](../0510-dif-pres-exch-attach/README.md) (DIF Presentation Exchange) requires presentations in JSON-LD syntax and lists only `ldp_vc` and `ldp_vp`. It defines nothing for SD-JWT key binding or mdoc device authentication. OpenID4VP 1.0 requests presentations with DCQL only and does not use Presentation Exchange.
- One RFC per credential format was considered. DCQL already names the format per Credential Query, a query can mix formats, and RFC 0454 does not allow several attachments for different presentations. One pair of identifiers is simpler.
- Tunnelling OpenID4VP itself over DIDComm was considered. Like the Digital Credentials API binding in OpenID4VP Appendix A, it would need its own response mode and its own Client Identifier and audience rules, and the thread, `ack` and `problem-report` of present-proof would be lost.
- A new present-proof version built around DCQL was considered, with the query and `vp_token` in the message body, protocol-level nonce and audience, and discovery through its protocol identifier. Present-proof 2.0 is what agents run today, [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md) uses the same registry for issuance, and DCQL fits the RFC 0454 state machine without changes. A new version would also tie protocol versions to DCQL versions. The formats in this RFC do not depend on the protocol, so a later version can reuse them.
- The `audience` could be derived by both sides from the DIDComm layer instead of being sent. Sending it lets the verifier pick which of its DIDs the presentation is bound to. A public DID such as a `did:web` or `did:webvh` ties the presentation to the verifier's real identity. A derived value can only be a DID both sides already know from the DIDComm layer, often just a pairwise connection DID or key. Only the verifier knows which of its DIDs it wants named. Deriving would also need the same DIDComm layer data as checking. The prover's check works on keys, not on DID strings, so an explicit value is as safe as a derived one. Sending it also gives one exact string to compare, while a DID can have several equivalent forms (for example the long and short forms of `did:peer:4`) and implementations compare `aud` as exact strings. RFC 0510 also has the verifier send its `domain`.
- The prefix `didcomm:` follows `origin:` in OpenID4VP and `ia:` in the [OpenID4VCI 1.1 editor's draft](https://github.com/openid/OpenID4VCI/blob/bde23b72150967a701b66ec4caffcb4d2b8a8627/1.1/openid-4-verifiable-credential-issuance-1_1.md). Together with the rule in [Audience](#audience), it keeps a DIDComm audience distinct from any OpenID4VP Client Identifier for the same DID.
- Reusing `OpenID4VPHandover` with the `audience` as the client id was considered. DIDComm has no response URI, and a separate first element string means an mdoc device signature for one protocol can never be accepted by the other.
- For `vc+sd-jwt`, an SD-JWT+KB of the credential (as for `dc+sd-jwt`) was considered. VC-JOSE-COSE section 3.2.2 defines a `vp+sd-jwt` presentation with an `EnvelopedVerifiableCredential` and gives no rules for a KB-JWT on a credential. A `vp+jwt` ([VC-JOSE-COSE section 3.1.2, Securing JSON-LD Verifiable Presentations with JOSE](https://www.w3.org/TR/vc-jose-cose/#securing-vps-with-jose)) with the same `EnvelopedVerifiableCredential` was also considered. This RFC uses `vp+sd-jwt` because it is the presentation VC-JOSE-COSE defines for SD-JWT credentials.
- A new identifier instead of `vc+sd-jwt` was considered. `vc+sd-jwt` matches the VC-JOSE-COSE `typ` and media type, and the [`dcql` library](https://github.com/openwallet-foundation-labs/identity-common-ts/tree/main/packages/dcql) (formerly dcql-ts) already treats `vc+sd-jwt` with `meta.type_values` as a W3C credential. OpenID4VP 1.0 uses only `dc+sd-jwt` for IETF SD-JWT VC, so the old meaning of `vc+sd-jwt` is left only in implementations of earlier drafts.
- For `vc+sd-jwt`, matching `type_values` against fully expanded type IRIs, as OpenID4VP Appendix B.1.1 does, was considered. In VCDM 2.0 every `type` term expands to an absolute IRI, an undefined term through the `@vocab` of a context such as `https://www.w3.org/ns/credentials/undefined-terms/v2`. So the relative IRI shortcut in Appendix B.1.1 never applies, and both sides would need the context documents of each credential. VC-JOSE-COSE has no expansion step, and the `dcql` library compares the `type` strings it is given. Accepting both forms was also considered. It would not spare the prover from expansion, and a verifier could not tell which form a prover supports.

## Prior art

- [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) defines DCQL, the `vp_token`, the nonce and audience rules and `OpenID4VPHandover`. This RFC reuses them over DIDComm.
- [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md) issues `vc+sd-jwt` credentials over issue-credential v2. This RFC is the presentation side.
- [RFC 0510](../0510-dif-pres-exch-attach/README.md) and [RFC 0771](../0771-anoncreds-attachments/README.md) share one identifier for proposals and requests. RFC 0771 also forbids a `nonce` in proposals.
- [Present Proof 3.0](https://github.com/decentralized-identity/waci-didcomm/blob/main/present_proof/present-proof-v3.md) is the DIDComm v2 native present-proof protocol. This RFC uses its attachment layout for present-proof 2.0 over DIDComm v2. It does not register formats for Present Proof 3.0, but they could be used with it unchanged.
- The [OpenID4VCI 1.1 editor's draft](https://github.com/openid/OpenID4VCI/blob/bde23b72150967a701b66ec4caffcb4d2b8a8627/1.1/openid-4-verifiable-credential-issuance-1_1.md) defines `OpenID4VCIIAEHandover` outside OpenID4VP with the same pattern as the DIDComm Handover. [ISO/IEC TS 18013-7:2025](https://www.iso.org/standard/91154.html) defines further handovers for remote mdoc presentation.

## Unresolved questions

- OpenID4VP or OpenID4VCI may later define an identifier for VCDM 2.0 credentials ([OpenID4VCI issue 194](https://github.com/openid/OpenID4VCI/issues/194)) and guidance on identifier collisions ([OpenID4VP issue 777](https://github.com/openid/OpenID4VP/issues/777)). If they do, this RFC should follow.
- Which status mechanisms should a verifier support for `vc+sd-jwt`?
- RFC 0881 allows `cnf` as `jwk` or `kid`. This RFC supports both. Should a verifier be able to require one of them?
- Should `transaction_data` be supported in a later version?
- When SD-JWT VC and Token Status List are published as RFCs, this RFC should reference those.

## Implementations

The following lists the implementations (if any) of this RFC. Please do a pull request to add your implementation. If the implementation is open source, include a link to the repo or to the implementation within the repo. Please be consistent in the "Name" field so that a mechanical processing of the RFCs can generate a list of all RFCs supported by an Aries implementation.

[Credo](https://github.com/openwallet-foundation/credo-ts) has an implementation of `dc+sd-jwt` and `vc+sd-jwt` in progress. It will be listed here when this RFC moves to DEMONSTRATED.

Name / Link | Implementation Notes
--- | ---
 | 
