# 🔐 SecretSplit

**Zero-knowledge file encryption with Shamir's Secret Sharing — single HTML file, no server, no dependencies.**

Encrypt any file and split the decryption key across multiple trusted parties. A configurable threshold (k-of-n) of share holders must cooperate to decrypt. No single party can recover the file alone (unless k = 1).

---

## Features

- **Shamir's Secret Sharing** over GF(256) — provably secure threshold scheme
- **AES-256-GCM** encryption via the browser's native Web Crypto API
- **PBKDF2** key derivation (200,000 iterations, SHA-256)
- **Zero server** — runs entirely from `file://` in any modern browser
- **ZIP packaging** — each trustee receives a ready-to-use package (share + encrypted file + decrypt-only app); one master ZIP bundles all packages
- **Trustee app** — the included `index.html` in each package is stripped to Decrypt & Reconstruct only
- **Pushover notifications** — optional alert when a file is successfully decrypted or when decryption fails
- **Dark / Light mode** — Forest Green + Teal palette, system auto default, persists via localStorage
- **Bilingual UI** — English / German toggle

---

## Usage

### Encrypt

1. Open `index.html` in Chrome or Firefox
2. Drop any file onto the drop zone
3. Set **k** (minimum shares needed to decrypt) and **n** (total shares to create)
4. Name each trustee
5. Optionally enter Pushover credentials for decryption notifications
6. Click **Encrypt & Generate Shares**
7. Click **Download All Packages** — one ZIP containing a per-trustee ZIP for each trustee

Each trustee ZIP contains:
- `share_<name>.txt` — their share file
- `<filename>.enc` — the encrypted file
- `index.html` — a decrypt-only version of the app

### Decrypt

1. Open `index.html` (either the main app or a trustee package)
2. Switch to the **Decrypt & Reconstruct** tab
3. Drop the `.enc` file
4. Drop at least **k** share files (from different trustees)
5. Click **Decrypt File** and download the restored file

---

## Cryptographic Design

```
File  ──►  AES-256-GCM  ──►  .enc  (raw ciphertext + 16-byte auth tag)
              ▲
        PBKDF2(masterSecret, salt, 200k, SHA-256)
              ▲
         masterSecret  (32 random bytes)
              │
     Shamir Split  k-of-n  over GF(256)
              │
    share_alice.txt  share_bob.txt  share_charlie.txt  …
```

### Share file format

```json
{
  "version": 1,
  "app": "SecretSplit",
  "trustee": "Alice",
  "shareIndex": 1,
  "k": 2,
  "n": 3,
  "shareData": "<base64 — 32 bytes>",
  "salt": "<base64 — 16 bytes>",
  "iv": "<base64 — 12 bytes>",
  "filename": "document.pdf",
  "pushover": "<base64 — AES-GCM ciphertext, encrypted with masterSecret>",
  "pushoverIv": "<base64 — 12 bytes>",
  "alertPushover": "<base64 — AES-GCM ciphertext, encrypted with shareData>",
  "alertPushoverIv": "<base64 — 12 bytes>"
}
```

### Encrypted file format

Raw binary — only the AES-GCM ciphertext with authentication tag appended. All metadata lives in the share files.

---

## Pushover Notifications

Two types of notification can be configured:

| Event | Trigger | Credential storage |
|---|---|---|
| Success | File successfully decrypted | Encrypted with `masterSecret` — requires k shares |
| Failure / Alert | Decryption attempt fails | Encrypted with per-share `shareData` — readable from 1 share |

Pushover credentials are never stored in plaintext. The success notification can only be sent after successful reconstruction (k shares needed); the alert notification is accessible to any single share holder.

---

## Security Notes

| Property | Detail |
|---|---|
| Master secret | 32 bytes from `crypto.getRandomValues` |
| KDF | PBKDF2-SHA256, 200 000 iterations |
| Encryption | AES-256-GCM — provides confidentiality and integrity |
| Secret sharing | Shamir over GF(2⁸) — information-theoretically secure |
| Key | Non-extractable `CryptoKey` — never touches JavaScript memory as raw bytes |
| Share leakage | Fewer than k shares reveal **zero** information about the secret |

**Threat model:** SecretSplit protects against an adversary who gains access to fewer than k share files. It does not protect against a compromised browser, a malicious operating system, or physical access to a device where shares are stored.

---

## Running the Tests

Open `test.html` in a browser. All 7 tests should pass (green):

| # | Test |
|---|---|
| 1 | GF(256) multiply — known values |
| 2 | Shamir 2-of-3 roundtrip |
| 3 | Shamir 3-of-5 roundtrip |
| 4 | Shamir 1-of-1 roundtrip |
| 5 | Under-threshold failure (1 share of 2-of-3) |
| 6 | AES-256-GCM roundtrip |
| 7 | PBKDF2 determinism |

---

## Browser Compatibility

Requires Web Crypto API (`crypto.subtle`). Supported in all modern browsers when served over HTTPS or from `file://`.

| Browser | Supported |
|---|---|
| Chrome 60+ | ✅ |
| Firefox 57+ | ✅ |
| Safari 15+ | ✅ |
| Edge 79+ | ✅ |

---

## License

GNU General Public License v3.0 — see [LICENSE](LICENSE) for details.
