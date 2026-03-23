# Zero-Knowledge Vault App — Technical Design

## 1. App Architecture

### 1.1 Pattern: MVVM + Unidirectional Data Flow

The app follows MVVM with Jetpack Compose for the UI layer. Each screen has a dedicated ViewModel that exposes a single `UiState` data class via `StateFlow`. User actions dispatch events to the ViewModel, which orchestrates domain logic and updates state. Compose observes state and re-renders.

```
View (Compose) → ViewModel → UseCase / Repository → Data Sources
     ↑                                                      |
     └────────── StateFlow<UiState> ←──────────────────────┘
```

### 1.2 Module Structure

```
:app                    — Application entry, navigation, DI wiring
:core:crypto            — Argon2id, HKDF, AES-256-GCM, key wrapping, memory zeroing
:core:keystore          — Android Keystore wrapper, biometric DEK caching
:core:network           — Ktor HTTP client, TLS pinning, auth interceptor
:core:storage           — EncryptedSharedPreferences for local settings
:core:common            — Shared models, error types, Result wrappers
:feature:auth           — Signup, login, password change, account recovery
:feature:vault          — Vault list, entry detail, add/edit, password generator
:feature:settings       — App settings, recovery token generation
```

### 1.3 Dependency Injection (Hilt)

Hilt provides DI scoping:

| Scope | What lives here |
|-------|----------------|
| `@Singleton` | Ktor `HttpClient`, `EncryptedSharedPreferences`, `CryptoEngine` |
| `@ActivityRetainedScoped` | `SessionManager` (holds DEK reference, session timer) |
| `@ViewModelScoped` | Individual ViewModels |

Key bindings:

- `CryptoEngine` — interface for all crypto operations, backed by `CryptoEngineImpl`
- `KeystoreManager` — wraps Android Keystore for biometric-gated DEK storage
- `VaultRepository` — coordinates network + crypto for vault CRUD
- `AuthRepository` — coordinates signup/login/recovery flows
- `SessionManager` — manages DEK lifecycle, background timer, biometric re-unlock

### 1.4 Navigation

Single-activity architecture with Compose Navigation. Navigation graph:

```
LoginScreen ←→ SignupScreen
     ↓
VaultListScreen → EntryDetailScreen → EditEntryScreen
     ↓                                      ↑
PasswordChangeScreen            AddEntryScreen
     ↓
RecoveryScreen
     ↓
SettingsScreen → GenerateRecoveryTokenScreen
```

A `NavGuard` composable checks `SessionManager.isUnlocked` on every navigation event. If the session has expired, it redirects to the biometric prompt or login screen.

---

## 2. Crypto Module Design

### 2.1 CryptoEngine Interface

```kotlin
interface CryptoEngine {
    suspend fun deriveKeys(password: CharArray, salt: ByteArray): DerivedKeys
    fun generateDek(): ByteArray
    fun encryptEntry(plaintext: ByteArray, dek: ByteArray): EncryptedBlob
    fun decryptEntry(blob: EncryptedBlob, dek: ByteArray): ByteArray
    fun wrapDek(dek: ByteArray, kek: ByteArray): ByteArray
    fun unwrapDek(wrappedDek: ByteArray, kek: ByteArray): ByteArray
    fun generateRecoveryToken(): RecoveryToken
    suspend fun deriveRecoveryKeys(token: ByteArray, salt: ByteArray): DerivedKeys
    fun generateSalt(): ByteArray
    fun zeroize(vararg arrays: ByteArray)
    fun zeroize(chars: CharArray)
}
```

### 2.2 Key Derivation

1. **Argon2id** — `Argon2id(password, salt)` produces a 32-byte master secret.
   - Parameters: memory=64MB, iterations=3, parallelism=4, output=32 bytes.
   - Library: Signal's `argon2` Android library (native JNI, well-audited).
2. **HKDF-Expand** — Derives two subkeys from the master secret:
   - `auth_key = HKDF-Expand(master, info="auth", length=32)`
   - `kek = HKDF-Expand(master, info="encrypt", length=32)`
   - Library: Tink's HKDF implementation or BouncyCastle `HKDFBytesGenerator`.
3. Master secret is zeroed immediately after HKDF derivation.

### 2.3 AES-256-GCM

