# JWT & OAuth2 — Interview-Ready Notes
### For Java Backend Developer Interviews

---

## 1. Authentication vs. Authorization (Foundation)

Before JWT/OAuth2, nail this distinction — interviewers love opening with it.

| | Authentication | Authorization |
|---|---|---|
| Question answered | "Who are you?" | "What are you allowed to do?" |
| Example | Logging in with username/password | Checking if a logged-in user can access `/admin` |
| Happens | First | After authentication |

- **OAuth2** is fundamentally an **authorization** framework (delegated access), not an authentication protocol.
- **JWT** is just a **token format** — a way to represent claims. It can carry authentication *or* authorization data.
- **OIDC (OpenID Connect)** is the layer built **on top of OAuth2** that adds authentication (who the user is), via the `id_token`.

**Interview soundbite:**
> "OAuth2 answers 'can this app access this resource on my behalf?' OpenID Connect, built on OAuth2, answers 'who is this user?' JWT is just the token format both often use to carry that information."

---

## 2. JWT (JSON Web Token)

### What it is
A JWT is a **compact, self-contained, URL-safe token format** (RFC 7519) used to represent claims (statements about a user or entity) that can be verified because it's **digitally signed**.

### Structure — 3 Parts, Base64Url-Encoded, Dot-Separated
```
header.payload.signature
```

Example:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

#### a) Header
```json
{ "alg": "HS256", "typ": "JWT" }
```
- `alg` — signing algorithm (`HS256`, `RS256`, etc.)
- `typ` — token type

#### b) Payload (Claims)
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "roles": ["ADMIN"],
  "iat": 1516239022,
  "exp": 1516242622
}
```
**⚠️ Payload is only Base64Url-encoded, NOT encrypted.** Anyone can decode and read it (e.g., on jwt.io). Never put secrets (passwords, credit card numbers) in the payload.

**Standard (Registered) Claims:**

| Claim | Meaning |
|---|---|
| `iss` | Issuer — who created/signed the token |
| `sub` | Subject — the entity the token is about (usually user ID) |
| `aud` | Audience — intended recipient(s) of the token |
| `exp` | Expiration time (Unix timestamp) — token invalid after this |
| `nbf` | Not Before — token not valid until this time |
| `iat` | Issued At |
| `jti` | JWT ID — unique identifier (useful for revocation/blocklisting) |

Custom claims (e.g., `roles`, `email`) can be added alongside these.

#### c) Signature
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```
The signature is what makes the token **tamper-evident**: if anyone modifies the header or payload, re-computing the signature on the server won't match, and verification fails.

### Signing Algorithms — Symmetric vs. Asymmetric

| | Symmetric (`HS256`) | Asymmetric (`RS256` / `ES256`) |
|---|---|---|
| Keys | Single shared secret | Private key (signs) + Public key (verifies) |
| Who can verify | Anyone with the shared secret | Anyone with the public key |
| Who can issue | Anyone with the shared secret | Only holder of the private key |
| Best for | Single service/monolith issuing & verifying its own tokens | Microservices — auth server signs with private key; any downstream service verifies with public key, without being able to *forge* tokens |
| Key distribution risk | Secret must be shared with every verifier — larger attack surface | Public key can be freely distributed (e.g., via JWKS endpoint) |

**Interview tip:** In a microservices architecture, `RS256` (or `ES256`) is generally preferred because it decouples the ability to *verify* a token from the ability to *issue* one — a compromised downstream service can't mint fake tokens.

### JWT ≠ Encryption
- JWT (as typically used, a **JWS** — JSON Web *Signature*) provides **integrity** and **authenticity**, not confidentiality.
- If you need the payload itself to be unreadable, you need a **JWE** (JSON Web *Encryption*) — a related but separate spec, less commonly used in practice.

### Java/Spring Example — Generating & Validating a JWT (using `jjwt` library)
```java
// Generating
String token = Jwts.builder()
    .setSubject("john.doe")
    .claim("roles", List.of("ADMIN"))
    .setIssuedAt(new Date())
    .setExpiration(new Date(System.currentTimeMillis() + 3600_000)) // 1 hour
    .signWith(SignatureAlgorithm.HS256, secretKey)
    .compact();

// Validating
Claims claims = Jwts.parser()
    .setSigningKey(secretKey)
    .parseClaimsJws(token)
    .getBody();

String username = claims.getSubject();
List<String> roles = claims.get("roles", List.class);
```

