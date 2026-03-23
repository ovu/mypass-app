# Vault UI Spec

## Overview

This spec defines the UI screens and interactions for the vault feature: listing entries, viewing/copying passwords, adding/editing entries, generating passwords, and displaying recovery tokens. All screens use Jetpack Compose with Material 3.

---

## 1. Vault List Screen

### Route
`/vault` — the main screen after login/unlock.

### Layout

```
┌─────────────────────────────────┐
│ MyPass                     [⚙]  │  ← TopAppBar with settings icon
├─────────────────────────────────┤
│ 🔍 Search passwords...          │  ← Search bar
├─────────────────────────────────┤
│ [All] [Login] [Card] [Note] ... │  ← Category filter chips (horizontal scroll)
├─────────────────────────────────┤
│ ┌─────────────────────────────┐ │
│ │ GitHub                      │ │  ← Entry card
│ │ john@example.com            │ │
│ │ Login · Updated 2 days ago  │ │
│ └─────────────────────────────┘ │
│ ┌─────────────────────────────┐ │
│ │ Netflix                     │ │
│ │ john@example.com            │ │
│ │ Login · Updated 1 week ago  │ │
│ └─────────────────────────────┘ │
│          ...                    │
├─────────────────────────────────┤
│                          [ + ]  │  ← FAB: add new entry
└─────────────────────────────────┘
```

### Behavior

**Search:**
- Filters entries by title, username, URL, and notes.
- Filtering happens in-memory on the decrypted entries (already in ViewModel).
- Debounced: 300ms after last keystroke.
- Empty state: "No entries match your search."

**Category Chips:**
- Categories: All, Login, Credit Card, Secure Note, Identity.
- Tapping a chip filters the list to that category.
- "All" is selected by default.

**Entry Cards:**
- Display: title (bold), username/email (subtitle), category + last-updated relative time.
- Tap: navigate to Entry Detail Screen.
- Long-press: shows context menu (Copy password, Copy username, Delete).

**Empty State (no entries):**
- Illustration + "Your vault is empty. Tap + to add your first password."

**Pull-to-Refresh:**
- Fetches latest encrypted entries from server, decrypts, and updates list.

### ViewModel

```kotlin
data class VaultListState(
    val entries: List<VaultEntryUi> = emptyList(),
    val filteredEntries: List<VaultEntryUi> = emptyList(),
    val searchQuery: String = "",
    val selectedCategory: Category = Category.ALL,
    val isLoading: Boolean = false,
    val error: String? = null
)

data class VaultEntryUi(
    val id: String,
    val title: String,
    val subtitle: String,    // username or card last-4
    val category: Category,
    val updatedAt: Instant
)
```

---

## 2. Entry Detail Screen

### Route
`/vault/{entryId}`

### Layout

```
┌─────────────────────────────────┐
│ ← GitHub                  [✏] [🗑]│  ← Back, Edit, Delete
├─────────────────────────────────┤
│                                 │
│ Username                        │
│ john@example.com         [📋]   │  ← Tap to copy
│                                 │
│ Password                        │
│ ••••••••••••             [👁][📋]│  ← Toggle visibility + copy
│                                 │
│ URL                             │
│ https://github.com       [📋]   │  ← Tap to copy
│                                 │
│ Notes                           │
│ My GitHub work account          │
│                                 │
│ ─────────────────────────────── │
│ Created: Jan 15, 2026           │
│ Modified: Mar 20, 2026          │
└─────────────────────────────────┘
```

### Behavior

**Password visibility:**
- Default: masked with dots.
- Tap eye icon: reveal password in monospace font. Auto-hide after 10 seconds.

**Copy to clipboard:**
- Tap copy icon next to any field.
- Show snackbar: "Password copied. Clipboard will clear in 30s."
- Schedule clipboard clear via `ClipboardManager.clearPrimaryClip()` after 30 seconds.
- On Android 13+, set `ClipDescription.EXTRA_IS_SENSITIVE` flag.
- If user copies another field before timer expires, restart timer for the new copy.

