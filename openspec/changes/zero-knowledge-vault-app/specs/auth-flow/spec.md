# Authentication Flows Spec

## Overview

This spec defines every authentication-related flow in the app: signup, login, password change, account recovery, biometric unlock, and session timeout. Each flow is described as a sequence of steps with explicit crypto operations, network calls, and error handling.

---

## 1. Signup Flow

### Trigger
User taps "Create Account" and submits email + master password (confirmed twice).

### Client-Side Validation
- Email: valid format, non-empty.
- Master password: minimum 12 characters. UI shows strength indicator (zxcvbn-style scoring, not enforced beyond minimum length).
- Confirmation: must match.

### Steps

1. **Generate salt**: `salt = SecureRandom(16 bytes)`
2. **Derive keys**: `DerivedKeys(auth_key, kek) = CryptoEngine.deriveKeys(password, salt)`
3. **Generate DEK**: `dek = CryptoEngine.generateDek()` — random 256-bit key
4. **Wrap DEK with KEK**: `encrypted_dek_primary = CryptoEngine.wrapDek(dek, kek)`
5. **Generate recovery token**: `recovery = CryptoEngine.generateRecoveryToken()` — 128-bit, Base32
6. **Generate recovery salt**: `salt_recovery = SecureRandom(16 bytes)`
7. **Derive recovery keys**: `DerivedKeys(recovery_auth_key, recovery_kek) = CryptoEngine.deriveRecoveryKeys(recovery.rawBytes, salt_recovery)`
8. **Wrap DEK with recovery KEK**: `encrypted_dek_recovery = CryptoEngine.wrapDek(dek, recovery_kek)`
9. **Zeroize**: kek, recovery_kek, master_secret (already zeroed inside deriveKeys)
10. **API call**: `POST /api/auth/register`
    ```json
    {
      "email": "user@example.com",
      "auth_key": "<base64>",
      "salt": "<base64>",
      "encrypted_dek_primary": "<base64>",
      "salt_recovery": "<base64>",
      "recovery_auth_key": "<base64>",
      "encrypted_dek_recovery": "<base64>"
    }
    ```
11. **On success**: Navigate to Recovery Token Display Screen (see vault-ui spec).
12. **Store DEK in session**: `SessionManager.unlock(dek)` — user is now logged in.
13. **Cache DEK for biometric**: If biometric is available, encrypt DEK with Keystore key and store ciphertext.
14. **Zeroize**: auth_key, recovery token (after user acknowledges).

### Error Handling

| Error | Behavior |
|-------|----------|
| Email already registered (409) | Show "Account already exists. Try logging in." |
| Network error | Show retry button. No state is persisted, so retry is safe. |
| Argon2 failure | Show generic error. Log to Crashlytics (no sensitive data). |

---

## 2. Login Flow

### Trigger
User enters email + master password and taps "Log In".

### Steps

1. **Fetch salt**: `GET /api/auth/login/salt?email=user@example.com` → `{ salt: "<base64>" }`
   - If user does not exist, server returns a fake salt (timing-safe, to prevent user enumeration).
2. **Derive keys**: `DerivedKeys(auth_key, kek) = CryptoEngine.deriveKeys(password, salt)`
3. **API call**: `POST /api/auth/login`
    ```json
    {
      "email": "user@example.com",
      "auth_key": "<base64>"
    }
    ```
4. **On success**: Server returns:
    ```json
    {
      "token": "<jwt>",
      "encrypted_dek_primary": "<base64>",
      "vault_entries": [
        { "id": "uuid", "data": "<base64>" }
      ]
    }
    ```
5. **Unwrap DEK**: `dek = CryptoEngine.unwrapDek(encrypted_dek_primary, kek)`
6. **Zeroize**: auth_key, kek
7. **Store DEK in session**: `SessionManager.unlock(dek)`
8. **Cache DEK for biometric**: Encrypt DEK with Keystore key, store ciphertext.
9. **Decrypt vault entries**: For each entry, `CryptoEngine.decryptEntry(entry.data, dek)` → cache decrypted entries in ViewModel (in-memory only).
10. **Navigate**: To Vault List Screen.

### Error Handling

| Error | Behavior |
|-------|----------|
| Wrong password (401) | Show "Incorrect email or password." (generic to prevent enumeration) |
| Rate limited (429) | Show "Too many attempts. Try again in X seconds." with countdown |
| Network error | Show retry button |
| DEK unwrap failure | This should not happen if server returned correct data. Show "Data integrity error. Contact support." |

---

## 3. Password Change Flow

### Trigger
User navigates to Settings → Change Password. User must enter old password + new password (confirmed twice).

### Precondition
User is currently authenticated (DEK is in session).

### Steps

