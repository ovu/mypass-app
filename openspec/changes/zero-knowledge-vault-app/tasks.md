# Implementation Tasks

## Phase 1: Foundation

### Task 1.1 — Project Structure Setup
- Create multi-module Gradle project with modules: `:app`, `:core:crypto`, `:core:keystore`, `:core:network`, `:core:storage`, `:core:common`, `:feature:auth`, `:feature:vault`, `:feature:settings`.
- Configure Kotlin 1.9+, Jetpack Compose BOM, Material 3.
- Add Hilt dependencies and configure `@HiltAndroidApp` Application class.
- Set up Compose Navigation with empty placeholder screens for each route.
- Set `FLAG_SECURE` on the Activity window.
- Configure R8/ProGuard for release builds.
- Verify: app builds, launches to a blank screen with Hilt injecting correctly.

### Task 1.2 — Core Common Module
- Define shared data types: `Category` enum, `VaultEntry` domain model, `EncryptedBlob` value class.
- Define `Result` wrapper (or use `kotlin.Result`) for domain error propagation.
- Define `CryptoException` sealed class (WrongPassword, IntegrityError, InvalidRecoveryToken, Argon2Failure, Unexpected).
- Verify: types compile, unit tests for data class equality and serialization.

---

## Phase 2: Crypto Module

### Task 2.1 — Argon2id Integration
- Add Signal argon2 library dependency (`org.signal:argon2`).
- Implement `suspend fun argon2id(password: CharArray, salt: ByteArray): ByteArray` on `Dispatchers.Default`.
- Parameters: memory=64MB, iterations=3, parallelism=4, output=32 bytes.
- Ensure password CharArray is converted to UTF-8 bytes then zeroed.
- Write unit tests with known test vectors from RFC 9106.
- Write instrumented test measuring execution time on device (target: 500ms-1000ms).

### Task 2.2 — HKDF-Expand
- Add BouncyCastle or Tink dependency for HKDF.
- Implement `fun hkdfExpand(prk: ByteArray, info: String, length: Int): ByteArray`.
- Implement `fun deriveKeys(password: CharArray, salt: ByteArray): DerivedKeys` composing Argon2id + HKDF.
- info strings: `"mypass-auth-key"` for auth_key, `"mypass-encryption-key"` for KEK.
- Zero master_secret after HKDF derivation.
- Write unit tests with HKDF test vectors from RFC 5869.

### Task 2.3 — AES-256-GCM Encrypt/Decrypt
- Implement `fun encryptEntry(plaintext: ByteArray, dek: ByteArray): EncryptedBlob`.
- Implement `fun decryptEntry(blob: EncryptedBlob, dek: ByteArray): ByteArray`.
- Nonce: 12 bytes from `SecureRandom`, prepended to ciphertext.
- GCM tag length: 128 bits.
- Map `AEADBadTagException` → `CryptoException.IntegrityError`.
- Write roundtrip unit tests. Write test for wrong-key decryption failure.

### Task 2.4 — DEK Generation and Wrapping
- Implement `fun generateDek(): ByteArray` — 32 random bytes.
- Implement `fun wrapDek(dek: ByteArray, kek: ByteArray): ByteArray` — AES-256-GCM.
- Implement `fun unwrapDek(wrappedDek: ByteArray, kek: ByteArray): ByteArray`.
- Map unwrap failure → `CryptoException.WrongPassword`.
- Write roundtrip tests. Write wrong-KEK test.

### Task 2.5 — Recovery Token Generation
- Implement `fun generateRecoveryToken(): RecoveryToken` — 128-bit random, Base32-encoded, hyphenated display.
- Implement `fun deriveRecoveryKeys(tokenBytes: ByteArray, salt: ByteArray): DerivedKeys` — same Argon2id + HKDF pipeline.
- Implement Base32 encode/decode (RFC 4648, uppercase, no padding).
- Write tests for token format (XXXX-XXXX-XXXX-XXXX), Base32 roundtrip, and key derivation from token.

