# Encrypt v1.0.0

**Encrypt** is an offline, zero-network authenticated encryption tool built with Flutter for Android and Windows. It enables two parties to exchange confidential, tamper-evident messages over untrusted communication channels using a pre-shared passphrase.

**Cryptographic algorithms:** AES-256-GCM and PBKDF2-HMAC-SHA256.

> [!WARNING]
> ENCv1 uses a static salt and only 10,000 PBKDF2 iterations. Weak passphrases may be vulnerable to offline dictionary attacks. Do not use this configuration to protect high-value secrets.

---

## Table of Contents

- [Features](#features)
- [Downloads](#downloads)
- [How It Works](#how-it-works)
- [Cryptographic Specification](#cryptographic-specification)
- [Message Format](#message-format)
- [Encryption and Decryption](#encryption-and-decryption)
- [Security Model](#security-model)
- [Limitations](#limitations)
- [Compatibility](#compatibility)
- [License](#license)

---

## Features

- **Authenticated encryption:** AES-256-GCM provides confidentiality and integrity.
- **Password-based key derivation:** PBKDF2-HMAC-SHA256 derives a 256-bit encryption key.
- **Cross-platform support:** Available for Android and Windows.
- **Offline operation:** Designed to process messages locally without a remote service.
- **Portable messages:** Base64-encoded ciphertext can be shared through SMS, email, or messaging applications.
- **Tamper detection:** Modified or corrupted messages fail authentication.
- **Versioned format:** The `ENCv1:` prefix identifies the message format.
- **No server dependency:** Encryption and decryption do not require a remote service.

## Downloads

Download the standalone release binaries from the [GitHub Releases](../../releases) page.

| Platform | Architecture | Release file |
|---|---|---|
| Android | ARM64 (`arm64-v8a`) | `app-arm64-v8a-release.apk` |
| Android | ARM32 (`armeabi-v7a`) | `app-armeabi-v7a-release.apk` |
| Windows | x64 (64-bit) | `Encrypt_Setup_v1.0.0.exe` |

Windows builds target Windows 10 and Windows 11. ARM64 is recommended for most modern Android smartphones; the ARM32 build is for compatible 32-bit devices.

---

## How It Works

Encrypt uses symmetric encryption. Both participants must know the same passphrase.

```text
SENDER
  |
  v
Plaintext + Passphrase
  |
  v
PBKDF2-HMAC-SHA256
  |
  v
256-bit Encryption Key
  |
  v
AES-256-GCM
  |
  v
Nonce + Ciphertext + Authentication Tag
  |
  v
Base64 Encoding
  |
  v
ENCv1:<Base64Payload>
  |
  v
Untrusted Communication Channel
  |
  v
RECEIVER
  |
  v
Decode + Derive Key + Verify Tag
  |
  +---- Valid ----> Return Plaintext
  |
  +---- Invalid --> Reject Message
```

The receiver derives the same encryption key from the passphrase and verifies the authentication tag before accepting the plaintext.

**Important:** The passphrase is not included in the encrypted message. It must be shared separately through a trusted channel.

---

## Cryptographic Specification

### Encryption Algorithm

| Parameter | ENCv1 value |
|---|---|
| Algorithm | AES-256-GCM |
| Key length | 256 bits (32 bytes) |
| Nonce length | 96 bits (12 bytes) |
| Authentication tag | 128 bits (16 bytes) |
| Padding | Not required |

### Key Derivation

| Parameter | ENCv1 value |
|---|---|
| Algorithm | PBKDF2-HMAC-SHA256 |
| Passphrase | User-provided secret |
| Salt | `CryptoApp_StaticSalt_v1` |
| Salt encoding | UTF-8 |
| Iteration count | 10,000 |
| Derived-key length | 32 bytes |

### Message Encoding

| Parameter | ENCv1 value |
|---|---|
| Prefix | `ENCv1:` |
| Binary layout | Nonce + Ciphertext + Authentication Tag |
| Text encoding | Base64 |

These parameters define the ENCv1 format and must remain consistent across compatible implementations.

### 1. Key Derivation — PBKDF2-HMAC-SHA256

PBKDF2 transforms a passphrase into a fixed-length cryptographic key.

Let:

- \(P\) = passphrase bytes.
- \(S\) = UTF-8 bytes of `CryptoApp_StaticSalt_v1`.
- \(c = 10{,}000\) = iteration count.
- \(dkLen = 32\) bytes = derived-key length.
- \(hLen = 32\) bytes = SHA-256 output length.

The derived key is:

\[
DK = \operatorname{PBKDF2\text{-}HMAC\text{-}SHA256}(P,S,c,dkLen)
\]

Since \(dkLen = hLen\), only one output block is required:

\[
DK = T_1 = \bigoplus_{i=1}^{c} U_i
\]

where:

\[
U_1 = \operatorname{HMAC\text{-}SHA256}
(P,S \parallel \operatorname{INT}_{32BE}(1))
\]

and:

\[
U_i = \operatorname{HMAC\text{-}SHA256}(P,U_{i-1})
\]

for \(2 \le i \le c\).

The result is a 32-byte key suitable for AES-256.

**Security consideration:** ENCv1 uses a static salt rather than a unique, randomly generated salt for each message. This permits attackers to reuse password-guessing work across messages. The low iteration count also makes weak passphrases easier to crack.

### 2. Authenticated Encryption — AES-256-GCM

AES-GCM combines counter-mode encryption with authentication using GHASH over the finite field \(GF(2^{128})\).

For encryption key \(K\), nonce \(N\), and plaintext \(P\):

\[
(C,T) = \operatorname{AES\text{-}256\text{-}GCM}(K,N,P)
\]

where:

- \(C\) = ciphertext.
- \(T\) = 16-byte authentication tag.
- \(N\) = 12-byte nonce.

#### A. Confidentiality

GCM uses AES counter mode to encrypt plaintext. The ciphertext length equals the plaintext length, so PKCS#7 padding is unnecessary.

#### B. Integrity and Authentication

GCM authenticates the encrypted data using GHASH and generates an authentication tag.

During decryption, the tag must be verified before plaintext is released. If verification fails, the message must be rejected.

The authentication tag demonstrates knowledge of the shared key, subject to the cryptographic assumptions. It does not independently establish the sender's identity. Anyone who knows the passphrase can generate valid messages.

#### C. Nonce Requirements

A nonce must be unique for every encryption under the same key.

ENCv1 uses a randomly generated 96-bit nonce. Random generation makes collisions unlikely for small message volumes, but does not guarantee uniqueness.

Applications encrypting large numbers of messages under a single key require a suitable nonce-management strategy.

---

## Message Format

Every encrypted message follows this format:

```text
ENCv1:<Base64Payload>
```

The Base64 payload decodes into three consecutive fields:

```text
+------------------+------------------+------------------+
| Nonce            | Ciphertext       | Authentication   |
| 12 bytes         | N bytes          | Tag: 16 bytes    |
+------------------+------------------+------------------+
```

### Binary Layout

- **Bytes 0–11:** Nonce.
- **Bytes 12 to 12 + N - 1:** Ciphertext.
- **Final 16 bytes:** Authentication tag.

The total decoded payload length is:

\[
L = 12 + N + 16 = 28 + N
\]

where \(N\) is the ciphertext length in bytes.

### Payload Validation

A conforming implementation should:

1. Verify the `ENCv1:` prefix.
2. Decode the Base64 payload.
3. Reject malformed Base64 or payloads shorter than 28 bytes.
4. Extract the nonce, ciphertext, and authentication tag.
5. Derive the key using the ENCv1 parameters.
6. Perform authenticated decryption.
7. Return plaintext only after successful authentication.

Empty plaintext is valid at the cryptographic level and produces a 28-byte binary payload.

Implementations should enforce reasonable maximum message sizes to prevent excessive memory consumption.

---

## Encryption and Decryption

### Encryption Process

1. Accept the plaintext and passphrase.
2. Encode the passphrase according to the established ENCv1 encoding rules.
3. Derive a 256-bit key using PBKDF2-HMAC-SHA256.
4. Generate a cryptographically secure 12-byte nonce.
5. Encrypt the plaintext using AES-256-GCM with a 128-bit authentication tag.
6. Serialize the nonce, ciphertext, and authentication tag in the specified order.
7. Encode the binary payload using Base64.
8. Prefix the result with `ENCv1:`.

### Decryption Process

1. Accept the encrypted message and passphrase.
2. Validate the version prefix and Base64 encoding.
3. Decode and validate the binary payload.
4. Extract the nonce, ciphertext, and authentication tag.
5. Derive the key using the same ENCv1 parameters.
6. Decrypt and verify the authentication tag.
7. Return plaintext only if authentication succeeds.

Incorrect passphrases, corrupted messages, and invalid authentication tags should produce a decryption failure without exposing sensitive diagnostic information.

---

## Security Model

Encrypt is designed to protect message contents when encrypted messages travel through untrusted communication channels.

### Security Properties

| Property | ENCv1 behavior |
|---|---|
| Confidentiality | Provided by AES-256-GCM, subject to key security and correct implementation |
| Ciphertext integrity | Provided by GCM authentication |
| Key-possession verification | A valid tag demonstrates knowledge of the shared key, subject to cryptographic assumptions |
| Sender identity | Not independently established |
| Replay prevention | Not provided |
| Forward secrecy | Not provided |
| Metadata confidentiality | Not provided |
| Compromised-device protection | Not provided |

### 1. Shared-Passphrase Model

Both participants must possess the same passphrase.

If an attacker obtains the passphrase, they can decrypt captured messages and generate new authenticated messages.

Use strong, unique passphrases and exchange them through a trusted channel. Sharing the passphrase alongside the ciphertext through the same compromised channel defeats the intended protection.

### 2. Static Salt

ENCv1 uses the application-wide static salt:

```text
CryptoApp_StaticSalt_v1
```

A static salt allows password-guessing work to be reused across messages encrypted using the same derivation parameters.

A unique salt per message would prevent this reuse, although it would not eliminate offline password guessing.

### 3. PBKDF2 Work Factor

ENCv1 uses 10,000 PBKDF2-HMAC-SHA256 iterations.

This iteration count provides limited resistance to offline dictionary attacks, particularly when passphrases are short, common, or predictable.

AES-256 does not compensate for a weak passphrase. Overall security depends substantially on passphrase entropy and the cost of deriving each candidate key.

### 4. Replay Attacks

An attacker can retransmit an unchanged, valid encrypted message even if they cannot read or modify it.

Applications that trigger actions based on received messages should implement authenticated sequence numbers, unique message identifiers, or another replay-prevention mechanism.

### 5. Local Device Exposure

Encrypt cannot guarantee confidentiality on a compromised device. Sensitive data may be exposed through:

- Malware or keyloggers.
- Clipboard history and synchronization.
- Screenshots or screen recording.
- Operating-system memory dumps.
- Debug logs and crash reports.
- Physical access to an unlocked device.

Avoid logging or unnecessarily persisting passphrases, derived keys, and plaintext.

---

## Limitations

- Requires an independently shared passphrase.
- Uses a static salt and a relatively low PBKDF2 iteration count.
- Does not independently verify sender identity.
- Does not provide replay protection or forward secrecy.
- Depends on correct cryptographic implementation and secure random-number generation.
- Cannot protect plaintext exposed on a compromised endpoint.
- Does not hide message length, timing, or communication patterns.
- Does not prevent an attacker with the passphrase from creating valid messages.

---

## Compatibility

ENCv1 defines a fixed cryptographic and serialization format.

A future version should use a distinct prefix, such as `ENCv2:`, with an explicitly documented cryptographic specification.

Potential improvements include:

- A cryptographically random, per-message salt.
- A stronger, benchmarked password-based key derivation configuration.
- Explicit passphrase encoding and normalization rules.
- Appropriate nonce-management safeguards.
- Authenticated protocol metadata and replay protection where required.
- Maximum payload sizes and uniform error handling.
- Cross-platform test vectors for Android and Windows.

Existing ENCv1 messages should remain decryptable for as long as backward compatibility is required.

New cryptographic parameters must never be applied to old messages implicitly. Each version should use its own clearly defined parsing, key derivation, and encryption rules.

---

## License

This project is licensed under the [MIT License](LICENSE).
