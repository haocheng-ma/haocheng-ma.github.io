---
layout: post
title: Key Management in TPM based Security
date: 2025-07-24
description: TPMs are ridiculously complex.
tags: security
categories: tpm
# giscus_comments: true
# related_posts: true
toc: true
---

A TPM (Trusted Platform Module) does not store full keys directly. Instead, it persists secret values called Seeds and derives keys from them deterministically. Two common seed types drive two key hierarchies: Endorsement and Storage.

| Seed type        | Key hierarchy          | Purpose                      |
| ---------------- | ---------------------- | ---------------------------- |
| Endorsement Seed | Endorsement Key (EK)   | Identifies the TPM           |
| Storage Seed     | Storage Root Key (SRK) | Protects keys for local apps |

{% include figure.liquid path="assets/img/2025-07-24-tpm-based-security/key_management.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. Endorsement and Storage types in TPM key management
</div>

TPM keys fall into two categories by use:

- **Restricted**: sign or decrypt TPM-internal state and data. Examples include signing boot measurements and proving that two keys live in the same TPM. EK, SRK, and AIK are all restricted.
- **Non-Restricted**: serve general purposes such as TLS client keys or document signing keys. Application keys fall in this category.

## Endorsement Key

The Endorsement Key (EK) is the TPM's identity, provisioned by the TPM manufacturer.

- The EK key pair is generated deterministically by `TPM2_CreatePrimary()` from a known template (the EK Template), the Endorsement Primary Seed (EPS), and a chosen key derivation function (KDF).
- The EK certificate is issued by the TPM manufacturer's CA and stored in TPM NVRAM.
- The EK certificate contains the EK public key along with fields such as the manufacturer name, component model, component version, and key ID.

For the EK Template format, see [Default EK Template (TPMT_PUBLIC): RSA 2048 (Storage)](https://trustedcomputinggroup.org/wp-content/uploads/TCG_IWG_EKCredentialProfile_v2p3_r2_pub.pdf#page=37).

The EK lives in a TPM Shielded Location. The private key never leaves the TPM; software can read only the public key.

Two related storage concepts apply:

- **Isolated Locations**: non-volatile storage with access control that rejects access from outside the system boundary.
- **Shielded Locations**: tamper-resistant storage, typically used for secret values.

## Storage Root Key

The TPM's storage hierarchy organizes keys as a tree:

- Keys such as the Attestation Key (AK) and application keys are created under this hierarchy.
- Their parent is the Storage Root Key (SRK), a root inside the TPM that protects and derives child keys.
- Key attestation for these keys is performed by the EK.

In other words, key attestation builds a chain of trust by signing: each link carries trust from the EK down to a specific key such as an AK.

## Attestation Key

The Attestation Key (AK) signs data produced inside the TPM. It has three defining properties:

- **Non-duplicable**: the TPM generates the AK internally and never exposes key material. The private key cannot be exported or copied.
- **Restricted**: the AK signs or decrypts only TPM-produced data, such as PCR (Platform Configuration Register) values. It cannot sign or encrypt arbitrary data, which prevents misuse as a general-purpose key.
- **Signing key for device identity (DevID)**: the AK public key certificate can serve as a device credential, issued by the manufacturer or another trusted party.

_"The 802.1AR standard defines a secure device identifier (DevID) as 'an identifier that is cryptographically bound to a device'."_

Device identities created during manufacturing are called IDevID/IAK. Their credentials last for the device's lifetime. Identities created during deployment or use are called LDevID/LAK, and their credentials are removed at device reset or end of life.

_"Credential is a combination of a private key and a certificate."_

The Attestation Identity Key (AIK) is a TPM 1.2 key type. AIKs perform only two signing operations:

- **TPM_Quote**: signs PCR values with the AIK to produce a hardware report of device identity and current state. Remote attestation uses this to prove a device's trustworthy state to a remote verifier.
- **TPM_CertifyKey**: produces a signed statement that another key (not the AIK itself) resides in the TPM's key hierarchy and is non-migratable. The certified key must actually have these properties.

## Credential Activation

Credential activation lets a remote party establish trust in a new key such as the IAK, which can then encrypt or attest on the device's behalf. It proves two things:

- The IAK (Identity Attestation Key) has the expected key attributes:

  ```c
  FlagFixedTPM             // Key cannot be exported; it must stay in the TPM.
  FlagSensitiveDataOrigin  // Key was generated inside the TPM, not imported.
  ```

- The IAK resides in the same TPM as the EK.

_Credential activation allows a remote party to verify one key is on the same TPM as another key, and that the key has a specific set of properties._

The flow is shown below. The activation key is the IAK; the anchor key is the EK.

{% include figure.liquid path="assets/img/2025-07-24-tpm-based-security/credential_activation.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. The flow of credential activation
</div>

- The client sends the IAK public key, the EK public key, and the IAK attributes to the server.
- The server checks the IAK attributes against policy, generates a secret, and builds a challenge:

  ```
  Challenge = ENC{secret} | aes_key1
              ENC{key1}  | aes_key2
              ENC{seed}  | EK_pub
  ```

  `aes_key1` is a randomly generated symmetric key. `aes_key2 = KDF(seed, IAK name)`, where KDF is a key derivation function and IAK name is the hash of the IAK public key blob (which includes attributes). All sensitive data is encrypted with the EK public key, so only the target TPM can decrypt it.

- The client passes the challenge to the TPM via the `ActivateCredential` command.
- The TPM can decrypt and return the secret only when both the matching EK and IAK reside inside it. A successful decryption proves that the IAK exists in this TPM and carries the correct attributes.

## Reference

1. [https://ericchiang.github.io/post/tpm-keys/]()
2. [https://tpm2-software.github.io/tpm2-tss/getting-started/2019/12/18/Remote-Attestation.html]()
3. [https://github.com/google/go-attestation/blob/master/docs/credential-activation.md]()
4. [https://github.com/tpm2dev/tpm.dev.tutorials]()