**Delete:**
- Tap trash icon → confirmation dialog: "Delete 'GitHub'? This action cannot be undone."
- On confirm: `DELETE /api/vault/entries/{id}` → remove from local list → navigate back.

**Edit:**
- Tap edit icon → navigate to Edit Entry Screen with pre-filled data.

### ViewModel

```kotlin
data class EntryDetailState(
    val entry: VaultEntryDetail? = null,
    val isPasswordVisible: Boolean = false,
    val isLoading: Boolean = false,
    val error: String? = null
)

data class VaultEntryDetail(
    val id: String,
    val title: String,
    val username: String,
    val password: CharArray,  // CharArray, not String
    val url: String,
    val notes: String,
    val category: Category,
    val createdAt: Instant,
    val updatedAt: Instant
)
```

---

## 3. Add / Edit Entry Screen

### Route
- Add: `/vault/add`
- Edit: `/vault/{entryId}/edit`

### Layout

```
┌─────────────────────────────────┐
│ ← Add Entry              [Save]│
├─────────────────────────────────┤
│                                 │
│ Category                        │
│ [Login ▼]                       │  ← Dropdown
│                                 │
│ Title *                         │
│ [________________________]      │
│                                 │
│ Username                        │
│ [________________________]      │
│                                 │
│ Password                        │
│ [__________________] [👁] [🎲]  │  ← Visibility toggle + generator
│                                 │
│ URL                             │
│ [________________________]      │
│                                 │
│ Notes                           │
│ [________________________]      │
│ [________________________]      │
│ [________________________]      │
│                                 │
└─────────────────────────────────┘
```

### Behavior

**Validation:**
- Title is required. All other fields are optional.
- Show inline error under Title if empty on save attempt.

**Password field:**
- Text input with visibility toggle.
- Generator button (dice icon) opens Password Generator bottom sheet.
- Password stored as `CharArray` in ViewModel, zeroed on screen exit.

**Save (Add):**
1. Serialize entry to JSON bytes.
2. `encrypted = CryptoEngine.encryptEntry(jsonBytes, dek)`
3. `POST /api/vault/entries` with encrypted blob.
4. On success: navigate back to Vault List. New entry appears at top.

**Save (Edit):**
1. Same serialization + encryption.
2. `PUT /api/vault/entries/{id}` with new encrypted blob.
3. On success: navigate back to Entry Detail with updated data.

**Discard changes:**
- Back button or back gesture with unsaved changes → "Discard changes?" dialog.

### Encrypted Blob Structure

The server stores each vault entry as an **opaque encrypted blob**. It has no plaintext columns — no title, no description, no metadata. All fields live inside the encrypted payload:

```
    SERVER SEES                        CLIENT SEES (after decryption)
    ───────────                        ────────────────────────────────
    ┌──────────────────┐               ┌──────────────────────────────┐
    │ id: UUID         │               │ title: "GitHub"              │
    │ user_id: UUID    │               │ username: "john@example.com" │
    │ encrypted_blob:  │  ──AES-GCM──▶ │ password: "s3cur3P@ss!"     │
    │   "a3F7x9...b64" │  decrypt with │ url: "https://github.com"   │
    │ nonce: "k9L...=" │  DEK + nonce  │ notes: "Work account"       │
    │ created_at: ts   │               │ category: "login"            │
    │ updated_at: ts   │               └──────────────────────────────┘
    └──────────────────┘
```

**Entry JSON Schema** (plaintext, before encryption):