### Spring Security — JWT Filter Pattern
```java
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            try {
                Claims claims = jwtService.validateAndParse(token);
                String username = claims.getSubject();

                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(username, null, extractAuthorities(claims));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            } catch (JwtException ex) {
                // invalid/expired token — leave context unauthenticated
            }
        }
        chain.doFilter(request, response);
    }
}
```
Registered before `UsernamePasswordAuthenticationFilter` in the security filter chain — so by the time Spring Security's authorization checks run, `SecurityContextHolder` already knows who the user is.

### JWT Pros & Cons

| Pros | Cons |
|---|---|
| **Stateless** — server doesn't need to store session data; scales horizontally with no shared session store | **Hard to revoke** — a valid, unexpired JWT stays valid even if you "log out" or ban the user, unless you build a blocklist |
| Self-contained — carries claims, no DB lookup needed to know who the user is | **Payload is readable** — never store sensitive data in it |
| Works well across microservices / different domains | If the secret/private key leaks, **every** issued token is compromised |
| Compact — fits in an HTTP header | Larger than an opaque session ID — adds a bit of overhead per request |

### The Revocation Problem — Common Follow-Up Question
Since JWTs are stateless by design, revoking one before its `exp` requires extra work:
- **Short expiration + refresh tokens** (see below) — most common approach.
- **Token blocklist** (e.g., in Redis, keyed by `jti`) — checked on every request; partially reintroduces state, but only for revoked tokens.
- **Versioning** — store a `tokenVersion` per user in the DB; increment it on logout/password change; JWT carries the version it was issued with; reject if it doesn't match current version.

### Access Token vs. Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| Purpose | Sent with every API request to prove identity/permissions | Used only to get a new access token |
| Lifetime | Short (minutes) | Long (days/weeks) |
| Stored where | Memory / short-lived storage on client | Secure storage (httpOnly cookie, secure server-side store) |
| Sent to | Resource server, on every call | Only to the Authorization Server's `/token` endpoint |
| If leaked | Limited damage window (expires soon) | Bigger risk — often rotated on each use (**refresh token rotation**) to limit blast radius |

**Interview soundbite:**
> "Short-lived access tokens limit the damage window if a token is stolen, while a long-lived refresh token — kept in more secure storage and rotated on use — lets the client get new access tokens without forcing the user to log in again."

---

## 3. OAuth2

### What it is
OAuth2 (RFC 6749) is an **authorization framework** that lets a third-party application get **limited access** to a user's resources on another service, **without ever seeing the user's credentials**.

**Classic example:** "Login with Google" / a photo-printing app accessing your Google Photos without you giving it your Google password.

### Core Roles

| Role | Description | Example |
|---|---|---|
| **Resource Owner** | The user who owns the data | You |
| **Client** | The application requesting access | The photo-printing app |
| **Authorization Server** | Issues tokens after authenticating the resource owner and getting consent | Google's OAuth server |
| **Resource Server** | Hosts the protected resources, validates the access token on each request | Google Photos API |

### Key Terms
- **Scope** — the specific permissions being requested (e.g., `read:photos`, `email`, `profile`).
- **Access Token** — credential the client uses to call the resource server.
- **Redirect URI** — where the authorization server sends the user back after consent, with an authorization code.
- **Client ID / Client Secret** — identify and authenticate the client app itself to the authorization server.

### OAuth2 Grant Types (Flows)

#### a) Authorization Code Grant (Most Common, Most Secure)
Used by server-side web apps (and, with PKCE, by mobile/SPA apps too).

```
1. Client redirects user to Authorization Server's /authorize endpoint
   (with client_id, redirect_uri, scope, state)
2. User logs in at the Authorization Server and grants consent
3. Authorization Server redirects back to redirect_uri with an authorization CODE
4. Client's backend exchanges the code (+ client_secret) for an access token
   by calling the /token endpoint — this exchange happens server-to-server,
   so the access token never touches the browser/user-agent directly
5. Client uses the access token to call the Resource Server
```

