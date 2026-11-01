# Encrypt (v2.0.0)

Encrypt is an offline, air-gapped public-key cryptographic tool for Android and Windows. Built on the **IETF RFC 9180 (Hybrid Public Key Encryption / HPKE)** specification, Encrypt eliminates shared passwords by providing asynchronous authenticated encryption using **Curve25519 (X25519)**, **HKDF-SHA256**, and **AES-256-GCM**.

Private keys are protected at rest using **Argon2id**-hardened vaults, and volatile heap memory is governed by a 15-minute auto-locking timer.

---

## Downloads

Download standalone binaries directly from the [Releases](../../releases) section:

| Platform | Target Architecture | File Name | Description |
| :--- | :--- | :--- | :--- |
| **Android** | ARM 64-bit (`arm64-v8a`) | `app-arm64-v8a-release.apk` | Recommended for all modern 64-bit Android smartphones |
| **Android** | ARM 32-bit (`armeabi-v7a`) | `app-armeabi-v7a-release.apk` | For legacy or budget 32-bit Android devices |
| **Windows** | x64 (64-bit Intel/AMD) | `Encrypt_Setup_v2.0.0.exe` | Standard Windows 10/11 desktop installer |

---

## Core Features

- **Asynchronous Public-Key Encryption:** Encrypt messages using the recipient's public key alone. No pre-shared passwords or synchronous handshakes required.
- **IETF RFC 9180 HPKE Architecture:** Implements the `DHKEM(X25519, HKDF-SHA256)` Key Encapsulation Mechanism combined with `AES-256-GCM` AEAD.
- **Argon2id Vault Protection:** At-rest private keys are sealed with memory-hard Argon2id key derivation, resisting GPU, ASIC, and distributed dictionary attacks.
- **Volatile Memory Auto-Lock:** Unlocked private keys reside in volatile heap memory and are wiped automatically after 15 minutes of inactivity.
- **Air-Gapped Operation:** Operates with zero network permissions (`android.permission.INTERNET` is omitted), ensuring keys and plaintexts never touch a network socket.
- **Local Contact Management:** Store, label, inspect, copy, and delete recipient public keys with 8-byte SHA-256 identity fingerprints.

---

## Cryptographic Architecture & Flow

```text
SENDER (Alice)                                              RECEIVER (Bob)
[ Recipient PK (pkR) ]                                      [ Master Password ]
         │                                                           │
         ▼                                                           ▼
Generate Ephemeral Keypair (ekS, epkS)                      [ Argon2id KDF ]
         │                                                           │
         ▼                                                           ▼
ECDH: Shared Secret = X25519(ekS, pkR)                      Decrypt Vault -> Private Key (skR)
         │                                                           │
         ▼                                                           ▼
HKDF-SHA256: Extract & Expand                               Paste "ENCv2:<Base64Payload>"
         │                                                           │
         ├──> Derived Symmetric Key (K)                              ▼
         └──> Base Nonce (N)                                Extract epkS (Bytes 0..31)
         │                                                           │
         ▼                                                           ▼
AES-256-GCM Seal(K, N, Plaintext)                           ECDH: Shared Secret = X25519(skR, epkS)
         │                                                           │
         ▼                                                           ▼
Pack: epkS [32B] + Ciphertext [NB] + Tag [16B]              HKDF-SHA256: Derive identical (K, N)
         │                                                           │
         ▼                                                           ▼
Serialize to Base64 -> Prefix "ENCv2:"                      AES-256-GCM Open & Verify Tag
         │                                                           │
         ▼                                                           ▼
Send via SMS, WhatsApp, Email, or Web                       [ Verified Plaintext Output ]
```

---

## Mathematical Specification

### 1. Key Encapsulation Mechanism: DHKEM(X25519, HKDF-SHA256)
Diffie-Hellman operations are performed over Curve25519 in Montgomery form:

$$E: y^2 = x^3 + 486662x^2 + x \pmod{2^{255} - 19}$$

1. **Sender Ephemeral Generation:**
   A secure 256-bit scalar $ek_S \in_R \mathbb{Z}_q$ is sampled using host CSPRNG. The sender calculates the ephemeral public point:
   $$epk_S = ek_S \cdot G$$

2. **Diffie-Hellman Shared Secret:**
   Given the recipient's static public point $pk_R = sk_R \cdot G$:
   $$\text{SharedSecret} = \text{X25519}(ek_S, pk_R) = ek_S \cdot (sk_R \cdot G) = sk_R \cdot (ek_S \cdot G) = \text{X25519}(sk_R, epk_S)$$

### 2. Key Derivation: HKDF (RFC 5869)
The shared secret is converted into a 256-bit symmetric key $K$ and 96-bit base nonce $N$ using HMAC-SHA256:

1. **Extract Stage:**
   $$\text{PRK} = \text{HMAC-SHA256}(\text{Salt} = \text{"HPKE-v1"}, \text{SharedSecret})$$

2. **Expand Stage:**
   Given the protocol context string $I = \text{"HPKE-AES256GCM"}$:
   $$T(1) = \text{HMAC-SHA256}(\text{PRK}, I \parallel \text{0x01})$$
   $$T(2) = \text{HMAC-SHA256}(\text{PRK}, T(1) \parallel I \parallel \text{0x02})$$
   $$\text{OKM} = T(1) \parallel T(2)$$
   $$K = \text{OKM}[0 \dots 31] \quad (256\text{-bit AEAD Key})$$
   $$N = \text{OKM}[32 \dots 43] \quad (96\text{-bit Initialization Vector})$$

