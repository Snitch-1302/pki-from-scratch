# PKI From Scratch

A hand-built Public Key Infrastructure using Python + OpenSSL: Root CA →
Intermediate CA → end-entity certificate issuance → revocation → CRL
generation.

Read the story behind this project (motivation, design decisions, lessons
learned) here: **[Hashnode article link]**

---

## What this is

A three-tier certificate authority hierarchy built manually with OpenSSL's
config-file-driven `openssl ca` workflow (not the stateless `-req`
shortcut), orchestrated via Python. Covers the full certificate lifecycle:
issuance, verification, revocation, and CRL publication.

```
Root CA (offline trust anchor, self-signed)
   └── Intermediate CA (signs end-entity certs)
         └── End-entity certificates (server / client)
```

## Tech stack

- **OpenSSL** — CA operations, key generation, signing, revocation, CRLs
- **Python 3.x** — orchestration of OpenSSL calls via `subprocess`
- **Git Bash** (Windows) — shell environment

## Project structure

```
pki-from-scratch/
├── rootca/
│   ├── certs/            # issued certs
│   ├── crl/               # generated CRLs
│   ├── newcerts/          # archive of every cert ever issued
│   ├── private/           # root CA private key (gitignored)
│   ├── openssl.cnf        # root CA config
│   ├── root-ca.crt        # root CA public certificate
│   ├── index.txt          # CA ledger (gitignored)
│   ├── serial              # next serial number (gitignored)
│   └── crlnumber           # next CRL number (gitignored)
├── intermediateca/
│   └── ...                 # same structure as rootca/
├── scripts/                 # Python orchestration scripts
└── README.md
```

> Private keys and CA ledger state (`index.txt`, `serial`, `crlnumber`) are
> excluded via `.gitignore`. Only public certs and configs are tracked.
> All Subject DNs used throughout are placeholders (e.g. `Learning Lab`,
> `PKI Lab Root CA`) — no real personal or organizational data.

## Prerequisites

- OpenSSL (v1.1.1+ or v3.x)
- Python 3.8+
- Git Bash or any POSIX-compatible shell (Windows users: avoid PowerShell
  for the setup commands below — brace expansion syntax differs)

## Setup

Clone the repo:

```bash
git clone https://github.com/<your-username>/pki-from-scratch.git
cd pki-from-scratch
```

### 1. Root CA

```bash
cd rootca
openssl genrsa -aes256 -out private/root-ca.key 4096
openssl req -x509 -new -key private/root-ca.key -sha256 -days 3650 \
  -out root-ca.crt -config openssl.cnf -extensions v3_ca
```

Verify:

```bash
openssl x509 -in root-ca.crt -text -noout
```

### 2. Intermediate CA

```bash
openssl genrsa -aes256 -out intermediateca/private/intermediate-ca.key 4096

openssl req -new -key intermediateca/private/intermediate-ca.key \
  -out intermediateca/intermediate-ca.csr -config intermediateca/openssl.cnf

openssl ca -config rootca/openssl.cnf -extensions v3_intermediate_ca \
  -days 1825 -notext -md sha256 \
  -in intermediateca/intermediate-ca.csr -out intermediateca/intermediate-ca.crt
```

Build the chain file (most specific first, root last):

```bash
cat intermediateca/intermediate-ca.crt rootca/root-ca.crt > intermediateca/ca-chain.crt
```

Verify:

```bash
openssl x509 -in intermediateca/intermediate-ca.crt -text -noout
```

Confirm the Intermediate's `Authority Key Identifier` matches the Root's
`Subject Key Identifier`, and `Basic Constraints` shows `CA:TRUE,
pathlen:0` (can sign end-entity certs, cannot create further sub-CAs).

### 3. Issue an end-entity certificate

```bash
openssl genrsa -out intermediateca/private/server.key 2048

MSYS_NO_PATHCONV=1 openssl req -new -key intermediateca/private/server.key \
  -out intermediateca/server.csr \
  -subj "/C=US/ST=California/O=Learning Lab/CN=server.pkilab.local" \
  -addext "subjectAltName=DNS:server.pkilab.local"

openssl ca -config intermediateca/openssl.cnf -extensions server_cert \
  -days 365 -notext -md sha256 \
  -in intermediateca/server.csr -out intermediateca/server.crt
```

Verify:

```bash
openssl x509 -in intermediateca/server.crt -text -noout
```

Confirm `Basic Constraints: CA:FALSE`, `Extended Key Usage: TLS Web Server
Authentication`, and `Subject Alternative Name: DNS:server.pkilab.local`
are all present, and that the Authority Key Identifier matches the
Intermediate's own Subject Key Identifier.

> Note: server keys are generated **without** `-aes256` — unlike CA keys,
> a server process needs to read its key automatically on startup with no
> human present to enter a passphrase. In production this gap is closed
> with strict filesystem permissions or a secrets manager, not left open.

### 4. Verify the chain

```bash
openssl verify -CAfile rootca/root-ca.crt \
  -untrusted intermediateca/intermediate-ca.crt intermediateca/server.crt
```

Expected output: `intermediateca/server.crt: OK`

Only `rootca/root-ca.crt` is passed as explicitly trusted (`-CAfile`) —
modeling a root baked into an OS/browser trust store. The intermediate is
supplied as `-untrusted`, a helper to complete the chain, not inherently
trusted itself; OpenSSL still verifies it chains back to the trusted root.

### 5. Revoke a certificate

*(instructions added once Phase 5 is complete)*

### 6. Generate a CRL

*(instructions added once Phase 6 is complete)*

## Status

- [x] Phase 0 — CA directory structure & bookkeeping
- [x] Phase 1 — Root CA
- [x] Phase 2 — Intermediate CA
- [x] Phase 3 — End-entity certificate issuance
- [x] Phase 4 — Chain verification
- [ ] Phase 5 — Revocation
- [ ] Phase 6 — CRL generation

## License

MIT (or your preferred choice)