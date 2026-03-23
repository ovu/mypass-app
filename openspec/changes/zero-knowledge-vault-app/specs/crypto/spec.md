# Crypto Module Spec

## Overview

The crypto module (`core:crypto`) implements all client-side cryptographic operations. It is a pure Kotlin module with no Android framework dependencies (except `java.security.SecureRandom`), making it testable on JVM. The module exposes a `CryptoEngine` interface consumed by the auth and vault features.

## Dependencies

- **Argon2**: Signal's `org.signal:argon2` library (native JNI for Android, pure-Java fallback for tests)
- **HKDF**: BouncyCastle `HKDFBytesGenerator` or Tink `com.google.crypto.tink.subtle.Hkdf`
- **AES-256-GCM**: `javax.crypto.Cipher` (standard JCE, hardware-accelerated)
- **SecureRandom**: `java.security.SecureRandom` for nonces, salts, DEK, recovery tokens

---

## 1. Key Derivation

### 1.1 Argon2id

**Input:**
- `password: CharArray` — user's master password (or recovery token bytes)
- `salt: ByteArray` — 16 bytes, randomly generated at signup

**Parameters:**
- Algorithm: Argon2id (hybrid, resistant to both side-channel and GPU attacks)
- Memory cost: 64 MB (65536 KiB)
- Time cost (iterations): 3
- Parallelism: 4
- Output length: 32 bytes

**Output:**
- `master_secret: ByteArray` — 32-byte derived key

**Performance target:** 500ms-1000ms on a mid-range Android device (Snapdragon 6-series, 2023). On first app launch, the module runs a benchmark with a throwaway password and adjusts iterations if needed. The chosen iteration count is stored in `EncryptedSharedPreferences`.

**Implementation notes:**
- `password` is converted to UTF-8 bytes for Argon2 input, then the char array and byte array are both zeroed.
- The function is `suspend` and runs on `Dispatchers.Default` to avoid blocking the main thread.

### 1.2 HKDF-Expand

After Argon2id produces the master secret, HKDF-Expand derives two purpose-specific keys:

**auth_key:**
- `HKDF-Expand(prk=master_secret, info="mypass-auth-key".toByteArray(), length=32)`
- Sent to the server as the authentication credential (server bcrypt-hashes it).

**KEK (Key Encryption Key):**
- `HKDF-Expand(prk=master_secret, info="mypass-encryption-key".toByteArray(), length=32)`
- Used locally to wrap/unwrap the DEK. Never leaves the device.

**Post-derivation:**
- `master_secret` is zeroed immediately after both HKDF-Expand calls complete.

### 1.3 Data Types

```kotlin
data class DerivedKeys(
    val authKey: ByteArray,   // 32 bytes — send to server
    val kek: ByteArray        // 32 bytes — local only
) {
    fun zeroize() {
        authKey.fill(0)
        kek.fill(0)
    }
}
```

---

## 2. DEK Generation

**When:** Signup only (one DEK per user account, for the lifetime of the account).

**How:**
```kotlin
fun generateDek(): ByteArray {
    val dek = ByteArray(32)
    SecureRandom().nextBytes(dek)
    return dek
}
```

**Properties:**
- 256-bit random key for AES-256-GCM.
- Generated once. Survives password changes (only the wrapping changes, not the DEK itself).
- The same DEK encrypts all vault entries for a given user.

---

## 3. Vault Entry Encryption

### 3.1 Encrypt

**Input:**
- `plaintext: ByteArray` — serialized vault entry (JSON bytes)
- `dek: ByteArray` — 32-byte DEK

**Process:**
1. Generate 12-byte random nonce via `SecureRandom`.
2. Initialize `Cipher` with `AES/GCM/NoPadding`, `GCMParameterSpec(128, nonce)`.
3. Encrypt plaintext → ciphertext + 16-byte auth tag (appended by JCE).
4. Return `EncryptedBlob(nonce || ciphertext_with_tag)`.