### Task 2.6 — Memory Zeroing
- Implement `fun zeroize(vararg arrays: ByteArray)` and `fun zeroize(chars: CharArray)`.
- Audit all crypto functions to ensure zeroing in `finally` blocks.
- Write test verifying that after `zeroize()`, array contents are all zero.

---

## Phase 3: Android Keystore + Biometric

### Task 3.1 — Keystore Manager
- Implement `KeystoreManager` class wrapping Android Keystore.
- `fun generateKey()`: Create AES-256-GCM key with `setUserAuthenticationRequired(true)`, `AUTH_BIOMETRIC_STRONG`, `setInvalidatedByBiometricEnrollment(true)`.
- `fun encryptDek(dek: ByteArray): ByteArray`: Encrypt DEK with Keystore key, return ciphertext.
- `fun getDecryptCipher(): Cipher`: Get initialized Cipher for BiometricPrompt.
- `fun decryptDek(cipher: Cipher, ciphertext: ByteArray): ByteArray`: Decrypt using authenticated Cipher.
- `fun deleteKey()`: Remove Keystore entry.
- `fun isKeyValid(): Boolean`: Check if key exists and is not invalidated.
- Write instrumented tests on device.

### Task 3.2 — Biometric Prompt Integration
- Create `BiometricHelper` composable/utility that wraps `BiometricPrompt`.
- Check biometric availability: `BiometricManager.canAuthenticate(BIOMETRIC_STRONG)`.
- Show prompt with title "Unlock MyPass", negative button "Use master password".
- On success: extract Cipher from `AuthenticationResult`, pass to `KeystoreManager.decryptDek()`.
- On failure/cancel: emit event to ViewModel for fallback to master password.
- Handle `KeyPermanentlyInvalidatedException` — clear Keystore, require master password.

### Task 3.3 — Session Manager
- Implement `SessionManager` as `@ActivityRetainedScoped` Hilt binding.
- State: `StateFlow<SessionState>` with values `LOCKED`, `UNLOCKED`, `BIOMETRIC_LOCKED`.
- `fun unlock(dek: ByteArray)` — store DEK reference, set UNLOCKED.
- `fun clearSession()` — zero DEK, delete Keystore key, set LOCKED.
- Observe `ProcessLifecycleOwner` for foreground/background transitions.
- On background: record timestamp. On foreground: check elapsed > 5 min → `BIOMETRIC_LOCKED`.
- Configurable timeout stored in EncryptedSharedPreferences.
- Write unit tests for state transitions and timeout logic.

---

## Phase 4: Auth Screens

### Task 4.1 — Signup Screen + Flow
- Build Compose UI: email field, master password field (with confirmation), "Create Account" button.
- Password strength indicator (length-based + zxcvbn-style).
- ViewModel: validate inputs, call `CryptoEngine` for full signup pipeline (salt, keys, DEK, recovery token, wrapping).
- Network call: `POST /api/auth/register`.
- On success: store DEK in session, cache in Keystore, navigate to Recovery Token Display.
- Error handling: duplicate email (409), network errors.
- Write UI tests for validation states and navigation.

### Task 4.2 — Login Screen + Flow
- Build Compose UI: email field, master password field, "Log In" button, "Forgot Password?" link.
- ViewModel: fetch salt, derive keys, authenticate, unwrap DEK, decrypt vault.
- Network calls: `GET /api/auth/login/salt`, `POST /api/auth/login`.
- On success: store DEK, cache in Keystore, navigate to Vault List.
- Error handling: wrong password (401), rate limiting (429), network errors.
- Write UI tests.

### Task 4.3 — Biometric Unlock Screen
- Build Compose UI: lock icon, "Unlock with fingerprint" prompt, "Use master password" button.
- Trigger `BiometricPrompt` on screen appear.
- On biometric success: decrypt DEK from Keystore, navigate to Vault List.
- On "Use master password": navigate to Login Screen.
- On Keystore invalidation: show message, navigate to Login Screen.