1. **Validate new password**: Same rules as signup (12+ chars, confirmation match).
2. **Derive old keys**: `DerivedKeys(old_auth_key, old_kek) = CryptoEngine.deriveKeys(old_password, current_salt)`
3. **Verify old password locally**: `CryptoEngine.unwrapDek(encrypted_dek_primary, old_kek)` — if this fails, the old password is wrong. Do NOT call the server.
4. **Generate new salt**: `new_salt = SecureRandom(16 bytes)`
5. **Derive new keys**: `DerivedKeys(new_auth_key, new_kek) = CryptoEngine.deriveKeys(new_password, new_salt)`
6. **Re-wrap DEK**: `new_encrypted_dek_primary = CryptoEngine.wrapDek(dek, new_kek)`
7. **Zeroize**: old_kek, new_kek, old_password, new_password
8. **API call**: `POST /api/auth/change-password`
    ```json
    {
      "old_auth_key": "<base64>",
      "new_auth_key": "<base64>",
      "new_salt": "<base64>",
      "new_encrypted_dek_primary": "<base64>"
    }
    ```
9. **On success**: Show confirmation. Session continues with existing DEK (unchanged).
10. **Zeroize**: old_auth_key, new_auth_key.

### Important
- Vault entries are NOT re-encrypted. Only the DEK wrapping changes.
- Existing recovery tokens remain valid (they wrap the same DEK).
- The biometric Keystore cache remains valid (same DEK, just re-wrapped for server storage).

### Error Handling

| Error | Behavior |
|-------|----------|
| Old password wrong (local unwrap fails) | Show "Current password is incorrect." |
| Old auth_key rejected (401) | Show "Authentication failed. Try again." |
| Network error | Show retry. The operation is idempotent from the user's perspective. |

---

## 4. Account Recovery Flow

### Trigger
User taps "Forgot Password?" on the login screen and enters their email + recovery token.

### Steps

1. **Collect input**: Email + recovery token (user types the XXXX-XXXX-XXXX-XXXX format).
2. **Parse token**: Strip hyphens, Base32-decode to 16 bytes. Validate length.
3. **API call**: `POST /api/auth/recovery/initiate`
    ```json
    {
      "email": "user@example.com"
    }
    ```
4. **On success**: Server returns:
    ```json
    {
      "salt_recovery": "<base64>",
      "encrypted_dek_recovery": "<base64>"
    }
    ```
5. **Derive recovery keys**: `DerivedKeys(recovery_auth_key, recovery_kek) = CryptoEngine.deriveRecoveryKeys(token_bytes, salt_recovery)`
6. **Unwrap DEK**: `dek = CryptoEngine.unwrapDek(encrypted_dek_recovery, recovery_kek)` — if this fails, the recovery token is wrong.
7. **Zeroize**: recovery_kek, token_bytes
8. **Prompt for new password**: User enters new master password (confirmed).
9. **Generate new salt**: `new_salt = SecureRandom(16 bytes)`
10. **Derive new keys**: `DerivedKeys(new_auth_key, new_kek) = CryptoEngine.deriveKeys(new_password, new_salt)`
11. **Re-wrap DEK**: `new_encrypted_dek_primary = CryptoEngine.wrapDek(dek, new_kek)`
12. **Zeroize**: new_kek, new_password
13. **API call**: `POST /api/auth/recovery/complete`
    ```json
    {
      "email": "user@example.com",
      "recovery_auth_key": "<base64>",
      "new_auth_key": "<base64>",
      "new_salt": "<base64>",
      "new_encrypted_dek_primary": "<base64>"
    }
    ```
14. **On success**: Log user in with the DEK. Navigate to Vault List.
15. **Zeroize**: new_auth_key, recovery_auth_key.

### Error Handling

| Error | Behavior |
|-------|----------|
| Recovery token wrong (unwrap fails) | Show "Invalid recovery token. Check and try again." |
| Email not found (404) | Show generic "Recovery failed." (prevent enumeration) |
| Network error | Show retry |

### Security Note
The recovery token used in this flow is NOT invalidated. It can be used again. If the user wants to invalidate old recovery tokens, they must do so explicitly from Settings (future feature).

---

## 5. Biometric Unlock Flow

### Trigger
App returns to foreground after >5 minutes, or user manually locks and re-unlocks.

### Precondition
- User has previously logged in with master password in this app session (Keystore key exists).
- Device has biometric hardware and at least one enrolled biometric.

### Steps

1. **SessionManager** detects `BIOMETRIC_LOCKED` state.
2. **UI** shows BiometricPrompt:
   ```kotlin
   BiometricPrompt.PromptInfo.Builder()
       .setTitle("Unlock MyPass")
       .setSubtitle("Use your fingerprint to unlock your vault")
       .setNegativeButtonText("Use master password")
       .setAllowedAuthenticators(BIOMETRIC_STRONG)
       .build()
   ```
