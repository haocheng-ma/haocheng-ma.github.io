---
layout: post
title: Key Management with a TPM
date: 2025-07-24
description: TPMs are ridiculously complex.
tags: security
giscus_comments: true
# related_posts: true
toc: true
---

A Trusted Platform Module (TPM) does not need to keep every key permanently loaded. TPM 2.0 organizes keys into hierarchies, each rooted in a secret seed from which the TPM can deterministically recreate primary keys. This post focuses on two of those hierarchies: Endorsement and Storage.

| Seed type        | Key hierarchy          | Purpose                      |
| ---------------- | ---------------------- | ---------------------------- |
| Endorsement Seed | Endorsement Key (EK)   | Identifies the TPM           |
| Storage Seed     | Storage Root Key (SRK) | Protects keys for local apps |

{% include figure.liquid path="assets/img/2025-07-24-tpm-based-security/key_management.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. The Endorsement and Storage hierarchies
</div>

The `restricted` attribute further limits what a TPM key may do:

- **Restricted keys** operate only on data that has the structure or provenance expected by the TPM. An Attestation Key, for example, signs TPM-generated attestation data, while a restricted decryption key can protect child objects. EKs, SRKs, and Attestation Keys are normally created as restricted keys.
- **Unrestricted keys** can perform general-purpose operations. A TLS client key or a document-signing key would usually belong here.

## Endorsement Key

The Endorsement Key (EK) is a cryptographic identity for the TPM. Its certificate lets another party associate the EK public key with a genuine TPM.

- `TPM2_CreatePrimary()` can reproduce the EK key pair from the Endorsement Primary Seed (EPS) and a known public template. The TPM performs the underlying key derivation internally.
- An EK certificate is usually issued during manufacturing by the TPM or platform manufacturer and stored in TPM non-volatile memory.
- The certificate contains the EK public key and may also identify the manufacturer, model, version, and key ID.

For the public-area format, see the TCG's [default RSA 2048 EK template](https://trustedcomputinggroup.org/wp-content/uploads/TCG_IWG_EKCredentialProfile_v2p3_r2_pub.pdf#page=37).

The EK is held in a TPM Shielded Location. Software can read its public key, but the private part must never be exposed outside the TPM.

Two related terms describe how a TPM protects data:

- **Isolated Locations** are protected by access controls at the TPM boundary.
- **Shielded Locations** also protect secret values from exposure or modification while the TPM operates on them.

## Storage Root Key

The Storage hierarchy organizes keys as a tree. Its root is the Storage Root Key (SRK), and keys such as Attestation Keys and application keys can be created beneath it.

The entire tree does not have to remain in TPM memory. A child key can be stored outside the TPM as a protected blob; its parent key protects the confidential part and verifies its integrity when the object is loaded again. Following the parent links eventually leads back to the SRK.

This storage relationship is not itself a certificate chain. When a remote service needs to connect a new Attestation Key to the TPM identified by an EK, it can use credential activation, described below.

## Attestation Key

An Attestation Key (AK) signs evidence produced by the TPM. A typical AK has three important properties:

- **Bound to one TPM**: attributes such as `fixedTPM` and `fixedParent` prevent the key from being moved to another TPM or parent. When the TPM also generates the sensitive material, the private key is never exposed in plaintext.
- **Restricted to attestation**: rather than signing arbitrary application data, the key signs TPM-generated structures, including reports containing Platform Configuration Register (PCR) values.
- **Usable as a device credential**: a manufacturer or another trusted party can certify the AK public key and use it as part of a device identity (DevID).

IEEE 802.1AR defines a secure device identifier (DevID) as an identifier cryptographically bound to a device.

An identity installed during manufacturing is an Initial Device Identifier (IDevID), with the corresponding key referred to here as an IAK. It normally lasts for the lifetime of the device. An identity installed later by an owner or operator is a Locally Significant Device Identifier (LDevID), with a corresponding LAK; it can be replaced or removed when the device is reset or retired.

In this context, a credential consists of the private key together with the certificate that binds its public key to the device identity.

Attestation Identity Key (AIK) is the TPM 1.2 name for this kind of key. An AIK supports two signing operations:

- **`TPM_Quote`** signs a report containing selected PCR values. A remote verifier can use the report as evidence of the device's measured state.
- **`TPM_CertifyKey`** signs a statement about another key in the TPM hierarchy, including whether that key is non-migratable.

## Credential Activation

Suppose a server already trusts an EK certificate and now needs to enroll a newly created IAK. Credential activation checks two facts before the server accepts that key:

- The IAK has the attributes required by the server's policy. For example:

  ```c
  FlagFixedTPM             // The key cannot move to another TPM.
  FlagSensitiveDataOrigin  // The TPM generated the key's sensitive material.
  ```

- The IAK resides in the same TPM as the EK.

In the terminology used in Figure 2, the IAK is the activation key and the EK is the anchor key.

{% include figure.liquid path="assets/img/2025-07-24-tpm-based-security/credential_activation.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. Credential activation flow
</div>

- The client sends the server the EK public key and the IAK's public area. The public area contains both the key and its attributes; hashing it produces the IAK Name.
- The server checks those attributes against its policy, generates a secret, and creates a credential for the IAK. A simplified view is:

  ```text
  encrypted_seed = Encrypt(EK_public, seed)
  symmetric_key  = KDF(seed, IAK_Name)
  credential     = EncryptAndAuthenticate(symmetric_key, secret, IAK_Name)
  challenge      = credential || encrypted_seed
  ```

  Only the TPM holding the EK private key can recover the seed. The IAK Name is part of the key derivation and integrity check, so changing the IAK public key or its attributes produces a different result.

- The client passes the challenge and handles for both keys to `TPM2_ActivateCredential`.
- The TPM uses the EK to recover the seed, recomputes the keys from the seed and the loaded IAK's Name, verifies the credential, and returns the secret. Success tells the server that this exact IAK, with the attributes it inspected, is loaded in the same TPM as the EK.

## Reference

1. [A Practical Guide to TPM 2.0](https://ericchiang.github.io/post/tpm-keys/)
2. [Remote Attestation with tpm2-tss](https://tpm2-software.github.io/tpm2-tss/getting-started/2019/12/18/Remote-Attestation.html)
3. [Credential Activation in go-attestation](https://github.com/google/go-attestation/blob/master/docs/credential-activation.md)
4. [tpm.dev Tutorials](https://github.com/tpm2dev/tpm.dev.tutorials)
