# Roadmap

`go-ruby-oidc/oidc` is grown **test-first**, each capability paired with the
rejection tests that pin its security behaviour rather than built in isolation.
The deterministic, interpreter-independent slice of OpenID Connect is
**complete**.

| Stage | What | Status |
| --- | --- | --- |
| Discovery | Fetch and parse `.well-known/openid-configuration` into `ProviderMetadata`, enforcing the issuer match. | **Done** |
| JWKS & key selection | Fetch, parse and cache the JSON Web Key Set (`KeySet` / `JWKSCache`); select the signing key by `kid`/`alg`; RSA/EC keys materialised from the JWK; one-shot refetch on a rotated key. | **Done** |
| ID-token verification | `Verifier` parses the ID token (via go-ruby-jwt), verifies the JWS signature (RS/ES/PS/HS; `none` rejected), and validates `iss`, `aud`, `exp`/`iat`/`nbf`, `nonce`, `azp`, `at_hash`/`c_hash`. | **Done** |
| Authorization-Code + PKCE | `Client.AuthCodeURL` (scope `openid`, state, nonce, PKCE S256) and `Client.Exchange` — swap the code for tokens via go-ruby-oauth2 and verify the returned ID token. | **Done** |
| UserInfo | Call the userinfo endpoint with the bearer access token and return the claims. | **Done** |
| HTTP seam & coverage | Injectable `Doer` for every network fetch; 100% coverage of every rejection branch, `gofmt` + `go vet` clean, green across 6 arches including the big-endian lane. | **Done** |

## Documented out-of-scope boundaries

These are **deliberate**, recorded so the module's surface is unambiguous:

- **No socket in the core.** The library performs the deterministic protocol
  logic; the actual HTTP transport is the host's, injected through `Doer`. This
  is why `rbgo` binds the transport rather than the reverse.
- **Built on, not reinventing, its siblings.** OAuth2 request/response
  construction is [go-ruby-oauth2](https://github.com/go-ruby-oauth2/oauth2); JWS
  parse/verify is [go-ruby-jwt](https://github.com/go-ruby-jwt/jwt). This module
  adds only the OIDC-specific pieces.
- **Deterministic protocol logic only.** Anything requiring a live Ruby binding
  or evaluation of arbitrary Ruby is the consumer's job.

See [Usage & API](api.md) for the surface and [Why pure Go](why.md) for the
deterministic/interpreter split.
