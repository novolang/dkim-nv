# dkim-nv

DomainKeys Identified Mail (DKIM) lets a domain take responsibility for
a message by attaching a signature to it, and lets a receiver check that
signature against a public key the domain publishes in the Domain Name
System. It is specified in
[RFC 6376](https://www.rfc-editor.org/rfc/rfc6376).
[RFC 8463](https://www.rfc-editor.org/rfc/rfc8463) adds the Ed25519
algorithm, [RFC 8301](https://www.rfc-editor.org/rfc/rfc8301) forbids
SHA-1, and [RFC 8601](https://www.rfc-editor.org/rfc/rfc8601) defines
the `Authentication-Results` header a verdict is written into. This
package brings all four to novo-lang as arithmetic over bytes the caller
already holds. It builds on
[crypto-nv](https://novo-lang.org/packages/crypto-nv) for SHA-256,
[ed25519-nv](https://novo-lang.org/packages/ed25519-nv) for the
signature, and [base64-nv](https://novo-lang.org/packages/base64-nv) for
the tags that carry binary.
[smtp-nv](https://novo-lang.org/packages/smtp-nv) is the package that
submits a message this one signs.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What DKIM is

A signer chooses some of a message's header fields and the body, reduces
them to a fixed written form, hashes them, signs the hash, and puts the
result in a new header field called `DKIM-Signature`. A verifier reads
that field, fetches the public key it names, and repeats the arithmetic.
RFC 6376 section 3 defines the whole of it.

Reducing a message to a fixed written form is called
**canonicalisation**, and it is the part that decides whether a
signature survives the journey. A message passes through servers that
refold header lines, change the case of field names, add or remove
trailing whitespace, and add or strip empty lines at the end of the
body. Section 3.4 defines two algorithms, and the header half and the
body half are chosen separately.

| Algorithm | What it does to a header | What it does to the body | Sections |
| --- | --- | --- | --- |
| `simple` | Nothing at all | Removes trailing empty lines | 3.4.1 and 3.4.3 |
| `relaxed` | Lowercases the field name, unfolds continuation lines, collapses each run of whitespace to one space, and removes trailing whitespace | Removes trailing whitespace on every line, collapses internal whitespace runs, and removes trailing empty lines | 3.4.2 and 3.4.4 |

Two hashes are computed, in one order. The **body hash** covers the
canonicalised body and goes in the signature's `bh=` tag. The
**signature** then covers the canonicalised signed header fields
followed by the `DKIM-Signature` field itself, which by then carries the
body hash. Section 3.7 fixes the order, and it is why a verifier can
reject a modified body without doing any public-key arithmetic.

The `DKIM-Signature` field's value is a list of `tag=value` pairs
separated by semicolons, defined in section 3.2 and listed in section
3.5.

| Tag | What it carries | Required |
| --- | --- | --- |
| `v=` | The version, which must be `1` | yes |
| `a=` | The algorithm | yes |
| `b=` | The signature, in base64 | yes |
| `bh=` | The body hash, in base64 | yes |
| `d=` | The signing domain, which is what a verdict is about | yes |
| `s=` | The selector, which with the domain names the DNS record | yes |
| `h=` | The signed field names, colon separated, in signing order | yes |
| `c=` | The canonicalisation pair; `simple/simple` when absent | no |
| `l=` | How many bytes of the canonicalised body are covered | no |
| `t=` | When it was signed, in Unix seconds | no |
| `x=` | When it expires, in Unix seconds | no |
| `i=` | The agent or user identifier | no |
| `q=` | How to fetch the key; `dns/txt` is the only method defined | no |
| `z=` | Copies of the header fields the signer saw, for a person to read | no |

The public key is a TXT record in the DNS, at
`<selector>._domainkey.<domain>`. Its value is another `tag=value` list,
defined in section 3.6.1.

| Tag | What it carries |
| --- | --- |
| `v=` | `DKIM1` when present |
| `k=` | The key type, `rsa` or `ed25519`; `rsa` when absent |
| `p=` | The public key, in base64; empty means the key is revoked |
| `h=` | The hash algorithms this key may be used with; empty means any |
| `s=` | The service types; `email` is the only one defined |
| `t=` | Flags; `y` means testing and `s` means strict identity |
| `n=` | A note for a person reading the DNS, never acted on |

Three algorithms can appear in `a=`, and this package can complete one
of them.

| Algorithm | Reference | Usable here |
| --- | --- | --- |
| `ed25519-sha256` | RFC 8463 | yes |
| `rsa-sha256` | RFC 6376 section 3.3.3 | no; nothing on the registry implements RSA |
| `rsa-sha1` | RFC 6376 section 3.3.3 | no; RFC 8301 forbids signing and verifying with it |

A verification answers one of five outcomes. RFC 6376 section 6.1
defines them and RFC 8601 section 2.7.1 gives them the names a log line
carries.

| Outcome | What it means | What a receiver does |
| --- | --- | --- |
| `pass` | The signature verified | Read the next rule below, about what a pass is worth |
| `fail` | A signature was present and does not verify | Treat the message as unvouched for |
| `none` | There was no signature | Nothing; most mail is unsigned |
| `temperror` | The check could not be completed and retrying might help | Try again later |
| `permerror` | The check could not be completed and retrying will not help | Treat the message as unvouched for |

This package performs no input or output. It opens no socket, resolves
no name and reads no clock. The key record arrives as the character
strings the caller fetched, and the current time arrives as a number the
caller read, so a verification is reproducible and can be tested against
RFC 6376's own vectors with no network.

## Install

```
novo pkg add dkim-nv
```

## Example

```novo
use dkimsign
use ed25519

fn main() [io]
    match ed25519.signing_key_from_seed([])
        Err(e) => println(e.message())
        Ok(k)  =>
            // A signer over a key held in memory. Signing with it costs
            // no effects, which is what lets a signing call stay pure.
            let signer = dkimsign.ed25519_signer(k, "k1")

            // What the signature will say about itself: the domain, the
            // recommended header names, relaxed on both halves, no
            // length tag and no timestamps.
            let request = dkimsign.default_request("example.test")

            // The header names and values in the order they appear, then
            // the body. The answer is the field value to write.
            match dkimsign.sign_message(signer, request,
                                        ["From", "Subject"],
                                        [" Alice <a@example.test>", " hello"],
                                        [])
                Err(e) => println(e.message())
                Ok(v)  => println("DKIM-Signature:${v}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: dkim-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `dkimcanon` | The two canonicalisation algorithms, for the header half and the body half, writing into a buffer the caller supplies. The length limit, the recommended field names, and the check for a bare line feed. |
| `dkimtag` | The two `tag=value` grammars, parsed and written: the `DKIM-Signature` field and the `_domainkey` TXT record. The algorithm names, the key flags, and the DNS name a key lives at. |
| `dkimsign` | Signing. The `DkimSigner` interface, an Ed25519 signer over a key held in memory, the body hash, the bytes a signature is computed over, and the finished field value. |
| `dkimverify` | Verifying. The five outcomes with the reason for each, the body hash check, the expiry check, the domain alignment check, and the `Authentication-Results` fragment. |
| `dkimerr` | The faults, which describe what a call could not do and never what a message is worth. |

## How to choose an entry point

**`dkimsign.sign_message` signs a whole message in one call.** Give it a
signer, a request, the header names and values in the order they appear,
and the body. It answers the `DKIM-Signature` field value. Use it
whenever the message is in memory.

**`dkimsign.body_hash`, `.signing_input` and `.sign_input` are the same
work in three steps.** Use them when the body is too large to hold. The
body hash is computed while the body goes past, the signing input is
built afterwards, and only the signing input reaches the key.

**`dkimverify.verify` checks one signature against one key record.**
`verify_all` does the same for a message carrying several. It answers a
list rather than stopping at the first success, because each signature
is a public-key operation and how many to attempt is the caller's
decision.

**The signer is yours to supply.** `DkimSigner` is an interface with one
signing method, and the effects that method declares are the effects a
signing call costs. `dkimsign.ed25519_signer` is the implementation for
a key held in memory, and it costs nothing. A key in a file or in a
hardware module is an implementation you write, and a call through it
costs what reading that key costs.

## The rules a user needs

1. **The `c=` tag names two algorithms, not one.** The header half and
   the body half are chosen separately, and `relaxed/simple` is a real
   combination. A single word means that algorithm for the header and
   `simple` for the body. RFC 6376 section 3.5. `canon_of_tag` parses
   the pair, and `absent_canon` is the `simple/simple` that an absent
   `c=` means.
2. **An empty body canonicalises to one CRLF, not to nothing.** Sections
   3.4.3 and 3.4.4. Getting this wrong gives the wrong body hash for
   every message with no body, which is every bounce.
3. **The `DKIM-Signature` field is itself signed, with `b=` emptied and
   no trailing CRLF.** Section 3.7. The `b=` value is removed because a
   hash cannot contain itself. The CRLF is left off because the field is
   the last thing hashed. `dkimcanon.signature_header_into` does both.
4. **The signed fields are hashed in `h=`'s order, not the message's.**
   Section 5.4. `dkimcanon.headers_into` takes the `h=` list and the
   message's fields as separate arguments for that reason.
5. **A field named in `h=` more times than the message has it is taken
   from the bottom up.** Section 5.4.2. A signer names a field twice to
   say there was exactly one, so that a server adding a second one
   breaks the signature. A name in `h=` that the message does not have
   at all is not a failure: it says the field was absent, and one that
   turns up later is not the signer's.
6. **`From` must be in `h=`.** Section 5.4 requires it, and
   `dkimsign.request_fault` refuses a request without it before anything
   is hashed. A signature over `From` alone is valid and nearly useless,
   which is what `dkimcanon.recommended_headers` exists to prevent.
7. **`bh=` is computed before `b=`, and `b=` covers it.** Section 3.7
   fixes the order. `dkimsign.unsigned_value` is the field value with
   `bh=` filled in and `b=` present and empty, and
   `dkimtag.finish_signature_value` is what puts the signature into it.
8. **`b=`, `bh=` and `p=` are standard base64 with padding.** Section
   3.5. They are not the URL-safe alphabet. Folding whitespace inside a
   value is ignored, and this package removes it rather than leaving it
   to the caller.
9. **`l=` says the signature covers only a prefix of the body.** Section
   3.4.5. It lets a mailing list append a footer without breaking the
   signature, and it lets anyone else append anything at all.
   `dkimcanon.body_limited_into` is a separate function so that a signer
   reaches for it deliberately, and `DkimVerdict.body_truncated` reports
   its use to the receiver. A body shorter than a positive `l=` does not
   verify.
10. **A duplicate tag is a syntax error, and an unknown tag is kept.**
    Section 3.2. Taking the last of two `d=` tags would let anyone who
    can append to the header override the domain a verifier already
    read. Unknown tags go in `extra`, because a verifier must ignore
    them and a relay must preserve them.
11. **A TXT record is the concatenation of every one of its character
    strings.** Section 3.6.2.2. Each string is at most 255 bytes and
    every RSA key is longer than one, so a caller that parsed the first
    string alone would parse a truncated key.
    `dkimtag.parse_key_record` takes the whole list.
12. **An empty `p=` is a revocation, not a missing record.** Section
    3.6.1. An absent record means the selector was never used. An empty
    key means it was, and every signature naming it is now invalid.
13. **A key's own flags narrow what it may do.** Section 3.6.1. `t=y`
    marks a testing key, and a failure against one is not evidence of
    anything. `t=s` requires `i=`'s domain to equal `d=` exactly rather
    than be a subdomain of it. `k=` and `h=` restrict the algorithms.
14. **Nothing here converts line endings.** A message uses CRLF and a
    body read from a file on a Unix machine does not.
    `dkimcanon.bare_lf_at` reports the first bare line feed and the
    caller fixes it, because signing rewritten bytes produces a
    signature that fails at the recipient with nothing to point at.
15. **The clock is an argument.** `now_unix` is a time the caller read,
    and `-1` skips the expiry check. RFC 6376 section 3.5 says `x=` must
    be greater than `t=`, and a verifier should treat a signature past
    `x=` as invalid.
16. **A `pass` says only that some domain signed the message.** Anyone
    can sign their own mail with their own domain and get a `pass`. The
    check that gives it meaning is alignment of the signing domain with
    the `From` header's domain, and it is not in RFC 6376 at all. It is
    DMARC's rule, [RFC 7489](https://www.rfc-editor.org/rfc/rfc7489)
    section 3.1, and `dkimverify.domains_aligned` is it.
17. **A new signature goes above the fields it signed.** That is where a
    verifier expects the newest one, and it is what makes a second
    signature over the first possible.

## What is not included

- **RSA.** `rsa-sha256` is what almost every DKIM signature in the world
  uses, and no package on the registry implements RSA.
  `dkimtag.algorithm_is_available` answers false for it, and a verifier
  meeting one answers `permerror` rather than `fail`, because "I cannot
  check this" and "this is forged" are different things to tell an
  operator. `ed25519-sha256` is the algorithm this package can complete
  today.
- **A resolver.** The `_domainkey` record arrives as the strings the
  caller fetched. `dkimtag.key_dns_name` produces the name to look up
  and `dkimverify.dns_failed` is the verdict for a lookup that did not
  answer.
- **DMARC policy.** `domains_aligned` is here because a verifier without
  it is dangerous. Fetching the `_dmarc` record, reading `p=reject` and
  deciding what to do with a message is a separate package.
- **Authenticated Received Chain.** RFC 8617's sealed chain is built on
  this package's canonicalisation and is its own specification.
- **Message parsing.** Header names and values arrive as two parallel
  lists, because that is what both a message builder and a message
  parser already hold.
- **A device probe.** Every function here is arithmetic over the
  caller's bytes, so the toolchain builds the package for a
  microcontroller with no heap allocator. No test asserts it, because a
  program that signs mail runs on a host.

## Related packages

- [smtp-nv](https://novo-lang.org/packages/smtp-nv) submits a message to
  a server. It is the consumer of this package: `smtpmsg.render`
  produces the whole message as bytes, the caller splits it at the first
  empty line into headers and body, signs, and writes the
  `DKIM-Signature` field above the fields it signed. Nothing in
  `smtpmsg`'s surface has to change, because it already takes the date
  and the message identifier as arguments, which is what makes a
  rendered message reproducible enough to sign.
- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) reads the
  TXT record this package parses. Its `RdataTxt` carries exactly the
  list of character strings `dkimtag.parse_key_record` takes. It is not
  a dependency, because this package never fetches a record.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies the
  SHA-256 used for the body hash and inside every signature. It also has
  the constant-time comparison a verifier needs.
- [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) supplies the
  signature itself. `dkimsign.ed25519_signer` takes one of its signing
  keys.
- [base64-nv](https://novo-lang.org/packages/base64-nv) decodes and
  encodes the `b=`, `bh=` and `p=` values.
- `std.crypto` in the standard library is OpenSSL, host only. A program
  that already links it can implement `DkimSigner` over its keys instead
  of using the Ed25519 signer here.

## Tests

```bash
novo test tests/dkimcanon_tests.nv   # 8 tests over the canonicalisations
novo test tests/dkimsign_tests.nv    # 14 tests over the tags, signing and the verdict
```

The reference data is RFC 6376's own: the worked canonicalisation in
section 3.4.5 and the signed message in appendix A. RFC 8463's appendix
A Ed25519 example is the signing vector. The canonicalisation edge cases
follow Python's [dkimpy](https://pypi.org/project/dkimpy/), and the
order in which a verifier runs its checks follows
[OpenDKIM](http://www.opendkim.org/).

The suite asserts the five mistakes that are easy to make and hard to
see: an empty body canonicalising to one CRLF rather than to nothing,
the signature field hashed with an empty `b=` and no trailing CRLF, a
bare word in `c=` meaning `simple` for the body, an empty `p=` meaning
revocation rather than absence, and a TXT record being the concatenation
of every string rather than the first. It also asserts that the signing
interface costs nothing over a key held in memory: the suite's own
signer declares no effects, and so does the function that drives it, so
the compiler checks the claim before an assertion runs.

The tests compile today and fail at run, each on the
`not implemented: dkim-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `dkimcanon.default_canon`, `.absent_canon`, `.canon_kind_name`, `.canon_tag`, `.canon_of_tag` | no |
| `dkimcanon.header_into`, `.headers_into`, `.signature_header_into` | no |
| `dkimcanon.body_into`, `.body_limited_into`, `.body_length`, `.bare_lf_at` | no |
| `dkimcanon.is_recommended_header`, `.recommended_headers` | no |
| `dkimtag.parse_tags`, `.tag_value`, `.parse_signature` | no |
| `dkimtag.write_signature_value`, `.finish_signature_value`, `.key_dns_name` | no |
| `dkimtag.parse_key_record`, `.key_is_revoked`, `.key_is_testing`, `.identity_allowed`, `.key_allows` | no |
| `dkimtag.algorithm_of_token`, `.algorithm_token`, `.algorithm_is_permitted`, `.algorithm_is_available` | no |
| `dkimsign.ed25519_signer`, and the `DkimSigner` implementation behind it | no |
| `dkimsign.default_request`, `.request_fault`, `.signature_field_name` | no |
| `dkimsign.body_hash`, `.signing_input`, `.unsigned_value` | no |
| `dkimsign.sign_message`, `.sign_input` | no |
| `dkimverify.outcome_name`, `.outcome_token`, `.unsigned`, `.dns_failed` | no |
| `dkimverify.verify`, `.verify_all`, `.body_hash_matches`, `.unsigned_fields` | no |
| `dkimverify.is_expired`, `.is_from_the_future`, `.domains_aligned` | no |
| `dkimverify.auth_results_fragment` | no |
| `dkimerr.is_caller_fault`, `.offset_of`, and the `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
