# sign-csr

A lightweight CA that signs CSRs via GitHub Actions, with certificates valid for **47 days**.

## How it works

1. **Drop a CSR** into `sign-server-47/` and commit.
2. The GitHub Action signs it with the CA key (stored as a secret).
3. The signed certificate and the CSR land in `certificates/<name>/`.
4. The `sign-server-47/` directory is cleaned out, ready for the next request.

## Directory structure

```
sign-csr/
├── ca/
│   ├── ca.crt        # CA certificate (committed to repo)
│   └── serial        # Serial number tracking (committed)
├── certificates/     # Signed certs + their CSRs
│   └── <name>/
│       ├── <name>.csr              # The original CSR
│       └── YYYYMMDD-<name>.pem    # Signed cert (date = expiry)
├── sign-server-47/   # Drop CSRs here to trigger signing
│   └── .gitkeep
└── .github/workflows/sign-csr.yml
```

## Usage

### Sign a new CSR

```bash
# 1. Clone the repo
git clone https://github.com/BartVanEynde/sign-csr.git
cd sign-csr

# 2. Create a CSR (example for a webserver)
openssl req -new -newkey rsa:2048 -nodes -keyout webserver01.key \
  -out sign-server-47/webserver01.csr \
  -subj "/CN=webserver01.example.com"

# 3. Commit and push
git add sign-server-47/webserver01.csr
git commit -m "Sign webserver01"
git push
```

The workflow runs automatically and commits back:
```
certificates/webserver01/
├── webserver01.csr
└── 20260820-webserver01.pem   valid until 2026-08-20
```

### Renew all certificates

To re-sign **every** existing CSR (for another 47 days) no revocation, old certs stay:

```bash
touch sign-server-47/renew-all.txt
git add sign-server-47/renew-all.txt
git commit -m "Renew all certificates"
git push
```

The old .pem files remain alongside the new ones both are valid until their respective expiry dates.

## Initial setup (one-time)

### 1. CA certificate (already in repo)

`ca/ca.crt` is the self-signed CA certificate. Clients that trust it will trust all certs signed by this workflow.

The CA cert is:

```
Subject:  C = BE, O = Home CA, CN = Sign-CSR CA 47
Expires:  2036-07-01
```

### 2. Set the CA key as a GitHub secret

The passwordless CA private key must be set as a **repository secret** so the workflow can sign with it.

**Steps:**
1. Go to your repo on GitHub: `https://github.com/BartVanEynde/sign-csr`
2. Click **Settings -> Secrets and variables -> Actions**
3. Click **New repository secret**
4. **Name:** `CA_KEY`
5. **Secret:** paste the full `-----BEGIN PRIVATE KEY-----` ... `-----END PRIVATE KEY-----` block (provided below)
6. Click **Add secret**

> Keep this key safe anyone with it can sign certificates trusted by your CA.

## Creating your own client keys and CSRs

```bash
# Generate a client key (passwordless)
openssl genrsa -out myapp.key 2048

# Create a CSR
openssl req -new -key myapp.key -out myapp.csr \
  -subj "/CN=myapp.example.com"
```

Then put `myapp.csr` in `sign-server-47/`, commit, and push.

## Security notes

- The CA key never appears in the repo it lives only as a GitHub Actions secret.
- The workflow verifies that the secret matches `ca/ca.crt` before signing.
- Auto-commits use `[skip ci]` to prevent re-trigger loops.
- Old certificates are **not** revoked both old and new remain valid until their respective end dates.