- Each vault entry encrypted with a unique 12-byte random nonce.
- `EncryptedBlob` = `nonce (12 bytes) || ciphertext || tag (16 bytes)`.
- Library: `javax.crypto.Cipher` with `AES/GCM/NoPadding` (hardware-accelerated on most devices).
- Nonce generated via `SecureRandom`.

### 2.4 DEK Wrapping

- DEK wrapped with KEK using AES-256-GCM (same primitive, different key).
- Stored format: `nonce || wrapped_dek || tag`.
- Recovery wrapping uses the same scheme but with KEK derived from recovery token.

### 2.5 Memory Management

- All sensitive byte arrays (`master_secret`, `dek`, `kek`, `password`) are explicitly zeroed after use via `Arrays.fill(array, 0)`.
- `CharArray` for passwords (never `String`, which is immutable and lingers in heap).
- `CryptoEngine.zeroize()` is called in `finally` blocks to ensure cleanup even on exceptions.

---

## 3. Android Keystore Integration

### 3.1 Purpose

After the user logs in with their master password, the DEK is encrypted and stored in Android Keystore so that subsequent unlocks can use biometrics instead of re-running Argon2id.

### 3.2 Key Spec

```kotlin
KeyGenParameterSpec.Builder("mypass_dek_wrapper", PURPOSE_ENCRYPT or PURPOSE_DECRYPT)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setKeySize(256)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationParameters(0, AUTH_BIOMETRIC_STRONG)
    .setInvalidatedByBiometricEnrollment(true)
    .build()
```

Key properties:
- **Biometric-gated**: Every encrypt/decrypt requires a fresh `BiometricPrompt` authentication.
- **Invalidated on biometric change**: If the user adds/removes a fingerprint, the key is invalidated and the user must re-enter the master password.
- **Not exportable**: Key material never leaves the TEE/StrongBox.

### 3.3 Flow

1. After successful master-password login, encrypt DEK with Keystore key → store ciphertext in `EncryptedSharedPreferences`.
2. On biometric re-unlock: `BiometricPrompt` → get `Cipher` from callback → decrypt DEK ciphertext → DEK is back in memory.
3. On app kill or device restart: Keystore entry is programmatically deleted in `SessionManager.clearSession()`.

### 3.4 Fallback

If biometric is unavailable (not enrolled, hardware absent), the app skips Keystore caching entirely. Every unlock requires the master password. The UI shows an informational message suggesting biometric setup.

---

## 4. Session Lifecycle

### 4.1 States

```
LOCKED → (master password) → UNLOCKED → (background >5min) → BIOMETRIC_LOCKED → (biometric) → UNLOCKED
   ↑                                                                                               |
   └───────────────────────── (app killed / device restart) ────────────────────────────────────────┘
```

### 4.2 SessionManager

```kotlin
@ActivityRetainedScoped
class SessionManager @Inject constructor(
    private val keystoreManager: KeystoreManager,
    private val lifecycle: ProcessLifecycleOwner
) {
    private var dek: ByteArray? = null
    private var backgroundTimestamp: Long = 0
    private val timeoutMs = 5 * 60 * 1000L // 5 minutes

    val state: StateFlow<SessionState> // LOCKED, UNLOCKED, BIOMETRIC_LOCKED

    fun unlock(dek: ByteArray) { ... }
    fun onBackground() { backgroundTimestamp = System.currentTimeMillis() }
    fun onForeground() {
        if (elapsed > timeoutMs) {
            zeroizeDek()
            state = BIOMETRIC_LOCKED
        }
    }
    fun biometricUnlock(cipher: Cipher) { dek = keystoreManager.decryptDek(cipher) }
    fun clearSession() { zeroizeDek(); keystoreManager.deleteKey() }
}
```

### 4.3 Lifecycle Observer

An `Application`-level `ProcessLifecycleOwner` observer tracks foreground/background transitions and calls `SessionManager.onBackground()` and `SessionManager.onForeground()`.

### 4.4 App Kill Handling

`SessionManager.clearSession()` is called from:
- `Application.onTerminate()` (best effort, not guaranteed)
- `Activity.onDestroy()` when `isFinishing == true`
- On next app launch if stale Keystore entry is detected

The Keystore wrapper key is deleted on every session clear, so a cold start always requires the master password.

---

## 5. Networking Layer

### 5.1 HTTP Client

Ktor client with OkHttp engine. Configured at `@Singleton` scope via Hilt.

### 5.2 TLS Pinning

