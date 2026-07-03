# go-ruby-oidc documentation

**OpenID Connect — discovery, JWKS, ID-token verification, Authorization-Code + PKCE and UserInfo — in pure Go, no cgo.**

`go-ruby-oidc/oidc` is a pure-Go (zero cgo) OpenID Connect library. It mirrors
the surface of Ruby's [`openid_connect`](https://github.com/nov/openid_connect)
gem where it maps cleanly onto deterministic protocol logic — **without any Ruby
runtime**. The module path is `github.com/go-ruby-oidc/oidc`.

It is a **standalone, reusable** module — a sibling of
[go-ruby-oauth2](https://github.com/go-ruby-oauth2/oauth2) and
[go-ruby-jwt](https://github.com/go-ruby-jwt/jwt), on which it builds — and the
OIDC layer bound into [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)
by `rbgo`. The dependency runs the other way: this library has **no dependency
on the Ruby runtime**, and it never opens a socket — every network round-trip is
an injectable `Doer` seam.

!!! success "Status: complete — 100% coverage, every rejection branch"
    Discovery, JWKS + key selection, ID-token signature-and-claim verification,
    the Authorization-Code + PKCE flow and UserInfo are implemented, at 100%
    coverage — every rejection branch — `gofmt` + `go vet` clean, CI green across
    the six 64-bit Go targets (including the big-endian lane).

## Quick taste

```go
// doer is your HTTP seam: oidc.DoerFunc(func(*oidc.HTTPRequest) (*oidc.HTTPResponse, error){...})
client, _ := oidc.DiscoverClient(oidc.Config{
    Doer:         doer,
    ClientID:     "myclient",
    ClientSecret: "mysecret",
    RedirectURI:  "https://app.example.com/callback",
    Scopes:       []string{"email", "profile"},
}, "https://accounts.example.com")

url := client.AuthCodeURL(oidc.AuthParams{State: "xyz", Nonce: "n-0S6", CodeVerifier: verifier})
tokens, _ := client.Exchange("the-code", verifier, "n-0S6") // id_token is verified
info, _ := client.UserInfo(tokens.Access.Token)
```

## What it consumes — and doesn't reinvent

| capability | source |
| --- | --- |
| authorization-URL + token-request construction, PKCE S256, token-response parsing | [`go-ruby-oauth2/oauth2`](https://github.com/go-ruby-oauth2/oauth2) |
| ID-token JWS parse + signature verify (RS/ES/PS/HS) | [`go-ruby-jwt/jwt`](https://github.com/go-ruby-jwt/jwt) |
| discovery, JWKS→key selection, OIDC claim validation, the flow orchestration | this package |

## Repositories

| Repo | What it is |
| --- | --- |
| [`oidc`](https://github.com/go-ruby-oidc/oidc) | the library — OpenID Connect in pure Go |
| [`docs`](https://github.com/go-ruby-oidc/docs) | this documentation site (MkDocs Material, versioned with mike) |
| [`go-ruby-oidc.github.io`](https://github.com/go-ruby-oidc/go-ruby-oidc.github.io) | the organization landing page (Hugo) |
| [`brand`](https://github.com/go-ruby-oidc/brand) | logo and brand assets |

## Principles

- **Pure Go, `CGO_ENABLED=0`** — trivial cross-compilation, a single static
  binary, no C toolchain.
- **`none` is always rejected**, and every claim-rejection branch is tested — the
  security posture is the point.
- **The HTTP round-trip is a host seam.** Nothing in the core opens a socket; the
  host (rbgo) binds `Doer`, tests mock it.
- **Standalone & reusable.** Built on go-ruby-oauth2 and go-ruby-jwt; no
  dependency on the Ruby runtime — the dependency runs the other way.
- **100% test coverage** is the target, enforced as a CI gate, across 6 arches.

## Where to go next

- [Why pure Go](why.md) — why OIDC's verification core is deterministic enough to
  live as a standalone, interpreter-independent Go library.
- [Usage & API](api.md) — the public surface and worked examples.
- [Roadmap](roadmap.md) — what is done and what is out of scope by design.

Source lives at [github.com/go-ruby-oidc/oidc](https://github.com/go-ruby-oidc/oidc).
