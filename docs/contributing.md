# Contributing

Contributions are welcome. `go-ruby-oidc/oidc` is built to a small set of
non-negotiable rules — they are what keep it pure-Go, correct, and secure. Please
read these before opening a pull request.

## Hard rules

- **Build from source — no vendoring.** Everything compiles from source. Being
  able to compile from source is a guarantee of independence.
- **100% test coverage target, enforced in CI.** New code ships with tests, and
  coverage is a CI gate. Fill the **rejection** branches — a security library is
  defined by what it refuses, not only what it accepts.
- **All GitHub content in English.** Issues, pull requests, commits, comments,
  and discussions are English-only.
- **`none` is always rejected.** No configuration path may accept an unsigned ID
  token. Signature verification is mandatory, against the key resolved from the
  provider's JWKS.
- **Pure Go, cgo disabled.** The whole point is a single static binary with no C
  toolchain. Code must build with `CGO_ENABLED=0`. If a feature seems to need C,
  it needs a pure-Go path instead.
- **No socket in the core.** All HTTP goes through the injectable `Doer`; the core
  never dials. Reuse go-ruby-oauth2 / go-ruby-jwt rather than reimplementing
  OAuth2 or JWS.

## Workflow

1. Pick or open an issue describing the change.
2. Work test-first: add the unit / rejection tests over the mocked `Doer`, then
   make them pass.
3. Run the full suite with coverage and confirm the gate is green:

    ```sh
    COVERPKG=$(go list ./... | paste -sd, -)
    go test -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
    go tool cover -func=cover.out | tail -1   # 100.0%
    ```

4. Open a PR in English, referencing the issue.

## Where things live

The library is in
[`github.com/go-ruby-oidc/oidc`](https://github.com/go-ruby-oidc/oidc). This
documentation site is in
[`github.com/go-ruby-oidc/docs`](https://github.com/go-ruby-oidc/docs). Start from
the [Usage & API](api.md) page and the [Roadmap](roadmap.md) to find the right
place for your change.
