# Why pure Go

`go-ruby-oidc/oidc` implements OpenID Connect in **pure Go, with cgo disabled**.
The slice of OIDC it covers is **deterministic and interpreter-independent**:
given a provider's metadata, its key set, and a token, whether that token is
*valid* is a pure function of those inputs — no live binding, no evaluation of
arbitrary Ruby. That is exactly the part that can — and should — live as a
standalone Go library, separate from any interpreter.

## Built on its siblings, reusable by anyone

This library sits on top of two sibling modules and adds only the OIDC-specific
logic:

- the OAuth2 authorization-URL and token-request construction come from
  [`go-ruby-oauth2/oauth2`](https://github.com/go-ruby-oauth2/oauth2);
- the ID token is a signed JWT parsed and signature-verified by
  [`go-ruby-jwt/jwt`](https://github.com/go-ruby-jwt/jwt);
- **this package** adds discovery, JWKS→key selection from a provider's key set,
  and the ID-token **claim** validation (iss, aud, exp, iat, nbf, nonce, azp,
  at_hash, c_hash) plus the flow orchestration.

It is **reusable standalone**: any Go program can import
`github.com/go-ruby-oidc/oidc` directly, with no Ruby runtime; and the dependency
runs the *other* way — `rbgo` binds this module as the OIDC layer for
[go-embedded-ruby](https://github.com/go-embedded-ruby/ruby), rather than this
module depending on the interpreter.

## The HTTP round-trip is a host seam

Every network fetch — discovery, JWKS, token exchange, userinfo — goes through an
injectable `Doer` interface. So:

- **the core never opens a socket** — a host binds `Doer` to its own HTTP client
  (go-ruby-net-http / faraday), and unit tests supply an in-memory mock;
- the token exchange reuses go-ruby-oauth2's request/response model through a
  small `Doer`→`oauth2.RoundTripper` adapter, so nothing is reimplemented;
- the deterministic verification logic is testable **without a network**, which
  is what lets it reach 100% coverage of every rejection branch.

## Why pure Go matters here

Because the library is CGO-free, it:

- cross-compiles to every Go target with no C toolchain, and links into a single
  static binary;
- has **no dependency on the Ruby runtime** — the dependency runs the other way;
- keeps the security-critical checks (`none` rejected, signature verified against
  the resolved key, exact `iss`/`aud`/`nonce` matching) in plain, auditable Go.

See [Usage & API](api.md) for the surface and [Roadmap](roadmap.md) for what is
in scope.
