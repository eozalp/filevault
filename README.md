# 🔒 FileVault

> **A self-contained, browser-based encrypted file vault. No server. No account. No cloud. Just a single `.vault.html` file that only you can open.**

---

## Table of Contents

- [Overview](#overview)
- [Core Philosophy](#core-philosophy)
- [Features](#features)
- [Cryptography & Security](#cryptography--security)
- [Getting Started](#getting-started)
  - [Creating Your First Vault](#creating-your-first-vault)
  - [Opening an Existing Vault](#opening-an-existing-vault)
- [Operation Modes](#operation-modes)
  - [📂 Open Mode](#-open-mode)
  - [⚡ Live Mode](#-live-mode)
- [Authentication](#authentication)
  - [Password Unlock](#password-unlock)
  - [Two-Factor Authentication (TOTP)](#two-factor-authentication-totp)
  - [ECDH Key-Based Unlock](#ecdh-key-based-unlock)
- [Working With Files](#working-with-files)
  - [Adding Files](#adding-files)
  - [Previewing Files](#previewing-files)
  - [Editing Text Files](#editing-text-files)
  - [Renaming & Removing Files](#renaming--removing-files)
  - [Downloading Files](#downloading-files)
- [Vault Features](#vault-features)
  - [Secure Note](#secure-note)
  - [Music Player](#music-player)
  - [Vault Sharing (ECDH)](#vault-sharing-ecdh)
  - [Vault Signing](#vault-signing)
  - [Idle Lock Timer](#idle-lock-timer)
- [Settings](#settings)
- [UI & Accessibility](#ui--accessibility)
- [Browser Compatibility](#browser-compatibility)
- [Technical Architecture](#technical-architecture)
- [Security Hardening Notes](#security-hardening-notes)
- [Known Limitations](#known-limitations)
- [License](#license)

---

## Overview

FileVault is a **zero-dependency, single-file web application** that lets you store, encrypt, and manage files entirely inside a self-contained `.vault.html` document. The vault file itself is the application — open it in any modern browser to decrypt and interact with its contents. Nothing is ever sent to a server.

The output of FileVault is a portable `.vault.html` file: a fully functional HTML document whose payload section holds all your encrypted data. You can store it on a USB drive, email it to yourself, or keep it in any cloud storage — the contents remain completely opaque without your password.

---

## Core Philosophy

| Principle | Implementation |
|-----------|----------------|
| **No network** | All crypto runs in the browser via [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API). No XHR, no fetch, no WebSocket. |
| **No accounts** | The vault file _is_ the identity. There is no registration, login service, or session management on any server. |
| **No dependencies** | Zero npm packages, zero CDN imports, zero external scripts. Everything ships in the single HTML file. |
| **Portable** | A `.vault.html` file works in Chrome, Firefox, Safari and any standards-compliant browser — on desktop, mobile, or an air-gapped machine. |
| **Local-first** | Your files never leave your device unless you explicitly share the vault. |

---

## Features

### 🔐 Encryption & Security
- **AES-256-GCM** encryption for all file payloads
- **PBKDF2** key derivation (SHA-256, high iteration count) from your password
- **Verification token** stored in metadata — unlocking with a wrong password fails fast without decrypting the full payload
- **Content Security Policy** (`default-src 'self' blob: data:`) injected via `<meta>` — blocks all external resource loads
- **Vault tamper detection** — code integrity check runs after every unlock (including the ECDH path)
- **Vault signing** — optional ECDSA digital signature on the vault; a badge shows ✅ Signed / ❌ Tampered / — Unsigned at open time

### 📁 File Management
- Add files via **drag-and-drop**, **file picker**, or **camera capture** (mobile)
- **Batch file addition** with background progress toast and cancellation support
- **Preview** images, video, audio, and text files in-app without extracting to disk
- **Inline text editing** — edit plaintext/code files in the preview pane and commit the change back into the vault
- **File rename** with single-tap ✏️ edit
- **File grid** view with category-coloured thumbnails (image, audio, video, doc, text, archive)
- **Search / filter** across all filenames in real time
- Incremental rendering for vaults with 300+ files to keep the UI responsive

### 🎵 Music Player
- Full-featured **aurora glass** music player surfaced from encrypted audio files
- Collapsible header with spinning disc art and track metadata
- **Playlist groups** — create named playlists, add/remove tracks, shuffle, loop
- Mini-player button in the nav bar for background listening
- Media Session API integration for lock-screen controls (where supported)

### 🔑 Multi-Factor & Key-Based Access
- **TOTP two-factor authentication** — SHA-1 · 6 digits · 30-second window · compatible with Google Authenticator, Authy, Bitwarden, 1Password
- **ECDH asymmetric unlock** — share the vault with a recipient by wrapping the AES key in their P-256 public key; they unlock with their private key without ever knowing your password
- **Vault sharing wizard** — 3-step UI: paste/load recipient public key → repack vault → share or download

### ⚡ Live Mode (File System Access API)
- On supported browsers, open the vault with a **writable file handle** and changes write back to disk immediately — no download step
- Falls back gracefully to **Open Mode** when the File System Access API is unavailable

### 🎨 Theming & Appearance
- **Auto dark mode** — follows OS preference (`prefers-color-scheme`)
- **Manual light/dark toggle** persisted in vault settings
- **Custom accent color** — pick any colour via a native color picker
- iOS-style design system: frosted-glass nav bar, bottom-sheet modals, HIG-compliant touch targets (44 × 44 pt minimum)

---

## Cryptography & Security

### Key Derivation

```
Password  ──┐
Salt (random 16 B) ──┤  PBKDF2-SHA256 (N iterations)  ──▶  Master Key (AES-256-GCM)
```

The derived key is used to:
1. Encrypt a small **verification blob** (`FILEVAULT_OK`) — fast fail on wrong password
2. Encrypt the **payload JSON** containing all file metadata and binary blobs

### File Payload Encryption

Each save operation:
1. Serialises the full file list (names, sizes, types, base64-encoded binary) into JSON
2. Encrypts the JSON with **AES-256-GCM** using a fresh 12-byte random IV
3. Encodes the ciphertext as Base64 and injects it into the vault's `<script id="v-meta">` tag
4. Optionally writes back in place (Live Mode) or downloads a new file (Open Mode)

### Chunked Processing

For large files, encryption and decryption run in chunks to avoid OOM on mobile devices:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `CHUNK_SIZE` | 4 MB | File read window |
| `ENC_CHUNK_SIZE` | 1 MB | Encryption chunk size |

### ECDH Sharing

```
Sender                              Recipient
──────                              ─────────
AES-256 master key                  P-256 key pair (generated offline)
     │                                   │
     │   ◀── recipient's public key ─────┘
     │
     ├── ECDH key agreement → shared secret
     ├── HKDF derive wrapping key
     └── AES-KW wrap(master key) → recipientBlock stored in vault metadata

Recipient opens vault:
  load private key → ECDH → HKDF → AES-KW unwrap → master key → decrypt vault
```

The vault can carry multiple `recipients` blocks, allowing several people to open the same vault with different private keys.

### Vault Signing (ECDSA)

An optional Ed25519/P-256 ECDSA signature over the vault's payload ciphertext is stored in metadata. On open, the signature is verified and a badge is displayed:

| Badge | Meaning |
|-------|---------|
| ✅ **Signed** | Signature present and valid |
| ❌ **Tampered** | Signature present but verification failed |
| — **Unsigned** | No signature in this vault |

---

## Getting Started

### Creating Your First Vault

1. Open `filevault_v5.html` in your browser (just double-click, or `file://` URL).
2. The app starts in **shell mode** — no vault loaded yet.
3. Choose a password and click **Create New Vault**.
4. Add files using the drop zone or file picker.
5. Click **Pack & Download Vault** to save your `.vault.html` file.

> **Tip:** Keep the downloaded `.vault.html` and discard the original `filevault_v5.html` — the vault file contains the full app.

### Opening an Existing Vault

1. Open your `.vault.html` file in a browser.
2. Choose **📂 Open Mode** or **⚡ Live Mode** (see below).
3. Select the same `.vault.html` file when prompted.
4. Enter your password (and TOTP code if 2FA is enabled).
5. Your files appear decrypted in the file list.

---

## Operation Modes

### 📂 Open Mode

Works in **every browser** including Safari, Firefox mobile, and `file://` origins.

- When you click **Pack & Download Vault**, a new `.vault.html` is downloaded.
- You must **replace the old file** with the newly downloaded one to persist changes.
- A **📍 Where Is My Vault?** modal appears after each download, with platform-specific instructions for finding the file in your Downloads folder.

### ⚡ Live Mode

Available in **Chromium-based browsers** (Chrome 86+, Edge 86+, Brave, Opera) on HTTPS or `localhost` origins.

- Uses the **File System Access API** (`showOpenFilePicker` with read/write permission).
- Changes are written **directly back to the original file** when you tap Save — no download step.
- A green toast confirms each successful write.
- Falls back silently to Open Mode if the API is unavailable (e.g., Firefox, iOS Safari).

| Feature | Open Mode | Live Mode |
|---------|-----------|-----------|
| Browser support | All | Chromium 86+ |
| Save mechanism | Download new file | Write to original |
| Works on `file://` | ✅ | ❌ |
| Works on iOS/Safari | ✅ | ❌ |
| Requires permission prompt | ❌ | ✅ (once per session) |

---

## Authentication

### Password Unlock

Enter your password in the **🔑 Password** field and tap **Unlock Vault**. The app derives the AES key via PBKDF2 and verifies against the stored verification blob before decrypting the full payload.

### Two-Factor Authentication (TOTP)

Enable TOTP from **⚙️ Settings → Two-Factor Auth**:

1. A random secret is generated and displayed as a copyable string.
2. On mobile, a deep-link button opens your authenticator app directly.
3. On desktop, enter the secret manually into Google Authenticator, Authy, Bitwarden, 1Password, or any RFC 6238-compatible app.
4. Enter the first 6-digit code to **verify** the setup — the secret is only saved after successful verification.

When TOTP is active, a second input field appears at unlock time. The live countdown timer shows the remaining seconds for the current code window, turning amber at 10 s and red at 5 s.

**TOTP parameters:** SHA-1, 6 digits, 30-second period, issuer tag `FileVault`.

### ECDH Key-Based Unlock

For password-free access or sharing scenarios:

1. The recipient generates a P-256 key pair and shares their **public key** (JSON file or paste).
2. The vault owner uses **↑ Share Vault** to wrap the AES master key with the recipient's public key and produce a re-packed vault.
3. The recipient opens the vault, loads their **private key** (paste or file), and unlocks without a password.

Key fingerprints are displayed for visual verification.

---

## Working With Files

### Adding Files

- **Drop zone** — drag files from your file manager onto the drop zone.
- **File picker** — tap **📁 Add Files** to open a native file picker (multi-select supported).
- **Camera capture** (mobile) — two buttons: **📷 Camera** (rear) and **🤳 Selfie** (front-facing) invoke the device camera directly.

Large batches (50+ files) are processed in controlled slices with a floating progress toast and a **Cancel** button.

### Previewing Files

Tap any file card (or the 🔍 icon in list view) to open the **Preview Modal**:

| File Type | Preview Behaviour |
|-----------|------------------|
| Images (JPEG, PNG, GIF, WebP, SVG…) | Rendered `<img>` element |
| Video (MP4, WebM, MOV…) | Native `<video>` player |
| Audio (MP3, AAC, FLAC, OGG…) | Native `<audio>` player |
| Plain text / code / JSON / CSV | Scrollable `<pre>` block (editable — see below) |
| PDF | Embedded `<iframe>` via blob URL |
| Other | Download prompt |

A progress bar with percentage is shown during decryption for all file types.

### Editing Text Files

When previewing a plaintext or code file, the preview becomes **editable**. A footer bar with two actions appears:

- **✅ Commit** — re-encrypts the edited content and replaces the file in the vault (staged until next save).
- **✗ Discard** — closes the editor without saving changes.

### Renaming & Removing Files

- **Rename:** Tap the ✏️ pencil icon on any file card. An inline input appears; press Enter or blur to confirm.
- **Remove:** Tap the ✖️ button on a file card. The card dims immediately (soft-delete); the file is purged on the next save.
- **Restore:** Tap the ✅ restore button on a soft-deleted card before saving.

### Downloading Files

Tap the ⬇️ download icon on any file card to decrypt and save that single file to your device.

---

## Vault Features

### Secure Note

A collapsible **🗒 Secure Note** section lives inside the vault. The note is encrypted with the same AES key as the files. A dot indicator on the section header shows whether a note is present. Useful for passwords, PINs, recovery codes, or any short sensitive text.

### Music Player

When the vault contains audio files, the **aurora glass** music player appears at the top of the unlocked view:

- **Collapsible header** with a spinning conic-gradient disc and current track info.
- **Playlist groups** — tap ＋ to create a named playlist; drag tracks between groups.
- **Controls:** play/pause, previous, next, shuffle, loop.
- **Media Session API** integration — lock-screen controls and "Now Playing" metadata on Android and iOS (where supported).
- **Mini-player button** in the nav bar keeps music playing while you browse the file list.
- Playlists with 120+ tracks are rendered incrementally (200 tracks per page) to prevent jank on large vaults.

### Vault Sharing (ECDH)

The **↑ Share Vault** button in the unlocked header opens a 3-step wizard:

| Step | Action |
|------|--------|
| 1 | Paste or load recipient's P-256 public key JSON; enter an optional label |
| 2 | Vault is re-packed with the recipient's wrapped key block added to metadata |
| 3 | Share via the native Share API (📤) or save to device (⬇️) |

The recipient receives a vault they can open with their private key — your password is never exposed.

### Vault Signing

In **⚙️ Settings → Signing**, generate an ECDSA key pair. The private key is exported as a JSON file for safekeeping. Subsequent saves sign the vault payload; the public key is embedded in the vault for verification on open.

### Idle Lock Timer

The vault auto-locks after a configurable period of inactivity (default: 5 minutes). The timer resets on any user interaction. Configure or disable from **⚙️ Settings → Auto-Lock**.

---

## Settings

Access via **⚙️** in the nav bar (visible when a vault is unlocked):

| Setting | Description |
|---------|-------------|
| **Vault Name** | Display name shown in the unlocked header |
| **Change Password** | Re-encrypts the vault with a new password |
| **Two-Factor Auth** | Enable/disable TOTP; manage the TOTP secret |
| **Signing** | Generate an ECDSA signing key pair; enable/disable signing |
| **Auto-Lock** | Set idle timeout duration (1 min – 1 hr, or Off) |
| **Accent Color** | Pick a custom accent color for the UI |
| **Theme** | Override system preference: Light / Dark / Auto |
| **Close Vault** | Lock and return to the open screen |

---

## UI & Accessibility

FileVault follows **Apple Human Interface Guidelines** for layout and interaction patterns, adapted for both mobile and desktop.

### Accessibility
- All interactive elements have `aria-label` or visible labels.
- Dynamic content regions use `role="status"` and `aria-live="polite"`.
- Modal dialogs use `role="dialog"`, `aria-modal="true"`, and safe `aria-hidden` management that avoids hiding focused elements.
- **Screen-reader-only** (`.sr-only`) labels on icon-only buttons.
- **Focus-visible** styles with a high-contrast 3 px accent-coloured outline for keyboard navigation.
- All touch targets meet the **44 × 44 pt minimum** HIG requirement.
- Full **`prefers-reduced-motion`** support — all animations are suppressed when the OS setting is active.

### Responsive Layout
| Breakpoint | Behaviour |
|------------|-----------|
| ≤ 375 px | Compact mode: smaller hero, tighter card grid, reduced font sizes |
| 375 – 599 px | Default mobile layout |
| ≥ 600 px | Tablet/desktop: centered card shell, dot-grid outer canvas, larger type, full-radius modals |

---

## Browser Compatibility

| Browser | Open Mode | Live Mode | TOTP | ECDH | Camera |
|---------|-----------|-----------|------|------|--------|
| Chrome / Edge 86+ (desktop) | ✅ | ✅ | ✅ | ✅ | N/A |
| Chrome Android | ✅ | ✅ | ✅ | ✅ | ✅ |
| Firefox (desktop) | ✅ | ❌ | ✅ | ✅ | N/A |
| Firefox Android | ✅ | ❌ | ✅ | ✅ | ✅ |
| Safari 15+ (macOS) | ✅ | ❌ | ✅ | ✅ | N/A |
| Safari (iOS 15+) | ✅ | ❌ | ✅ | ✅ | ✅ |
| Samsung Internet | ✅ | ✅* | ✅ | ✅ | ✅ |
| Brave | ✅ | ✅ | ✅ | ✅ | N/A |

\* Live Mode availability depends on the Samsung Internet version's File System Access API support.

> **Minimum requirement:** A browser with **Web Crypto API** support (`window.crypto.subtle`). All modern browsers since 2019 qualify.

---

## Technical Architecture

FileVault is architected as a **single IIFE** (Immediately Invoked Function Expression) with no module bundler, framework, or build step.

```
filevault_v5.html
├── <meta http-equiv="Content-Security-Policy">   ← XSS defence
├── <script id="v-meta" type="application/json">  ← Vault metadata / payload (null in shell)
├── <style>                                        ← ~2000 lines of CSS (design system + components)
├── <body>
│   ├── #app
│   │   ├── .nav-bar                               ← Sticky frosted-glass navigation
│   │   ├── #top-notices                           ← Collapsible notification area
│   │   ├── #prog                                  ← Progress bar (fixed on long ops)
│   │   ├── #ui-locked                             ← Lock screen (mode pick + auth)
│   │   ├── #ui-unlocked                           ← Vault view (file list + player + note)
│   │   └── Modals: preview, settings, share,      ← iOS bottom sheets
│   │            signing, whereabouts, vaultinfo,
│   │            help/tour
│   └── #toast                                     ← Floating status toasts
└── <script>                                       ← ~4500 lines of app logic
    ├── Environment detection (detectAppEnv)
    ├── OPFS temp cleanup
    ├── Crypto helpers (PBKDF2, AES-GCM, ECDH, ECDSA, TOTP)
    ├── State management (filesInVault, addedFiles, deletedIds, vaultSettings)
    ├── UI renderers (renderList, renderMusicPlayer, renderTOTPSettings…)
    ├── File operations (batchAddFiles, packVault, extractFile, previewFile…)
    ├── Live Mode (VAULT_FILE_HANDLE, writeBack)
    ├── Share wizard
    └── Interactive tour (self-contained slide engine, no dependencies)
```

### Key State Variables

| Variable | Type | Purpose |
|----------|------|---------|
| `AES_KEY` | `CryptoKey` | Decrypt-capable AES-256-GCM key |
| `AES_KEY_ENC` | `CryptoKey` | Encrypt-capable version of the same key |
| `STREAM_FILE` | `File` | The currently open `.vault.html` blob |
| `VAULT_FILE_HANDLE` | `FileSystemFileHandle` | Writable handle (Live Mode only) |
| `VAULT_PAYLOAD_OFFSET` | `number` | Byte offset of the `<!-- VAULT_PAYLOAD -->` marker |
| `filesInVault` | `Array` | Decrypted file metadata + base64 blobs |
| `addedFiles` | `File[]` | Staged new files not yet encrypted |
| `deletedIds` | `Set<string>` | Soft-deleted file IDs (O(1) lookup) |
| `vaultSettings` | `Object` | Name, theme, TOTP secret, accent color, idle timeout, note… |

### OPFS Usage

When available (`HAS_OPFS`), the Origin Private File System is used as a temporary write buffer during the vault repack operation, avoiding large in-memory `Blob` allocations. Any orphaned `fv_*` OPFS entries from crashed sessions are cleaned up on startup.

---

## Security Hardening Notes

The following mitigations were applied during development (selected):

- **`N14`** — ECDH unlock path now also runs `checkCodeIntegrity()` (was previously skipping tamper detection).
- **`PM3`** — OPFS temp file cleanup is gated on `HAS_OPFS` — prevents API errors on browsers or `file://` origins without OPFS.
- **`PS4`** — `deletedIds` uses a `Set` for O(1) membership tests (was an `Array.includes` O(n) scan).
- **`PS3`** — Files are sorted by insertion order after decryption to prevent order-based information leakage.
- **`PH6`** — Long decrypt and extract operations expose a cancellation flag (`_opCancelled`) to allow the user to abort mid-operation cleanly.
- **Picker guard** — A `_pickerActive` semaphore prevents concurrent `showOpenFilePicker`/`showSaveFilePicker` calls, which cause silent failures in some browsers.
- **`safeAriaHidden`** — The `aria-hidden` setter checks that no descendant of the target element currently holds focus before applying, preventing keyboard traps.
- **Settings allowlist** — `KNOWN_SETTINGS_KEYS` restricts which keys from a decrypted payload are applied to `vaultSettings`, preventing injection of arbitrary config via a crafted vault.

---

## Known Limitations

- **No key stretching UI feedback** — PBKDF2 iteration count is fixed at compile time. Very weak passwords cannot be warned against at key-derivation time.
- **Base64 payload overhead** — Binary files are stored as Base64 inside the JSON payload, adding ~33% size overhead. A future version may switch to a binary container format.
- **Single-threaded encryption** — Web Workers are not used; encryption of very large files (>500 MB) will block the main thread briefly between chunks. A Worker-based pipeline is planned.
- **Live Mode requires HTTPS or localhost** — The File System Access API is not available on `file://` origins or plain HTTP.
- **No incremental save** — Each save re-encrypts and re-serialises the entire payload. For vaults with many large files, this can take several seconds.
- **TOTP secret stored in vault** — The TOTP secret is part of the encrypted vault metadata. Anyone who knows your password can extract it. TOTP is a second factor against remote/automated attacks, not against someone who already has your password *and* your vault file.

---

## License

This project is released under the **MIT License**.

```
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

<div align="center">

**FileVault** — Your files, encrypted on your device.  
No accounts. No servers. No cloud. Just a file.

</div>
