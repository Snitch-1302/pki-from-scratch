# PKI From Scratch

A hand-built Public Key Infrastructure using Python + OpenSSL: Root CA →
Intermediate CA → end-entity certificate issuance → revocation → CRL
generation.

Read the story behind this project (motivation, design decisions, lessons
learned) here: https://quietbytes.hashnode.dev/series/pki-from-scratch

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

- **OpenSSL** - CA operations, key generation, signing, revocation, CRLs
- **Python 3.x** - orchestration of OpenSSL calls via `subprocess`
- **Git Bash** (Windows) - shell environment

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
> `PKI Lab Root CA`) - no real personal or organizational data.

## Prerequisites

- OpenSSL (v1.1.1+ or v3.x)
- Python 3.8+
- Git Bash or any POSIX-compatible shell (Windows users: avoid PowerShell
  for the setup commands below - brace expansion syntax differs)

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

> Note: server keys are generated **without** `-aes256` - unlike CA keys,
> a server process needs to read its key automatically on startup with no
> human present to enter a passphrase. In production this gap is closed
> with strict filesystem permissions or a secrets manager, not left open.

### 4. Verify the chain

```bash
openssl verify -CAfile rootca/root-ca.crt \
  -untrusted intermediateca/intermediate-ca.crt intermediateca/server.crt
```

Expected output: `intermediateca/server.crt: OK`

Only `rootca/root-ca.crt` is passed as explicitly trusted (`-CAfile`) -
modeling a root baked into an OS/browser trust store. The intermediate is
supplied as `-untrusted`, a helper to complete the chain, not inherently
trusted itself; OpenSSL still verifies it chains back to the trusted root.

### 5. Revoke a certificate

```bash
openssl ca -config intermediateca/openssl.cnf -revoke intermediateca/server.crt
```

Only the CA that issued a certificate can revoke it - this updates that
CA's own `index.txt`, flipping the entry's status from `V` to `R` and
recording a revocation timestamp.

### 6. Generate a CRL

```bash
openssl ca -config intermediateca/openssl.cnf -gencrl \
  -out intermediateca/crl/intermediate-ca.crl
```

Inspect it:

```bash
openssl crl -in intermediateca/crl/intermediate-ca.crl -text -noout
```

Confirm the revoked serial appears under `Revoked Certificates`, and note
the `Next Update` field - a CRL has its own expiry and is expected to be
periodically republished.

**Verify revocation is actually enforced** - note that `openssl verify`
does *not* check revocation status unless explicitly told to:

```bash
openssl verify -CAfile rootca/root-ca.crt -untrusted intermediateca/intermediate-ca.crt \
  -CRLfile intermediateca/crl/intermediate-ca.crl -crl_check intermediateca/server.crt
```

Expected output: `error 23 at 0 depth lookup: certificate revoked`

`-crl_check` is required to enable revocation checking at all - supplying
a CRL via `-CRLfile` alone is silently ignored without it.

## Status

- [x] Phase 0 - CA directory structure & bookkeeping
- [x] Phase 1 - Root CA
- [x] Phase 2 - Intermediate CA
- [x] Phase 3 - End-entity certificate issuance
- [x] Phase 4 - Chain verification
- [x] Phase 5 - Revocation
- [x] Phase 6 - CRL generation

## License

MIT (or your preferred choice)
