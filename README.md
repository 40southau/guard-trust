# guard-trust

Guard signing public keys, published for independent verification.

`trust-manifest.json` lists every Cloud KMS signing key Guard uses to sign evidence, **per
environment and per key version**, with each key's PEM and the SHA-256 of its DER
SubjectPublicKeyInfo.

Nothing in this repository is secret. Everything in it is a public key.

## Verify a key yourself

```bash
# 1. Pin what you fetched. Cite the commit, not the file.
git log -1 --format='%H %cI'

# 2. Recompute a key's fingerprint independently.
#    Extract a publicKeyPem from the manifest into key.pem, then:
openssl pkey -pubin -in key.pem -outform DER | openssl dgst -sha256 | awk '{print "sha256:"$NF}'
#    The result must equal that entry's spkiSha256, string for string.
```

**The `awk` is not decoration.** That one openssl command prints three different strings depending on
which openssl you have — `SHA2-256(stdin)= 4af8…` on OpenSSL 3, `SHA256(stdin)= 4af8…` on OpenSSL
1.1, and a bare `4af8…` on the LibreSSL that ships as `/usr/bin/openssl` on macOS — while this
manifest stores `sha256:4af8…`. Same hex, four different strings. The `awk` normalises all of them so
you are comparing a value rather than reading past a prefix, and a mismatch means a mismatch.

Match a Guard record to its key on **`keyVersionName`**, which is byte-identical to the
`keyVersionId` stamped into every Guard attestation. Match exactly. Never coerce a record to a
version it was not signed under.

## Why you can check this history rather than trust it

The manifest carries **no timestamp**, deliberately. It changes if and only if a key changes, so
every commit here is a key event and the publication time is a commit date rather than a date we
assert about ourselves. That only means something if the history is hard to rewrite, so:

```bash
# Every commit is signed. Plain HTTPS — no account, no token, no permission from us:
curl -s https://api.github.com/repos/40southau/guard-trust/commits \
  | jq -r '.[] | "\(.sha[0:8]) verified=\(.commit.verification.verified) \(.commit.verification.reason)"'

# The signing key is published on the GitHub account, so you can confirm who signed:
curl -s https://api.github.com/users/savvymonkeyfoo/ssh_signing_keys
```

`jq` only formats; the raw JSON carries the same fields if you would rather not install it.

**Both of those are measured, not assumed** — the point of this page is that you need nothing from
us, so the commands have to actually work with nothing:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://api.github.com/users/savvymonkeyfoo/ssh_signing_keys
# 200
curl -s -o /dev/null -w '%{http_code}\n' https://api.github.com/repos/40southau/guard-trust/commits
# 200
```

**One wrinkle, stated so a correct check does not look like a failure.** The initial commit
(`db33f33`) is signed by **GitHub's own web-flow key** — it was created through the web UI — while
every commit after it is SSH-signed with the key above. The API reports `verified=true` for both. A
local `git log --show-signature` will verify the SSH commits and report the web-flow one as
unverified, because verifying that one needs GitHub's PGP key rather than ours. That is a key your
machine does not have, not an unsigned commit in this history.

Branch protection on `main` additionally refuses force-pushes and deletions and requires verified
signatures, admin included. **Take that as our claim, not as your evidence** — reading a
repository's protection settings requires admin access you do not have. Your own check needs no
permission and is stronger: **record the commit SHA you fetched.** If this history is ever
rewritten, your pinned SHA stops resolving, and no setting we could change makes a missing commit
look present.

**One lifecycle dependency, stated because nothing currently watches it.** Signature verification
depends on the signing key remaining published on the GitHub account. If it is removed, every
historical signature here stops verifying — the commits are unchanged, but GitHub can no longer
attribute them. Pinning a SHA is unaffected by that and is the check to rely on.

## What this does not prove

Possession of these keys proves nothing about any individual Guard record. As of this writing there
is **no export bundle** (Guard does not hand you a portable set of records and signed chain heads)
and **no verifier you can run** (chain-verification code exists in Guard's libraries and is invoked
by nothing a customer can reach). Verifying a record end to end still requires access Guard
controls. Publishing these keys makes Guard's signatures *checkable in principle*; it does not make
any record *verified*.

`whatThisDoesNotProve` in the manifest carries the same boundary, so it travels with the file.

## Rotation

Guard's signing keys do not auto-rotate; versions are created deliberately. When one is added, this
manifest is regenerated and **every prior version is kept** — records signed under an earlier
version still need that version's key. A `DISABLED` key version is deliberately never published: it
may be disabled because it is compromised.

A key declared in Guard's infrastructure code but absent from a project is recorded under
`declaredButAbsentFromEstate` rather than omitted, so a key present in one environment and missing
in another is legible as a stated fact rather than looking like an omission.
