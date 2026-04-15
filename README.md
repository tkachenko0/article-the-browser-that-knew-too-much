# The Browser That Knew Too Much

_XSS, CSRF, OAuth 2.0, JWTs, the BFF Pattern, and are you ready for Claude Mythos?_

Web security is one of those domains where the gap between knowing a concept and applying it correctly is surprisingly wide. XSS, CSRF, MITM...
Everyone's heard of them. But the path from "I know what XSS is" to "my token storage is actually safe" is full of non obvious decisions.

And right now, that gap matters more than ever.

In April 2026, Anthropic announced **Claude Mythos Preview**, a new tier of model, larger and more capable than any Opus before it, with cybersecurity skills the industry is still processing. Mythos reads a codebase, hypothesizes vulnerabilities, runs the actual application to confirm or reject them, and produces a bug report with a proof-of-concept exploit. In pre-release testing it found thousands of previously unknown zero-day vulnerabilities across every major operating system and browser, including a 27-year-old flaw in OpenBSD and a 16-year-old flaw in FFmpeg, bugs that had survived decades of human review and millions of automated tests.

The implication is direct: soon, customers, auditors, and non-technical stakeholders will be able to point an agent at a vendor's application and get a structured security report without writing a single line of code.

> _A note before we start: don't take my word for any of this. Security is a field where second opinions matter. Read the specs, check what others have implemented, and form your own conclusions. What follows is my attempt to connect a lot of dots that had been floating separately in my head and I may have gotten some of them wrong. If you spot something, say so in the comments._

## The Classic Attacks (That Still Work)

### XSS - Cross-Site Scripting

XSS is old. It's been on the OWASP Top 10 for decades. It also keeps happening.

The concept is simple: an attacker finds a way to inject JavaScript into your page. Once that script runs in a victim's browser, it has access to everything JavaScript can touch, cookies, localStorage, sessionStorage, runtime variables. It can silently exfiltrate tokens, session IDs, or form data to an attacker controlled server.

**"But I'm using React/Vue/Angular..."**

Modern frameworks escape output by default, which helps. But it doesn't eliminate the risk:

- `dangerouslySetInnerHTML` in React and `v-html` in Vue bypass escaping entirely
- Third-party libraries can be vulnerable or, as happened with `axios@1.14.1` in early 2025, they can be _deliberately compromised_ in a supply chain attack that installs a Remote Access Trojan on developer machines
- Browser extensions running on your site are outside your control entirely

**The key conclusion:**

> _"Any data accessible to JavaScript is accessible to an attacker via XSS."_

It doesn't matter if it's in localStorage, sessionStorage, or a runtime variable. If JavaScript can read it, so can an attacker.

### CSRF - Cross-Site Request Forgery

Where XSS tries to _steal_ your credentials, CSRF tries to _use_ them without you knowing.

The attack exploits the fact that browsers automatically attach cookies to requests, regardless of where the request originates. An attacker can craft a malicious page that silently sends a POST request to your bank (or your app), and the browser will helpfully include the session cookie.

The victim doesn't lose their token. They just unknowingly transfer money, change their email, or perform whatever action the attacker engineered.

A key characteristic of CSRF attacks: they look like normal user activity in server logs. They're systematically underreported.

```mermaid
sequenceDiagram
    participant V as Victim browser
    participant E as evil.com
    participant A as api.yourapp.com

    V->>E: visits malicious page
    E->>V: hidden form targeting api.yourapp.com
    V->>A: POST /transfer (browser auto-attaches session cookie)
    A->>A: executes transfer ✓
    Note over A,V: browser blocks JS from reading the response (CORS),<br/>but the server already acted on the request
```

### "But… doesn't CORS protect us from CSRF?"

Not really.

**CORS** (Cross-Origin Resource Sharing) is a browser mechanism that _relaxes_ the Same-Origin Policy. By default, JavaScript on `evil.com` cannot read the _response_ from `api.yourapp.com`. CORS allows you to selectively open that up.

But CSRF doesn't need to read the response.

Certain requests, simple ones like a `GET` or a plain `POST` with a standard content type, are sent by the browser _before_ any CORS preflight check.
Sometimes the server receives and executes the request, only then does the browser decide whether to show the response to JavaScript.
The HTTP method matters: endpoints that mutate state should never be reachable via `GET`, and relying on CORS alone to block `POST` requests is not safe either.

Also worth noting: CORS is enforced by _browsers_. Tools like `curl`, Postman, or any backend-to-backend call ignore it entirely. CORS is not server-side security, it's a browser policy.

### MITM - Man in the Middle

MITM attacks intercept traffic between client and server. Common vectors: public Wi-Fi, DNS poisoning, compromised routers.

