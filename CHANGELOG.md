# Changelog

All notable changes to dkim-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `dkimsign` — `DkimSigner[e]`, the load-bearing interface: a key lives
  in memory, in a file or in a hardware module, and the effect parameter
  is what lets one signing call cost what the caller's key costs.
  `DkimEd25519Signer` is the `[]` one.
- `dkimcanon` — `simple` and `relaxed` for header and body, chosen
  separately, written into a caller's buffer, with the `l=` limit as its
  own function.
- `dkimtag` — the `DKIM-Signature` and `_domainkey` tag grammars,
  parsed and written, with unknown tags kept.
- `dkimverify` — the five-way verdict with its reason, the domain and
  the selector on it, and DMARC's alignment check.
- `dkimerr` — the faults, which are about a CALL and never about a
  message.

### Notes

- **RSA is the missing primitive**, named: `rsa-sha256` is what almost
  every DKIM signature uses, nothing on this grid implements RSA, and
  `dkimtag.algorithm_is_available` answers false for it.  A verifier
  meeting one answers `permerror` rather than `fail`, because "I cannot
  check this" is a different thing to tell an operator.  The missing row
  is `rsa-nv` — `core`/`crypto`, over bigint-nv — and it is not a DKIM
  package: TLS, JWT and every certificate chain want the same thing.
- **`pass` alone means less than everyone thinks**, so
  `dkimverify.domains_aligned` is here: DMARC's rule, in the package
  that would otherwise ship without it.
- **No resolver and no clock.**  The TXT record and `now` are both
  values the caller supplies, which is what makes a verification
  reproducible.
- **What smtp-nv calls** is four calls in the README, none of which
  changes `smtpmsg`'s existing surface.
- **Five quiet mistakes are tests**: an empty body canonicalises to one
  CRLF and not to nothing; the signature header is hashed with an empty
  `b=` and no trailing CRLF; a bare word in `c=` means that algorithm
  for the header and `simple` for the body; an empty `p=` is a
  revocation and not a missing record; and a TXT record is the
  concatenation of every string, not the first one.

### Design notes

- **The shape the missing RSA row would take.**  `rsa-nv`, a `core`
  package in the `crypto` category: PKCS#1 v1.5 signing and
  verification with SHA-256, over the modular exponentiation beneath
  it.  bigint-nv is on the registry already and is where the arithmetic
  would come from, and asn1-nv reads the `SubjectPublicKeyInfo` a `p=`
  tag carries.  It is not a DKIM package: crypto-nv 0.1.3 carries
  SHA-1, SHA-256, SHA-512 and HMAC and no public-key arithmetic;
  ed25519-nv and p256-nv are the two public-key packages and neither is
  RSA; jwt-nv declares an RSA key type over no implementation; and
  tls-nv's own interface names RSA verification as one of its missing
  primitives.  When it lands, `dkimtag.algorithm_is_available` is the
  one function that changes.
- **Why the signer is an interface with an effect parameter.**  A
  private key lives in memory, in a file the program read at startup,
  or in a hardware module or cloud key service, and those cost nothing,
  file access and network access respectively.  A package that took a
  key as bytes would serve the first two and be unusable for the third.
  One that took a callback would have to name the widest of the three,
  which would make this a `host` package and put it out of reach of
  every `core` consumer, smtp-nv's message builder included.
- **Why the verdict is not a `Result`.**  Every way a message can be
  wrong is a verdict, and a fault is only ever something the caller
  did.  A verifier that reported a malformed header as a fault would
  have its caller writing an error branch for a message that is simply
  unsigned.
- **Why `body_length` is -1 for absent.**  `l=0` is legal and means "I
  signed no body", which is also how a message gets signed and then
  completely rewritten, so zero and absent cannot share a
  representation.
- **Why `algorithm_is_available` is published.**  A gap in what this
  project implements is data a consumer can branch on rather than a
  paragraph in a README it cannot.
