# TikTok Reverse Engineering - Mobile and Web API


A technical overview of TikTok's internal API security, covering the signature algorithms used to authenticate client-server communication. This repository provides Python implementations for both Mobile and Web signing pipelines, as well as the TTEncrypt layer.

**Disclaimer:** Interacting with private APIs without authorization may violate the terms of service of the platform. Proceed with caution and at your own risk.

#### Update 28/03/2026 : Every algorithm (Mobile/Web) has been updated. Thanks for the stars ❤.
Adding other more useful things at 500 Stars 💕

---

## Project Structure

```
Mobile/          # Mobile signing pipeline (Android/TikTok app)
├── metasec.py   # Main Metasec class — entry point for all mobile signatures
├── native.py    # Bit manipulation utilities (rotate, reverse, byteswap…)
├── exception.py # Custom exceptions
├── cipher/
│   ├── AES.py   # AES wrapper (OFB mode)
│   ├── RC4.py   # RC4 stream cipher
│   └── SIMON.py # SIMON block cipher (128-bit, 72 rounds)
├── helpers/
│   ├── argus.py # X-Argus generation (protobuf + SM3 + SIMON + AES)
│   └── ladon.py # X-Ladon key derivation and encoding
└── protobuf/
    └── protobuf.py # Custom protobuf serializer/reader

Web/             # Web signing pipeline (browser / tiktok.com)
├── bogus.py     # X-Bogus signature (double-MD5 + RC4 + shifted base64)
├── gnarly.py    # ChaCha20-based PRNG used for web entropy
├── compress.py  # LZW compressor
├── base.py      # Node.js-compatible base64 encode/decode
└── ressource.py # RC4 + execjs base64 helper (shifted alphabet)

TTEncrypt/
└── ttencrypt.py # TT payload encryption (AES + custom S-boxes + gzip)
```

---

## 1. Mobile — `Metasec`

The `Metasec` class (`Mobile/metasec.py`) is the single entry point for signing mobile API requests. It produces four headers in one call:

| Header | Description |
|--------|-------------|
| `X-Argus` | Main mobile signature — protobuf → SM3 → SIMON → AES |
| `X-Gorgon` | Secondary integrity header derived from params + payload + cookies |
| `X-Ladon` | Lightweight token bound to `app_id` and `license_id` |
| `X-Khronos` | Current Unix timestamp |

### Quick usage

```python
from Mobile import Metasec

signer = Metasec()  # Uses built-in default SIGN_KEY
# Or provide your own key:
# signer = Metasec(key="<base64-encoded-32-byte-key>")

headers = signer.sign(
    url="https://api.tiktokv.com/aweme/v1/feed/?aid=1233&...",
    app_id=1233,
    app_version="25.1.1",
    app_launch_time=1700000000,
    device_type="SM-G973N",
    sdk_version="v04.04.09-boa-hotfix",
    sdk_version_code=44409,
    license_id=123456,
    device_id="1234567890123456789",   # optional
    device_token="...",                # optional
    dyn_seed="...",                    # optional (versions 1-8 supported)
    dyn_version=1,                     # optional
    payload=None,                      # hex string or None
    cookies=None                       # raw cookie string or None
)
# headers = {"x-argus": "...", "x-gorgon": "...", "x-ladon": "...", "x-khronos": "..."}
```

### X-Argus internals

1. **Protobuf serialization** — device/app/request metadata packed with the custom `ProtoBuf` encoder.
2. **SM3 hashing** — query string and body each hashed with the Chinese SM3 standard (similar to SHA-256).
3. **SIMON cipher** — serialized protobuf encrypted with SIMON-128/256, 72 rounds.
4. **AES-OFB** — SIMON output further encrypted with AES (key/IV derived from `SIGN_KEY`).
5. **Base64 output** — prefixed with two control bytes, then standard base64-encoded.

### Dynamic seed (`dyn_encode`)

`dyn_version` 1–8 are supported. Each version applies a different combination of MD5 hashing, byte reversal, XOR masks, and bit manipulation to produce the `dyn` field embedded in the protobuf.

---

## 2. Web — X-Bogus

`Web/bogus.py` (`Signer` class) signs browser-side requests appended as `&X-Bogus=<value>`.

**Pipeline:**
1. Double-MD5 of URL params and request body.
2. RC4-encrypt the User-Agent with key `[0,1,14]`, then base64 + MD5.
3. Assemble a salt array (timestamp, magic `536919696`, hashes).
4. Filter → scramble → RC4-encrypt (key `[255]`).
5. Prefix `\x02\xFF`, encode with shifted base64 alphabet `Dkdpgh4ZKsQB80/Mfvw36XI1R25-WUAlEi7NLboqYTOPuzmFjJnryx9HVGcaStCe`.

### Other web modules

| File | Role |
|------|------|
| `gnarly.py` | ChaCha20-based PRNG seeded from timestamp + random values |
| `compress.py` | LZW compressor used to shrink payloads before signing |
| `base.py` | Node.js-compatible base64 (matches JS `btoa` byte-for-byte) |
| `ressource.py` | RC4 + execjs `btoa` + shifted-alphabet re-encode |

---

## 3. TTEncrypt

`TTEncrypt/ttencrypt.py` encrypts the raw request payload before it is sent. It uses:

- A custom AES-based cipher with four hardcoded S-box tables (`dword_0`–`dword_3`).
- gzip compression of the plaintext before encryption.
- A fixed magic header `[0x74, 0x63, 0x05, 0x10, 0x00, 0x00]`.

---

## 4. Operational Notes

-   **IP Reputation** — Residential IPs are treated with much less scrutiny than datacenter ranges.
-   **Device warm-up** — A fresh `device_id` must be activated through a series of realistic requests before core endpoints return valid data.
-   **Algorithm versioning** — Constants and logic change with each app update; continuous RE is required to stay current.
-   **Geographic restrictions** — Many endpoints behave differently or fail based on the client's IP region.


