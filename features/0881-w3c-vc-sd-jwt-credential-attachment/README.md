# Aries RFC 0881: W3C SD-JWT Verifiable Credential (`vc+sd-jwt`) Attachment format for requesting and issuing credentials

- Authors: George Mulhearn (Anonyome Labs)
- Status: [PROPOSED](/README.md#proposed)
- Since: 2025-12-10
- Supersedes:
- Start Date: 2025-12-10
- Tags: [feature](/tags.md#feature), [protocol](/tags.md#protocol), [credentials](/tags.md#credentials)

## Summary

This RFC registers an attachment format for use in the [issue-credential V2](../0453-issue-credential-v2/README.md) protocol based on W3C Verifiable Credentials with [SD-JWT](https://www.w3.org/TR/vc-jose-cose/#with-sd-jwt) securing mechanism from the [VC Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/).

## Motivation

The Issue Credential protocol needs an attachment format to be able to exchange W3C credentials secured with SD-JWT. SD-JWT enables selective disclosure of claims, allowing holders to present only the claims they choose to share with a verifier. The `vc+sd-jwt` media type is defined in the [W3C VC-JOSE-COSE](https://www.w3.org/TR/vc-jose-cose/) specification targeting VC Data Model 2.0.

## Tutorial

Complete examples of messages are provided in the [reference section](#reference).

## Reference

### Credential Offer Attachment Format

Format identifier: `didcomm/w3c-vc-sd-jwt-offer@v1.0`

This format is used to offer a credential to a potential holder. The JSON structure might look like this:

```json
{
  "binding_required": true,
  "binding_method": {
    "didcomm_signed_attachment": {
      "algs_supported": ["ES256", "EdDSA"],
      "did_methods_supported": ["key", "jwk"],
      "nonce": "b19439b0-4dc9-4c28-b796-99d17034fb5c"
    }
  },
  "selectively_disclosable_claims": [
    "$.credentialSubject.degree.name",
    "$.credentialSubject.degree.type"
  ],
  "credential": {
    "@context": [
      "https://www.w3.org/ns/credentials/v2",
      "https://www.w3.org/ns/credentials/examples/v2"
    ],
    "type": ["VerifiableCredential", "UniversityDegreeCredential"],
    "issuer": "did:key:z6MkodKV3mnjQQMB9jhMZtKD9Sm75ajiYq51JDLuRSPZTXrr",
    "validFrom": "2020-01-01T19:23:24Z",
    "validUntil": "2021-01-01T19:23:24Z",
    "credentialSubject": {
      "degree": {
        "type": "BachelorDegree",
        "name": "Bachelor of Science and Arts"
      }
    }
  }
}
```

A complete [`offer-credential` message from the Issue Credential protocol 2.0](../0453-issue-credential-v2/README.md#offer-credential) might look like this:

```json
{
  "@id": "284d3996-ba85-45d9-964b-9fd5805517b6",
  "@type": "https://didcomm.org/issue-credential/2.0/offer-credential",
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "format": "didcomm/w3c-vc-sd-jwt-offer@v1.0"
    }
  ],
  "offers~attach": [
    {
      "@id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "mime-type": "application/json",
      "data": {
        "base64": "ewogICJiaW5kaW5nX3JlcXVpcmVkIjogdHJ1ZSwK...(clipped)...fQp9"
      }
    }
  ]
}
```

- `binding_required` - Optional. Boolean indicating whether the credential MUST be bound to the holder. If omitted, the credential is not required to be bound to the holder. If set to `true`, the credential MUST be bound to the holder using at least one of the binding methods defined in `binding_method`.
- `binding_method` - Required if `binding_required` is `true`. Object containing key-value pairs of binding methods supported by the issuer to bind the credential to a holder. See [Binding Methods](#binding-methods) for a registry of default binding methods supported as part of this RFC.
- `selectively_disclosable_claims` - Optional. Array of strings indicating which claims in the issued credential will be selectively disclosable. Each string is a [JSONPath](https://www.rfc-editor.org/rfc/rfc9535) expression that points to a single, unambiguous location within the `credential` object (e.g. `$.credentialSubject.degree.name`). Only dot-notation member accessors and array indices (e.g. `$.credentialSubject.addresses[0].street`) are permitted — wildcards, filters, array slices, and recursive descent MUST NOT be used. If omitted, the issuer does not communicate which claims will be selectively disclosable ahead of issuance.
- `credential` - Required. The credential to be issued. The credential MUST conform to VC Data Model 2.0. The credential MUST NOT contain any proofs. Some properties MAY be omitted if they will only be available at time of issuance, such as `validFrom`, `issuer`, `credentialSubject.id`, `credentialStatus`.

#### Credential Offer Exceptions

To allow for validation of the `credential` in the offer, the `credential` MUST be conformant to the VC Data Model 2.0, except for the following exceptions:

- `validFrom` can be omitted, or set to a placeholder value.
- `issuer` (or `issuer.id` if issuer is an object) can be omitted.
- `credentialSubject.id` can be omitted.
- `credentialStatus` can be omitted entirely, or `credentialStatus.type` can be present with other dynamic fields omitted.

### Credential Request Attachment Format

Format identifier: `didcomm/w3c-vc-sd-jwt-request@v1.0`

This format is used to request issuance of a credential. The JSON structure might look like this:

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
  "@id": "7293daf0-ed47-4295-8cc4-5beb513e500f",
  "@type": "https://didcomm.org/issue-credential/2.0/request-credential",
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "13a3f100-38ce-4e96-96b4-ea8f30250df9",
      "format": "didcomm/w3c-vc-sd-jwt-request@v1.0"
    }
  ],
  "requests~attach": [
    {
      "@id": "13a3f100-38ce-4e96-96b4-ea8f30250df9",
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
        "base64": "<base64-encoded-json-attachment-content>",
        "jws": {
          "protected": "eyJhbGciOiJFZERTQSIsImtpZCI6ImRpZDprZXk6... (bytes omitted)",
          "signature": "3dZWsuru7QAVFUCtTd0s7uc1peYEijx4eyt5... (bytes omitted)"
        }
      }
    }
  ]
}
```

- `binding_proof` - Required if `binding_required` is `true` in the offer. Object containing key-value pairs of proofs for the binding to the holder. The keys MUST match keys of the `binding_method` object from the offer. See [Binding Methods](#binding-methods) for a registry of default binding methods supported as part of this RFC.

### Credential Attachment Format

Format identifier: `didcomm/w3c-vc-sd-jwt@v1.0`

This format is used to transmit a verifiable credential with SD-JWT securing mechanism. The JSON structure might look like this:

```json
{
  "credential": "eyJhbGciOiJFUzI1NiJ9.eyJfc2QiOi...(clipped)...~"
}
```

A complete [`issue-credential` message from the Issue Credential protocol 2.0](../0453-issue-credential-v2/README.md#issue-credential) might look like this:

```json
{
  "@id": "284d3996-ba85-45d9-964b-9fd5805517b6",
  "@type": "https://didcomm.org/issue-credential/2.0/issue-credential",
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "format": "didcomm/w3c-vc-sd-jwt@v1.0"
    }
  ],
  "credentials~attach": [
    {
      "@id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "mime-type": "application/json",
      "data": {
        "json": {
          "credential": "eyJhbGciOiJFUzI1NiJ9.eyJfc2QiOi...(clipped)...~"
        }
      }
    }
  ]
}
```

- `credential` - Required. The SD-JWT in compact serialization format. The credential MUST conform to the VC Data Model 2.0 and use the `application/vc+sd-jwt` media type as defined in [VC-JOSE-COSE section 6.1.3, application/vc+sd-jwt](https://www.w3.org/TR/vc-jose-cose/#vc-json-sd-jwt).

It is up to the issuer to decide which claims are selectively disclosable, except that the top-level `@context`, `type`, `issuer`, `validFrom`, `validUntil`, `credentialStatus`, `credentialSchema`, `relatedResource` and `cnf` properties, and any property inside them, MUST NOT be selectively disclosable. A verifier needs them to check that the credential is authentic and still valid ([RFC 9901 section 9.7, Selectively Disclosable Validity Claims](https://www.rfc-editor.org/rfc/rfc9901.html#section-9.7), [VC-JOSE-COSE section 3.2.1](https://www.w3.org/TR/vc-jose-cose/#securing-with-sd-jwt)). `selectively_disclosable_claims` in the offer MUST NOT point to any of them. If `selectively_disclosable_claims` was provided in the offer, the issued SD-JWT MUST make at least those claims selectively disclosable. If `binding_required` was `true` in the offer, the issued SD-JWT MUST include a `cnf` (confirmation) claim bound to the holder's key as provided through the binding proof. When presenting the credential, the holder proves possession of the `cnf` key as the presentation format defines, for example by signing a `vp+sd-jwt` ([VC-JOSE-COSE section 3.2.2](https://www.w3.org/TR/vc-jose-cose/#securing-vps-sd-jwt)) with it. A Key Binding JWT ([RFC 9901 section 4.3](https://www.rfc-editor.org/rfc/rfc9901.html#section-4.3)) is not used for this.

### Binding Methods

The attachment format supports different methods to bind the credential to the receiver of the credential. In the offer message the issuer can indicate which binding methods are supported in the `binding_method` object. Each key represents the id of the supported binding method.

This section defines a set of binding methods supported by this attachment format, but other binding methods may be used. Based on the binding method, the request needs to include a `binding_proof` object where the key matches the key of the binding method from the offer.

#### DIDComm Signed Attachment

Identifier: `didcomm_signed_attachment`

This binding method leverages [DIDComm signed attachments](../../concepts/0017-attachments/README.md#signing-attachments) to bind a credential to a specific key and/or identifier.

##### Binding Method in Offer

```json
{
  "didcomm_signed_attachment": {
    "algs_supported": ["ES256", "EdDSA"],
    "did_methods_supported": ["key", "jwk"],
    "jwk_supported": true,
    "nonce": "b19439b0-4dc9-4c28-b796-99d17034fb5c"
  }
}
```

- `algs_supported` - Required. List of strings indicating the JSON Web Algorithms supported by the issuer for verifying the signed attachment. The list MUST contain at least one value. The values MUST be a valid algorithm identifier as defined in the [JSON Web Signature and Encryption Algorithms](https://www.iana.org/assignments/jose/jose.xhtml#web-signature-encryption-algorithms) registry.
- `did_methods_supported` - Required. List of strings indicating which DID methods are supported by the issuer for binding the credential to the holder. The list MUST contain at least one value unless `jwk_supported` is `true`. Values should ONLY include the method identifier of the DID method (e.g. `key`, `jwk`, `web`).
- `jwk_supported` - Optional. Boolean indicating whether the issuer accepts a signed attachment with a bare key in a `jwk` header instead of a DID URL in `kid`. Defaults to `false`.
- `nonce` - Required. Nonce to be used in the request to prevent replay attacks of the signed attachment.

##### Binding Proof in Request

The binding proof in the request points to an appended attachment containing the signed attachment.

```json
{
  "didcomm_signed_attachment": {
    "attachment_id": "<@id of the attachment>"
  }
}
```

- `attachment_id` - Required. The id of the appended attachment included in the request message that contains the signed attachment.

###### Signed Attachment Content

The attachment MUST be signed by including a signature in the `jws` field of the attachment. The data MUST be a JSON document encoded in the `base64` field of the attachment.

**JWS Payload:**

```json
{
  "nonce": "<nonce from the offer binding_method>"
}
```

- `nonce` - Required. The `nonce` from the `didcomm_signed_attachment` object within `binding_method` from the credential offer.

**Protected Header:**

```json
{
  "alg": "ES256",
  "kid": "did:key:zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169#zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169"
}
```

With a bare key in `jwk` instead of `kid`:

```json
{
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "fyNYMN0976ci7xqiSdag3buk-ZCwgXU4kz9XNkBlNUI",
    "y": "hW2ojTNfH7Jbi8--CJUo3OCbH3y5n91g-IMA9MLMbTU"
  }
}
```

- `alg` - Required. A digital signature algorithm identifier as per IANA "JSON Web Signature and Encryption Algorithms" registry. MUST NOT be `none` or an identifier for a symmetric algorithm (MAC). MUST match one of the `algs_supported` entries from the offer `binding_method` object.
- `kid` - Required unless `jwk` is present. JOSE Header containing an absolute DID URL pointing to a specific key in a DID document. The DID method of the DID URL MUST match one of the `did_methods_supported` from the offer `binding_method` object. The verification method it dereferences to MUST be in the `authentication` relationship of the resolved DID document ([DID Core section 5.3.1, Authentication](https://www.w3.org/TR/did-1.0/#authentication)).
- `jwk` - Required unless `kid` is present. JOSE Header containing the holder's public key ([RFC 7515 section 4.1.3, "jwk" (JSON Web Key) Header Parameter](https://www.rfc-editor.org/rfc/rfc7515.html#section-4.1.3)). It MUST NOT contain private key members or `x5u`. It MUST NOT be used unless the offer `binding_method` object has `jwk_supported` set to `true`.

The header MUST NOT have both `kid` and `jwk`, as for the `jwt` proof type of [OpenID4VCI Appendix F.1](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#appendix-F.1).

The issuer MUST verify the signature with the key named by `kid` or `jwk` and MUST check the `nonce`, `alg`, `kid` and `jwk` rules above before it issues a credential bound to that key.

##### Binding in Credential

The issued SD-JWT MUST include a `cnf` (confirmation) claim ([VC-JOSE-COSE section 4.1.3, cnf](https://www.w3.org/TR/vc-jose-cose/#cnf), [RFC 7800](https://www.rfc-editor.org/rfc/rfc7800.html)) with exactly one member, taken from the protected header of the signed attachment:

- `cnf.kid` - When the header has `kid`. It is the same DID URL string.
- `cnf.jwk` - When the header has `jwk`. It is the same public key ([RFC 7800 section 3.2, Representation of an Asymmetric Proof-of-Possession Key](https://www.rfc-editor.org/rfc/rfc7800.html#section-3.2)). It MUST NOT contain private key members or `x5u`.

## Drawbacks

- There is currently no attachment format defined for a credential proposal. This makes it impossible for a holder to initiate the issuance of a credential using this attachment format.
- This RFC only covers issuance. A separate attachment format is needed for presenting SD-JWT credentials via [Present Proof v2 (RFC 0454)](../0454-present-proof-v2/README.md).

## Rationale and alternatives

- [RFC 0593: JSON-LD Credential Attachment](../0593-json-ld-cred-attach/README.md) supports Linked Data Proof credentials but does not support SD-JWT.
- [RFC 0809: W3C Data Integrity Credential Attachment](../0809-w3c-data-integrity-credential-attachment/README.md) supports Data Integrity proofs and served as the structural basis for this RFC.
- The `hlindy-zkp-v1.0` format is restricted to the Hyperledger Indy network.
- [OpenID for Verifiable Credential Issuance](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) provides SD-JWT credential issuance but is not a DIDComm-based protocol.

## Prior art

This attachment format is structurally based on [RFC 0809: W3C Data Integrity Credential Attachment](../0809-w3c-data-integrity-credential-attachment/README.md), adapted for the SD-JWT securing mechanism.

## Unresolved questions

- Should the holder be able to propose specific claims as selectively disclosable?

## Implementations

The following lists the implementations (if any) of this RFC. Please do a pull request to add your implementation. If the implementation is open source, include a link to the repo or to the implementation within the repo. Please be consistent in the "Name" field so that a mechanical processing of the RFCs can generate a list of all RFCs supported by an Aries implementation.

| Name / Link | Implementation Notes |
| ----------- | -------------------- |
|             |