**Output format:**
```
[ nonce: 12 bytes ][ ciphertext + GCM tag: variable + 16 bytes ]
```

### 3.2 Decrypt

**Input:**
- `blob: ByteArray` — the encrypted blob (nonce || ciphertext_with_tag)
- `dek: ByteArray` — 32-byte DEK

**Process:**
1. Extract first 12 bytes as nonce.
2. Remaining bytes are ciphertext_with_tag.
3. Initialize `Cipher` in decrypt mode with `GCMParameterSpec(128, nonce)`.
4. Decrypt and verify authentication tag.
5. If tag verification fails, throw `CryptoException.IntegrityError`.

### 3.3 EncryptedBlob Type

```kotlin
@JvmInline
value class EncryptedBlob(val data: ByteArray) {
    val nonce: ByteArray get() = data.copyOfRange(0, 12)
    val ciphertext: ByteArray get() = data.copyOfRange(12, data.size)
}
```

### 3.4 Nonce Uniqueness

Each encryption call generates a fresh random 12-byte nonce. With 2^96 possible values and the expected number of vault entries per user being <10,000, the collision probability is negligible (far below 2^-32). No nonce counter or nonce tracking is needed.

---

## 4. DEK Wrapping

### 4.1 Wrap (Encrypt DEK with KEK)

**Input:**
- `dek: ByteArray` — 32-byte DEK
- `kek: ByteArray` — 32-byte KEK (derived from master password or recovery token)

**Process:** Same as vault entry encryption: AES-256-GCM with a random 12-byte nonce.

**Output:** `wrapped_dek: ByteArray` — `nonce (12) || encrypted_dek (32) || tag (16)` = 60 bytes total.

### 4.2 Unwrap (Decrypt DEK with KEK)

**Input:**
- `wrapped_dek: ByteArray` — 60 bytes
- `kek: ByteArray` — 32-byte KEK

**Process:** AES-256-GCM decrypt. If the tag check fails (wrong password), throw `CryptoException.WrongPassword`.

### 4.3 Error Handling

The unwrap operation is the primary place where a wrong master password is detected. If `AEADBadTagException` is thrown by JCE, it means the KEK is wrong (derived from a wrong password). This is mapped to a user-facing "Incorrect password" error.

---

## 5. Recovery Token

### 5.1 Generation

**When:** At signup, and when the user explicitly requests a new recovery token from settings.

**Process:**
1. Generate 128 bits (16 bytes) via `SecureRandom`.
2. Base32-encode (RFC 4648, no padding, uppercase alphabet: `A-Z2-7`).
3. Format for display: insert hyphens every 4 characters → `XXXX-XXXX-XXXX-XXXX`.

```kotlin
data class RecoveryToken(
    val rawBytes: ByteArray,       // 16 bytes — used for key derivation
    val displayString: String      // "ABCD-EFGH-JKLM-NPQR"
) {
    fun zeroize() { rawBytes.fill(0) }
}
```

### 5.2 Recovery Key Derivation

The recovery token is used exactly like a master password:
1. Generate a fresh 16-byte `salt_recovery`.
2. `recovery_master = Argon2id(token_bytes, salt_recovery)` — same params as master password.
3. `recovery_auth_key = HKDF-Expand(recovery_master, info="mypass-auth-key", 32)` — stored on server for recovery authentication.
4. `recovery_kek = HKDF-Expand(recovery_master, info="mypass-encryption-key", 32)`.
5. `encrypted_dek_recovery = AES-256-GCM(dek, recovery_kek)`.

### 5.3 Recovery DEK Wrapping

Same wrapping scheme as primary DEK wrapping (section 4.1), but using the recovery-derived KEK.

The server stores:
- `salt_recovery` — needed to re-derive keys during recovery
- `bcrypt(recovery_auth_key)` — to authenticate the recovery request (same pattern as master password auth)
- `encrypted_dek_recovery` — DEK wrapped with recovery KEK

The recovery token itself is **never** stored on client or server.

### 5.4 Recovery Token Management

Two operations are supported:

