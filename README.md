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
