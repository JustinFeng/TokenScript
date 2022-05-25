### eip: 5???

### title: asserting authenticity of Client Script for Token Contracts

### description: Provide a method to assert the authenticity of client script for token contracts, and a way for revokation

### author: Weiwu (@weiwu-zhang), Tore Frederiksen (@jot2re)

### discussions-to:

### status: Draft

### type: Standards Track

### category: ERC

### created: 2s022-05-03

### requires: 

### Abstract

This ERC describes how to assert the authenticity of the script related to some token or smart contract, regardless of how the script was obtained.

### Motivation

Often NFT authors want to provide some user functionality to their tokens through client scripts. This should be done safely, without opening the user to potential scams. Refer to ERC xxxx examples of such scripts.

Although ERC xxxx specified a way to obtain a set of client scripts through URI, it is inapplicable for token contracts that was issued before the creation of ERC xxxx. Furthermore, it lack the finess to address situations such as

- a smart contract might have different scripts for different environments or use-cases. Take a subway token as an example. It might invoke a minimal script at the POS to be sent through NFC (Internet might be slow or inaccessible underground), while advanced functions for user retention, such as rewarding user mascot NFT for continued use, or carbon credit for buying carbon-neutral airfare.
- in a specific use case, a token's script is often localized, and it doesn't make sense to download and load all language translations.
- a specific use-case might be compatible with only a specific version of the token's script.

ERC xxxx returns an all-purpose, one-version of script for use on the client side ignoring the nuances.

This ERC offers a way to assert authenticity of such client scripts disregarding how it is obtained, and can work with smart contracts *prior* to the publication of this ERC.

### Overview

Although the *token/smart contract author* and the *client script author* can be the same person/team, we will assume they are different people in this ERC, and the case that they are the same person/team can be implied.

The steps needed to ensure script authenticity can be summarized as follows:

1. The *script author* creates a *script signing key*, which has an associates *verification key address*.

2. The *smart contract author*, using the *smart contract deployment key*, signs a certificate, including expiry info and whose subject public key info contains the *script signing key's* associated *verification key*.

3. Any client script which is deployed gets signed by the *script signing key* and with a certificate attached.

This process is a deliberate copy of the TLS certification, based on X.509, which has stood the test of time. 

The authenticity of the client script may be obtained through the `scriptURI()` function call, as in ERC5XX0, or supplied separately by the use-cases. However, this ERC is applicable to any code or data that is signed, and a client must validate the signature in the way specified in this ERC. In real life use-cases, the client scripts can be either supplied by aforementioned `scriptURI()` or offered to the client (wallet) in anyway the wallet can work with, even through NFC connections or QR code.

### Specification
The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY” and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

#### Format of the certificate and signature

The certificate for the *script signing key* MUST be in the X.509 format, in accordance with [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280).
Furthermore, the certificate MUST by signed by the *smart contract author* as the issuer, using the *smart contract deployment key*. 

We furthermore make the following requirements of the content of this certificate:

- The `issuer` field MUST be populated with a Common Name, which MUST be the address of the verification key associated with the *smart contract deployment key*. E.g. `CN=0x12345678901234567890`.

- The `SubjectPublicKeyInfo` field  MUST contain the public part of the *script signing key*.

- The `extensions` field SHOULD be set to include `KeyUsage` (see RFC 5280 sec. 4.2.1.3) with bit 0 set, to indicate that the *script signing key* is used for signing only. Furthermore the *Extended Key Usage* extensions SHOULD also be included, with only the *id-kp-codeSigning* identifier set (See RFC 52080 sec. 4.2.1.12). 