**Why the extra "code" step instead of returning the token directly?** The code is short-lived and exchanged over a secure back-channel with the client secret — it never appears in browser history or gets exposed to the front-end/user-agent, unlike a token passed directly in a redirect URL.

#### b) Authorization Code + PKCE (Proof Key for Code Exchange)
**Mandatory in modern OAuth2 best practice** for public clients (SPAs, mobile apps) that **can't safely store a client secret**.

```
1. Client generates a random "code_verifier" and derives a "code_challenge" = SHA256(code_verifier)
2. Authorization request includes code_challenge
3. Authorization Server stores it, returns the authorization code as usual
4. When exchanging the code for a token, client sends the original code_verifier
5. Authorization Server hashes it and compares to the stored code_challenge — must match
```
Prevents an intercepted authorization code from being exchanged by an attacker, since they wouldn't have the original `code_verifier`.

#### c) Client Credentials Grant
**Machine-to-machine** — no user involved at all. Used for service-to-service communication.
```
1. Client authenticates directly with client_id + client_secret at /token
2. Authorization Server returns an access token scoped to the client itself
```

#### d) Resource Owner Password Credentials (ROPC) — ⚠️ Legacy, Avoid
Client collects the user's username/password directly and exchanges them for a token.
- Defeats the whole point of OAuth2 (client sees raw credentials).
- **Deprecated in OAuth 2.1** — only mentioned for legacy/trusted first-party apps, and even then discouraged.

#### e) Implicit Grant — ⚠️ Deprecated
Used to be recommended for SPAs; returned the access token directly in the redirect URL fragment (no code exchange step).
- **Deprecated in OAuth 2.1** in favor of Authorization Code + PKCE, because tokens exposed in the URL/browser history are more vulnerable to leakage (referrer headers, browser history, logs).

### Grant Type Comparison

| Grant Type | Use Case | User Involved? | Client Secret Needed? | Status |
|---|---|---|---|---|
| Authorization Code | Server-side web apps | ✅ Yes | ✅ Yes | ✅ Recommended |
| Authorization Code + PKCE | SPAs, mobile apps | ✅ Yes | ❌ No (uses code_verifier instead) | ✅ Recommended, mandatory for public clients |
| Client Credentials | Service-to-service (M2M) | ❌ No | ✅ Yes | ✅ Recommended for its use case |
| Resource Owner Password | Legacy first-party apps | ✅ Yes (credentials shared with client) | ✅ Yes | ⚠️ Deprecated |
| Implicit | Old SPA pattern | ✅ Yes | ❌ No | ⚠️ Deprecated |

**Interview one-liner:**
> "Authorization Code (with PKCE for public clients) is the go-to for anything involving a user; Client Credentials is for service-to-service calls with no user; Implicit and Password grants are legacy and shouldn't be used in new systems."

---

## 4. OAuth2 + JWT — How They Fit Together

OAuth2 defines the **flow** for obtaining a token; it doesn't mandate a **format** for the token itself. In practice, most modern Authorization Servers (Keycloak, Auth0, Okta, AWS Cognito) issue **JWTs as access tokens**, because:
- They're **self-contained** — the Resource Server can validate them locally (signature + `exp` check) without a network round-trip back to the Authorization Server.
- They can carry **scopes/roles as claims**, which the Resource Server uses directly for authorization decisions.

### OpenID Connect (OIDC) — Authentication on Top of OAuth2
OIDC adds a third token type: the **`id_token`** — always a JWT, always represents the authenticated user's identity (not just "access to a resource").

| Token | Purpose | Always a JWT? |
|---|---|---|
| Access Token | Authorize API calls to a resource server | Often, but not required by spec |
| Refresh Token | Get a new access token | Usually opaque, not a JWT |
| ID Token (OIDC only) | Prove who the user is | ✅ Always a JWT |

### Spring Security — OAuth2 Resource Server (Validating Incoming JWTs)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```
```properties
# application.properties — points to the Authorization Server's public keys (JWKS endpoint)
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://auth-server.example.com
```
Spring Security automatically:
1. Fetches the Authorization Server's public keys from its **JWKS (`.well-known/jwks.json`)** endpoint.
2. Validates the incoming JWT's signature, `exp`, and `iss` claim.
3. Populates `SecurityContextHolder` with the token's claims (scopes → authorities).

