# Aries RFC 0884: IETF SD-JWT VC (`dc+sd-jwt`) Attachment format for requesting and issuing credentials

- Authors: [Tarun Vadde](mailto:vaddeofficial@gmail.com) (2060.io)
- Status: [PROPOSED](/README.md#proposed)
- Since: 2026-10-30
- Status Note: Initial draft.
- Supersedes:
- Start Date: 2026-10-30
- Tags: [feature](/tags.md#feature), [protocol](/tags.md#protocol), [credentials](/tags.md#credentials)

## Summary

This RFC registers an attachment format for use in the [issue-credential V2](../0453-issue-credential-v2/README.md) protocol based on IETF [SD-JWT-based Verifiable Digital Credentials](https://datatracker.ietf.org/doc/draft-ietf-oauth-sd-jwt-vc/) (SD-JWT VC), with the media type `application/dc+sd-jwt`. The formats work over DIDComm v1 and DIDComm v2.

## Motivation

The Issue Credential protocol has no attachment format for IETF SD-JWT VCs. [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md) issues W3C VC Data Model 2.0 credentials secured with SD-JWT (`vc+sd-jwt`), which have a different payload, different required claims and a different way to prove holder binding. SD-JWT VC is the format that [OpenID for Verifiable Credential Issuance](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) (OpenID4VCI) and [OpenID for Verifiable Presentations](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) (OpenID4VP) use as `dc+sd-jwt`, and it is commonly paired with Token Status List for revocation. With this format an issuer can issue the same credential over DIDComm that it issues over OpenID4VCI, and a holder can present it over DIDComm with [RFC 0882](https://github.com/decentralized-identity/aries-rfcs/pull/882) or over OpenID4VP.

## Tutorial

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY" and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

Complete examples of messages are provided in the [reference section](#reference).

## Reference

Credentials issued with this format are SD-JWT VCs, and [SD-JWT VC](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22) and [RFC 9901](https://www.rfc-editor.org/rfc/rfc9901.html) apply to them unless this RFC says otherwise.

### Credential Offer Attachment Format

Format identifier: `didcomm/dc-sd-jwt-offer@v1.0`

This format is used to offer a credential to a potential holder. The JSON structure might look like this:

```json
{
  "binding_required": true,
  "binding_method": {
    "didcomm_signed_attachment": {
      "algs_supported": ["ES256", "EdDSA"],
      "did_methods_supported": ["key", "jwk"],
      "nonce": "jypsHZPgS3pcEdDm-bKnxA"
    }
  },
  "selectively_disclosable_claims": [
    ["given_name"],
    ["family_name"],
    ["birthdate"],
    ["address", "locality"],
    ["nationalities", 0]
  ],
  "credential": {
    "iss": "did:example:issuer",
    "vct": "https://credentials.example.com/identity_credential",
    "given_name": "Erika",
    "family_name": "Mustermann",
    "birthdate": "1963-08-12",
    "address": {
      "locality": "Berlin",
      "country": "DE"
    },
    "nationalities": ["DE"],
    "status": {
      "status_list": {}
    }
  }
}
```

A complete [`offer-credential` message from the Issue Credential protocol 2.0](../0453-issue-credential-v2/README.md#offer-credential) might look like this:

```json
{
  "@id": "8b5c4e5a-1d0f-4a5e-9a43-2f0b8f6c1e27",
  "@type": "https://didcomm.org/issue-credential/2.0/offer-credential",
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "c1e5f7a2-3b4d-4e6f-8a9b-0c1d2e3f4a5b",
      "format": "didcomm/dc-sd-jwt-offer@v1.0"
    }
  ],
  "offers~attach": [
    {
      "@id": "c1e5f7a2-3b4d-4e6f-8a9b-0c1d2e3f4a5b",
      "mime-type": "application/json",
      "data": {
        "base64": "ewogICJiaW5kaW5nX3JlcXVpcmVkIjogdHJ1ZSwK...(clipped)...IH0KfQ=="
      }
    }
  ]
}
```

- `binding_required` - Optional. Boolean indicating whether the credential MUST be bound to the holder. If omitted, the credential is not required to be bound to the holder. If set to `true`, the credential MUST be bound to the holder using at least one of the binding methods defined in `binding_method`.
- `binding_method` - Required if `binding_required` is `true`. Object containing key-value pairs of binding methods supported by the issuer to bind the credential to a holder. When `binding_required` is `false` or omitted, the issuer MAY still include `binding_method`, and the holder MAY then choose to send a binding proof. See [Binding Methods](#binding-methods) for a registry of default binding methods supported as part of this RFC.
- `selectively_disclosable_claims` - Optional. Array of claims path pointers indicating which claims in the issued credential will be selectively disclosable. A claims path pointer is a non-empty array of strings and non-negative integers, as defined in [OpenID4VP section 7.1, Semantics for JSON-based credentials](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-7.1). A string selects an object property and an integer selects an array element. Pointers start at the root of `credential`, which is the root of the SD-JWT VC payload. Each pointer MUST select exactly one claim that exists in `credential`. The `null` component (all elements of an array) MUST NOT be used. If omitted, the issuer does not communicate which claims will be selectively disclosable ahead of issuance.
- `credential` - Required. The JWT Claims Set of the SD-JWT VC to be issued, with every claim in plain form. It MUST conform to [SD-JWT VC section 2.2.2, JWT Claims Set](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.2.2), with two exceptions: it MUST NOT contain `cnf`, and its `status` MAY name the status mechanisms that will be used with their members omitted. For example `{"status_list": {}}` says the credential will reference a Token Status List. It MUST contain `vct`. It MUST NOT contain `_sd`, `_sd_alg` or `...` entries.

A claims path pointer in `selectively_disclosable_claims` MUST NOT select a claim that [SD-JWT VC section 2.2.2.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.2.2.3) does not allow to be selectively disclosed, or anything inside such a claim.

The holder MUST reject an offer that does not meet the rules above by sending a `problem-report` with the code `issuance-abandoned`.

The issuer MUST NOT offer a pointer that selects a claim the Type Metadata of `vct` marks `sd` `never`. A claim marked `sd` `always` is selectively disclosable in the issued credential even if no pointer selects it. The holder does not need Type Metadata to check an offer.

#### Issuance-time Claims

The issued credential can differ from the offered `credential` only in these claims:

- `iss`, `sub` and `vct#integrity`. If the offer omits one, the issuer MAY add it.
- `iat`, `nbf` and `exp`. If the offer has one, the issued value can differ. If the offer omits one, the issuer MAY add it.
- `status`. If the offer omits it, the issuer MAY add it. If the offer names a mechanism with its members omitted, the issued `status` has the full value for that mechanism.
- `cnf`. The issuer sets it from the binding proof (see [Binding in Credential](#binding-in-credential)).

### Credential Request Attachment Format

Format identifier: `didcomm/dc-sd-jwt-request@v1.0`

This format is used to request issuance of a credential. A request with this format MUST be sent in response to an `offer-credential` message with a `didcomm/dc-sd-jwt-offer@v1.0` offer in the same thread, because the credential content and the binding nonce come from the offer. The issuer MUST reject a request that has no such offer by sending a `problem-report` with the code `issuance-abandoned`.

When the issuer sent the `offer-credential` over a connection, the holder MUST authcrypt the `request-credential` over that connection. The issuer MUST reject a `request-credential` in that thread unless it is authcrypted and its sender key is the holder's key for that connection.

The JSON structure might look like this:

```json
{
  "binding_proof": {
    "didcomm_signed_attachment": {
      "attachment_id": "attachment-0"
    }
  }
}
```

A complete [`request-credential` message from the Issue Credential protocol 2.0](../0453-issue-credential-v2/README.md#request-credential) might look like this:

```json
{
  "@id": "f3a1c9d2-6e7b-4c8d-9e0f-1a2b3c4d5e6f",
  "@type": "https://didcomm.org/issue-credential/2.0/request-credential",
  "~thread": { "thid": "8b5c4e5a-1d0f-4a5e-9a43-2f0b8f6c1e27" },
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "4d6e8f0a-2b3c-4d5e-8f7a-9b0c1d2e3f4a",
      "format": "didcomm/dc-sd-jwt-request@v1.0"
    }
  ],
  "requests~attach": [
    {
      "@id": "4d6e8f0a-2b3c-4d5e-8f7a-9b0c1d2e3f4a",
      "mime-type": "application/json",
      "data": {
        "base64": "ewogICJiaW5kaW5nX3Byb29mIjogewogICAgImRpZGNvbW1fc2lnbmVkX2F0dGFjaG1lbnQiOiB7CiAgICAgICJhdHRhY2htZW50X2lkIjogImF0dGFjaG1lbnQtMCIKICAgIH0KICB9Cn0="
      }
    }
  ],
  "~attach": [
    {
      "@id": "attachment-0",
      "mime-type": "application/json",
      "data": {
        "base64": "eyJub25jZSI6ICJqeXBzSFpQZ1MzcGNFZERtLWJLbnhBIn0",
        "jws": {
          "protected": "eyJhbGciOiJFUzI1NiIsImtpZCI6ImRpZDprZXk6ekRuYWVyRGFURjVCWEVhdkNyZlJaRWszMTZkcGJMc2ZQRFozV0o1aFJUUEZVMjE2OSN6RG5hZXJEYVRGNUJYRWF2Q3JmUlpFazMxNmRwYkxzZlBEWjNXSjVoUlRQRlUyMTY5In0",
          "signature": "...(clipped)"
        }
      }
    }
  ]
}
```

- `binding_proof` - Required if `binding_required` is `true` in the offer. Object containing key-value pairs of proofs for the binding to the holder. The keys MUST match keys of the `binding_method` object from the offer. See [Binding Methods](#binding-methods) for a registry of default binding methods supported as part of this RFC.

If the offer set `binding_required` to `true` and the request has no `binding_proof` for a method in `binding_method`, the issuer MUST NOT issue the credential and sends a `problem-report` with the code `issuance-abandoned`.

### Credential Attachment Format

Format identifier: `didcomm/dc-sd-jwt@v1.0`

This format is used to transmit an SD-JWT VC. The JSON structure might look like this:

```json
{
  "credential": "eyJhbGciOiJFUzI1NiIsInR5cCI6ImRjK3NkLWp3dCIsImtpZCI6IiNrZXktMSJ9.eyJpc3MiOiJkaWQ6ZXhhbXBs...(clipped)~WyIyR0xDNDJzS1F2ZUNmR2ZyeU5STjl3IiwgImdpdmVuX25hbWUiLCAiRXJpa2EiXQ~...(clipped)~"
}
```

A complete [`issue-credential` message from the Issue Credential protocol 2.0](../0453-issue-credential-v2/README.md#issue-credential) might look like this:

```json
{
  "@id": "2a7e9c1b-5d3f-4b6a-8c0e-7f1d2b3a4c5e",
  "@type": "https://didcomm.org/issue-credential/2.0/issue-credential",
  "~thread": { "thid": "8b5c4e5a-1d0f-4a5e-9a43-2f0b8f6c1e27" },
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "9e0f1a2b-3c4d-4e5f-a6b7-c8d9e0f1a2b3",
      "format": "didcomm/dc-sd-jwt@v1.0"
    }
  ],
  "credentials~attach": [
    {
      "@id": "9e0f1a2b-3c4d-4e5f-a6b7-c8d9e0f1a2b3",
      "mime-type": "application/json",
      "data": {
        "json": {
          "credential": "eyJhbGciOiJFUzI1NiIsInR5cCI6ImRjK3NkLWp3dCIsImtpZCI6IiNrZXktMSJ9.eyJpc3MiOiJkaWQ6ZXhhbXBs...(clipped)~WyIyR0xDNDJzS1F2ZUNmR2ZyeU5STjl3IiwgImdpdmVuX25hbWUiLCAiRXJpa2EiXQ~WyJlbHVWNU9nM2dTTklJOEVZbnN4QV9BIiwgImZhbWlseV9uYW1lIiwgIk11c3Rlcm1hbm4iXQ~WyI2SWo3dE0tYTVpVlBHYm9TNXRtdlZBIiwgImJpcnRoZGF0ZSIsICIxOTYzLTA4LTEyIl0~WyJlSThaV205UW5LUHBOUGVOZW5IZGhRIiwgImxvY2FsaXR5IiwgIkJlcmxpbiJd~WyJRZ19PNjR6cUF4ZTQxMmExMDhpcm9BIiwgIkRFIl0~"
        }
      }
    }
  ]
}
```

- `credential` - Required. The SD-JWT VC in the compact serialization of [RFC 9901 section 4, SD-JWT and SD-JWT+KB Data Formats](https://www.rfc-editor.org/rfc/rfc9901.html#section-4), with all of its Disclosures and without a Key Binding JWT. It uses the `application/dc+sd-jwt` media type as defined in [SD-JWT VC section 2.1, Media Type](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.1). The header `typ` MUST be `dc+sd-jwt`.

One attachment holds one credential.

The issued SD-JWT VC MUST meet these rules:

- `vct` MUST be the `vct` of the offer.
- The payload, after processing the Disclosures as in [RFC 9901 section 7.1, Verification of the SD-JWT](https://www.rfc-editor.org/rfc/rfc9901.html#section-7.1), MUST have the claims of the offered `credential` with the same values, except for the changes allowed in [Issuance-time Claims](#issuance-time-claims). No other claim can be added.
- The issuer decides which claims are selectively disclosable, within the rules of [Selective Disclosure](#selective-disclosure). If `selectively_disclosable_claims` was provided in the offer, the issued SD-JWT MUST make at least those claims selectively disclosable.
- `cnf` follows [Binding in Credential](#binding-in-credential).
- The issuer key can be found as described in [Issuer Key](#issuer-key).
- If `status` references a Token Status List, it follows [Status](#status).

#### Selective Disclosure

Which claims can be selectively disclosed follows [SD-JWT VC section 2.2.2.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.2.2.3) and, when the Type Metadata of `vct` has claim metadata, its `sd` values ([SD-JWT VC section 5.6.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-5.6.4)).

#### Issuer Key

The holder and later verifiers need to find the issuer key for the `iss` of the credential ([SD-JWT VC section 2.5, Issuer Verification Key Discovery and Validation](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.5)). The issuer MUST use one of these:

- One of the two mechanisms defined in SD-JWT VC section 2.5, JWT VC Issuer Metadata or Inline X.509 Certificates.
- DID-based key discovery. `iss` is a DID and the header `kid` MUST be a DID URL of that DID: either relative, such as `#key-1`, and resolved against `iss`, or absolute with the same DID as `iss`. A relative `kid` is RECOMMENDED. The key is the verification method that DID URL dereferences to, and it MUST be in the `assertionMethod` relationship of the resolved DID document.

When `iss` is a DID, the issuer MUST use DID-based key discovery and the credential MUST NOT have an `x5c` header. A holder that permits DID-based key discovery MUST use only that mechanism when `iss` is a DID, so that an attacker cannot change which mechanism is used, for example by adding an `x5c` header ([SD-JWT VC section 7.3, Ecosystem-specific Public Key Verification Methods](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-7.3)). [RFC 0882](https://github.com/decentralized-identity/aries-rfcs/pull/882) defines how its verifiers find the issuer key of a `dc+sd-jwt` presentation.

#### Status

When the credential references a Token Status List, `status` follows [Token Status List section 6.2, Referenced Token in JOSE](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21#section-6.2) and the `uri` of `status_list` MUST be an `https` URI. The Status List Token SHOULD be signed by the issuer of the credential, with a key that can be found the same way as the credential's issuer key.

[RFC 0882](https://github.com/decentralized-identity/aries-rfcs/pull/882) defines how its verifiers check the `status` claim. An issuer that uses a status mechanism other than Token Status List SHOULD do so only when the verifiers of the credential support it.

#### Holder Verification

When the holder receives an `issue-credential` message with this format, it MUST check that:

1. `credential` is an SD-JWT in compact serialization without a Key Binding JWT ([RFC 9901 section 7.2, Processing by the Holder](https://www.rfc-editor.org/rfc/rfc9901.html#section-7.2)).
2. The SD-JWT VC verifies as in [SD-JWT VC section 2.4, Verification and Processing](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.4), the header `typ` is `dc+sd-jwt`, and the issuer key was found with a mechanism of [Issuer Key](#issuer-key) that the holder's policy permits. A holder that permits DID-based key discovery MUST use only that mechanism when `iss` is a DID, and MUST reject the credential if it has an `x5c` header.
3. No claim that [SD-JWT VC section 2.2.2.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.2.2.3) does not allow to be selectively disclosed, and nothing inside one, is in a Disclosure or holds an `_sd` or `...` digest.
4. `vct` and the other claims match the offer as required in [Credential Attachment Format](#credential-attachment-format).
5. Every claims path pointer in the offer's `selectively_disclosable_claims` selects a claim that is selectively disclosable in the issued credential.
6. The binding matches the request. If the request carried a binding proof, `cnf` MUST be present and follow [Binding in Credential](#binding-in-credential): `cnf.kid` is the same string as the `kid` of the signed attachment, or `cnf.jwk` is the same public key as its `jwk`, for example compared by their [RFC 7638](https://www.rfc-editor.org/rfc/rfc7638) thumbprints. If the request carried no binding proof, `cnf` MUST NOT be present.
7. If `status` references a Token Status List, it has the form required in [Status](#status).

If a check fails, the holder MUST NOT store the credential and sends a `problem-report` with the code `issuance-abandoned`, as described in [RFC 0453](../0453-issue-credential-v2/README.md).

#### Example Credential

The issued SD-JWT VC from the example `issue-credential` message above.

Issuer-signed JWT header:

```json
{
  "alg": "ES256",
  "typ": "dc+sd-jwt",
  "kid": "#key-1"
}
```

Issuer-signed JWT payload:

```json
{
  "iss": "did:example:issuer",
  "iat": 1793350800,
  "exp": 1951117200,
  "vct": "https://credentials.example.com/identity_credential",
  "_sd": [
    "0HZmnSIPz337kSWe7C34l--88gzJi-eBJ2Vz_HJwATg",
    "I00fcFUoDXCucp5yy2ujqPssDVGaWNiUliNz_awD0gc",
    "Lai6IU6d7GQagXR7AvGTrnXgSld3z8EIg_fv3fOZ1Wg"
  ],
  "address": {
    "_sd": ["aWdEwsZvJfFQeBPWRsGf9W_6dnY-O0TUYvwQyYJqfcE"],
    "country": "DE"
  },
  "nationalities": [{ "...": "3j6R6Qrii_oY-llhg021nA2silD3yG4tr3P1z9FWyjE" }],
  "status": {
    "status_list": {
      "idx": 412,
      "uri": "https://issuer.example.com/statuslists/1"
    }
  },
  "cnf": {
    "kid": "did:key:zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169#zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169"
  },
  "_sd_alg": "sha-256"
}
```

The Disclosures decode to:

```text
["2GLC42sKQveCfGfryNRN9w", "given_name", "Erika"]
["eluV5Og3gSNII8EYnsxA_A", "family_name", "Mustermann"]
["6Ij7tM-a5iVPGboS5tmvVA", "birthdate", "1963-08-12"]
["eI8ZWm9QnKPpNPeNenHdhQ", "locality", "Berlin"]
["Qg_O64zqAxe412a108iroA", "DE"]
```

`cnf.kid` is the `kid` of the request's signed attachment. A signed attachment with a `jwk` header gives `cnf.jwk` with that key instead.

### Binding Methods

The attachment format supports different methods to bind the credential to the receiver of the credential. In the offer message the issuer can indicate which binding methods are supported in the `binding_method` object. Each key represents the id of the supported binding method.

This section defines a set of binding methods supported by this attachment format, but other binding methods may be used. Based on the binding method, the request needs to include a `binding_proof` object where the key matches the key of the binding method from the offer.

#### DIDComm Signed Attachment

Identifier: `didcomm_signed_attachment`

This is the binding method of the same name defined in [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md#didcomm-signed-attachment), including its `jwk_supported` option. This RFC adds these rules:

- The issuer MUST create a new `nonce` for every offer from a cryptographically secure random source with at least 120 bits of entropy. A version 4 UUID made with such a source ([RFC 9562 section 5.4, UUID Version 4](https://www.rfc-editor.org/rfc/rfc9562.html#section-5.4), 122 random bits) meets this, and so does the base64url encoding of 16 or more random bytes. The issuer stores it with the protocol thread. When the issuer sends more than one offer in a thread, only the nonce of the last offer is valid.
- The JWS payload is the base64url encoding, without padding, of the bytes decoded from `data.base64`. Writers SHOULD put that unpadded base64url string in `data.base64`, as [RFC 0017](../../concepts/0017-attachments/README.md#base64url) recommends, and the issuer verifies the signature over the re-encoded payload.
- If a check of the signed attachment fails, the issuer sends a `problem-report` with the code `issuance-abandoned`.

An implementation that shares code with RFC 0881 can apply these rules to both formats.

##### Binding in Credential

If the request carried a `didcomm_signed_attachment` binding proof, the issued SD-JWT VC MUST include a `cnf` claim ([SD-JWT VC section 2.2.2.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-2.2.2.3)) taken from the signed attachment as in [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md#binding-in-credential): `cnf.kid` for a `kid` header and `cnf.jwk` for a `jwk` header.

If the request carried no binding proof, the issued credential MUST NOT have a `cnf` claim.

### DIDComm v2

Issue-credential 2.0 messages with these formats can also be sent over [DIDComm v2.1](https://identity.foundation/didcomm-messaging/spec/v2.1/). The attachment content and the format identifiers do not change. These rules replace the DIDComm v1 message layout and the connection rule in [Credential Request Attachment Format](#credential-request-attachment-format):

- Layout: this layout is used until a specification defines issue-credential 2.0 over DIDComm v2. Each attachment that would be in `offers~attach`, `requests~attach` or `credentials~attach` goes in the top-level `attachments` ([Attachments](https://identity.foundation/didcomm-messaging/spec/v2.1/#attachments)). It has an `id`, a `media_type` of `application/json`, the format identifier in `format`, and the same `data` object as in DIDComm v1, with the content in `data.json` or `data.base64`. `body` has no `formats` and none of these `~attach` arrays. Other fields such as `comment`, `goal_code`, `replacement_id` and `credential_preview` stay in `body`.
- Binding proof: the signed attachment of a [`didcomm_signed_attachment`](#didcomm-signed-attachment) binding proof goes in the top-level `attachments` instead of `~attach`, with a `media_type` of `application/json` and no `format`. Its `id` is the `attachment_id` of the binding proof. Its `data` object, including `data.jws`, stays the same as in DIDComm v1.
- Sender: when the `offer-credential` has a `to` with one DID, the holder MUST authcrypt the `request-credential` from that DID, or from a DID it rotated to with `from_prior`. The issuer MUST reject a `request-credential` in that thread unless it is authcrypted, its `from` matches the sender key ([Message Layer Addressing Consistency](https://identity.foundation/didcomm-messaging/spec/v2.1/#message-layer-addressing-consistency)) and the DID of that `from`, without any path or query, is the DID in the `to` of the offer, another form of that DID defined by its DID method, or a DID that DID rotated to with a valid `from_prior`.

### Presenting the Credential

A credential issued with this format is an IETF SD-JWT VC and can be presented under the `dc+sd-jwt` Credential Format of [RFC 0882](https://github.com/decentralized-identity/aries-rfcs/pull/882), or with OpenID4VP. A credential without `cnf` can only answer a DCQL query that sets `require_cryptographic_holder_binding` to `false`. Holder binding uses only `cnf`, not the keys of the DIDComm connection the credential was issued over.

### Specifications

This RFC uses these versions:

- [RFC 9901: Selective Disclosure for JSON Web Tokens](https://www.rfc-editor.org/rfc/rfc9901.html)
- [SD-JWT-based Verifiable Digital Credentials (SD-JWT VC), draft -22](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22)
- [Token Status List, draft -21](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21)
- [OpenID for Verifiable Presentations 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html), for the claims path pointer syntax
- [RFC 7800: Proof-of-Possession Key Semantics for JSON Web Tokens](https://www.rfc-editor.org/rfc/rfc7800.html)
- [Decentralized Identifiers (DIDs) v1.0](https://www.w3.org/TR/did-1.0/), W3C Recommendation
- [DIDComm Messaging v2.1](https://identity.foundation/didcomm-messaging/spec/v2.1/)

### Security considerations

The signed attachment payload has only the `nonce`, so a binding proof is not tied to one issuer. A dishonest issuer can copy another issuer's nonce into its own offer, pass the holder's proof on to that issuer and get back a credential bound to the holder's key, which the holder then receives from an issuer it never talked to.

### Privacy considerations

The privacy considerations of [RFC 9901 section 10](https://www.rfc-editor.org/rfc/rfc9901.html#section-10), [SD-JWT VC section 8](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-8) and [Token Status List section 12](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-status-list-21#section-12) apply.

## Drawbacks

- There is currently no attachment format defined for a credential proposal, and a request can only answer an offer. This makes it impossible for a holder to initiate the issuance of a credential using this attachment format.
- There is no way for the holder to say which signature algorithms it supports for the issuer signature. The issuer chooses.
- Each issuance gives one credential. Batch issuance, as in [OpenID4VCI section 8.3, Credential Response](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-8.3), is not covered.
- SD-JWT VC and Token Status List are not yet published as RFCs.

## Rationale and alternatives

- [RFC 0881](../0881-w3c-vc-sd-jwt-credential-attachment/README.md) issues W3C VCDM 2.0 credentials secured with SD-JWT. An SD-JWT VC payload is a flat JWT Claims Set with `vct` instead of a VCDM credential, has its own list of claims that are never selectively disclosable, and proves holder binding with a Key Binding JWT. Earlier SD-JWT VC drafts used the `typ` `vc+sd-jwt`, which now names VCDM 2.0 secured with SD-JWT. A separate format with `typ` `dc+sd-jwt` keeps the two credential types apart and keeps both RFCs simple.
- The structure and the `didcomm_signed_attachment` binding method are the same as in RFC 0881, so an implementation can share most code between the two formats.
- `selectively_disclosable_claims` uses claims path pointers instead of the JSONPath subset of RFC 0881. The same syntax is used by OpenID4VCI claims metadata ([Appendix B.2](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#appendix-B.2), [Appendix C](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#appendix-C)), by SD-JWT VC Type Metadata ([section 5.6.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-sd-jwt-vc-22#section-5.6.1)) and by DCQL in OpenID4VP and RFC 0882. Claim names that contain `.` or `[` need no escaping. An issuer can take the pointers from its Type Metadata or OpenID4VCI configuration.
- The holder picks the binding by how it signs the signed attachment: a DID URL in `kid` gives `cnf.kid` and a bare key in `jwk` gives `cnf.jwk`, the way the OpenID4VCI `jwt` proof binds the credential to the DID or to the key ([OpenID4VCI Appendix F.1](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#appendix-F.1)). The issuer never turns one into the other. A holder that wants `cnf.jwk`, which SD-JWT VC recommends, signs with `jwk`.
- RFC 0881 requires a binding key named by `kid` to be in the `authentication` relationship. RFC 0882 verifiers require this for a `cnf.kid` key, and a key the holder's DID only lists for other purposes should not become a credential key.
- [OpenID4VCI Appendix A.3, IETF SD-JWT VC](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#appendix-A.3) issues the same credential but is not a DIDComm-based protocol. Its parameters map onto this format: `vct` and the claims stay the same, `cryptographic_binding_methods_supported` values `did:<method>` map to `did_methods_supported` and the value `jwk` to `jwk_supported`, and `proof_signing_alg_values_supported` maps to `algs_supported`. The OpenID4VCI `jwt` proof carries `aud` and `iat` as well as the nonce. The signed attachment here keeps only the nonce, as in RFC 0881, so that its payload stays the same across RFC 0809, RFC 0881 and this RFC. See [Security considerations](#security-considerations) for what this leaves out.

## Prior art

This attachment format is structurally based on [RFC 0881: W3C SD-JWT Verifiable Credential Attachment](../0881-w3c-vc-sd-jwt-credential-attachment/README.md) and [RFC 0809: W3C Data Integrity Credential Attachment](../0809-w3c-data-integrity-credential-attachment/README.md), adapted for IETF SD-JWT VC as issued by [OpenID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html).

## Unresolved questions

- Should `selectively_disclosable_claims` allow the `null` component to make every element of an array selectively disclosable?
- Should the signed attachment payload add `aud`, `iat` or a `typ` header, as in the OpenID4VCI `jwt` proof? Adding these would change the binding method shared with RFC 0809 and RFC 0881. See [Security considerations](#security-considerations).
- When SD-JWT VC and Token Status List are published as RFCs, this RFC should reference those.

## Implementations

The following lists the implementations (if any) of this RFC. Please do a pull request to add your implementation. If the implementation is open source, include a link to the repo or to the implementation within the repo. Please be consistent in the "Name" field so that a mechanical processing of the RFCs can generate a list of all RFCs supported by an Aries implementation.

| Name / Link | Implementation Notes |
| ----------- | -------------------- |
|             |
