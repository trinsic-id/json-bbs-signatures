# BBS+ Signature Suite for LDP using JSON Pointer Normalization

This document describes a BBS signature suite for Linked Data Proofs. The suite uses [JSON Pointer Normalization](https://trinsic-id.github.io/json-ptr-n11n-spec/) to generate verifiable data suitable for use with BBS signature algorithms.

## Motivation

The main goal of this signature suite is to define additional BBS signature scheme that utilize simpler algorithms compared to existing ones. This is useful in situations where JSON-LD processors or suites are unavailable. The existing [BBS+ Signatures 2020](https://w3c-ccg.github.io/ldp-bbs2020/) uses JSON-LD processors and data canonizalization using URDNA2015 in contrast to this suite which uses simpler normalization algorithm based on JSON Pointer document flattening.
This normalization algorithm can utilize selective disclosure and ZKP features defines in the BBS spec.

## JSON Pointer

[JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) is an IETF standards track specification that defines a string syntax for identifying a specific value within a JavaScript Object Notation (JSON) document. For example, in the document below:

```json
{
  "foo": {
    "bar": "baz"
  },
  "this": "that",
  "allow": [ true, 42 ]
}
```

The JSON pointer of the value `"baz"` is `/foo/bar`, the value at JSON Pointer `/this` is `"that"`, etc.

### Canonicalized Format using JSON Pointer

Any JSON object can be rewritten in pointer format. This approach provides stable algorithm for transformation of data. The above example can be canonicalized as:

```json
{
  "/foo/bar": "baz",
  "/this": "that",
  "/allow/0": true,
  "/allow/1": 42
}
```

## Linked Data Proofs - Canonicalization Algorithm

[Section 10 - Creating New Proof Types](https://w3c-ccg.github.io/ld-proofs/#creating-new-proof-types) in Linked Data Proofs 1.0 document outlines a recommendation for creating new proof types. We can use the `canonicalizationAlgorithm` term to specify the use of JSON Pointer canonicalization in the BBS Signature suite.

```json
{
  "type": "BlsBbsSignature2020",
  "canonicalizationAlgorithm": "https://datatracker.ietf.org/doc/rfc6901/",
  "created": "2021-07-02T20:34:57Z",
  "proofPurpose": "assertionMethod",
  "verificationMethod": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619#test"
}
```

## Signature Algorithm

To produce signature using this scheme, the followoing steps are performed in order:

- Given an input document, add a document proof object in the `proof` field. The `proof` must not contain the `proofValue` element
- Normalize the entire document with proof using [JSON Pointer Normalization](https://trinsic-id.github.io/json-ptr-n11n-spec/)
- Transform the normalized document into an array of stringified statements, where each statement represents the normalized entry object of each field
- Sign the statements using BBS signature algorithm
- Append the signature value to the `proof.proofValue` field of the original document in base64url format

### Example

Given an unsigned VC document with `proof` object appended:

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1", "https://www.w3.org/2018/credentials/examples/v1"],
  "type": ["VerifiableCredential", "UniversityDegreeCredential"],
  "credentialSchema": {
    "id": "did:example:cdf:35LB7w9ueWbagPL94T9bMLtyXDj9pX5o",
    "type": "did:example:schema:22KpkXgecryx9k7N6XN1QoN3gXwBkSU8SfyyYQG"
  },
  "issuer": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619",
  "credentialSubject": {
    "givenName": "Jane",
    "familyName": "Doe",
    "degree": {
      "type": "BachelorDegree",
      "name": "Bachelor of Science and Arts",
      "college": "College of Engineering"
    }
  },
  "proof": {
    "type": "BlsBbsSignature2020",
    "canonicalizationAlgorithm": "https://datatracker.ietf.org/doc/rfc6901/",
    "created": "2021-07-02T20:34:57Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619#test"
  }
}
```

Normalized document using JSON Pointer addressing

```json
{
  "/@context/0": "https://www.w3.org/2018/credentials/v1",
  "/@context/1": "https://www.w3.org/2018/credentials/examples/v1",
  "/type/0": "VerifiableCredential",
  "/type/1": "UniversityDegreeCredential",
  "/credentialSchema/id": "did:example:cdf:35LB7w9ueWbagPL94T9bMLtyXDj9pX5o",
  "/credentialSchema/type": "did:example:schema:22KpkXgecryx9k7N6XN1QoN3gXwBkSU8SfyyYQG",
  "/issuer": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619",
  "/credentialSubject/givenName": "Jane",
  "/credentialSubject/familyName": "Doe",
  "/credentialSubject/degree/type": "BachelorDegree",
  "/credentialSubject/degree/name": "Bachelor of Science and Arts",
  "/credentialSubject/degree/college": "College of Engineering",
  "/proof/type": "BlsBbsSignature2020",
  "/proof/created": "2021-07-02T20:34:57Z",
  "/proof/proofPurpose": "assertionMethod",
  "/proof/verificationMethod": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619#test"
}
```

Normalization of each statement into stringified statements, suitable for hashing and signing using BBS.

```js
[
  '{"/@context/0":"https://www.w3.org/2018/credentials/v1"}',
  '{"/@context/1":"https://www.w3.org/2018/credentials/examples/v1"}',
  '{"/credentialSchema/id":"did:example:cdf:35LB7w9ueWbagPL94T9bMLtyXDj9pX5o"}',
  '{"/credentialSchema/type":"did:example:schema:22KpkXgecryx9k7N6XN1QoN3gXwBkSU8SfyyYQG"}',
  '{"/credentialSubject/degree/college":"College of Engineering"}',
  '{"/credentialSubject/degree/name":"Bachelor of Science and Arts"}',
  '{"/credentialSubject/degree/type":"BachelorDegree"}',
  '{"/credentialSubject/familyName":"Doe"}',
  '{"/credentialSubject/givenName":"Jane"}',
  '{"/issuer":"did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619"}',
  '{"/proof/created":"2021-07-02T20:34:57Z"}',
  '{"/proof/proofPurpose":"assertionMethod"}',
  '{"/proof/type":"BlsBbsSignature2020"}',
  '{"/proof/verificationMethod":"did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619#test"}',
  '{"/type/0":"VerifiableCredential"}',
  '{"/type/1":"UniversityDegreeCredential"}'
]
```

## Derivation Algorithm

Proof derivation algorithm is similar to the signature algorithm, with the added difference of calculating the indices of revealed statements

### Selective Disclosure

This suite only makes recommendations on some methods that systems can utilize to communicate data disclosure.

#### JSON Pointer

Applications can construct a set of pointer statements to request data for disclosure by the holder party. For example:

```json
[
  "/credentialSubject/familyName",
  "/credentialSubject/degree"
  "/issuer"
]
```

Applying this projection to the originally signed VC, we can obtain the derived document.

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1", "https://www.w3.org/2018/credentials/examples/v1"],
  "issuer": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619",
  "credentialSubject": {
    "givenName": "Jane",
    "degree": {
      "type": "BachelorDegree",
      "name": "Bachelor of Science and Arts",
      "college": "College of Engineering"
    }
  },
  "proof": {
    "type": "https://mattrglobal.github.io/bbs-signatures-spec/#name-sign",
    "created": "2021-07-02T20:34:57Z",
    "proofPurpose": "assertionMethod",
    "verificationMethod": "did:example:Wz4eUg7SetGfaUVCn8U9d62oDYrUJLuUtcy619#test"
  }
}
```


#### JSON Path

Alternatively, JSON Path syntax can also be used to communicate disclosure. For example

```json
[
  "$.credentialSubject.familyName",
  "$.credentialSubject.degree"
  "$.issuer"
]
```


### Generating Proof Data

- Use the selective disclosure method to obtain a projection of the original document
- Normalize the original document and projected document
- Transform the original document and projected document into string statements
- Obtain the indices that correspond to the revealed statements in relation to the original document
- Create BBS proof of signature using the projected document statements, the set of revealed indices, and the original `signatureValue`