- If [revocation option 1](#Revocation) is used, then `extensions` MUST also include `cRLDistributionPoints` (see RFC 5280 4.2.1.13) which MUST contain at least one `distributionPoint` element, containing a `fullName` element. The `fullName` MUST be a single `IA5String` for a `uniformResourceIdentifier` which is an URI pointing to a Certificate Revocation List.

- The `notBefore` and `notAfter` field SHOULD be set to limit lifetime of the *script signing key* reasonably.

- The `version` field SHOULD be grater than 2, to indicate that the certificate is **not** a regular X.509 certificate.


We require the signature to be done as a JWS according to [RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515). Concretely we have the following requirements to the elements contained in the JWS:

- The `x5u` header MUST be included and stored as part of the `JWS Protected Header`. Furthermore, its URI MUST point to the X.509 certificate of the *script signing key*.

- The `payload` member MUST be exactly the URL base64 encoding of the client script *or* an a Keccak of the client script.


#### Format to attach signature

This ERC does not specify how a wallet client obtains the signature as this can be realized in multiple ways. The simplest of which is to embed the client script in the `payload` of the JWS, as discussed above. This ensure that only a single URI is needed in order to locate both the client script, its signature and the signing key certificate. 

If the client script is stored in a directory (e.g. on a webserver), i.e. with a file-name instead of with a hash digest identifier, then the JWS can simply be stored using the same URI as the client script, with ".jws" or ".sig" appended. 


#### Revocation

While the X.509 certificate SHOULD be issued with a limited life-time, key leaks can still happen, and thus it should be possible to revoke an already issued X.509 certificate. 

This ERC does not dictate how or if it should be done. But it is required that the `notBefore` and `notAfter` fields in the X.509 certificate are significantly constrained if the no revocation mechanism is used.

We furthermore define 2 OPTIONAL revocation mechanisms:

1. Using Certificate Revocation Lists (CRL). The *smart contract author* published a signed list of revoked certificates at one or more URIs. If this option is used, then the X.509 certificate MUST contain information of how to access the CRL, as already discussed. This ERC does not dictate the format of the CRL but recommend keeping it as simple as possible. For example letting it be a JWS, signed with the *smart contract deployment key* where the `payload` isa URL base64 encoding of a comma-separated list of (hex) Keccak hash digests of each revoked X.509 certificate. In this example, the signature of the CRL SHOULD be validated against the address of the *smart contract deployment key* which issued the X.509 certificate, when checking for revocation. Furthermore, when checking the revocation list it MUST be verified that the Keccak Hash digest of the X.509 certificate is not included in the CRL.

2. Storing a list of revoked verification key addresses in the smart Contract. A smart contract `function revokeVerificationAddr(address memory revokedVerificationAddr)` could be added to the token smart contract, which MUST append `revokedVerificationAddr` to a list, which can then be returned through a `function revokedVerificationAddrs() external view returns(address[] memory)`. In this case, the address of the `SubjectPublicKeyInfo` in the X.509 certificate MUST be validated to **not** be in the list of returned `revokedVerificationAddrs`.


#### Validation

When it comes to validating the authenticity of client script, the following steps MUST be completed. If any of these steps fails, then the script MUST be rejected:

1. The client script, JWS and X.509 and the address of the *smart contract deployment key* MUST be fetched. Note that if the client script is embedded as the `payload `of the JWS, then an URI of the JWS uniquely defines how to learn both the signature (embedded in the JWS), the client script (embedded in the JWS), and the X.509 certificate (through the URI in the `x5u` header). 

2. The JWS signature is validated according to RFC 7515, using the public key contained in the `SubjectPublicKeyInfo` field of the X.509 certificate.

3. If the `payload` of the JWS does **not** contain the script, then it MUST be validated that the `payload` is the Keccak hash digest of the client script fetched.

4. The X.509 certificate is validated according to RFC 5280, with the exception that a value of `version` SHOULD be accepted if and only if it is greater than 2, and that the public key used to sign it is recovered from the signature itself. Furthermore, the recovered public key MUST be checked to represent the same address as stored in the Common Name in the `issuer` field.

5. The address stored in the Common Name of the `issuer` field in the X.509 certificate is validated to be equal to the address of the *smart contract deployment key* .

6. If the client script/JWS is pointed to by a `scriptURI` in accordance to ERC 5XX0, then it SHOULD also be validated that the URI of the client script/JWS is contained in the array returned by the smart contract method `scriptURI()`.

7. If a [revocation mechanism](#Revocation) is used, then it should validated that the X.509 certificate has not been revoked.

### Concerning Proxy Contracts


