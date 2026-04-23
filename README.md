# 🔒 Stealth-LSB Vault

> **Advanced LSB steganography tool — AES-256-GCM encrypted, 100% local, zero backend.**  
> Hide text or files inside innocent-looking images. Nothing ever leaves your browser.

[![License: MIT](https://img.shields.io/badge/License-MIT-emerald.svg)](LICENSE)
[![No dependencies](https://img.shields.io/badge/dependencies-zero-blue.svg)](index.html)
[![Works offline](https://img.shields.io/badge/offline-ready-success.svg)](index.html)

---

## ✨ Features

- **LSB Steganography** — imperceptibly modifies the Least Significant Bit of R, G, B channels in every pixel (Alpha channel is never touched).
- **AES-256-GCM encryption** — your payload is encrypted *before* injection. Even with the image, nothing can be read without the password.
- **PBKDF2-SHA256 key derivation** — 120 000 iterations with a random 128-bit salt per encode operation. Brute-force resistant.
- **Hide text or files** — encode a secret message or any binary file (PDF, ZIP, image…) up to the image's capacity.
- **Lossless PNG export** — the stego image is always saved as PNG. JPEG compression would destroy the hidden bits.
- **Web Worker offloading** — pixel-level bit manipulation runs on a background thread. The UI stays perfectly responsive during large operations.
- **Zero dependencies** — a single `index.html` file. No Node.js, no build step, no npm. Open and use.
- **No network calls** — the entire application runs client-side. Your secrets never touch a server.

---

## 🚀 Quick Start

```bash
# Option 1 — just open the file
open index.html

# Option 2 — serve locally (some browsers restrict file:// for Web Workers)
python3 -m http.server 8080
# then visit http://localhost:8080
```

No installation. No build. That's it.

---

## 🖥️ Usage

### Encoding (hiding data)

1. Switch to **ENCODE** mode (default).
2. Drag & drop a **cover image** (PNG, JPG, WEBP, BMP — any format works as input).
3. The **Capacity Monitor** tells you how many bytes that image can hide.
4. Choose your payload:
   - **Text tab** — type or paste your secret message.
   - **File tab** — select any file (≤ 10 MB).
5. Enter a strong **master password**.
6. Click **ENCODE & INJECT**.
7. The browser downloads a `*_steg.png` file — share this image.

### Decoding (extracting data)

1. Switch to **DECODE** mode.
2. Load the `*_steg.png` file via drag & drop.
3. Enter the **same password** used during encoding.
4. Click **EXTRACT & DECODE**.
5. If it was text — it appears in the result panel with a copy button.
   If it was a file — it downloads automatically (with its original filename and MIME type).

---

## 🔬 Binary Payload Format

Understanding the exact byte structure is essential for interoperability and forensic analysis.

### Structure embedded in LSBs

```
Offset   Length   Field
──────   ──────   ─────────────────────────────────────────────────────
  0        4      Magic: "SLSB" (0x53 0x4C 0x53 0x42)
  4        1      Version: 0x01
  5        1      Type: 0x00 = text  |  0x01 = file
  6       16      PBKDF2 Salt (random, 128-bit)
 22       12      AES-GCM IV (random, 96-bit)
 34        4      Encrypted data length (uint32, big-endian)
 38        N      AES-GCM ciphertext  (includes 16-byte auth tag)
──────   ──────
TOTAL FIXED HEADER: 38 bytes
```

### After AES-GCM decryption

```
Offset   Length   Field
──────   ──────   ─────────────────────────────────────────────────────
  0        4      Metadata JSON byte-length (uint32, big-endian)
  4        M      Metadata JSON (UTF-8 encoded)
 4+M       R      Raw payload: UTF-8 text  OR  binary file bytes
```

**Metadata JSON examples:**
```json
{ "type": "text" }
{ "type": "file", "filename": "secret.pdf", "mime": "application/pdf" }
```

### LSB bit indexing

For bit index `i` (0-based, MSB first per byte):
```
pixel   = Math.floor(i / 3)      →  which pixel
channel = i % 3                  →  0=R, 1=G, 2=B
bytePos = Math.floor(i / 8)      →  which byte of payload
bitPos  = 7 - (i % 8)            →  which bit (MSB first)
data[pixel * 4 + channel] = (data[pixel * 4 + channel] & 0xFE) | bit
```

**Capacity formula:** `⌊ width × height × 3 / 8 ⌋` bytes

---

## 🔐 Security Notes

| Aspect | Detail |
|---|---|
| Cipher | AES-256-GCM (authenticated encryption — detects tampering) |
| KDF | PBKDF2-SHA256, 120 000 iterations, 128-bit random salt |
| IV | 96-bit random, unique per encode operation |
| Auth tag | 128-bit (GCM default) — wrong password = immediate rejection |
| LSB impact | ±1 per channel at most — visually imperceptible |
| Image format | **Always export as PNG**. JPEG is lossy and will destroy all hidden data |
| Steganalysis | Standard Chi-square and RS attacks will detect LSB-modified images. This tool provides **encryption**, not steganalysis resistance. |

> **Important:** This tool provides strong *cryptographic* protection of the payload. A determined adversary with the stego image can statistically detect that *something* is hidden (via steganalysis), even if they cannot read it. For maximum deniability, use large cover images and keep the payload small relative to capacity.

---

## 🏗️ Technical Architecture

```
index.html (single file)
│
├── Vue 3 (CDN)           — reactive UI, state management
├── Tailwind CSS (CDN)    — styling, dark theme
│
├── Main Thread
│   ├── Web Crypto API    — PBKDF2, AES-GCM (crypto.subtle)
│   ├── Canvas API        — image loading, pixel access, PNG export
│   └── Vue App           — UI logic, user interactions
│
└── Web Worker (inline Blob)
    └── LSB Engine        — bit-level pixel manipulation (off main thread)
```

Data flow — encoding:
```
User input
  → TextEncoder / FileReader (ArrayBuffer)
  → buildEmbeddablePayload() → PBKDF2 salt+key → AES-GCM encrypt
  → Worker: encodeLSB(pixels, encryptedBlob)
  → Canvas.putImageData()
  → canvas.toBlob('image/png') → download
```

Data flow — decoding:
```
Stego PNG loaded to Canvas
  → Worker pass 1: extract 38-byte header → validate magic + read encLen
  → Worker pass 2: extract header + ciphertext
  → parseEmbeddablePayload() → PBKDF2 → AES-GCM decrypt
  → parse meta JSON → display text  OR  trigger file download
```

---

## 📁 Project Structure

```
stealth-lsb-vault/
├── index.html      ← The entire application (HTML + CSS + JS, ~500 lines)
├── README.md       ← This file
└── .gitignore
```

---

## 🛠️ Development

No build toolchain required. Edit `index.html` directly.

For a local dev server (recommended over `file://` to avoid Worker restrictions):

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .

# PHP
php -S localhost:8080
```

---

## ⚠️ Known Limitations

- **No steganalysis resistance** — standard statistical tools (StegExpose, SteghideDetect) can detect LSB modification. Payloads are strongly encrypted, but their *presence* may be detectable.
- **PNG only for decode** — if you re-save the stego image as JPEG (even once), the LSB data is destroyed permanently.
- **Max payload ~10 MB** — limited by the file input filter and browser memory.
- **No alpha channel used** — Alpha LSB is intentionally skipped to avoid transparency artifacts.
- **Browser security** — `crypto.subtle` requires a secure context (`https://` or `localhost`). Opening via `file://` may work in some browsers but is not guaranteed.

---

## 🤝 Contributing

Contributions are welcome. Some ideas for enhancement:

- [ ] Adaptive bit-depth encoding (2 or 3 LSBs per channel for higher capacity)
- [ ] JPEG quantization-aware encoding for lossy-resistant stego
- [ ] Drag & drop paste from clipboard (`Ctrl+V` image paste)
- [ ] Steganalysis detection mode (Chi-square test visual)
- [ ] Multi-image splitting for large payloads
- [ ] PWA support (offline service worker)

---

## 📜 License

MIT © 2024 — Free to use, modify, and distribute.

---

> *"The best hidden message is one whose existence is not suspected."*