### Spring Security — OAuth2 Client (Login with Google/GitHub/etc.)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2Login(Customizer.withDefaults()); // handles the full Authorization Code flow
        return http.build();
    }
}
```
```properties
spring.security.oauth2.client.registration.google.client-id=YOUR_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_CLIENT_SECRET
spring.security.oauth2.client.registration.google.scope=openid,profile,email
```
Spring Boot auto-configures the redirect endpoints, code exchange, and token handling — you just declare the provider.

---

## 5. Common Interview Questions & Answers

**Q: Is JWT the same as OAuth2?**
> No. OAuth2 is an authorization *framework* defining flows for obtaining tokens. JWT is a *token format*. OAuth2 access tokens are often (but not required to be) JWTs.

**Q: How do you invalidate/revoke a JWT before it expires?**
> JWTs are stateless by design, so there's no built-in revocation. Options: keep access tokens short-lived and rely on refresh token expiry/rotation; maintain a server-side blocklist keyed by `jti`; or store a `tokenVersion` per user and reject tokens issued before the current version.

**Q: Where should the client store a JWT — localStorage or a cookie?**
> `localStorage` is accessible to JavaScript, making it vulnerable to **XSS**. An **httpOnly, Secure, SameSite** cookie isn't readable by JS (mitigating XSS token theft) but is vulnerable to **CSRF** unless paired with CSRF protections. Many production setups use httpOnly cookies + CSRF tokens, or keep the access token in memory only (not persisted) and rely on a refresh token in an httpOnly cookie.

**Q: What's the difference between authentication and authorization in the context of a JWT?**
> The JWT itself is just a signed data container — whether it represents "this is who the user is" (authentication, like an OIDC `id_token`) or "this client can do X" (authorization, like an OAuth2 access token with scopes) depends on how it's issued and used, not on the token format itself.

**Q: Why use RS256 over HS256 in a microservices setup?**
> With `HS256`, every service that needs to *verify* a token must also hold the shared secret — meaning any of them could also *forge* tokens if compromised. With `RS256`, only the Authorization Server holds the private (signing) key; every other service only needs the public key to verify, so a compromised downstream service can't mint valid tokens.

**Q: What happens if an access token is stolen?**
> The attacker can act as the user until the token expires — this is exactly why access tokens are kept short-lived (minutes). Additional mitigations: audience (`aud`) validation, IP/device binding where feasible, and refresh token rotation so a stolen refresh token becomes unusable after first use.

**Q: What is the `state` parameter for in the Authorization Code flow?**
> A random value the client generates before redirecting to the Authorization Server, and verifies matches on the callback. It's the primary defense against **CSRF attacks on the OAuth2 redirect flow** — without it, an attacker could trick a user into completing an authorization flow initiated by the attacker.

---

## Quick-Fire Interview Summary Table

| Topic | One-Line Answer |
|---|---|
| JWT | Self-contained, signed token format (header.payload.signature); payload is readable, not encrypted |
| JWT signing | `HS256` = shared secret (symmetric); `RS256` = private key signs, public key verifies (asymmetric, preferred for microservices) |
| JWT revocation | Hard by design — mitigate with short expiry, blocklists, or token versioning |
| Access vs Refresh token | Access = short-lived, sent on every request; Refresh = long-lived, only used to mint new access tokens |
| OAuth2 | Authorization framework — lets a client get limited, delegated access without seeing user credentials |
| OAuth2 roles | Resource Owner (user), Client (app), Authorization Server (issues tokens), Resource Server (hosts data) |
| Authorization Code Grant | Standard flow for apps with a backend — code exchanged server-side for a token |
| PKCE | Adds a code_verifier/code_challenge pair — required for public clients (SPAs, mobile) that can't hold a client secret |
| Client Credentials Grant | Machine-to-machine, no user involved |
| Deprecated grants | Implicit and Resource Owner Password Credentials — avoid in new systems |
| OIDC | Adds authentication on top of OAuth2 via a JWT `id_token` |
| Spring Security (Resource Server) | `.oauth2ResourceServer(oauth2 -> oauth2.jwt(...))` — validates incoming JWTs against the issuer's JWKS |
| Spring Security (OAuth2 Client) | `.oauth2Login(...)` — handles the full Authorization Code flow for "Login with X" |