3. **On biometric success**: `BiometricPrompt.AuthenticationResult` provides a `Cipher` object (initialized by the TEE after biometric verification).
4. **Decrypt DEK**: `dek = keystoreManager.decryptDek(cipher, storedCiphertext)` — uses the Cipher from the callback to decrypt the DEK ciphertext stored in EncryptedSharedPreferences.
5. **Resume session**: `SessionManager.unlock(dek)`. Navigate to Vault List.

### Fallback: "Use master password"
If the user taps the negative button, navigate to the full login screen. This re-runs Argon2id.

### Error Handling

| Error | Behavior |
|-------|----------|
| Biometric not recognized | BiometricPrompt handles retries internally (up to 3 attempts, then 30s lockout) |
| Keystore key invalidated (biometric enrollment changed) | Show "Biometric data changed. Please log in with your master password." Clear Keystore entry. |
| Biometric locked out (too many attempts) | Show "Biometric locked. Use master password." |
| Biometric not enrolled | Skip biometric entirely, require master password |

---

## 6. Session Timeout

### Mechanism

`SessionManager` observes app lifecycle via `ProcessLifecycleOwner`:

```kotlin
// In Application.onCreate()
ProcessLifecycleOwner.get().lifecycle.addObserver(sessionManager)
```

### Lifecycle Events

**ON_STOP (app backgrounded):**
1. Record `backgroundTimestamp = SystemClock.elapsedRealtime()`.
2. DEK remains in memory (for quick return).

**ON_START (app foregrounded):**
1. Calculate `elapsed = now - backgroundTimestamp`.
2. If `elapsed > timeoutMs` (default 5 minutes):
   - `dek.fill(0)` — zero the DEK in memory.
   - Set state to `BIOMETRIC_LOCKED`.
   - If biometric is not available, set state to `LOCKED` (requires master password).
3. If `elapsed <= timeoutMs`:
   - Session continues. No action needed.

### Configurable Timeout
Users can change the timeout duration in Settings. Options: 1 min, 2 min, 5 min (default), 15 min, 30 min. Stored in `EncryptedSharedPreferences`.

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| Device locked by user (power button) | Same as background — timer starts |
| Phone call interrupts app | Same as background — timer starts |
| App killed by OS (low memory) | On next launch, Keystore key is present but session is fresh → biometric prompt |
| Device reboot | Keystore key is deleted on session clear. Cold start requires master password. |
| Split-screen / picture-in-picture | App remains in foreground; no timeout triggered |

---

## 7. Sequence Diagrams

### 7.1 Signup Sequence

```
User              App                 CryptoEngine           Server
 |                 |                       |                    |
 |--email+pass---->|                       |                    |
 |                 |---deriveKeys--------->|                    |
 |                 |<--auth_key, kek-------|                    |
 |                 |---generateDek-------->|                    |
 |                 |<--dek----------------|                    |
 |                 |---wrapDek(dek,kek)--->|                    |
 |                 |<--encrypted_dek-------|                    |
 |                 |---generateRecovery--->|                    |
 |                 |<--token, recovery_kek-|                    |
 |                 |---wrapDek(dek,rkek)-->|                    |
 |                 |<--encrypted_dek_rec---|                    |
 |                 |---POST /register----------------------------->|
 |                 |<--201 Created---------------------------------|
 |<--show token----|                       |                    |
 |--"I saved it"-->|                       |                    |
 |                 |---zeroize(token)----->|                    |
```

### 7.2 Login Sequence

```
User              App                 CryptoEngine           Server
 |                 |                       |                    |
 |--email+pass---->|                       |                    |
 |                 |---GET /salt---------------------------------->|
 |                 |<--salt---------------------------------------|
 |                 |---deriveKeys(pass,salt)>|                   |
 |                 |<--auth_key, kek--------|                   |
 |                 |---POST /login(auth_key)---------------------->|
 |                 |<--jwt, encrypted_dek, entries-----------------|
 |                 |---unwrapDek(edek,kek)->|                    |
 |                 |<--dek-----------------|                    |
 |                 |---decryptEntries----->|                    |
 |                 |<--plaintext entries---|                    |
 |<--vault list----|                       |                    |
```

### 7.3 Biometric Re-unlock Sequence

```
User              App                 Keystore        BiometricPrompt
 |                 |                     |                  |
 |--open app------>|                     |                  |
 |                 |--check timeout----->|                  |
 |                 |  (>5min: LOCKED)    |                  |
 |                 |--show prompt--------|----------------->|
 |--fingerprint----|---------------------|----------------->|
 |                 |<--Cipher (authed)---|------------------|
 |                 |--decrypt(cipher)--->|                  |
 |                 |<--dek---------------|                  |
 |<--vault list----|                     |                  |
```