The standard defense is HTTPS, which encrypts traffic so an eavesdropper sees only noise.

HTTPS is necessary but not sufficient. The cookie configuration matters too.

## Secure Token Storage

Here's where it all comes together. After understanding the attack surface, the question is: **where should we store authentication tokens?**

Let's look at the options honestly:

| Storage              | JS Access | XSS Vulnerable | CSRF Vulnerable  | Auto-sent |
| -------------------- | --------- | -------------- | ---------------- | --------- |
| `localStorage`       | Yes       | Yes            | No               | No        |
| `sessionStorage`     | Yes       | Yes            | No               | No        |
| JS variable          | Yes       | Yes            | No               | No        |
| Normal cookie        | Yes       | Yes            | Yes              | Yes       |
| **HTTP-only cookie** | **No**    | **No**         | **Configurable** | **Yes**   |

The first three rows, localStorage, sessionStorage, JS variables, all share the same problem: JavaScript can read them, which means XSS can steal them.

Normal cookies are actually _worse_: they're XSS-vulnerable _and_ CSRF-vulnerable.

The only option that removes JavaScript access by design is an **HTTP-only cookie**. Because JS can't read it, XSS can't steal it. The CSRF risk remains, but it can be addressed through cookie configuration.

> _"By keeping the token outside of JavaScript's reach, even if an XSS vulnerability occurs, the attacker will not be able to steal the user's authentication credentials."_

### Cookie Attributes That Matter

Once you're using HTTP-only cookies, the configuration of those cookies becomes critical:

| Attribute  | Purpose                      | Protects Against |
| ---------- | ---------------------------- | ---------------- |
| `HttpOnly` | Prevents JavaScript access   | XSS              |
| `Secure`   | Sent only over HTTPS         | MITM             |
| `SameSite` | Restricts cross-site sending | CSRF             |
| `Path`     | Limits cookie scope          | Scope abuse      |
| `Max-Age`  | Limits lifetime              | Long exposure    |

All five matter. `SameSite=Strict` or `SameSite=Lax` addresses most CSRF risk at the cookie level.

