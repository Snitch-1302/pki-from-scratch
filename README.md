# PKI From Scratch

A hand-built Public Key Infrastructure using Python + OpenSSL — a Root CA, an
Intermediate CA, end-entity certificate issuance, revocation, and CRL
generation, built to actually *understand* the trust chain that underpins
TLS and IAM systems, not just run a tutorial's copy-paste commands.

This project was built while transitioning from an Operations background
(Goldman Sachs) into Cybersecurity Engineering, after completing an
integrated Master's in Cybersecurity at PSG College of Technology (June
2026). The goal wasn't just "make a padlock appear in a browser" — it was to
be able to explain, in an interview, *why* every line of every config file
and every command exists.

> **Note on privacy:** all Subject DNs, org names, and hostnames used
> throughout this project are placeholders (`Learning Lab`, `PKI Lab Root
> CA`, `example.lab`, etc.) — no personal information was ever entered into
> any certificate, so none needed to be stripped before making this repo
> public.

> **Note on secrets:** all private keys and CA state files (`index.txt`,
> `serial`, `crlnumber`) are excluded via `.gitignore`. Only public
> certificates and configuration files are tracked in this repo.

---

## Why this project exists

Most "I built a PKI" tutorials stop at getting a green padlock in a browser.
That's maybe 60% of what a PKI actually does. The other 40% — revocation,
CRLs, the reason an intermediate CA exists at all — is where the real
understanding lives, and it's the part most junior projects skip entirely.
This project deliberately goes through all of it.

---

## Trust Chain Overview

```
Root CA (self-signed, offline trust anchor)
   │
   │  signs
   ▼
Intermediate CA (signs end-entity certs day-to-day)
   │
   │  signs
   ▼
End-Entity Certificates (server / client certs)
```

**Why three tiers instead of just a root signing everything directly?**
If the Root CA's key is ever compromised, *every single certificate ever
issued under it* becomes untrustworthy — the entire hierarchy has to be
rebuilt and re-trusted everywhere. Keeping the root offline and doing all
day-to-day signing through an intermediate means that if the intermediate is
ever compromised, you revoke *just* the intermediate and re-issue a new one
— the root itself, and the trust everyone has already placed in it, stays
intact. This is the same reason organizations keep root keys in HSMs or on
air-gapped machines and only bring them online for rare signing ceremonies.

---

## Phase 0 — CA Bookkeeping Structure

Before touching any cryptography, a real Certificate Authority needs a
**stateful ledger**, not just a keypair. A CA that can only *issue*
certificates but has no record of what it issued can never revoke anything
— there'd be nothing to revoke *from*.

```
rootca/
├── certs/        # issued certs, by convention
├── crl/          # generated CRLs land here
├── newcerts/     # permanent archive of every cert ever issued, by serial
├── private/      # the CA's own private key — chmod 700
├── index.txt     # THE ledger: serial, status (V/R/E), expiry, subject DN
├── serial        # next serial number to hand out (starts at 1000)
└── crlnumber     # next CRL sequence number
```

`index.txt` is the file that actually makes revocation possible: when a cert
is revoked, `openssl ca -revoke` looks it up here by serial number and flips
its status from `V` (valid) to `R` (revoked). Without this file, "revoked"
isn't a concept OpenSSL has any way to track.

`serial` guarantees every certificate this CA ever issues has a **globally
unique** serial number — which is exactly what lets a CRL unambiguously say
"serial 1000 is revoked" with no chance of confusing it with another cert.

This structure was created twice — once for `rootca/`, once later for
`intermediateca/` — since each CA in the hierarchy keeps its own independent
ledger.

**A real gotcha I hit here:** I generated this folder structure from the
wrong working directory (parallel to the actual git repo instead of inside
it), and only caught it because `git add` failed with `fatal: not a git
repository`. It's a good early reminder that in PKI work — same as in
Git — *where you're standing* when you run a command matters as much as the
command itself.

---

## Phase 1 — Root CA

### Generating the key

```bash
openssl genrsa -aes256 -out private/root-ca.key 4096
```

- **`-aes256`** encrypts the private key file itself with a passphrase.
  Filesystem permissions (`chmod 700`) only stop other users on the same
  machine — they do nothing if the raw file is ever copied off the machine
  (a leaked backup, a stolen laptop). An AES-256-encrypted key is useless
  without the passphrase even if the file itself is stolen. This is defense
  in depth: two independent barriers instead of one.
