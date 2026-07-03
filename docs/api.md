# Usage & API

The public API lives at the module root (`github.com/go-ruby-oidc/oidc`). It is
**OIDC-shaped but Go-idiomatic**: the pieces mirror the OpenID Connect protocol
(discovery, JWKS, ID-token verification, the code flow, userinfo), while the
surface follows Go conventions — value types, no global state, an explicit HTTP
seam and `errors.Is`-matchable failures.

!!! success "Status: implemented"
    The library is built and importable as `github.com/go-ruby-oidc/oidc`, bound
    into `rbgo` as the OIDC layer; see [Roadmap](roadmap.md).

## Install

```sh
go get github.com/go-ruby-oidc/oidc
```

## The HTTP seam — `Doer`

In place of opening a socket, every network round-trip goes through a `Doer`:

```go
type Doer interface {
    Do(req *HTTPRequest) (*HTTPResponse, error)
}
```

A host binds it to go-ruby-net-http / faraday; tests supply an in-memory mock.
The token exchange reuses go-ruby-oauth2's request/response model via a small
`Doer`→`oauth2.RoundTripper` adapter, so nothing in this package opens a
connection.

## Worked example — the login round-trip

```go
// 1. Discover the provider and build a client.
client, err := oidc.DiscoverClient(oidc.Config{
    Doer:         doer,
    ClientID:     "myclient",
    ClientSecret: "mysecret",
    RedirectURI:  "https://app.example.com/callback",
    Scopes:       []string{"email", "profile"},
}, "https://accounts.example.com")

// 2. Build the authorization URL (openid scope + state + nonce + PKCE S256).
verifier := "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
authURL := client.AuthCodeURL(oidc.AuthParams{
    State: "xyz", Nonce: "n-0S6_WzA2Mj", CodeVerifier: verifier,
})

// 3. On the redirect, exchange the code — the returned id_token is verified.
tokens, err := client.Exchange("the-code", verifier, "n-0S6_WzA2Mj")
fmt.Println(tokens.Claims.Subject(), tokens.Access.Token)

// 4. Fetch UserInfo with the access token.
ui, err := client.UserInfo(tokens.Access.Token)
fmt.Println(ui.Subject())
```

## Verifying an ID token directly

```go
v := &oidc.Verifier{
    Issuer:   "https://accounts.example.com",
    ClientID: "myclient",
    Keys:     keySet, // *oidc.KeySet or *oidc.JWKSCache
    Nonce:    "n-0S6_WzA2Mj",
}
claims, err := v.Verify(idToken)
// err is errors.Is-matchable: ErrInvalidIssuer, ErrInvalidAudience, ErrExpired,
// ErrInvalidIat, ErrNotYetValid, ErrInvalidNonce, ErrInvalidAzp, ErrInvalidHash,
// ErrInvalidToken — all under ErrOIDC.
```

## Surface

```go
// Discovery
func Discover(doer Doer, issuer string) (*ProviderMetadata, error)
func ParseProviderMetadata(data []byte) (*ProviderMetadata, error)

// JWKS
func ParseJWKS(data []byte) (*KeySet, error)
func FetchJWKS(doer Doer, uri string) (*KeySet, error)
func NewJWKSCache(doer Doer, uri string, ttl time.Duration) *JWKSCache
func (s *KeySet) Select(kid, alg string) (*Key, error)     // also *JWKSCache
type KeySource interface{ Select(kid, alg string) (*Key, error) }

// ID token
type Verifier struct {
    Issuer, ClientID string
    Keys             KeySource
    HMACSecret       []byte
    Nonce            string
    Leeway           time.Duration
    AccessToken, Code string
    Algorithms       []string
}
func (v *Verifier) Verify(idToken string) (*IDTokenClaims, error)

// Flow
func NewClient(cfg Config) (*Client, error)
func DiscoverClient(cfg Config, issuer string) (*Client, error)
func (c *Client) AuthCodeURL(p AuthParams) string
func (c *Client) Exchange(code, codeVerifier, nonce string) (*Tokens, error)
func (c *Client) UserInfo(accessToken string) (*UserInfo, error)
func (c *Client) Verifier() *Verifier
```

## What is validated

The ID token's JWS signature is verified against the key resolved from the
provider's JWKS by `kid`/`alg` (**`none` is always rejected**), and the OIDC
claims are checked: `iss` exact match, `aud` contains the client id, `exp` /
`iat` / `nbf` with configurable leeway, `nonce`, `azp` (required when there are
multiple audiences), and `at_hash` / `c_hash` when present. Every one of those is
an explicit rejection branch with a matching `errors.Is` sentinel — and every one
is exercised by the test suite.
