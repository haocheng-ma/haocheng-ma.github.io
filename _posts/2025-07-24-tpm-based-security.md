---
layout: post
title: Key management in TPM based security
date: 2025-07-24
description: TPMs are ridiculously complex.
tags: tpm measurement attestation
categories: cc
# giscus_comments: true
# related_posts: true
toc:
  sidebar: left
---

TPM（可信平台模块）本质上并不会直接存储完整的密钥，而是持久化存储称为 Seed（种子）的秘密值，并通过这些 Seed 确定性地派生出各种密钥。两类常见的 Seed 和对应的密钥体系是 Endorsement 和 Stroage，如下所示：

| Seed 类型              | 密钥体系                      | 用途简述                                       |
| -------------------- | --------------------------- | ------------------------------------------ |
| Endorsement Seed | Endorsement Key（EK）   | 识别 TPM 的密钥      |
| Storage Seed     | Storage Root Key（SRK） | 本地应用使用的密钥 |

{% include figure.liquid path="assets/img/2025-07-24-tpm-based-security/key_management.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. Endorcement and Storage types in TPM Key Management
</div>

上述密钥按照用途分为两类：Restricted 和 Non-Restricted。
- Restricted：用于签名或者解密 TPM 的状态和数据，例如，签名启动度量值、证明某个密钥位于同一个 TPM，EK、SRK、AIK 都在此类；
- Non-Restricted：用于一般用途，例如，TLS 客户端的密钥、签名普通文档的密钥，应用密钥（Application Key）属于此类。

## Endorsement Key

Endorsement Key（EK）是 TPM 的身份标识，由 TPM 制造商配置。
- EK 公私钥对由已知模板（EK Template）、Endorsement Primary Seed（EPS）和选定的密钥派生函数（KDF）、通过 `TPM2_CreatePrimary()` 确定性生成；
- EK 证书由TPM制造商的证书颁发机构（CA）签发并将其存储到 TPM NVRAM；
- EK 证书包含EK公钥以及其他字段，例如，TPM 制造商名称、组件型号、组件版本、密钥 ID 等。

EK Template 的具体格式可查阅：[Default EK Template (TPMT_PUBLIC): RSA 2048 (Storage)](https://trustedcomputinggroup.org/wp-content/uploads/TCG_IWG_EKCredentialProfile_v2p3_r2_pub.pdf#page=37)

EK 被存储在 TPM Shielded Location，私钥永远不能暴露，公钥可从 TPM 内读取。
 
关于 TPM 的安全存储有两个概念：
- Isolated Locations：安全系统的非易失性存储，具有访问控制能力，不允许来自系统边界外的访问；
- Shielded Locations：具有防篡改能力的 Shielded Location，通常用于存储秘密值；

## Storage Root Key


在 TPM 的 Storage 体系结构中，密钥的创建遵循树状层级结构：
- 比如 Attestation Key（AK）或应用密钥，都是在 TPM 的存储体系下生成的；
- 这些密钥的父密钥是 SRK（Storage Root Key），SRK 是 TPM 中的一个根密钥，用于保护和派生其他密钥；
- 而这些密钥的 key attestation（密钥认证） 则由 EK（Endorsement Key） 来实现。

换句话说，Key attestation 是一种通过签名链建立信任的过程，核心目的是构建一条从 EK 一直到某个具体密钥（如 AK）的信任链。

## Attestation Key

Attestation Key（AK）是一种用于对 TPM 内部数据进行签名的密钥，具有以下关键特性：
- non-duplicable（不可复制）：AK 是由 TPM 内部直接生成的，并且密钥材料不会暴露给 TPM 外部，这确保了其私钥无法被导出或复制，增强了安全性。
- Restricted（受限用途）：AK 只能用于对 TPM 生成的数据进行签名或解密，例如 PCR（平台配置寄存器）值。它不能用于任意数据的签名或加密，以防止其被滥用为通用密钥。
- 签名密钥，用于设备身份（DevID）：AK 的公钥证书可以作为设备的身份凭据，由制造商或受信任方签发。

*"The 802.1AR standard defines a secure device identifier (DevID) as “an identifier that is cryptographically bound to a device”."*

设备制造商在制造阶段创建的设备身份称为 IDevID/IAK，IDevID “Credential” 在设备整个生命周期内使用。用户在部署/应用阶段创建的设备身份称为 LDevID/LAK，LDevID “Credential” 会在设备复位或寿命结束时被移除。

*"Credential is a combination of a private key and a certificate."*

Attestation Identity Key（AIK）是 TPM 1.2 中的一种密钥类型，AIK 仅被允许执行两种基于签名的操作：
- TPM_Quote：使用 AIK 对 PCR 寄存器值进行签名，生成代表设备身份和当前状态的硬件报告，常用于远程证明场景，也就是向远端证明设备当前的可信状态。
- TPM_CertifyKey：生成一个签名声明，用以证明另一个密钥（不是 AIK 本身）位于 TPM 的密钥存储体系中，且不可迁移（non-migratable）。显然，所认证的那个密钥必须实际具备这些属性。

## Credential Activation

Credential activation 能够远程地建立对新密钥（例如，IAK）的信任，该密钥随后可以代表设备进行加密或证明，需要证明如下两点：
- IAK（Identity Attestation Key）具备预期的密钥属性，例如：

   ```c
   FlagFixedTPM             // 密钥不能被导出，必须留在 TPM 中；
   FlagSensitiveDataOrigin  // 密钥必须由 TPM 内部生成，而不是导入；
   ```

- IAK 确实与 EK 存在于相同的 TPM 中。

*Credential activation allows a remote party to verify one key is on the same TPM as another key, and that the key has a specific set of properties.*

大概的流程如下：Activation key 代指 IAK，Anchor key 代指 EK。

{% include figure.liquid path="assets/img/2025-07-24-tpm-based-security/credential_activation.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. The Flow of Credential Activation
</div>

- Client 将 IAK 的公钥、EK 的公钥和 IAK 的密钥属性发送给 Server；
- Server 验证 IAK 的属性是否满足安全策略、生成秘密值并构造 Challenge，形式如下，然后返回给 Client：

     ```
     Challenge = ENC{secret} | aes_key1
                 ENC{key1}  | aes_key2
                 ENC{seed}  | EK_pub
     ```

     其中，`aes_key1` 是随机生成的对称密钥；`aes_key2 = KDF(seed, IAK name)`：KDF 是密钥派生函数，IAK name 是 IAK 公钥 Blob（包含属性）的哈希；所有敏感数据都用 EK 公钥加密，确保只能由目标 TPM 解密。

- Client 将 Challenge 发送给 TPM，调用 `ActivateCredential` 命令；
- TPM 内部只有在拥有与 Challenge 匹配的 EK 和 IAK 的前提下，才能正确解密并返回秘密值。成功解密即意味着：IAK 存在于该 TPM 中；IAK 拥有正确的密钥属性。


## Reference

1. [https://ericchiang.github.io/post/tpm-keys/]()
2. [https://tpm2-software.github.io/tpm2-tss/getting-started/2019/12/18/Remote-Attestation.html]()
3. [https://github.com/google/go-attestation/blob/master/docs/credential-activation.md]()
4. [https://github.com/tpm2dev/tpm.dev.tutorials]()