### 3. Payload Encryption: AES-256-GCM (NIST SP 800-38D)
Confidentiality is provided by AES in Counter (CTR) mode, and ciphertext authenticity is verified over Galois field $GF(2^{128})$:

- **Ciphertext Generation:**
  $$C_i = P_i \oplus \text{AES}_K(N \parallel \text{incr}_{32}(i))$$
- **Authentication Tag ($T$):**
  Evaluated over the field polynomial $f(x) = x^{128} + x^7 + x^2 + x + 1$ using GHASH:
  $$T = \text{GHASH}_H(C) \oplus \text{AES}_K(N \parallel 0^{31}1)$$

### 4. At-Rest Vault Key Protection: Argon2id
To securely store private key $sk_R$ in device preferences:

$$\text{VaultKey} = \text{Argon2id}(\text{Password}, \text{Salt}, t = 3, m = 65{,}536\text{ KiB}, p = 4, \text{keyLen} = 32)$$

The raw 32-byte private key is then encrypted via AES-256-GCM using $\text{VaultKey}$ and written to persistent storage alongside its 128-bit authentication tag and salt.

---

## Wire Format Specification

ENCv2 serializes all cryptographic components into a single ASCII string:

```text
ENCv2:<Base64( ) Auth Ciphertext Ephemeral Key Public Tag [16B] [32B] [NB] ||>
```

### Binary Frame Layout

```text
 0                            31 32                     N+31 N+32                 N+47
+-------------------------------+---------------------------+-------------------------+
|  Ephemeral PubKey (32 Bytes)  |    Ciphertext (N Bytes)   |   Auth Tag (16 Bytes)   |
+-------------------------------+---------------------------+-------------------------+
 \_____________________________/ \_________________________/ \_______________________/
    Sender's Ephemeral X25519         AES-CTR Stream Output      128-bit GHASH Tag
             Point
```

### Parsing & Verification Rules
- The binary frame must be **at least 48 bytes** in length ($32\text{ bytes Ephemeral PK} + 0\text{ bytes Ciphertext} + 16\text{ bytes Tag}$).
- Payloads missing the `ENCv2:` prefix or failing Base64 validation are rejected before invoking cryptographic primitives.
- If tag verification fails, decryption halts immediately; **no plaintext fragments are emitted to application memory**.

---

## Usage Guide

### 1. Set Up Your Identity
1. Launch Encrypt and tap **Set Up Your Identity**.
2. Enter a master password. The app executes Argon2id to derive a vault-encryption key and generates your persistent X25519 keypair.
3. Your **Public Key** and 8-byte **Fingerprint** are displayed. Tap the copy icon to share your public key with anyone who wants to send you messages.

### 2. Add Contacts
1. In the **Send To (Contact)** section, tap the **Add Contact** icon (`+`).
2. Enter a name/label and paste the recipient's Base64 public key.
3. Tap **Add**. The app validates the 32-byte key length and stores the contact locally.

### 3. Encrypt a Message
1. Select the intended recipient from the contact dropdown.
2. Enter your plaintext in the input box.
3. Tap **Encrypt**.
4. Tap the copy icon in the **Result** card and share the generated `ENCv2:...` payload over any platform (SMS, chat app, email).

### 4. Decrypt a Message
1. Tap **Unlock** and enter your master password (unlocked keys remain valid for 15 minutes before automatically locking).
2. Paste the `ENCv2:...` payload into the input box.
3. Tap **Decrypt**. The verified plaintext will appear in the **Result** card.

---

## Security Model & Limitations

While ENCv2 provides IND-CCA2 security against active channel adversaries, deployments must consider these explicit boundaries:

1. **Public-Key Authenticity (Trust-on-First-Use):**
   - Encrypt is intentionally decentralized and does not use centralized Public Key Infrastructure (PKI) or certificate authorities.
   - Users must verify contacts' 8-byte fingerprints out-of-band (e.g., in-person or over a voice call) to prevent adversary key-substitution (Man-in-the-Middle) attacks.

2. **Absence of Cryptographic Timestamps (Replay Attacks):**
   - `ENCv2:` payloads do not embed monotonic counters or timestamps.
   - An attacker can intercept a valid ciphertext and re-send it to the recipient at a later date. Applications requiring strict replay protection should include an application-level timestamp within the message text.

3. **Forward Secrecy vs. Recipient Compromise:**
   - **Sender Forward Secrecy:** Guaranteed. A fresh ephemeral keypair ($ek_S, epk_S$) is generated per message and destroyed immediately after encryption.
   - **Recipient Key Compromise:** If an attacker compromises the recipient's master password and local vault, all historical messages delivered to that recipient's static public key can be decrypted.

4. **Operating System & Clipboard Exposure:**
   - Decrypted plaintexts copied to the operating system clipboard can be read by other applications with clipboard access.
   - Devices infected with screen recorders, keyloggers, or hardware root compromises are outside the protection perimeter.

---

## License

This project is licensed under the [MIT License](LICENSE).
