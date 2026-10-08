# VaultKeep — Password Manager

A single-file, zero-dependency, **client-side password manager** built with vanilla HTML/CSS/JavaScript and the native Web Crypto API. All vault data is encrypted in the browser with a key derived from your master password — the app never sees or stores your plaintext secrets.

> Cybersecurity learning project. See [Limitations](#limitations) before using it with real accounts.

---

## Table of Contents

- [Features](#features)
- [Getting Started](#getting-started)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Usage Guide](#usage-guide)
- [Security Model](#security-model)
- [Configuration](#configuration)
- [Data Storage](#data-storage)
- [Keyboard & Accessibility](#keyboard--accessibility)
- [Limitations](#limitations)
- [License](#license)

---

## Features

### Security
- **Zero-knowledge encryption** — master password is never stored; plaintext exists only in memory while unlocked
- **PBKDF2-SHA256** key derivation with **600,000 iterations** and a unique 16-byte random salt per user
- **AES-256-GCM** authenticated encryption with a fresh random 12-byte IV on every save
- **Non-extractable keys** — the derived key cannot be exported from the browser
- **Timing-attack mitigation** on login (dummy key derivation for nonexistent usernames)
- **Exponential lockout** after 5 failed logins: 30s → 60s → 120s … capped at 900s
- **Generic error messages** — never reveals whether a username exists
- **Clipboard auto-clear** 20 seconds after copying a password
- **Inactivity auto-lock** (5/10/15/30 min or never; default 10 min)
- **Strict Content-Security-Policy** (`default-src 'none'`)
- No hand-written cryptography — only the native Web Crypto API

### Password Generator
- 8–64 character length slider with four selectable character sets
- CSPRNG (`crypto.getRandomValues`) with rejection sampling — no `Math.random()`
- Unbiased Fisher–Yates shuffle; guarantees at least one character from each selected set
- Ambiguous characters (`0/O`, `1/l/I`) excluded by default
- Live strength meter and weak-password detection (dictionary, sequences, repeats)

### Vault
- Add, edit, search, filter, and delete credentials with categories (Social, Banking, Education, Work, Shopping, Other)
- Show/hide and copy passwords per entry; inline password generation
- **Reused-password detection** with badges
- **Security dashboard** with a 0–100 score ring and five health metrics
- **Audit log** (capped at 200 events + failed login attempts) — never logs secrets
- **Change master password** in place (re-encrypts the whole vault with a new salt)
- Dark/light theme (manual toggle + system preference), responsive mobile layout
- Accessible: ARIA labels, live regions, `:focus-visible` outlines, `prefers-reduced-motion` support

---

## Getting Started

### Requirements
- Any modern browser (Chrome, Edge, Firefox, Safari) with Web Crypto API support
- No installation, no dependencies, no build step, no server

### Run

```powershell
# Option 1 — open the file (Web Crypto may require HTTPS or localhost)
start index.html

# Option 2 — serve locally (optional)
python -m http.server 8080
# then open http://localhost:8080/
```

### Deploy

The project is deployed to GitHub Pages from the `main` branch by the workflow in `.github/workflows/pages.yml`. On the repository's **Settings → Pages**, set the build and deployment source to **GitHub Actions**. Each push to `main` then publishes `index.html` at:

https://gowdavikitha886-hue.github.io/password-manager/

Use the HTTPS site so browsers enable the Web Crypto API. Vault data remains in that browser's local storage and is not synchronized.

---

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                       REGISTRATION                          │
│                                                             │
│  master password ──┐                                        │
│                    ├─► PBKDF2-SHA256 (600k iters, salt)     │
│  random 16B salt ──┘              │                         │
│                                   ▼                         │
│                        AES-256-GCM key                      │
│                        (non-extractable)                    │
│                                   │                         │
│              vault JSON ──────────┴──► ciphertext ──►        │
│                                       localStorage          │
├─────────────────────────────────────────────────────────────┤
│                          LOGIN                              │
│                                                             │
│  master password + stored salt ─► re-derive key ─► decrypt  │
│      success = correct password · failure = wrong password  │
└─────────────────────────────────────────────────────────────┘
```

1. **Register** — choose a username and a master password (≥12 chars, ≥ Medium strength). A random salt is generated and stored alongside your encrypted vault.
2. **Derive** — PBKDF2-SHA256 stretches the master password into an AES-256-GCM key that is never persisted or exported.
3. **Encrypt** — the entire vault (credentials, audit log, settings) is encrypted with a fresh IV on every save and written as ciphertext to `localStorage`.
4. **Unlock** — on login, the key is re-derived from the entered password and the stored salt. Decryption failure means the password is wrong.
5. **Auto-lock** — on timeout or logout, the key and all plaintext are cleared from memory.

---

## Project Structure

```
password-manager/
├── .github/workflows/pages.yml
├── index.html                    # The entire application
├── README.md
└── VaultKeep-Documentation.pdf
```

### Internal layout of `index.html`

| Lines | Section | Responsibility |
|------:|---------|----------------|
| 1–7 | Document head | Meta tags, CSP policy, title |
| 8–47 | Styles | CSS variables, light/dark themes, responsive layout, dialogs, toasts |
| 53–76 | Crypto helpers | Key derivation, encrypt/decrypt, CSPRNG, password generator, strength scoring |
| 77–81 | Storage layer | `localStorage` read/write, app state, save, audit logging |
| 82–104 | Auth & session | Register, login with lockout, lock/unlock, activity listeners |
| 105–113 | UI utilities | `h()` DOM builder, toasts, strength meter, clipboard, date formatting |
| 114–126 | Auth view | Login/register screens with live strength meter |
| 128–131 | App shell | Sidebar navigation and theme toggle |
| 132–147 | Credential dialog | Add/edit form with validation and inline generation |
| 148–162 | Vault view | Searchable, filterable credential list with reuse badges |
| 163–167 | Generator view | Interactive password generator |
| 168–173 | Security view | Score ring, stats, recent events |
| 174–175 | Audit view | Activity and failed-login table |
| 176–185 | Settings view | Auto-lock interval, master password change |
| 186–188 | Concepts view | Built-in educational page on the security concepts used |
| 189–191 | Bootstrap | Root renderer and theme initialization |

---

## Usage Guide

### Getting started
1. Open the file in a browser.
2. Switch to the **Register** tab, enter a username and a strong master password.
3. Your vault is created empty and encrypted immediately.

### Adding a credential
1. Go to **Vault → Add credential**.
2. Fill in name, URL (must start with `http://` or `https://`), username, and password — or click **Generate** to create one.
3. Pick a category, optionally add notes, then save.

### Generating a password
1. Open **Generator**.
2. Adjust the length slider and toggle character sets.
3. Click **Copy** (auto-clears after 20s) or **Use in new credential**.

### Checking your security posture
- Open **Security** for a 0–100 score ring, counts of weak/reused/old passwords, and recent activity.
- Open **Audit** for a full history of vault events and failed login attempts.

### Changing your master password
1. Open **Settings → Change master password**.
2. Verify your current password, then enter a new one (≥12 chars, ≥ Medium).
3. The vault is re-encrypted with a **new salt** — previous keys become useless.

### Locking
- Click **Lock/Logout** in the sidebar, or let the inactivity timer lock automatically.

---

## Security Model

| Threat | Mitigation |
|--------|-----------|
| Offline attack on stored data | 600k-iteration PBKDF2 makes brute-forcing expensive |
| Credential stuffing / wrong guesses | Exponential lockout after 5 failures (30s → 900s cap) |
| Username enumeration | Identical error text; dummy key derivation for unknown users |
| Timing attacks on login | Dummy derivation normalizes response time |
| Data tampering | AES-GCM authentication tag detects any modification |
| IV reuse | Fresh random 96-bit IV generated on every save |
| Key extraction | Keys marked non-extractable by Web Crypto |
| Clipboard shoulder-surfing | Copied secrets cleared after 20 seconds |
| Unattended session | Inactivity auto-lock (default 10 min) |
| Malicious resource loading | `default-src 'none'` CSP; no external resources |
| Weak user passwords | 12-char minimum + strength meter + weak-password dictionary |

### Cryptographic parameters

| Parameter | Value |
|-----------|-------|
| KDF | PBKDF2-SHA256 |
| Iterations | 600,000 |
| Salt | 16 random bytes (per user) |
| Cipher | AES-256-GCM |
| IV | 12 random bytes (per encryption) |
| Key type | Non-extractable, in-memory only |
| RNG | `crypto.getRandomValues` (rejection sampling) |

---

## Configuration

All configuration is in-file — there are no environment variables.

| Setting | Location | Default | Notes |
|---------|----------|---------|-------|
| `ITER` | line 54 | `600000` | PBKDF2 iterations — **changing this breaks existing vaults** |
| Auto-lock interval | in-app Settings | 10 min | 5 / 10 / 15 / 30 / Never |
| Theme | in-app toggle | system preference | Persisted in `localStorage["pm_theme"]` |
| CSP policy | line 6 meta tag | `default-src 'none'` | Restricts all external resource loading |
| Weak-password dictionary | `COMMON`, line 70 | 10 entries | Local common-password check |

---

## Data Storage

Everything lives in `localStorage` under two keys:

| Key | Contents |
|-----|----------|
| `pm_users` | `{ [username]: { salt, iter, fails, until, failTimes, blob } }` — `blob` is base64 ciphertext of the entire vault |
| `pm_theme` | `"dark"` or `"light"` |

The decrypted vault structure:

```json
{
  "items": [
    {
      "id": "uuid",
      "name": "Example",
      "url": "https://example.com",
      "user": "alice",
      "pw": "secret",
      "cat": "Work",
      "notes": "",
      "changed": 1696800000000
    }
  ],
  "log": [{ "t": 1696800000000, "e": "login" }],
  "settings": { "lock": 10 }
}
```

Plaintext never touches disk — only ciphertext is persisted.

---

## Keyboard & Accessibility

- Semantic form labels and buttons throughout
- `role="alert"` for errors, `role="status"` for toasts (announced by screen readers)
- Visible `:focus-visible` outlines on all interactive elements
- `prefers-reduced-motion` disables animations
- Responsive layout: horizontal navigation on screens ≤800px

---

## Limitations

This is an **educational project**, stated honestly both here and in the in-app Concepts page:

- A browser-only app cannot defend against malware, keyloggers, or XSS running on the same device.
- `localStorage` can be cleared by the user, browser cleanup, or a different browser profile — **export your data elsewhere** if you need backups.
- There is no synchronization, import/export, or multi-device support.
- No independent security audit has been performed.
- **Do not use it for real accounts.** For production use, prefer a vetted manager (Bitwarden, KeePass, 1Password).

---

## Testing

No automated test suite exists. Recommended manual checks:

1. Register → log out → log in with the correct password.
2. Log in with a wrong password 5 times and confirm the lockout timer engages.
3. Add a credential → reload the page → confirm it persists and decrypts.
4. Copy a password and confirm the clipboard clears after 20 seconds.
5. Wait for the auto-lock timeout and confirm plaintext is cleared.
6. Change the master password and confirm old password no longer works.

---

## License

No license file is included. All rights reserved by the author by default. This is an academic/cybersecurity coursework project.