Certificate pinning via OkHttp `CertificatePinner`:
- Pin the leaf certificate's SHA-256 hash.
- Include one backup pin for certificate rotation.
- Pinning failures throw `SSLPeerUnverifiedException`, caught and shown as a connectivity error (never silently bypassed).

### 5.3 Auth Interceptor

An OkHttp interceptor attaches the JWT session token to every authenticated request. On 401 responses, it clears the session and navigates to the login screen.

### 5.4 API Contract

All requests and responses use JSON. The app communicates with these backend endpoints:

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/auth/register` | Signup |
| POST | `/api/auth/login` | Login |
| POST | `/api/auth/recovery/initiate` | Start recovery |
| POST | `/api/auth/recovery/complete` | Complete recovery |
| POST | `/api/auth/change-password` | Password change |
| GET | `/api/vault/entries` | List all entries |
| POST | `/api/vault/entries` | Create entry |
| PUT | `/api/vault/entries/{id}` | Update entry |
| DELETE | `/api/vault/entries/{id}` | Delete entry |

### 5.5 Error Handling

Network errors are mapped to sealed class `NetworkResult<T>`:
- `Success(data: T)`
- `ApiError(code: Int, message: String)`
- `NetworkError(cause: Throwable)` — timeout, no connectivity, TLS failure

ViewModels never see raw exceptions; they pattern-match on `NetworkResult`.

---

## 6. Local Encrypted Storage

### 6.1 Scope

Local storage is used ONLY for non-sensitive settings. Vault data is NEVER cached locally.

### 6.2 Implementation

`EncryptedSharedPreferences` from AndroidX Security library, backed by Tink's AES-256-SIV for keys and AES-256-GCM for values.

### 6.3 What Is Stored

| Key | Value | Purpose |
|-----|-------|---------|
| `biometric_dek_ciphertext` | Encrypted DEK blob | For biometric re-unlock |
| `session_user_id` | User identifier | To fetch correct vault on biometric re-unlock |
| `timeout_duration_ms` | Long | User-configurable session timeout |
| `clipboard_clear_delay_sec` | Int | Seconds before clipboard auto-clear (default 30) |
| `last_sync_timestamp` | Long | For UI display only |

### 6.4 What Is NOT Stored

- Master password (never persisted anywhere)
- DEK in plaintext
- auth_key
- KEK
- Recovery token
- Vault entries (even encrypted — the server is the source of truth)

---

## 7. Security Design Decisions

### 7.1 FLAG_SECURE

`FLAG_SECURE` is set on the `Activity` window. This prevents:
- Screenshots via system screenshot button
- Screen recording
- Recent apps thumbnail showing vault contents
- Screen sharing / casting

Applied in `onCreate()` of the single Activity:
```kotlin
window.setFlags(FLAG_SECURE, FLAG_SECURE)
```

### 7.2 Clipboard Auto-Clear

When the user copies a password:
1. Copy to clipboard via `ClipboardManager`.
2. Schedule a `Handler.postDelayed` to clear the clipboard after 30 seconds (configurable).
3. On Android 13+, use `ClipDescription.setExtra(PersistableBundle("android.content.extra.IS_SENSITIVE"))` to prevent clipboard content from appearing in keyboard suggestions.
4. If the app is killed before the timer fires, the clipboard is NOT cleared (OS limitation). This is documented to the user.

### 7.3 No Logging of Sensitive Data

- Timber (or no-op logger in release) strips all logs in release builds.
- Debug builds use a custom `Timber.Tree` that redacts any log message containing known sensitive field names.
- OkHttp logging interceptor is excluded from release builds entirely.

### 7.4 Root / Emulator Detection

The app does NOT block rooted devices (users may have legitimate reasons). Instead, it shows a one-time warning: "This device appears to be rooted. Security guarantees may be reduced."

### 7.5 Obfuscation

- ProGuard/R8 enabled for release builds with full minification.
- Crypto class names are not specifically obfuscated beyond standard R8 (security through obscurity is not relied upon).

### 7.6 WebView / Deep Link Attack Surface

- The app contains NO WebViews.
- Deep links are not supported (reduces attack surface).
- No exported Activities, Services, or BroadcastReceivers beyond the launcher Activity.

### 7.7 Compose-Specific Security

- Password fields use `visualTransformation = PasswordVisualTransformation()`.
- Text fields for sensitive input use `KeyboardOptions(autoCorrect = false, keyboardType = KeyboardType.Password)` to prevent keyboard from caching input.