```json
{
  "title": "GitHub",
  "username": "john@example.com",
  "password": "s3cur3P@ss!",
  "url": "https://github.com",
  "notes": "Work account",
  "category": "login"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Human-readable label for the entry |
| `username` | No | Username or email |
| `password` | No | The stored password (as plaintext inside the encrypted blob) |
| `url` | No | Associated URL |
| `notes` | No | Free-text notes |
| `category` | Yes | One of: `login`, `credit_card`, `secure_note`, `identity` |

Timestamps (`created_at`, `updated_at`) are stored server-side on the unencrypted row — they don't reveal vault content and are needed for sorting.

**Why no plaintext metadata on the server:** Even knowing *which sites* a user has accounts on is a privacy leak. A stolen database reveals only opaque blobs and UUIDs. Search, sort by title, and category filtering all happen client-side after decryption.

---

## 4. Password Generator

### Trigger
Tap the dice icon on the password field in Add/Edit Entry Screen. Opens as a bottom sheet.

### Layout

```
┌─────────────────────────────────┐
│ Generate Password               │
├─────────────────────────────────┤
│                                 │
│ ┌─────────────────────────────┐ │
│ │ xK#9mPq$vR2nL@8w           │ │  ← Generated password (monospace)
│ └─────────────────────────────┘ │
│            [🔄 Regenerate]      │
│                                 │
│ Length                          │
│ ──────────●───────────── 16    │  ← Slider (8-64, default 16)
│                                 │
│ ☑ Uppercase (A-Z)              │
│ ☑ Lowercase (a-z)              │
│ ☑ Digits (0-9)                 │
│ ☑ Symbols (!@#$%^&*)           │
│ ☐ Exclude ambiguous (0O, 1lI) │
│                                 │
│        [Use This Password]      │  ← Fills password field + dismisses
└─────────────────────────────────┘
```

### Behavior

**Generation:**
- Uses `SecureRandom` to pick characters from the enabled character sets.
- At least one character from each enabled set is guaranteed (shuffle after generation).
- Regenerates automatically when any option changes.
- Default: length 16, all sets enabled, ambiguous excluded.

**Character sets:**
- Uppercase: `ABCDEFGHIJKLMNOPQRSTUVWXYZ`
- Lowercase: `abcdefghijklmnopqrstuvwxyz`
- Digits: `0123456789`
- Symbols: `!@#$%^&*()-_=+[]{}|;:',.<>?/~`
- Exclude ambiguous: removes `0O`, `1lI`, `` ` `` from enabled sets.

**Validation:**
- At least one character set must be enabled. If user tries to disable the last one, show toast and re-enable it.
- Minimum length: 8. Maximum: 64.

**"Use This Password":**
- Writes the generated password into the parent screen's password field as a `CharArray`.
- Dismisses the bottom sheet.

### Strength Indicator

Below the generated password, show a strength bar:
- Weak (red): <40 bits entropy
- Fair (orange): 40-60 bits
- Strong (green): 60-80 bits
- Very strong (dark green): >80 bits

Entropy calculated as: `length * log2(charset_size)`.

---

## 5. Recovery Token Display Screen

### Trigger
- After signup (automatic navigation).
- From Settings → "Generate New Recovery Token".

### Layout

```
┌─────────────────────────────────┐
│ ← Recovery Token                │
├─────────────────────────────────┤
│                                 │
│  ⚠ IMPORTANT                   │
│                                 │
│  Write down this recovery token │
│  and store it somewhere safe.   │
│  It is the ONLY way to recover  │
│  your account if you forget     │
│  your master password.          │
│                                 │
│  This token will NOT be shown   │
│  again.                         │
│                                 │
│ ┌─────────────────────────────┐ │
│ │                             │ │
│ │    ABCD-EFGH-JKLM-NPQR     │ │  ← Large monospace, high contrast
│ │                             │ │
│ └─────────────────────────────┘ │
│                                 │
│ ☐ I have written down this      │
│   token in a safe place         │
│                                 │
│       [Continue]                │  ← Disabled until checkbox checked
│                                 │
└─────────────────────────────────┘
```

### Behavior

**Display:**
- Token shown in large monospace font (minimum 24sp), high contrast (black on white background).
- Screen has `FLAG_SECURE` (like all screens), so no screenshots.
- No copy button — user must physically write it down. This is a deliberate UX choice to discourage digital storage of the token.

**Checkbox + Continue:**
- "Continue" button is disabled (grayed out) until the checkbox is checked.
- Tapping "Continue" calls `RecoveryToken.zeroize()` and navigates forward:
  - After signup: to Vault List.
  - From settings: back to Settings with a success snackbar.

**Back navigation:**
- Back button / gesture shows a confirmation dialog: "If you leave without saving your recovery token, you won't be able to see it again. Leave anyway?"
- If user confirms leave, token is zeroed and discarded.

### Recovery Token Management (from Settings)

Settings shows two options under a "Recovery Tokens" section:

```
┌─────────────────────────────────┐
│ Recovery Tokens                 │
├─────────────────────────────────┤
│                                 │
│ You have 1 recovery token(s).   │
│                                 │
│ [Replace Recovery Token]        │  ← Revokes old, generates new
│ [Add Additional Token]          │  ← Keeps old, adds new
│                                 │
└─────────────────────────────────┘
```

**Replace Recovery Token** (default action — for "I lost my token"):
1. User taps "Replace Recovery Token".
2. Confirmation dialog: "This will revoke your current recovery token. If you still have it written down, it will stop working. Continue?"
3. Prompt for master password (to re-verify identity and derive DEK).
4. Generate new recovery token + salt + wrapping.
5. `PUT /api/v1/auth/recovery/replace` with new `recovery_salt`, `recovery_auth_key`, `encrypted_dek_recovery`. Server deletes all existing recovery DEK rows and inserts the new one.
6. On success: navigate to Recovery Token Display Screen.
7. After user acknowledges: return to Settings with snackbar "Recovery token replaced."

**Add Additional Token** (for "I want a backup in a second location"):
1. User taps "Add Additional Token".
2. Prompt for master password.
3. Generate new recovery token + salt + wrapping.
4. `POST /api/v1/auth/recovery/add` with new `recovery_salt`, `recovery_auth_key`, `encrypted_dek_recovery`. Server inserts a new recovery DEK row alongside existing ones.
5. On success: navigate to Recovery Token Display Screen.
6. After user acknowledges: return to Settings with snackbar "Additional recovery token added. You now have N recovery token(s)."

**Token count:** The Settings screen shows the current number of recovery tokens (fetched from `GET /api/v1/auth/recovery/count`).

---

## 6. Common UI Patterns

### 6.1 Loading States

All network operations show:
- Inline `CircularProgressIndicator` for button actions (button text replaced with spinner).
- Full-screen loading overlay for initial data fetch (vault list on login).
- Skeleton loading for vault list on pull-to-refresh (entries show shimmer placeholders).

### 6.2 Error States

- Network errors: snackbar with "Retry" action at bottom of screen.
- Validation errors: inline text below the offending field, in `MaterialTheme.colorScheme.error`.
- Fatal errors (data integrity): full-screen error with "Contact Support" and "Log Out" buttons.

### 6.3 Confirmation Dialogs

Used for destructive actions:
- Delete entry
- Leave without saving changes
- Leave recovery token screen
- Log out

Format: `AlertDialog` with title, message, "Cancel" (dismiss) and "Confirm" (destructive, in error color) buttons.

### 6.4 Accessibility

- All interactive elements have `contentDescription`.
- Password fields use `semantics { password() }` to prevent TalkBack from reading them aloud.
- Minimum touch target: 48dp.
- Color contrast ratios meet WCAG AA (4.5:1 for text, 3:1 for large text).

### 6.5 Theme

- Material 3 dynamic color (Android 12+), with fallback to a static blue-grey palette.
- Dark mode supported, following system setting.
- No custom fonts — system default for consistency and performance.