- **`4096`-bit** — root CAs use a larger key size than end-entity certs
  because roots are long-lived (10 years here) and sit at the very top of
  the trust hierarchy. If a root key is ever broken, every certificate under
  it has to be re-trusted from scratch — so it gets the largest safety
  margin against future factoring/cryptanalytic advances. End-entity certs
  in this project use 2048-bit, since they're short-lived and far less
  catastrophic to replace.

### `openssl.cnf` — the CA's rulebook

This config file is what `openssl ca` consults for *every* operation —
issuing, revoking, generating a CRL. Key sections:

- **`[ CA_default ]`** — points at the bookkeeping files from Phase 0
  (`database = index.txt`, `serial = serial`, etc.), and sets
  `default_md = sha256` (SHA-1/MD5 are cryptographically broken for this
  purpose) and `policy = policy_strict`.
- **`[ policy_strict ]`** — controls which Subject DN fields must *match*
  between the CA and anything it signs. For example, `organizationName =
  match` means the CA will refuse to sign a CSR claiming a different
  organization than its own — this is an enforced rule, not a suggestion.
- **`[ v3_ca ]`** — the extensions applied to the Root CA's own certificate:
  - `basicConstraints = critical, CA:true` — this is the single field that
    makes a certificate a *CA* certificate at all, as opposed to a regular
    end-entity cert. `critical` means any verifier that doesn't understand
    this extension must reject the certificate outright.
  - `keyUsage = critical, keyCertSign, cRLSign` — restricts the key to only
    signing certificates and CRLs. It deliberately does *not* include things
    like `digitalSignature` — a root key isn't meant to be used for
    anything except minting trust.

### Generating the self-signed certificate

```bash
openssl req -x509 -new -key private/root-ca.key -sha256 -days 3650 \
  -out root-ca.crt -config openssl.cnf -extensions v3_ca
```

The `-x509` flag is what turns this from "generate a CSR asking someone else
to sign me" into "sign myself directly" — the defining trait of a root,
since there's no one above it to ask.

### Verifying the result

```bash
openssl x509 -in root-ca.crt -text -noout
```

Confirmed:
- **Issuer == Subject** (`C=US, ST=California, O=Learning Lab, CN=PKI Lab
  Root CA`) — proof of self-signature.
- **`Basic Constraints: critical, CA:TRUE`**
- **`Key Usage: critical, Certificate Sign, CRL Sign`**
- **Subject Key Identifier == Authority Key Identifier** — this is expected
  *only* for a self-signed root, since it is its own issuer. Once the
  Intermediate CA is signed, its Authority Key Identifier will instead point
  to the *Root's* Subject Key Identifier — that link is the actual mechanism
  a verifier uses to walk up a certificate chain.
- **Public key: 4096 bit**, validity ~10 years.

---

## Repository hygiene

Two real mistakes made (and fixed) during this project, kept here
deliberately as a note to future-me:

1. Ran the initial directory-creation command in PowerShell, which doesn't
   support Bash brace expansion (`{certs,crl,...}}`) — switched to Git Bash
   and stayed there for consistency for the rest of the project.
2. The initial `.gitignore` was only the generic GitHub Python template — it
   didn't actually exclude `*.key`, `private/`, or the CA state files
   (`index.txt`, `serial`, `crlnumber`). Caught this with `git status` before
   the first real commit and added PKI-specific rules before any key
   material could be committed.

---

## Status

- [x] Phase 0 — CA directory structure & bookkeeping
- [x] Phase 1 — Root CA (key + self-signed certificate)
- [ ] Phase 2 — Intermediate CA
- [ ] Phase 3 — End-entity certificate issuance
- [ ] Phase 4 — Chain verification
- [ ] Phase 5 — Revocation
- [ ] Phase 6 — CRL generation

---

## Tech

- Python 3.x (orchestration/automation of OpenSSL calls via `subprocess`)
- OpenSSL (CLI, config-file-driven `openssl ca` workflow — not the
  stateless `openssl x509 -req` shortcut, since that bypasses the CA
  database and can't support revocation)
- Git Bash (Windows)
