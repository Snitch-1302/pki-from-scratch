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

*(instructions added once Phase 2 is complete)*

### 3. Issue an end-entity certificate

*(instructions added once Phase 3 is complete)*

### 4. Verify the chain

*(instructions added once Phase 4 is complete)*

### 5. Revoke a certificate

*(instructions added once Phase 5 is complete)*

### 6. Generate a CRL

*(instructions added once Phase 6 is complete)*

## Status

- [x] Phase 0 — CA directory structure & bookkeeping
- [x] Phase 1 — Root CA
- [ ] Phase 2 — Intermediate CA
- [ ] Phase 3 — End-entity certificate issuance
- [ ] Phase 4 — Chain verification
- [ ] Phase 5 — Revocation
- [ ] Phase 6 — CRL generation

## License

MIT
