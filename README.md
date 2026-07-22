# X.509 Certificate Formats & Conversion Guide

A practical, single-page reference for X.509 certificate formats (PEM, DER, PKCS#7, PKCS#12, PKCS#8, JKS) and how to convert between them.

## Format Overview

| Format | Encoding | Contains | Typical extension |
|---|---|---|---|
| **PEM** | Base64 (ASCII, `-----BEGIN...-----`) | cert, key, chain, or all combined | `.pem` `.crt` `.cer` `.key` |
| **DER** | Raw binary | single cert or key | `.der` `.cer` |
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
> `certutil -decode` strips the `-----BEGIN/END-----` markers and base64-decodes the body - this is the correct direction for PEM→DER.


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
> `certutil -encode` base64-encodes binary input and wraps it with PEM headers - correct direction for DER→PEM.


---

### PEM/DER → PKCS#7 (.p7b)

PKCS#7 holds certs/chains only - no private key.

**OpenSSL - single cert**
```bash
openssl crl2pkcs7 -nocrl -certfile cert.pem -out cert.p7b
```

**OpenSSL - full chain**
```bash
openssl crl2pkcs7 -nocrl -certfile cert.pem -certfile intermediate.pem -certfile root.pem -out chain.p7b
```

**Windows certutil**
> Not directly supported by a single certutil command. In practice this is done via the Certificates MMC snap-in (`certmgr.msc`) → Export → "Cryptographic Message Syntax Standard - PKCS #7", or scripted via PowerShell's `Export-Certificate`/CMS cmdlets rather than certutil.

