# X.509 Certificate Formats & Conversion Guide

A practical, single-page reference for X.509 certificate formats (PEM, DER, PKCS#7, PKCS#12, PKCS#8, JKS) and how to convert between them.

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
