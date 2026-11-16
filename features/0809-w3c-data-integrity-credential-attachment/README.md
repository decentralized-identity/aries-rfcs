# Aries RFC 0809: W3C Verifiable Credential Data Integrity Attachment format for requesting and issuing credentials

- Authors: Timo Glastra (Animo Solutions)
- Status: [DEMONSTRATED](/README.md#demonstrated)
- Since: 2024-01-10
- Status Note: Implemented in [Aries Cloud Agent Python](https://github.com/hyperledger/aries-cloudagent-python) and [Credo](https://github.com/openwallet-foundation/credo-ts)
- Supersedes: [Aries RFC 0593: JSON-LD Credential Attachment](https://github.com/hyperledger/aries-rfcs/blob/main/features/0593-json-ld-cred-attach/README.md)
- Start Date: 2023-12-18
- Tags: [feature](/tags.md#feature), [protocol](/tags.md#protocol), [credentials](/tags.md#credentials), [test-anomaly](/tags.md#test-anomaly)

## Summary

This RFC registers an attachment format for use in the [issue-credential V2](../0453-issue-credential-v2/README.md) protocol based on W3C Verifiable Credentials with [Data Integrity Proofs](https://www.w3.org/TR/vc-data-integrity/) from the [VC Data Model](https://www.w3.org/TR/vc-data-model/#linked-data-proofs).

## Motivation

The Issue Credential protocol needs an attachment format to be able to exchange W3C verifiable credentials. It is desirable to make use of specifications developed in an open standards body, such as the [Credential Manifest](https://identity.foundation/credential-manifest/) for which the attachment format is described in [RFC 0511: Credential-Manifest Attachment format](../0511-dif-cred-manifest-attach/README.md). However, the _Credential Manifest_ is not finished and ready yet, and therefore there is a need to bridge the gap between standards.

## Tutorial

Complete examples of messages are provided in the [reference section](#reference).

## Reference

### Credential Offer Attachment Format

Format identifier: `didcomm/w3c-di-vc-offer@v0.1`

```json
{
  "options": {
    "binding_method": {
      "anoncreds_link_secret": {
        "nonce": "1234",
        "cred_def_id": "did:key:z6MkwXG2WjeQnNxSoynSGYU8V9j3QzP3JSqhdmkHc6SaVWoT/credential-definition",
        "key_correctness_proof": "<key_correctness_proof>"
      },
      "didcomm_signed_attachment": {
        "algs_supported": ["EdDSA"],
        "did_methods_supported": ["key", "web"],
        "nonce": "1234"
      }
    }
  },
  "credential": {
    "@context": [
      "https://www.w3.org/ns/credentials/v2",
    ],
    "type": ["VerifiableCredential"],
    "credentialSubject": {
      "height": 175,
      "age": 28,
      "name": "Alex",
      "sex": "male"
    }
  }
}
```
- `binding_method` - Optional. Object containing key-value pairs of binding methods supported by the issuer to bind the credential to a holder. If the value is omitted, this indicates the issuer does not support any binding methods for issuance of the credential. See [Binding Methods](#binding-methods) for a registry of default binding methods supported as part of this RFC.
- `credential` - Required. The credential to be issued. The credential MUST be compliant with the **first** `data_model_versions_supported` entry version of VC Data Model, except for the omission of a set required keys that may only be known at the time of issuance. See [Credential Offer Exceptions](#credential-offer-exceptions) for a list of exceptions. The credential MUST NOT contain any proofs. Some properties MAY be omitted if they will only be available at time of issuance, such as `issuanceDate`, `issuer`, `credentialSubject.id`, `credentialStatus`, `credentialStatus.id`.

#### Credential Offer Exceptions

To allow for validation of the `credential` according to the corresponding VC Data Model version, the `credential` in the offer MUST be conformant to the corresponding VC Data Model version, except for the exceptions listed below. This still allows the credential to be validated, knowing which deviations are possible.

The list of exception is as follows:

- `issuanceDate` (v1.1) or `validFrom` (v2.0) can be omitted, or set to a placeholder value.
- `issuer` (or `issuer.id` if issuer is an object) can be omitted
- `credentialSubject.id` can be omitted
- `credentialStatus`
  - Either the whole `credentialStatus` can be omitted
  - OR the `credentialStatus.type` can be present, but other required fields that are dynamic can be omitted (such as the `statusListIndex` and `statusListCredential` in case of [Bitstring Status List](https://www.w3.org/TR/vc-bitstring-status-list/))

### Credential Request Attachment Format

Format identifier: `didcomm/w3c-di-vc-request@v0.1`

This format is used to request a verifiable credential. The JSON structure might look like this:

```json
{
  "data_model_version": "2.0",
  "binding_proof": {
    "anoncreds_link_secret": {
      "entropy": "<random-entropy>",
      "cred_def_id": "did:key:z6MkwXG2WjeQnNxSoynSGYU8V9j3QzP3JSqhdmkHc6SaVWoT/credential-definition",
      "blinded_ms": {},
      "blinded_ms_corectness_proof": {},
      "nonce": "<random-nonce>"
    },
    "didcomm_signed_attachment": {
      "attachment_id": "<@id of the attachment>"
    }
  }
}
```

- `data_model_version` - Required. The data model version of the credential to be issued. The value MUST be a valid data model version and match one of the values from the `data_model_versions_supported` offer.
- `binding_proof` - Required if `binding_required` is `true` in the offer. Object containing key-value pairs of proofs for the binding to the holder. The keys MUST match keys of the `binding_method` object from the offer. See [Binding Methods](#binding-methods) for a registry of default binding methods supported as part of this RFC.

### Credential Attachment Format

Format identifier: `didcomm/w3c-di-vc@v0.1`

This format is used to transmit a verifiable credential. The JSON structure might look like this:

```json
{
  "credential": {
    // vc with proof object or array
  }
}
```

- `credential` - The signed credential. MUST be a valid verifiable credential object with one or more proofs and MUST conform to VC Data Model version as defined in the `data_model_version` of the request.

It is up to the issuer to the pick an appropriate cryptographic suite to sign the credential. The issuer may use the cryptographic binding material provided by the holder to select the cryptographic suite. For example, when the `anoncreds_link_secret` binding method is used, the issuer should use an `DataIntegrityProof` with the `anoncredsvc-2023` cryptographic suite. When a holder provides a signed attachment as part of the binding proof using the `EdDSA` JWA alg, the issuer could use a `DateIntegrityProof` with the `eddsa-rdfc-2022` cryptographic suite. However, it is not required for the cryptographic suite used for the signature on the credential to be in any way related to the cryptographic suite used for the binding proof, unless the binding method explicitly requires this (for example the `anoncreds_link_secret` binding method).

A complete [`issue-credential` message from the Issue Credential protocol 2.0](../0453-issue-credential-v2/README.md#issue-credential) might look like this:

```json
{
  "@id": "284d3996-ba85-45d9-964b-9fd5805517b6",
  "@type": "https://didcomm.org/issue-credential/2.0/issue-credential",
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "format": "didcomm/w3c-di-vc@v0.1"
    }
  ],
  "credentials~attach": [
    {
      "@id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "mime-type": "application/json",
      "data": {
        "base64": "ewogICAgICAgICAgIkBjb250ZXogWwogICAgICAg...(clipped)...RNVmR0SXFXZhWXgySkJBIgAgfQogICAgICAgIH0="
      }
    }
  ]
}
```

### Binding Methods

The attachment format supports different methods to bind the credential to the receiver of the credential. In the offer message the issuer can indicate which binding methods are supported in the `binding_methods` object. Each key represents the id of the supported binding method.

This section defines a set of binding methods supported by this attachment format, but other binding methods may be used. Based on the binding method, the request needs to include a `binding_proof` object where the key matches the key of the binding method from the offer.

#### AnonCreds Link Secret

Identifier: `anoncreds_link_secret`

This binding method is intended to be used in combination with a credential containing an AnonCreds proof.

##### Binding Method in Offer

The structure of the binding method in the offer MUST match the structure of the [Credential Offer](https://hyperledger.github.io/anoncreds-spec/#credential-offer) as defiend in the AonCreds specification, with the exclusion of the `schema_id` key.

```jsonc
{
  "nonce": "1234",
  "cred_def_id": "did:key:z6MkwXG2WjeQnNxSoynSGYU8V9j3QzP3JSqhdmkHc6SaVWoT/credential-definition",
  "key_correctness_proof": {
    /* key correctness proof object */
  }
}
```

##### Binding Proof in Request

The structure of the binding proof in the request MUST match the structure of the [Credential Request](https://hyperledger.github.io/anoncreds-spec/#credential-request) as defined in the AnonCreds specification.

```jsonc
{
  "anoncreds_link_secret": {
    "entropy": "<random-entropy>",
    "blinded_ms": {
      /* blinded ms object */
    },
    "blinded_ms_corectness_proof": {
      /* blinded ms correctness proof object */
    },
    "nonce": "<random-nonce>"
  }
}
```

##### Binding in Credential

The issued credential should be bound to the holder by including the blinded link secret in the credential as defined in the [Issue Credential section](https://hyperledger.github.io/anoncreds-spec/#issue-credential) of the AnonCreds specification. Credential bound using the AnonCreds link secret binding method MUST contain an proof with `proof.type` value of `DataIntegrityProof` and `cryptosuite` value of `anoncredsvc-2023`, and conform to the [AnonCreds W3C Verifiable Credential Representation](https://hyperledger.github.io/anoncreds-spec/#w3c-verifiable-credentials-representation).

#### DIDComm Signed Attachment

Identifier: `didcomm_signed_attachment`

This binding method leverages [DIDComm signed attachments](https://github.com/hyperledger/aries-rfcs/blob/main/concepts/0017-attachments/README.md#signing-attachments) to bind a credential to a specific key and/or identifier.

##### Binding Method in Offer

```jsonc
{
  "didcomm_signed_attachment": {
    "algs_supported": ["EdDSA"],
    "did_methods_supported": ["key"],
    "nonce": "b19439b0-4dc9-4c28-b796-99d17034fb5c"
  }
}
```

- `algs_supported` - Required. List of strings indicating the Json Web Algorithms supported by the issuer for verifying the signed attachment. The list MUST contain at least one value. The values MUST be a valid algorithm identifier as defined in the [JSON Web Signature and Encryption Algorithms](https://www.iana.org/assignments/jose/jose.xhtml#web-signature-encryption-algorithms) registry.
- `did_methods_supported` - Required. List of strings indicating which did methods are supported by the issuer for binding the credential to the holder. The list MUST contain at least one value. Values should ONLY include the method identifier of the did method. Examples values include `key` or `web`.
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

- `attachment_id` - The id of the appended attachment included in the request message that contains the signed attachment.

###### Signed Attachment Content

The attachment MUST be signed by including a signature in the `jws` field of the attachment. The data MUST be a JSON document encoded in the `base64` field of the attachment. The structure of the signed attachment is described below.

**JWS Payload**

```json
{
  "nonce": "<request_nonce>",
}
```

- `nonce` - Required. The `nonce` from the `didcomm_signed_attachment` object within `binding_method` from the credential offer

**Protected Header**

```json
{
  "alg": "EdDSA",
  "kid": "did:key:z6MkkwiqX7BvkBbi37aNx2vJkCEYSKgHd2Jcgh4AUhi4YY1u#z6MkkwiqX7BvkBbi37aNx2vJkCEYSKgHd2Jcgh4AUhi4YY1u"
}
```

- `alg`: REQUIRED. A digital signature algorithm identifier such as per IANA "JSON Web Signature and Encryption Algorithms" registry. MUST NOT be none or an identifier for a symmetric algorithm (MAC). MUST match one of the `algs_supported` entries from the offer `binding_method` object.
- `kid`: REQUIRED. JOSE Header containing the DID URL pointing to a specific key in a did document. The did method of the DID URL MUST match with one of the `did_methods_supported` from the offer `binding_method` object.

A signed binding request attachment appended to a request message might look like this:

```json
{
  "@id": "284d3996-ba85-45d9-964b-9fd5805517b6",
  "@type": "https://didcomm.org/issue-credential/2.0/request-credential",
  "comment": "<some comment>",
  "formats": [
    {
      "attach_id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "format": "didcomm/w3c-di-vc-request@v0.1"
    }
  ],
  "~attach": [
    {
      "@id": "123",
      "mime-type": "application/json",
      "data": {
        "base64": "<base64-encoded-json-attachment-content>",
        "jws": {
          "protected": "eyJhbGciOiJFZERTQSIsImlhdCI6MTU4Mzg4... (bytes omitted)",
          "signature": "3dZWsuru7QAVFUCtTd0s7uc1peYEijx4eyt5... (bytes omitted)"
        }
      }
    }
  ],
  "credentials~attach": [
    {
      "@id": "5b38af88-d36f-4f77-bb7a-2f04ab806eb8",
      "mime-type": "application/json",
      "data": {
        "base64": "ewogICAgICAgICAgIkBjb250ZXogWwogICAgICAg...(clipped)...RNVmR0SXFXZhWXgySkJBIgAgfQogICAgICAgIH0="
      }
    }
  ]
}
```

##### Binding in Credential

The issued credential should be bound to the holder by including the did in the credential as `credentialSubject.id` or `holder`.

## Drawbacks

- While it has a similar setup and structure compared to OpenID for Verifiable Credential Issuance, this attachment format only focuses on issuance of W3C Verifiable Credentials. Therefore another (but probably quite similar) attachment format needs to be defined to support issuance of non-W3C VCs
- There is currently no way for an issuer or holder to indicate which cryptographic suites they support for the signature of the credential. Currently it's at the discretion of the issuer to decide which cryptographic suite to use. In a future (minor) version we can add an optional way for a) the issuer to indicate which cryptographic suites they support, and b) the holder to indicate which cryptographic suites they support.
- There is currently no attachment format defined for a credential proposal. This makes it impossible for a holder to initiate the issuance of a credential using this attachment format. In a future (minor) version the proposal attachment format can be added.

## Rationale and alternatives

[RFC 0593: JSON-LD Credential Attachment](https://github.com/hyperledger/aries-rfcs/blob/main/features/0593-json-ld-cred-attach/README.md), [W3C VC API](https://w3c-ccg.github.io/vc-api/) allows issuance of credentials using only linked data signatures, while [RFC 0592: Indy Attachment](https://github.com/hyperledger/aries-rfcs/blob/main/features/0592-indy-attachments/README.md) supports issuance of AnonCreds credentials. This attachment format aims to support issuance of both previous attachment formats (while for AnonCreds it now being in the W3C model), as well as supporting additional features such as issuance W3C JWT VCs, credentials with multiple proofs, and cryptographic binding of the credential to the holder.

## Prior art

The attachment format in this RFC is heavily inspired by [RFC 0593: JSON-LD Credential Attachment](https://github.com/hyperledger/aries-rfcs/blob/main/features/0593-json-ld-cred-attach/README.md), [W3C VC API](https://w3c-ccg.github.io/vc-api/) and [OpenID for Verifiable Credential Issuance](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html).

## Unresolved questions
