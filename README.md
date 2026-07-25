# X.509 Certificate Formats & Conversion Guide

> **Status: Draft / Work in progress**  
> This reference is currently under development. Examples and recommendations may be updated as additional validation and testing are completed.

A practical, single-page reference for X.509 certificate formats (PEM, DER, PKCS#7, PKCS#12, PKCS#8, JKS) and how to convert between them.


## Table of Contents

- [Format Overview](#format-overview)
- [Conversions](#conversions)
  - [PEM → DER](#pem--der)
  - [DER → PEM](#der--pem)
  - [PEM Certificates → PKCS#7 (.p7b)](#pem-certificates--pkcs7-p7b)
  - [PKCS#7 → PEM Certificates](#pkcs7--pem-certificates)
  - [PEM → PKCS#12 (.p12/.pfx)](#pem--pkcs12-p12pfx)
  - [PKCS#12 → PEM](#pkcs12--pem)
  - [PKCS#1 ↔ PKCS#8 (private key)](#pkcs1--pkcs8-private-key)
  - [PKCS#12 ↔ JKS](#pkcs12--jks)
- [Building Certificate Chains / Bundles](#building-certificate-chains--bundles)



## Format Overview

| Format | Encoding | Contains | Typical extension |
|---|---|---|---|
| **PEM** | Base64 (ASCII, `-----BEGIN...-----`) | cert, key, chain, or all combined | `.pem` `.crt` `.cer` `.key` |
| **DER** | Raw binary ASN.1 | certificate, key, CSR, PKCS#7, etc. | `.der` `.cer` |
| **PKCS#7** | Base64 or binary | cert chain only, **no private key** | `.p7b` `.p7c` |
| **PKCS#12** | Binary, password-protected | cert + private key + chain, bundled | `.p12` `.pfx` |
| **PKCS#8** | Base64 or binary | private key, standard format | `.key` `.pem` |
| **JKS** | Proprietary binary, password-protected | Java-specific keystore (certs + keys) | `.jks` |

**PEM/DER are encodings** (how bytes are represented). **PKCS#7/PKCS#12/JKS are containers** (what's bundled together). A `.crt` file could be PEM or DER inside - the extension doesn't guarantee it.

## Conversions

### PEM → DER

**OpenSSL**
```bash
openssl x509 -in cert.pem -outform der -out cert.der
```

**Windows certutil**
```cmd
certutil -decode cert.pem cert.der
```
> `certutil -decode` Base64-decodes a PEM/Base64-encoded certificate to its binary DER representation. PEM headers are ignored during decoding.

**Python (cryptography)**
```python
from cryptography import x509
from cryptography.hazmat.primitives import serialization

with open("cert.pem", "rb") as f:
    cert = x509.load_pem_x509_certificate(f.read())

with open("cert.der", "wb") as f:
    f.write(cert.public_bytes(serialization.Encoding.DER))
```

---

### DER → PEM

**OpenSSL**
```bash
openssl x509 -in cert.der -inform der -out cert.pem -outform pem
```

**Windows certutil**
```cmd
certutil -encode cert.der cert.pem
```
> `certutil -encode` Base64-encodes the DER certificate and wraps it in standard `-----BEGIN CERTIFICATE-----` / `-----END CERTIFICATE-----` PEM headers.

---

### PEM Certificates → PKCS#7 (.p7b)

PKCS#7 holds certificates/chains only - **no private key**.

**OpenSSL - single certificate**
```bash
openssl crl2pkcs7 -nocrl -certfile cert.pem -out cert.p7b
```

**OpenSSL - full chain**
```bash
openssl crl2pkcs7 -nocrl -certfile cert.pem -certfile intermediate.pem -certfile root.pem -out chain.p7b
```

> By default, OpenSSL outputs a **PEM-encoded PKCS#7**. Add `-outform DER` to generate a binary PKCS#7 (`.p7b`), which is commonly used on Windows.

**Windows certutil**

> Not directly supported by a single `certutil` command. In practice, this is done via the Certificates MMC snap-in (`certmgr.msc`) → **Export** → **Cryptographic Message Syntax Standard – PKCS #7**, or scripted via PowerShell's `Export-Certificate`/CMS cmdlets rather than `certutil`.

---

### PKCS#7 → PEM Certificates

**OpenSSL**
```bash
openssl pkcs7 -in cert.p7b -print_certs -out certs.pem
```
> Extracts all certificates from the PKCS#7 container as individual `-----BEGIN CERTIFICATE-----` PEM blocks.

**Windows certutil**

> There is no direct `certutil` equivalent that extracts all certificates as individual PEM files.

```cmd
certutil -dump cert.p7b
```

> `certutil -dump` displays the certificate(s) contained in the PKCS#7 container for inspection, but it does not export them as PEM certificates. Use OpenSSL when you need PEM-encoded certificates.

---

### PEM → PKCS#12 (.p12/.pfx)

**OpenSSL - certificate + private key**
```bash
openssl pkcs12 -export -in cert.pem -inkey key.pem -out cert.p12 -name "my-cert"
```

**OpenSSL — certificate + private key + chain**
```bash
openssl pkcs12 -export -in cert.pem -inkey key.pem -certfile chain.pem -out cert.p12 -name "my-cert"
```
> `chain.pem` typically contains one or more intermediate CA certificates (and optionally the root CA). The private key specified by `-inkey` must match the certificate.

**Windows certutil**

> Not directly supported for building a PKCS#12 from a standalone PEM certificate and private key. `certutil -mergePFX` only combines existing PKCS#12 (PFX) files. The typical Windows-native approach is to import the certificate and private key into the certificate store, then use PowerShell's `Export-PfxCertificate`. For direct PEM certificate + key → PKCS#12 conversion, OpenSSL is the standard tool, including on Windows.

---

### PKCS#12 → PEM

**OpenSSL - everything (cert + key + chain) into one file**
```bash
openssl pkcs12 -in cert.p12 -out bundle.pem -nodes
```
> Exports the private key, leaf certificate, and CA certificates (if present) into a single PEM file. Remove -nodes (or use -noenc in newer OpenSSL versions) to keep the private key encrypted.

**OpenSSL - certificate only**
```bash
openssl pkcs12 -in cert.p12 -clcerts -nokeys -out cert.pem
```
> Extracts the leaf certificate and excludes the private key and CA certificates.

**OpenSSL - private key only**
```bash
openssl pkcs12 -in cert.p12 -nocerts -nodes -out key.pem
```
> Extracts the private key. Remove -nodes (or use -noenc in newer OpenSSL versions) to keep the exported private key encrypted.

**OpenSSL - CA chain only (excludes leaf certificate)**
```bash
openssl pkcs12 -in cert.p12 -cacerts -nokeys -out chain.pem
```
> Extracts only CA certificates stored in the PKCS#12 container. The leaf certificate is excluded.

**OpenSSL 3.x - legacy PKCS#12 files**
```
openssl pkcs12 -legacy -in cert.p12 -cacerts -nokeys -out chain.pem
```
>OpenSSL 3.x disables some legacy PKCS#12 algorithms by default. If the .p12/.pfx file was created by older tools (for example, older Windows certificate exports) and uses algorithms such as RC2 or 3DES, OpenSSL may fail with an algorithm error. Add -legacy to enable support for legacy PKCS#12 algorithms.

**Windows certutil**
```cmd
certutil -exportPFX -p "yourpassword" My "CertCommonNameOrSerial" cert.p12
```
> certutil -exportPFX exports a certificate and private key already stored in the Windows certificate store (for example, the My store). It does not decode or convert an existing standalone .p12/.pfx file into PEM. For standalone PKCS#12 → PEM conversion, use OpenSSL.


---

### PKCS#1 ↔ PKCS#8 (private key)

PKCS#1 (`BEGIN RSA PRIVATE KEY`) is RSA-only, legacy.
PKCS#8 (`-----BEGIN PRIVATE KEY-----` or `-----BEGIN ENCRYPTED PRIVATE KEY-----`) is a standard algorithm-independent private key format that supports RSA, EC, Ed25519, and other algorithms. Modern tools and libraries commonly prefer PKCS#8.

**OpenSSL - PKCS#1 → PKCS#8, unencrypted**
```bash
openssl pkcs8 -topk8 -nocrypt -in rsa_pkcs1.pem -out pkcs8.pem
```

**OpenSSL - PKCS#1 → PKCS#8, encrypted output**
```bash
openssl pkcs8 -topk8 -in rsa_pkcs1.pem -out pkcs8_encrypted.pem
```
>Creates an encrypted PKCS#8 private key (-----BEGIN ENCRYPTED PRIVATE KEY-----) protected with a passphrase.

**OpenSSL - PKCS#8 → PKCS#1** (RSA only)
```bash
openssl rsa -in pkcs8.pem -out pkcs1.pem
```
>Converts an RSA private key from PKCS#8 to traditional PKCS#1 format (-----BEGIN RSA PRIVATE KEY-----). Not applicable to EC, Ed25519, or other non-RSA keys.

**Windows certutil**
> Not applicable — Windows CryptoAPI/CNG manages private keys independently of PEM container formats and does not provide a direct PKCS#1 ↔ PKCS#8 conversion workflow. This distinction is mainly relevant in OpenSSL and PEM-based environments.


---

### PKCS#12 ↔ JKS

**OpenSSL**
> Not applicable - OpenSSL has no JKS support; JKS is a Java-specific keystore format.

**Windows certutil**
> Not applicable - JKS isn't used by Windows CryptoAPI/CNG; it's exclusively a Java Keystore format.


---

## Building Certificate Chains / Bundles

Order matters: **leaf → intermediate(s) → root**, in that sequence, in one file.

```bash
cat leaf.pem intermediate.pem root.pem > fullchain.pem
```

For servers (nginx/Apache), the root is usually **omitted** - clients already trust it via their OS/browser trust store:

```bash
cat leaf.pem intermediate.pem > fullchain.pem
```

For a P12 that includes the chain:

```bash
openssl pkcs12 -export -in leaf.pem -inkey key.pem -certfile intermediate.pem -out bundle.p12
```

---