**Replace (revoke old + create new):**
- Generates new `salt_recovery`, `recovery_auth_key`, `encrypted_dek_recovery`
- Server deletes ALL existing recovery DEK rows for this user and inserts the new one
- Use case: user lost their recovery token and wants a fresh one
- Old token becomes permanently useless (its `auth_hash` and `encrypted_dek` are deleted)

**Add additional:**
- Generates new `salt_recovery_N`, `recovery_auth_key_N`, `encrypted_dek_recovery_N`
- Server inserts alongside existing recovery DEK rows
- Use case: user wants backup tokens in multiple locations (e.g., safe + spouse)
- All existing tokens remain valid

Each recovery token is independent — each has its own salt, auth hash, and wrapped DEK. During recovery, the server tries `bcrypt.verify` against each recovery DEK row's `auth_hash` until one matches (or all fail).

### 5.5 Token Lifecycle

1. Token is generated and displayed to the user.
2. User acknowledges they have written it down (checkbox + confirm button).
3. `RecoveryToken.zeroize()` is called.
4. The token is never stored on the device or server.

---

## 6. Memory Zeroing

### 6.1 Policy

Every sensitive byte array and char array MUST be zeroed after use. "After use" means as soon as the value is no longer needed in the current operation.

### 6.2 Sensitive Values and Lifetimes

| Value | Lifetime |
|-------|----------|
| `password: CharArray` | Zeroed after Argon2id returns |
| `master_secret` | Zeroed after HKDF-Expand produces both subkeys |
| `auth_key` | Zeroed after being sent to the server |
| `kek` | Zeroed after DEK unwrap completes |
| `dek` | Held in `SessionManager` while session is active; zeroed on session clear |
| `recovery_token.rawBytes` | Zeroed after user acknowledges the display |
| `recovery_kek` | Zeroed after recovery DEK wrapping completes |

### 6.3 Implementation Pattern

```kotlin
suspend fun deriveAndUnwrap(password: CharArray, salt: ByteArray, wrappedDek: ByteArray): ByteArray {
    var masterSecret: ByteArray? = null
    var keys: DerivedKeys? = null
    try {
        masterSecret = argon2id(password, salt)
        password.fill('\u0000')

        keys = hkdfExpand(masterSecret)
        masterSecret.fill(0)
        masterSecret = null

        val dek = unwrapDek(wrappedDek, keys.kek)
        return dek
    } finally {
        password.fill('\u0000')
        masterSecret?.fill(0)
        keys?.zeroize()
    }
}
```

The `finally` block guarantees zeroing even if an exception occurs mid-flow.

### 6.4 Limitations

- JVM garbage collection may copy byte arrays during compaction. There is no way to prevent this on standard Android. This is an accepted risk, mitigated by prompt zeroing to minimize the window.
- `String` objects are never used for passwords or keys (they are immutable and cannot be zeroed).

---

## 7. Error Types

```kotlin
sealed class CryptoException : Exception() {
    object WrongPassword : CryptoException()     // AEADBadTagException during DEK unwrap
    object IntegrityError : CryptoException()     // AEADBadTagException during vault entry decrypt
    object InvalidRecoveryToken : CryptoException()
    data class Argon2Failure(override val cause: Throwable) : CryptoException()
    data class Unexpected(override val cause: Throwable) : CryptoException()
}
```

---

## 8. Testing Strategy

- **Unit tests (JVM):** Test encrypt/decrypt roundtrips, key derivation determinism (with fixed salt/password), nonce uniqueness, wrapping/unwrapping, error cases (wrong key).
- **Integration tests (instrumented):** Test Argon2id performance on real device, SecureRandom behavior.
- **Property-based tests:** For any `plaintext`, `encrypt(plaintext, dek)` followed by `decrypt(result, dek)` returns original plaintext. Decrypting with a different DEK always fails.
- **Known-answer tests:** Use test vectors from Argon2 RFC (RFC 9106) and HKDF RFC (RFC 5869) to verify implementation correctness.
