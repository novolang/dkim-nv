# Changelog

All notable changes to dkim-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
