# Zero-Knowledge Vault App

## Problem

Users need a mobile app to securely store and retrieve passwords. All cryptographic operations must happen on-device — the server must never see plaintext data. The app must also handle account recovery via a token that is shown once and never stored on the server.

## Solution

Build an Android app that implements client-side **zero-knowledge encryption** for a password vault. The app handles all key derivation, encryption, and decryption locally. The backend is treated as an opaque encrypted blob store.

### Crypto Architecture (client-side)

**Key Derivation (on every login)**
1. Run `Argon2id(master_password, salt)` → 32-byte master secret
2. `auth_key = HKDF-Expand(master, info="auth", length=32)` → sent to server for login
3. `kek = HKDF-Expand(master, info="encrypt", length=32)` → decrypts DEK locally

**Vault Encryption**
- DEK (Data Encryption Key): random 256-bit AES key, generated at signup
- Each vault entry encrypted with `AES-256-GCM(entry, DEK, unique_nonce)`
- DEK encrypted with KEK → stored on server as `encrypted_dek_primary`

**Recovery Token (generated at signup)**
1. Generate 128-bit random token, Base32-encode for display (e.g., `ABCD-EFGH-JKLM-NPQR`)
2. Derive KEK₂ from recovery token via Argon2id + HKDF
3. Encrypt DEK with KEK₂ → stored on server as `encrypted_dek_recovery`
4. Show token to user ONCE with strong warning to write it down
5. Discard token from memory immediately

**Multiple Recovery Tokens**
- Users can generate additional recovery tokens at any time
- Each adds one `encrypted_dek_recovery_N` blob to the server
- No limit on count (each is ~80 bytes)

### Session Management

| State | Behavior |
|-------|----------|
| First unlock | Master password → Argon2id → KEK → decrypt DEK |
| DEK caching | Store DEK encrypted in Android Keystore, gated by BiometricPrompt |
| App backgrounded < 5 min | DEK stays in memory, no re-auth |
| App backgrounded > 5 min | DEK cleared, biometric to re-unlock (instant, no Argon2) |
| App killed / device restart | Keystore entry cleared, full master password required |

### Argon2 Parameters (tuned for mid-range Android)

- Algorithm: Argon2id
- Memory: 64 MB
- Iterations: 3
- Parallelism: 4
- Salt: 16 bytes random
- Target: 500ms–1000ms on mid-range device (benchmark and adjust)

### Key Flows

**Signup**
1. User sets master password
2. App generates: salt, DEK (random), recovery token
3. Derives auth_key + KEK from master password
4. Encrypts DEK with KEK → `encrypted_dek_primary`
5. Encrypts DEK with recovery-derived KEK₂ → `encrypted_dek_recovery`
6. Sends to server: auth_key, salt, salt_recovery, encrypted_dek_primary, encrypted_dek_recovery
7. Shows recovery token to user with strong "write this down" prompt

**Login**
1. User enters master password
2. App derives auth_key, sends to server
3. Server verifies, returns salt + encrypted_dek_primary + encrypted vault entries
4. App derives KEK, decrypts DEK, decrypts vault entries
5. Stores DEK in Android Keystore for biometric re-unlock

**Password Change**
1. User provides old + new password
2. App decrypts DEK with old KEK (proves knowledge of old password)
3. Re-derives auth_key + KEK from new password with fresh salt
4. Re-encrypts DEK with new KEK
5. Sends new auth_key + salt + encrypted_dek_primary to server
6. Vault entries untouched

**Account Recovery**
1. User enters recovery token
2. App derives KEK₂, decrypts DEK
3. User sets new master password
4. App re-wraps DEK with new KEK, sends new auth credentials to server

**Vault CRUD**
1. New entry: encrypt with DEK + unique nonce, send encrypted blob to server
2. Read entry: fetch encrypted blob, decrypt with DEK
3. Update: re-encrypt, send new blob
4. Delete: tell server to remove entry

## Tech Stack

- **Language**: Kotlin
- **UI**: Jetpack Compose
- **Crypto**: Tink (Google) or Bouncy Castle for AES-256-GCM
- **Argon2**: Signal's argon2 Android library or libsodium-jni
- **Biometric**: AndroidX Biometric + Android Keystore
- **Networking**: Ktor client or Retrofit
- **Local storage**: Encrypted SharedPreferences (for settings only, NOT vault data)

## Non-Goals

- No iOS version (Android only for now)
- No web client
- No vault sharing between users
- No autofill service (future enhancement)
- No cloud backup of vault keys

## Security Invariants

1. Master password NEVER leaves the device
2. DEK NEVER leaves the device in plaintext
3. Recovery token is shown ONCE and discarded from memory
4. Server is untrusted — all crypto is local
5. If user loses both master password and all recovery tokens, data is permanently lost

## Open Questions

- Exact Kotlin crypto library choice (Tink vs BouncyCastle vs libsodium)
- Clipboard security (auto-clear copied passwords after N seconds?)
- Screenshot prevention (FLAG_SECURE on all screens?)
- Offline mode (cache encrypted vault locally for offline access?)
