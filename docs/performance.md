# Performance

`go-ruby-oidc/oidc` is the pure-Go library that
[`rbgo`](https://github.com/go-embedded-ruby/ruby) binds for OpenID Connect. This
page records how the module's performance is characterised, as part of the
ecosystem-wide per-module parity work.

## What dominates the cost

Unlike the numeric / collection modules, an OIDC verification is **I/O-bound and
crypto-bound**, not loop-bound:

- the network round-trips (discovery, JWKS, token, userinfo) go through the host's
  `Doer` and are **outside** this library — their latency is the provider's and
  the transport's, not the module's;
- the deterministic work this library does per token is a **single JWS signature
  verification** (RSA/EC/HMAC, delegated to
  [go-ruby-jwt](https://github.com/go-ruby-jwt/jwt) over Go's `crypto`) plus a
  handful of string/timestamp claim comparisons.

So the meaningful figure is the **CPU cost of one `Verify`** — the signature
check plus claim validation — with the network mocked out. That is what the
library's own benchmarks target; the JWKS cache (`JWKSCache`) exists precisely so
the key-set fetch is amortised across verifications rather than repeated per
token.

## Reference-parity framing

The parity question for this module is narrow: *the signature math is Go's
`crypto/rsa` · `crypto/ecdsa` · `crypto/hmac`, the same primitives every runtime
ultimately calls.* There is no interpreter dispatch on the hot path — a verified
token is a pure function of `(token, key, expected claims)`. The per-token CPU
cost is therefore dominated by the underlying `crypto` verify, and claim
validation is negligible beside it.

!!! note "No fabricated numbers"
    This page deliberately does **not** publish a benchmark table: doing so
    honestly requires a committed, reproducible harness (a Go driver plus a
    reference workload) measured on a named host, exactly as the numeric modules
    do. That harness is the tracked follow-up for this module. When it lands, the
    measured `ns/op` for `Verify` (per signature family) and for a cache-hit vs
    cache-miss JWKS lookup will be recorded here, with the host, dates and method
    stated — never estimated.

## Reproducing locally

The library's unit benchmarks run with the standard toolchain against the mocked
`Doer`, so no network or provider is required:

```sh
go test -run=^$ -bench=. -benchmem ./...
```