If you want to experiment with how cookie attributes affect real attack scenarios, this playground walks through it interactively:  
[tkachenko0/cookies-playground](https://github.com/tkachenko0/cookies-playground)

## Content Security Policy

Even with good storage practices, XSS is still a threat if an attacker can inject scripts into your page. **Content Security Policy (CSP)** is a secondary defense that limits the damage.

A CSP is a response header that tells the browser: "Only load scripts, styles, and images from these trusted sources. Block everything else."

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.com
```

Beyond blocking attacks, CSP enforces good engineering habits:

- Inline `<script>` blocks? Blocked - logic lives in files
- Random third-party CDN dropped in without review? Blocked
- A library that uses `eval()`? Breaks visibly, forcing a conscious decision
- Undeclared analytics, chat widgets, or payment SDKs? Blocked - everything external must be declared

CSP also has a `report-to` directive that sends violation data to your endpoint, giving you visibility into attempted injections or misconfigurations.

**Bad patterns should fail fast and loud.** CSP makes that happen.

If you want to see CSP in action and understand what it blocks (and why), this playground lets you experiment: [tkachenko0/csp-playground](https://github.com/tkachenko0/csp-playground)

## OAuth 2.0

Once you understand token storage, the next question is how tokens get issued in the first place. That's where **OAuth 2.0** comes in.

OAuth 2.0 is an authorization framework that allows an application to access resources on behalf of a user, without ever handling the user's credentials. The application gets a _token_ that represents delegated permission, not a password.

**Four key roles:**

- **Resource Owner** - the user
- **Client** - your application
- **Authorization Server** - issues tokens (e.g., Auth0, Cognito, Keycloak)
- **Resource Server** - your API, which validates tokens on each request

### Which Flow Should You Use?

OAuth 2.0 defines several "grant types" (flows). For SPAs, the right choice is:

**Authorization Code Flow with PKCE** — and here's why.

The basic Authorization Code flow works like this: the user authenticates with the Authorization Server, which returns a short-lived _code_ to the client. The client exchanges that code for tokens. This is secure for server-side apps that can store a **Client Secret**, used to authenticate the client during the exchange.

SPAs can't store a Client Secret securely. Any secret embedded in frontend code is exposed. And even without a secret, the Authorization Code alone is a liability: if intercepted in transit, an attacker can exchange it for tokens before your app does.

**PKCE (Proof Key for Code Exchange)** solves this:

1. Before the flow starts, the client generates a random `code_verifier`
2. It sends a hashed version (`code_challenge`) to the Authorization Server
3. When exchanging the code for tokens, the client proves it holds the original `code_verifier`

Even if an attacker intercepts the authorization code, they can't exchange it without the verifier.

```mermaid
sequenceDiagram
    participant B as Browser / Client
    participant IDP as Authorization Server

    B->>B: generate code_verifier (random)
    B->>B: code_challenge = SHA256(code_verifier)
    B->>IDP: GET /authorize?code_challenge=...&state=...
    IDP->>B: redirect to login
    B->>IDP: user authenticates
    IDP->>B: redirect with code=ABC
    B->>IDP: POST /token — code=ABC + code_verifier
    IDP->>IDP: SHA256(code_verifier) === stored code_challenge?
    IDP->>B: ✅ access_token, id_token, refresh_token
```

### OAuth 2.0 — What Can Go Wrong

Several implementation mistakes routinely turn up in real applications:

**Missing state parameter validation** The `state` parameter is a random value the client generates and includes in the authorization request. The server echoes it back, and the client _must_ verify it matches. Without this check, an attacker can craft a CSRF attack on the OAuth flow itself, forcing a victim to link their account to the attacker's identity provider account.

**Missing nonce validation** Similar to state, the `nonce` binds the ID token to a specific session. Skipping it opens replay attack vectors.

**Weak or absent token validation** APIs must verify all of these on every token:

- `iss` (issuer): is this from the expected authorization server?
- `aud` (audience): is this token intended for this API?
- `exp` (expiration): is this token still valid?
- Signature: has the token been modified?

Skipping any of these is not "good enough." An expired token, a token from the wrong tenant, or a token forged by algorithm confusion all look valid to code that doesn't check.

## JWTs Structure and Vulnerabilities

Access tokens are commonly issued as **JSON Web Tokens (JWTs)**. A JWT is a self-contained, signed JSON structure, the server can verify its authenticity without a database lookup.

A JWT has three Base64-encoded parts separated by dots:

```
header.payload.signature
```

- **Header**: algorithm (`alg`) and token type
- **Payload**: claims - `sub`, `iat`, `exp`, `iss`, `aud`
- **Signature**: cryptographic proof the token hasn't been tampered with

Algorithms can be symmetric (shared secret, e.g. HS256) or asymmetric (private key signs, public key verifies, e.g. RS256).

### JWT Attack Patterns

**Signature not verified**
Some implementations decode the token to read claims without verifying the signature. An attacker can modify `sub` to impersonate any user. This still shows up in penetration tests.

**Algorithm None attack**
The JWT spec includes `"alg": "none"`, meaning no signature is required. If a server accepts this, an attacker can craft arbitrary tokens with no secret required. Fix: **whitelist allowed algorithms explicitly** and never blacklist.

**Algorithm Confusion**
If the server uses RS256 (asymmetric), the public key is by definition public. An attacker can switch the header to HS256 (symmetric) and sign the token using the _public key as the secret_. If the server doesn't enforce which algorithm it expects, it will verify this as valid.

Fix: **explicitly enforce the expected algorithm**. Never let the token header dictate how it gets verified.

**Missing claim validation**
A valid signature doesn't mean the claims are valid. `exp`, `iss`, `aud`, and `sub` must all be checked independently. A token issued for a different application, or one that expired last year, is still cryptographically valid.

For testing JWT vulnerabilities: [jwt.io](https://jwt.io) for inspection, and `jwt_tool.py` for attack simulation.

## The Backend for Frontend (BFF) Pattern

After all of the above, one conclusion is unavoidable:

> _"An authentication flow that is free from XSS, CSRF, and MITM cannot reside in the frontend. It must be handled either as an additional layer between frontend and backend, or entirely on the backend."_

The **Backend for Frontend (BFF)** pattern is the architectural response to this.

The BFF pattern introduces a thin backend layer, co-located with the frontend, acting as its proxy, that owns the entire authentication flow:

```mermaid
flowchart TD
    B[Browser SPA] -->|all requests| BFF[BFF Proxy]
    BFF -->|OAuth 2.0 + PKCE| IDP[Identity Provider]
    BFF -->|proxies with X-User-* headers| API[Backend API]
    BFF -->|sets HttpOnly + Secure + SameSite cookies| B
    IDP -->|tokens| BFF
    API -->|response| BFF
```

**How it works:**

1. User clicks "Login" in the SPA
2. The BFF redirects to the Identity Provider
3. The Identity Provider redirects back to the BFF with the authorization code
4. The BFF exchanges the code for tokens (server-side, with PKCE and state validation)
5. The BFF sets an HTTP-only, Secure, SameSite cookie, tokens never touch the browser's JavaScript
6. Subsequent API calls from the SPA go through the BFF, which attaches the token server-side

```mermaid
sequenceDiagram
    participant B as Browser (SPA)
    participant BFF as BFF Proxy
    participant IDP as Identity Provider
    participant API as Backend API

    B->>BFF: GET /auth/login
    BFF->>BFF: generate state, nonce, code_verifier
    BFF->>B: set HttpOnly cookies (state, nonce, verifier)
    BFF->>B: 302 redirect to IDP /authorize
    B->>IDP: user authenticates
    IDP->>BFF: 302 callback with code + state
    BFF->>BFF: validate state cookie
    BFF->>IDP: POST /token (code + code_verifier)
    IDP->>BFF: tokens
    BFF->>BFF: verify JWT signatures + validate claims
    BFF->>B: set HttpOnly auth cookies + csrf_token cookie
    B->>BFF: POST /api/data + X-CSRF-Token header
    BFF->>BFF: validate CSRF token match
    BFF->>API: proxied request + X-User-Sub / X-User-Email
    API->>B: response
```

**The result:**

> _"The frontend doesn't know OAuth exists. The backend doesn't know tokens exist."_

The SPA just makes HTTP requests. Authentication is handled entirely outside JavaScript's reach.

> **Demo note:** In the demo accompanying this article, the BFF passes the access token, refresh token, and ID token directly as HTTP-only cookies for simplicity. This is purely for demonstration purposes. In a real-world implementation, the BFF can issue its own opaque session token or a self-signed JWT, store the refresh token in a server-side in-memory store (e.g. Redis), and never expose the original OAuth tokens to the browser at all. The details of that internal token management strategy, session design, refresh logic, Redis integration, are out of scope for this article, but worth exploring as a natural next step.

### BFF Security Checklist

A properly implemented BFF addresses the full threat model:

- **XSS**: HTTP-only cookies, JavaScript can never read the token
- **CSRF**: Two layers — `SameSite=Strict` on auth cookies, plus the **Double Submit Cookie Pattern**: after login, the BFF sets a separate `csrf_token` cookie with `httpOnly: false` so the frontend can read it via JavaScript and send it as an `X-CSRF-Token` header on every `/api/*` request. The BFF validates the match using `crypto.timingSafeEqual` to prevent timing attacks. An attacker on a different origin can cause the browser to send the cookie automatically, but cannot read its value to forge the header.
- **MITM**: `Secure=true` flag, tokens only travel over HTTPS
- **JWT**: Signature verification via JWKS, explicit algorithm whitelist, full claim validation (`iss`, `aud`, `exp`, `sub`), plus token consistency check — if only one of `id_token`/`access_token` is present, the session is considered tampered and all cookies are cleared
- **OAuth 2.0**: State parameter validation, nonce validation, PKCE (`code_challenge`/`code_verifier`)

### BFF Notes and Trade-offs

- The BFF doesn't have to be a generic proxy. It can be implemented as a library for a specific framework, eliminating the extra network hop
- For mobile apps, cookie-based flows are more complex since mobile isn't a browser

The open-source implementation referenced throughout this article is available at [tkachenko0/oauth2-bff-proxy](https://github.com/tkachenko0/oauth2-bff-proxy). It supports AWS Cognito, Microsoft Entra ID, and Keycloak out of the box, and the README documents every security decision in detail — including exactly why each cookie attribute is set the way it is, and how the Double Submit Cookie Pattern is wired up end-to-end.

## Wrapping Up

The thread connecting everything here is a single principle: **security by design means keeping sensitive data in places where the attack surface can't reach it**.

- Tokens accessible to JavaScript can be stolen via XSS
- Cookies without proper attributes can be hijacked or forged
- OAuth flows without state, nonce, and PKCE can be subverted
- JWTs without full claim validation can be replayed or forged
- Frontend-only authentication architectures concentrate risk in the worst possible place

The BFF pattern is the architectural expression of this principle: by moving authentication out of the browser, you remove an entire class of vulnerabilities by construction rather than by defense.

## Resources and Playgrounds

**Interactive tools (open source):**

- [csp-playground](https://github.com/tkachenko0/csp-playground) - Learn CSP by seeing and preventing XSS attacks
- [cookies-playground](https://github.com/tkachenko0/cookies-playground) - Experiment with cookie attributes and their security impact
- [TokenLeaks](https://github.com/tkachenko0/TokenLeaks) - Browser extension that shows what sensitive data a web page exposes to JavaScript

**Reference:**

- [OWASP XSS](https://owasp.org/www-community/attacks/xss/)
- [Auth0 - OAuth 2.0](https://auth0.com/docs/get-started/authentication-and-authorization-flow)
- [Auth0 - PKCE Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/)
- [jwt.io](https://jwt.io)
- [RFC 7519 - JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)

> _If something here is wrong, incomplete, or could be explained better, I'd genuinely like to know. Drop a comment below — this is the kind of topic where the discussion is often more valuable than the article._