---

## Phase 5: Vault Screens

### Task 5.1 — Vault List Screen
- Build Compose UI: search bar, category filter chips, scrollable entry list, FAB.
- ViewModel: hold decrypted entries in memory, filter by search query and category.
- Search: debounced 300ms, filter by title/username/url/notes.
- Pull-to-refresh: fetch + decrypt latest entries from server.
- Empty state with illustration.
- Navigation: tap entry → Detail, tap FAB → Add Entry.
- Write UI tests for search filtering and category selection.

### Task 5.2 — Entry Detail Screen
- Build Compose UI: field rows (username, password, URL, notes), copy buttons, edit/delete actions.
- Password visibility toggle with 10-second auto-hide.
- Copy-to-clipboard with 30-second auto-clear and snackbar.
- Delete with confirmation dialog + API call.
- Write UI tests for copy behavior and delete confirmation.

### Task 5.3 — Add Entry Screen
- Build Compose UI: category dropdown, title (required), username, password (with visibility + generator button), URL, notes.
- ViewModel: validate title non-empty, serialize to JSON, encrypt with DEK, POST to server.
- Discard-changes dialog on back navigation.
- On save success: navigate back to Vault List.
- Write UI tests for validation and save flow.

### Task 5.4 — Edit Entry Screen
- Reuse Add Entry composables with pre-filled data.
- ViewModel: same as Add but uses PUT instead of POST.
- Track dirty state for discard-changes dialog.
- On save success: navigate back to Entry Detail with updated data.

---

## Phase 6: Password Generator

### Task 6.1 — Password Generator Bottom Sheet
- Build Compose UI: generated password display, regenerate button, length slider (8-64), character set checkboxes, strength indicator, "Use This Password" button.
- Generation logic: `SecureRandom`, guarantee at least one char from each enabled set.
- Strength bar: entropy calculation `length * log2(charset_size)`.
- Auto-regenerate on option change.
- Validate at least one charset enabled.
- Return generated password as `CharArray` to parent screen.
- Write unit tests for generation logic (charset enforcement, length bounds, entropy calculation).

---

## Phase 7: Password Change + Recovery

### Task 7.1 — Password Change Screen + Flow
- Build Compose UI: current password, new password (with confirmation), "Change Password" button.
- ViewModel: verify old password locally (unwrap DEK), derive new keys, re-wrap DEK, API call.
- Network call: `POST /api/auth/change-password`.
- On success: show confirmation, session continues with same DEK.
- Error handling: wrong old password (local check), server rejection, network errors.
- Write UI tests.

### Task 7.2 — Account Recovery Screen + Flow
- Build Compose UI (multi-step):
  - Step 1: email + recovery token input (formatted XXXX-XXXX-XXXX-XXXX).
  - Step 2: new master password (with confirmation).
- ViewModel:
  - Step 1: parse token, fetch recovery data, derive recovery keys, unwrap DEK.
  - Step 2: derive new keys from new password, re-wrap DEK, complete recovery.
- Network calls: `POST /api/auth/recovery/initiate`, `POST /api/auth/recovery/complete`.
- On success: log user in, navigate to Vault List.
- Error handling: invalid token, network errors.
- Write UI tests for multi-step flow.

### Task 7.3 — Recovery Token Generation + Display
- Build Recovery Token Display Screen (see vault-ui spec).
- Implement "Generate New Recovery Token" flow from Settings.
- Prompt for master password before generating.
- Network call: `POST /api/auth/recovery/add`.
- Checkbox + Continue behavior with token zeroing.
- Back-navigation confirmation dialog.
- Write UI tests.

---

## Phase 8: Networking Layer

