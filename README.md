## Phase 0 — CA Bookkeeping Structure
Before any cryptography, a real Certificate Authority needs a stateful ledger — not just a keypair. I set up:
- `index.txt` — the CA's database of every issued cert (status, expiry, serial, subject)
- `serial` / `crlnumber` — incrementing counters ensuring every cert and every CRL revision has a unique ID
- `certs/`, `newcerts/`, `crl/`, `private/` — storage conventions used by OpenSSL's `openssl ca` workflow

This is what separates a real CA (revocation-capable) from a one-off `openssl x509 -req` shortcut, which never writes to a ledger at all.

## Phase 1 — Root CA
Generated a 4096-bit RSA keypair, AES-256 encrypted at rest (`openssl genrsa -aes256`), then created a self-signed X.509 certificate (`openssl req -x509`) marking it as a CA via `basicConstraints: CA:TRUE` and restricting its key usage to `keyCertSign` + `cRLSign` only — the principle of least privilege applied to a cryptographic key.

Verified the result with `openssl x509 -text -noout`, confirming self-signature (Issuer == Subject) and matching SKI/AKI (expected only for a root, since it signs itself).
