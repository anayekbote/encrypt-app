Encrypt v1.0.0

Encrypt is an offline, zero-network authenticated encryption application built with Flutter for Android and Windows. It enables two parties to exchange confidential, tamper-evident messages over untrusted communication channels using a pre-shared passphrase.

The application uses AES-256-GCM for authenticated encryption and PBKDF2-HMAC-SHA256 for deriving a 256-bit encryption key from the passphrase.

«[!WARNING]
ENCv1 is a legacy format with a static salt and only 10,000 PBKDF2 iterations. Weak passphrases may be vulnerable to offline dictionary attacks. Do not use this configuration to protect high-value secrets.»

---

Table of Contents

- "Features" (#features)
- "Downloads" (#downloads)
- "How It Works" (#how-it-works)
- "Cryptographic Specification" (#cryptographic-specification)
- "Message Format" (#message-format)
- "Encryption and Decryption" (#encryption-and-decryption)
- "Security Model" (#security-model)
- "Limitations" (#limitations)
- "Compatibility" (#compatibility)
- "License" (#license)

Features

- Authenticated encryption: AES-256-GCM provides confidentiality and integrity.
- Password-based key derivation: PBKDF2-HMAC-SHA256 derives a 256-bit key.
- Cross-platform support: Android and Windows.
- Offline operation: Designed to encrypt and decrypt messages locally.
- Portable messages: Base64-encoded payloads can be shared through SMS, email, or messaging applications.
- Tamper detection: Invalid authentication tags cause decryption to fail.
- Versioned format: The "ENCv1:" prefix identifies messages encrypted using version 1.
- No server dependency: Message encryption and decryption do not require a remote service.

Downloads

Download precompiled binaries from the "GitHub Releases" (../../releases) page.

Platform| Architecture| Release file
Android| ARM64 ("arm64-v8a")| "app-arm64-v8a-release.apk"
Android| ARM32 ("armeabi-v7a")| "app-armeabi-v7a-release.apk"
Windows| x64| "Encrypt_Setup_v1.0.0.exe"

System Requirements

- Android: Compatible ARM-based Android device.
- Windows: 64-bit Windows 10 or Windows 11.
- Communication: Any channel capable of transmitting the encrypted ASCII message.

Use only release binaries obtained from a trusted source.

How It Works

Encrypt uses a symmetric cryptographic model: both parties must know the same passphrase.

                    SENDER
                       |
             Plaintext + Passphrase
                       |
                       v
              PBKDF2-HMAC-SHA256
                       |
                       v
                 256-bit Key
                       |
                       v
                 AES-256-GCM
                       |
                       v
           Nonce + Ciphertext + Tag
                       |
                       v
                Base64 Encoding
                       |
                       v
             ENCv1:<Base64Payload>
                       |
                       v
             Untrusted Communication
                       |
                       v
                    RECEIVER
                       |
              Decode and Validate
                       |
                       v
             Derive the Same Key
                       |
                       v
               Verify GCM Tag
                       |
             +---------+---------+
             |                   |
          Valid Tag           Invalid Tag
             |                   |
             v                   v
       Return Plaintext       Reject Message

The receiver derives the same key from the passphrase and the version-specific salt. AES-GCM verifies the authentication tag before accepting the decrypted plaintext.

Important: The encrypted message does not contain the passphrase. The parties must establish it separately through a secure channel.

Cryptographic Specification

Parameter| ENCv1 specification
Encryption algorithm| AES-256-GCM
Key length| 256 bits (32 bytes)
Key derivation| PBKDF2-HMAC-SHA256
Salt| Static UTF-8 string
Salt value| "CryptoApp_StaticSalt_v1"
Iteration count| 10,000
Derived-key length| 32 bytes
Nonce / IV| 96 bits (12 bytes)
Authentication tag| 128 bits (16 bytes)
Binary payload| Nonce + ciphertext + tag
Text encoding| Base64
Message prefix| "ENCv1:"
Padding| None required by GCM

These parameters describe the ENCv1 format. They should not be silently changed in a way that breaks compatibility with previously encrypted messages.

1. Key Derivation: PBKDF2-HMAC-SHA256

PBKDF2 transforms the passphrase into a fixed-length encryption key.

Let:

- (P) = passphrase bytes, using the application's existing encoding rules.
- (S) = UTF-8 bytes of "CryptoApp_StaticSalt_v1".
- (c = 10{,}000) = iteration count.
- (dkLen = 32) bytes = derived-key length.
- (hLen = 32) bytes = SHA-256 output length.

The derived key is:

[
DK=\operatorname{PBKDF2}(P,S,c,dkLen)
]

Because (dkLen=hLen), only one PBKDF2 output block is required:

[
DK=T_1=U_1\oplus U_2\oplus\cdots\oplus U_c
]

where:

[
U_1=\operatorname{HMAC\text{-}SHA256}
(P,S\parallel\operatorname{INT}_{32BE}(1))
]

and:

[
U_i=\operatorname{HMAC\text{-}SHA256}(P,U_{i-1})
]

for (2\le i\le c).

The result is a 32-byte key suitable for AES-256.

Security consideration: The salt is static rather than randomly generated per message. It therefore does not prevent attackers from reusing password-guessing computations across messages. The iteration count is also low by modern password-storage standards.

2. Authenticated Encryption: AES-256-GCM

AES-GCM combines counter-mode encryption with an authentication mechanism based on GHASH over (GF(2^{128})).

For a key (K), nonce (N), and plaintext (P):

[
(C,T)=\operatorname{AES\text{-}256\text{-}GCM}(K,N,P)
]

where:

- (C) is the ciphertext.
- (T) is the 16-byte authentication tag.
- (N) is a 12-byte nonce generated for the encryption operation.

Confidentiality

GCM uses AES counter mode to encrypt the plaintext. The ciphertext length equals the plaintext length, so no PKCS#7 padding is required.

Integrity and Authentication

GCM authenticates the encrypted data using GHASH and a tag derived from the encryption key and nonce.

During decryption, the authentication tag must be verified. If verification fails, the message must be rejected without exposing the candidate plaintext to the caller.

The authentication tag detects unauthorized modifications. It does not establish the sender's real-world identity: anyone possessing the shared passphrase can create valid messages.

Nonce Requirements

The nonce must be unique for each encryption under a given key. ENCv1 uses a randomly generated 96-bit nonce.

Random generation makes collisions unlikely for small message volumes, but does not guarantee uniqueness. High-volume applications sharing one key need a carefully designed nonce-management strategy.

Message Format

Every ENCv1 message is represented as an ASCII string:

ENCv1:<Base64Payload>

The Base64 payload decodes to the following binary structure:

Offset  Length           Field
------  ---------------  ----------------
0       12 bytes         Nonce
12      N bytes          Ciphertext
12 + N  16 bytes         Authentication Tag

Equivalently:

+------------------+-------------------+------------------+
| Nonce            | Ciphertext        | Authentication   |
| 12 bytes         | N bytes           | Tag (16 bytes)   |
+------------------+-------------------+------------------+

The total decoded payload size is:

[
L=12+N+16=28+N
]

where (N) is the ciphertext length in bytes.

Payload Validation

A conforming implementation should:

1. Check the "ENCv1:" prefix.
2. Decode the Base64 payload.
3. Reject malformed Base64 and payloads shorter than 28 bytes.
4. Extract the 12-byte nonce, ciphertext, and 16-byte authentication tag.
5. Derive the encryption key using the ENCv1 parameters.
6. Perform authenticated decryption.
7. Return plaintext only after successful authentication.

Empty plaintext is valid at the cryptographic level and produces a payload containing a nonce and authentication tag.

Implementations should also enforce sensible maximum message sizes to prevent excessive memory consumption.

Encryption and Decryption

Encryption

1. Accept the plaintext and passphrase.
2. Convert the passphrase according to the established ENCv1 encoding rules.
3. Derive a 256-bit key using PBKDF2-HMAC-SHA256.
4. Generate a cryptographically secure 12-byte nonce.
5. Encrypt the plaintext using AES-256-GCM with a 128-bit tag.
6. Serialize the nonce, ciphertext, and tag in the specified order.
7. Encode the binary payload as Base64.
8. Prefix the result with "ENCv1:".

Decryption

1. Accept the encrypted message and passphrase.
2. Validate the version prefix and Base64 encoding.
3. Decode and validate the binary payload.
4. Extract the nonce, ciphertext, and authentication tag.
5. Derive the key using the same ENCv1 parameters.
6. Decrypt and verify the authentication tag.
7. Return the plaintext only if authentication succeeds.

Incorrect passphrases, corrupted messages, and modified authentication tags should produce a generic decryption failure rather than exposing sensitive diagnostic details.

Security Model

Encrypt is designed to protect message contents when encrypted messages are transmitted over untrusted channels.

Security Properties

Property| ENCv1 behavior
Confidentiality| Provided by AES-256-GCM, subject to key security
Ciphertext integrity| Provided by GCM authentication
Authentication of key possession| Successful verification demonstrates knowledge of the shared key, subject to cryptographic assumptions
Sender identity| Not independently established
Replay prevention| Not provided
Forward secrecy| Not provided
Metadata confidentiality| Not provided
Protection against compromised endpoints| Not provided

Shared-Passphrase Model

Both participants must possess the same passphrase. If an attacker obtains it, the attacker can decrypt captured messages and generate new authenticated messages.

Passphrases should be strong, unique, and exchanged through a trusted channel. A passphrase transmitted alongside the ciphertext through the same compromised channel provides little protection.

Offline Password Attacks

An attacker who captures an encrypted payload can attempt to guess the passphrase offline. The static salt and 10,000 PBKDF2 iterations make weak or predictable passphrases particularly vulnerable.

AES-256 does not protect against guesses of the passphrase itself. Security depends substantially on passphrase entropy and the cost of key derivation.

Replay Attacks

An attacker cannot ordinarily modify a valid ciphertext without causing authentication failure, but can retransmit an unchanged valid message.

Applications that use messages to trigger actions should implement authenticated sequence numbers, unique message identifiers, or another replay-prevention mechanism.

Local Device Exposure

The application cannot guarantee confidentiality on a compromised device. Plaintext, passphrases, and derived keys may be exposed through:

- Malware or keyloggers.
- Clipboard history and synchronization.
- Screenshots or screen recording.
- Operating-system memory dumps.
- Debug logs or crash reports.
- Physical access to an unlocked device.

Sensitive values should not be written to logs or persisted unnecessarily.

Limitations

- Requires an independently shared passphrase.
- Uses a static salt and a relatively low PBKDF2 iteration count.
- Does not establish an independently verifiable sender identity.
- Does not provide replay protection or forward secrecy.
- Depends on correct cryptographic implementation and secure random-number generation.
- Cannot protect data after plaintext is exposed on a compromised endpoint.
- Does not hide message length, timing, or communication patterns.

Compatibility and Future Versions

ENCv1 defines a fixed cryptographic and serialization format. A future version should use a distinct prefix, such as "ENCv2:", with an explicitly documented derivation and encryption specification.

A newer format should consider:

- A cryptographically random, per-message salt.
- A stronger, benchmarked password-based key derivation configuration.
- Strict nonce management.
- Explicit encoding and normalization rules for passphrases.
- Optional authenticated protocol metadata and replay protection where required.
- Maximum payload sizes and uniform error handling.
- Test vectors for interoperability between Android and Windows.

Existing ENCv1 messages should remain decryptable for as long as backward compatibility is required. Newer parameters must not be applied to old messages implicitly.

License

This project is licensed under the "MIT License" (LICENSE).