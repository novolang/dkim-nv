# dkim-nv

DKIM for novo-lang: canonicalise, hash, sign over a key the caller
supplies, and verify against a DNS record the caller already fetched —
answering a five-way verdict rather than a boolean.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

A port of Python's [`dkimpy`](https://pypi.org/project/dkimpy/) and
Rust's [`dkim`](https://docs.rs/dkim) crate, cut to the layer design.
[RFC 6376](https://www.rfc-editor.org/rfc/rfc6376) is the protocol;
[RFC 8463](https://www.rfc-editor.org/rfc/rfc8463) adds Ed25519;
[RFC 8301](https://www.rfc-editor.org/rfc/rfc8301) forbids SHA-1; and
[RFC 8601](https://www.rfc-editor.org/rfc/rfc8601) is the
`Authentication-Results` header a verdict goes into.

| module | holds | rows |
| --- | --- | --- |
| `dkimerr` | the faults — what a CALL could not do, never what a message is worth | `[]` |
| `dkimcanon` | `simple` and `relaxed`, for header and body, into a caller's buffer | `[]` |
| `dkimtag` | the `DKIM-Signature` and `_domainkey` grammars, parsed and written | `[]` |
| `dkimsign` | **`DkimSigner[e]`** — signing over the caller's key | `[]`, `[e]` |
| `dkimverify` | the verdict, with every reason it can be | `[]` |

## The load-bearing interface

```novo norun:pseudo
pub trait DkimSigner[e]
    fn dkim_sign(self, data: [u8]) -> Result<[u8], DkimFault> [e]
    fn dkim_algorithm(self) -> DkimAlgorithm [e]
    fn dkim_selector(self) -> Str [e]
```

**`DkimSigner[e]` is why this package can be `core` at all.**  A private
key lives in one of three places and they cost different things: in
memory, where signing is arithmetic and costs *nothing*; in a file the
program read at startup, which is the same once it is read; and in a
hardware module or a cloud key service, where signing is a network call.
A package that took a key as bytes would work for the first two and be
unusable for the third.  One that took a callback returning `Result`
would have to name an effect row, and the only honest row for "whatever
the caller's key does" is `[net, io, fs]` — which would make this
package `host` and put it out of reach of every `core` consumer,
including smtp-nv's message builder.

So the signer is the caller's, `sign_message` is `[e]`, and a caller
signing with an in-memory Ed25519 key pays exactly nothing.  The test
suite's own `DkimTape` implements `DkimSigner[]` and the function that
drives it carries no effect row at all, so the claim is checked by the
compiler before a single assertion runs.

**The second half of the same decision is that there is no resolver.**
The `_domainkey` TXT record arrives as the character-strings the caller
fetched — which is exactly the shape of dns-codec-nv's
`RdataTxt(strings: [[u8]])`, and which must be *concatenated* before
parsing, because a TXT string is at most 255 bytes and every RSA key is
longer than one.  That is what makes a verifier testable against RFC
6376's own vectors with no network, and what lets a caller with a DNS
cache use it.

## The verdict is five-way, and a boolean would lose three of them

```novo norun:pseudo
pub enum DkimOutcome
    DkimPass       // the domain in d= vouched for this message
    DkimFail       // somebody claimed it did and the arithmetic says otherwise
    DkimNone       // nobody claimed anything
    DkimTempError  // ask again later — the DNS did not answer
    DkimPermError  // asking again will not help
```

RFC 6376 § 6.1 and RFC 8601 § 2.7.1 both define these five, and they are
five different instructions to the program above.  A verifier that
answered `Bool` collapses the last three into `fail`, and a mail system
that quarantined on it throws away mail from every domain whose DNS was
briefly slow.

**And `pass` alone means less than everyone thinks.**  It says that
*some* domain signed the message; a forger who signs with their own
domain gets a `pass` every time.  The check that makes it mean something
is **alignment** — the signing domain against the `From` header's — and
it is not in RFC 6376 at all: it is DMARC's, RFC 7489 § 3.1.  There is
no DMARC package on this grid, so `dkimverify.domains_aligned` is here,
because a verifier that shipped without it would be shipping the
mistake.

## The one example that will work

```novo
use dkimsign
use ed25519

fn main() [io]
    match ed25519.signing_key_from_seed([])
        Err(e) => println(e.message())
        Ok(k)  =>
            let signer = dkimsign.ed25519_signer(k, "k1")
            match dkimsign.sign_message(signer,
                                        dkimsign.default_request("example.test"),
                                        ["From", "Subject"],
                                        [" Alice <a@example.test>", " hello"],
                                        [])
                Err(e) => println(e.message())
                Ok(v)  => println("DKIM-Signature:${v}")
```

## Adding it, and checking it

```console
$ novo pkg add dkim-nv
$ novo pkg build
$ novo test tests/dkimsign_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: dkim-nv.<module>.<fn>`.  That is
what an interface release looks like from the outside, and it is how the
first implementation will know it is finished.

## The missing primitive, named

**`rsa-sha256` is what almost every DKIM signature in the world uses,
and nothing on this grid implements RSA.**  crypto-nv 0.1.2 carries
SHA-1, SHA-256, SHA-512 and HMAC and no public-key arithmetic;
ed25519-nv and p256-nv are the two public-key packages and neither is
RSA; jwt-nv declares an RSA key type over no implementation; and
tls-nv's own interface names RSA verification as one of its four missing
primitives.

So `dkimtag.algorithm_is_available(DkimRsaSha256)` answers **false**,
and a verifier meeting one answers `DkimPermError` — not `DkimFail`.
The distinction is the point: "I cannot check this" and "this is forged"
are different things to tell an operator, and only one of them is a
reason to quarantine a message.

The missing row is **`rsa-nv`** — `core`, category `crypto`: PKCS#1 v1.5
signing and verification with SHA-256, and the modular exponentiation
under it.  bigint-nv is on the registry already and is where the
arithmetic would come from; asn1-nv reads the `SubjectPublicKeyInfo` a
`p=` tag carries.  It is not a DKIM package — TLS, JWT and every
certificate chain need the same thing — which is why it does not belong
here.

`ed25519-sha256` is the algorithm this package **can** complete, and
RFC 8463 is deployed; it is also not what most mail is signed with, and
saying so here is more useful than letting a reader conclude this
package can verify their inbox.

## What smtp-nv's message builder calls

smtp-nv named this row: its README says DKIM "is a canonicalisation, a
hash and an RSA or Ed25519 signature over selected headers, and it
belongs in its own `core` package that `smtpmsg` would then depend on".
Four calls, in this order, and none of them changes `smtpmsg`'s existing
surface:

1. **`smtpmsg.render`** already produces the whole message as `[u8]`.
   Split it at the first empty line — the headers it wrote, and the
   body — which it already knows because it wrote both.
2. **`dkimsign.default_request(domain)`**, then set `signed_headers` to
   the intersection of `dkimcanon.recommended_headers()` and the fields
   the message actually has.  `From` is required and `request_fault`
   refuses a request without it.
3. **`dkimsign.sign_message(signer, request, names, values, body)`**,
   where `signer` is the caller's.  The row is `[e]` — so `smtpmsg`
   stays `[]` for a caller with an in-memory key, and a caller with a
   hardware key pays that key's effects and no others.
4. **Prepend `DKIM-Signature:` and the returned value** above every
   header it signed.  That is where a verifier expects the newest
   signature and what makes a second signature over the first possible.

What smtp-nv should **add** is one field on its message value — the
signer — and one refusal: a message whose `Date` or `Message-ID` is
generated at render time cannot be signed reproducibly, and `smtpmsg`
already makes both fields rather than generating them, which is exactly
the property signing needs.  Nothing else moves.

## What is NOT here, and why

- **No DNS.**  A `core` package has no resolver.  `dkimtag.key_dns_name`
  produces the name to look up and `dkimverify.dns_failed` is the
  verdict for a lookup that did not answer; the `[net]` is the host's,
  and dns-codec-nv is what parses the response.
- **No DMARC policy.**  `domains_aligned` is here because a verifier
  without it is dangerous, but fetching `_dmarc`, reading `p=reject` and
  deciding what to do with a message is a policy engine and a different
  row.
- **No ARC.**  RFC 8617's sealed chain is built on this package's
  canonicalisation and is its own specification; it would be
  `arc-nv` over this one.
- **No message parsing.**  Headers arrive as parallel `names` and
  `values` lists because that is what both a builder and a parser
  already hold, and a package that parsed RFC 5322 as well would be two
  packages — smtp-nv's README makes the same split.
- **No line-ending conversion.**  `dkimcanon.bare_lf_at` reports a bare
  LF and the caller fixes it, because a canonicaliser that silently
  rewrote line endings would sign bytes the message does not contain and
  the failure would arrive at the recipient with nothing to point at.
- **No device claim.**  A microcontroller does not sign mail.

## What widened, and what did not

- **Nothing widened.**  Every row is `[]` except the `[e]` that
  `DkimSigner[e]` binds, which the `effect-budget` audit row counts as
  inside the `core` budget.
- **`l=` is its own function rather than an optional argument**, and
  that is a deliberate friction.  The length tag lets a mailing list
  append a footer without breaking a signature, and lets an attacker
  append anything at all; a caller should have to reach for it, and
  `DkimVerdict.body_truncated` reports its use to whoever receives the
  message.
- **The verdict is not a `Result`.**  Every way a *message* can be wrong
  is a verdict, and a `DkimFault` is only ever something the *caller*
  did.  A verifier that reported a malformed header as a fault would
  have its caller writing an error branch for a message that is simply
  unsigned.
- **`body_length` is -1 for absent and 0 is a real value.**  `l=0` means
  "I signed no body", which is legal and which is also how a message
  gets signed and then completely rewritten — so the two cannot share a
  representation.
- **`dkimtag.algorithm_is_available` exists because a grid gap is not a
  design flaw**, and publishing it as data rather than as a paragraph
  means a consumer can branch on it.  When `rsa-nv` lands, one function
  changes.

## The reference implementation

[`dkimpy`](https://pypi.org/project/dkimpy/) for the canonicalisation
edge cases and [OpenDKIM](http://www.opendkim.org/) for the verifier's
ordering.  The test vectors are RFC 6376's appendix A signed message,
RFC 8463's appendix A Ed25519 example, and § 3.4.5's worked
canonicalisation.

## Licence

Apache-2.0.