### Task 8.1 — Ktor HTTP Client Setup
- Configure Ktor client with OkHttp engine at `@Singleton` scope.
- JSON serialization with kotlinx.serialization.
- Auth interceptor: attach JWT to authenticated requests, handle 401 → session clear.
- Base URL configuration (build flavors: debug vs release).
- Request/response logging (debug builds only, no sensitive data).
- Write unit tests with mock engine.

### Task 8.2 — TLS Certificate Pinning
- Configure OkHttp `CertificatePinner` with SHA-256 pins for the server certificate.
- Include one backup pin for rotation.
- On pin failure: surface user-facing error ("Cannot verify server identity. Check your network.").
- Pinning disabled in debug builds for local development.
- Write test verifying pinning rejects wrong certificates.

### Task 8.3 — API Data Transfer Objects
- Define request/response DTOs for all endpoints using `@Serializable`.
- DTOs: `RegisterRequest`, `LoginSaltResponse`, `LoginRequest`, `LoginResponse`, `ChangePasswordRequest`, `RecoveryInitiateRequest`, `RecoveryInitiateResponse`, `RecoveryCompleteRequest`, `VaultEntryDto`, `CreateEntryRequest`, `UpdateEntryRequest`.
- Base64 encoding/decoding for all byte array fields.
- Write serialization roundtrip tests.

---

## Phase 9: Local Storage

### Task 9.1 — Encrypted SharedPreferences Setup
- Configure `EncryptedSharedPreferences` with Tink master key.
- Define typed accessor class `AppPreferences` for: `biometric_dek_ciphertext`, `session_user_id`, `timeout_duration_ms`, `clipboard_clear_delay_sec`, `last_sync_timestamp`.
- Provide as `@Singleton` via Hilt.
- Write instrumented tests for read/write roundtrip.

---

## Phase 10: Security Hardening

### Task 10.1 — Clipboard Security
- Implement `ClipboardHelper` that copies text and schedules auto-clear after configurable delay (default 30s).
- On Android 13+: set `EXTRA_IS_SENSITIVE` on clip description.
- Use `Handler.postDelayed` for clear scheduling. Cancel pending clears on new copy.
- Write unit tests for scheduling logic.

### Task 10.2 — Sensitive Input Handling
- Audit all password/token input fields for `PasswordVisualTransformation`, `autoCorrect = false`, `keyboardType = KeyboardType.Password`.
- Ensure passwords flow through `CharArray` (never `String`) from UI to crypto layer.
- Verify `FLAG_SECURE` is set on the Activity.
- Add root/emulator detection with one-time warning dialog.

### Task 10.3 — Release Build Hardening
- Verify ProGuard/R8 minification is enabled and tested.
- Strip all Timber/logging in release builds.
- Remove OkHttp logging interceptor from release.
- Verify no exported components beyond launcher Activity.
- Test release APK manually.

---

## Phase 11: Integration Testing

### Task 11.1 — Crypto Integration Tests
- End-to-end test: signup key generation → DEK wrapping → vault entry encrypt/decrypt → password change re-wrapping → recovery token unwrap.
- Test with known inputs for deterministic verification.
- Test wrong-password and wrong-token error paths.

### Task 11.2 — Auth Flow Integration Tests
- Test signup → login → logout → biometric re-unlock full cycle using mock server.
- Test password change → re-login with new password.
- Test account recovery → re-login.
- Test session timeout → biometric re-unlock.
- Use Hilt test rules and mock network responses.

### Task 11.3 — Vault CRUD Integration Tests
- Test add entry → list → view → edit → delete full cycle.
- Verify encrypted blobs sent to server are not plaintext (parse and assert non-JSON).
- Test pull-to-refresh fetches and decrypts correctly.

### Task 11.4 — Security Integration Tests
- Verify `FLAG_SECURE` is set (instrumented test checking window flags).
- Verify clipboard is cleared after timeout.
- Verify DEK is zeroed after session clear (inspect `SessionManager` state).
- Verify Keystore key is deleted after session clear.
