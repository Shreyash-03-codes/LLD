# API Security Basics

## Learning Objectives

- Separate authentication, who you are, from authorization, what you may do, and map each to a status.
- Carry the caller's identity in a token, a signed JWT or an API key, and verify it before the request does anything.
- Apply the two checks, identity then permission, so an unauthenticated caller gets a `401` and an authenticated one without the right gets a `403`.

## Introduction

Every API sits on a network that will try to use it wrong. Security is not a feature you bolt on at the end; it is two checks that run on every request before the request touches any state. First, prove who is calling. Second, check whether that caller is allowed to do what it is asking. Get the first right and the request proceeds; get either wrong and the response is a clear, coded rejection.

## Problem Statement

The failure is treating security as one blob instead of two checks. A single `if (token != null)` at the top of every method conflates "there is a token" with "the caller is allowed." The results are the classic misconfigurations: an endpoint that only checks the token exists, so any signed token, even for the wrong tenant, gets in; or an endpoint that checks nothing, trusting the caller's network. And when the two checks are not distinct, the status codes lie, a `401` returned when the caller was actually authenticated but not allowed, which the client then mis-handles.

The other classic is putting the checks in the wrong place, or trusting the client's claims. The token that is verified nowhere, or the `role` field that comes in the request body and is taken on faith, lets a caller set its own permissions. The server has to be the one to verify and the server has to be the one to decide.

## Core Concept

### The two checks

Authentication is the first check: who are you? The caller presents a credential, a token, and the server verifies it is genuine and unexpired. The answer to this check is an identity, a principal, and it says nothing yet about what that identity may do. A valid login proves "you are the user with id 91", not "you may read that order."

Authorization is the second check: what may you do? Given the identity, the server decides whether this identity is permitted to perform the requested operation on the requested resource. The check uses a role, a scope, or a rule, and it is where the "should this request succeed" decision lives.

The two checks run in order, and each has its own rejection:

| Check | Question | Fails with |
|-------|----------|-----------|
| Authentication | Who is calling? | `401` unauthenticated |
| Authorization | May this identity do it? | `403` forbidden |

The order matters. You authenticate before you authorize, because you cannot decide the permission of an identity you have not established. The status codes keep the distinction visible to the client.

### The token

The credential that travels with the request is a token. The two dominant forms are a signed JWT and an API key.

A JWT is a self-contained token: a header, a claim set (who, when, what scope), and a signature over both, so the server can verify the token without a database lookup. The verification is: check the signature against the secret or the public key, check the expiry, check the issuer. If all three pass, the claims are the caller's identity.

```java
Jws<Claims> jws = Jwts.parserBuilder()
        .setSigningKey(secretKey)
        .build()
        .parseClaimsJws(rawToken);
String userId = jws.getBody().getSubject();
```

An API key is simpler: an opaque secret the server looks up, usually tied to a client and its limits. It is easier to revoke and harder to embed data in, and it is the typical bearer for server-to-server clients. Both boil down to the same boundary: the server verifies the presented credential, and the caller does not get to choose its own identity.

### The checks belong in a filter, not the controller

The security check is a cross-cutting concern, so it lives before the controller. In Spring it is a security filter chain: the request passes through the filters, the token is extracted and verified, the identity is put into the context, and only then does the request reach the handler. A controller that receives a request has already been authenticated; a controller that still checks the token is repeating work the layer did.

```java
@Bean
SecurityFilterChain chain(HttpSecurity http) {
    return http
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(...))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .build();
}
```

The filter is where the two checks get their distinct answers. `authenticated()` enforces authentication; `hasRole` enforces authorization. The wiring is declarative, and a controller authorizes nothing, it just runs.

Diagram: the two checks on the request's way in

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 400" font-family="Menlo, Consolas, monospace">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#33475b"/>
    </marker>
  </defs>

  <rect x="40" y="40" width="120" height="50" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="100" y="72" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Client</text>

  <line x1="160" y1="65" x2="260" y2="65" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="210" y="52" text-anchor="middle" font-size="11" fill="#5a6b7a">token</text>

  <rect x="262" y="40" width="170" height="50" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="347" y="62" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Authenticate</text>
  <text x="347" y="81" text-anchor="middle" font-size="11" fill="#5a6b7a">verify the JWT</text>

  <line x1="432" y1="65" x2="540" y2="65" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <rect x="545" y="45" width="150" height="50" rx="8" fill="#fdecea" stroke="#a94442" stroke-width="1.5"/>
  <text x="620" y="73" text-anchor="middle" font-size="12" fill="#a94442">401 no identity</text>

  <line x1="347" y1="90" x2="347" y2="148" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="262" y="150" width="170" height="50" rx="10" fill="#f7f9fb" stroke="#33475b" stroke-width="1.5"/>
  <text x="347" y="172" text-anchor="middle" font-size="12" font-weight="bold" fill="#1a2733">Authorize</text>
  <text x="347" y="191" text-anchor="middle" font-size="11" fill="#5a6b7a">check the scope</text>

  <line x1="432" y1="175" x2="540" y2="175" stroke="#9aa7b4" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#arrow)"/>
  <rect x="545" y="155" width="150" height="50" rx="8" fill="#fdecea" stroke="#a94442" stroke-width="1.5"/>
  <text x="620" y="183" text-anchor="middle" font-size="12" fill="#a94442">403 no rights</text>

  <line x1="347" y1="200" x2="347" y2="298" stroke="#33475b" stroke-width="1.5" marker-end="url(#arrow)"/>

  <rect x="262" y="300" width="170" height="50" rx="10" fill="#eef6ee" stroke="#4a8a4a" stroke-width="1.5"/>
  <text x="347" y="322" text-anchor="middle" font-size="12" font-weight="bold" fill="#2f6b2f">Access</text>
  <text x="347" y="341" text-anchor="middle" font-size="11" fill="#4a8a4a">handler runs</text>
</svg>
```

The token is verified first; a bad or missing one answers `401` and stops. Then the scope is checked; a caller that is identified but not permitted answers `403`. Only after both does the handler run. The two rejections are distinct, and the client can branch on them.

### Least privilege is a design habit

Authorization is not one global "is admin" bit. It is the shape of the permission model: a user may read their own orders, an auditor may read the logs, a service account may write to the queue and nothing else. The scope is the smallest set that does the job, and the API expresses it as a role or a scope on the token, checked per endpoint. An endpoint with no authorization check is an endpoint that trusts everyone who got past the door.

### Transport and tokens

Security does not stop at the checks. The API runs over TLS, so the token and the payload are not readable in transit. The token itself is short-lived, an access token expires in minutes to an hour, and a long-lived capability is the refresh path, not the token. The short life is what limits the damage if a token leaks: a leaked token is a problem for its lifetime, not forever.

## Real Production Usage

Spring Security is the production home of this. The filter chain above, with `oauth2ResourceServer` and `authorizeHttpRequests`, is the standard shape, and the JWT verification is a library call, `jjwt`, `nimbus`, backed by the issuer's key. OAuth2 is the protocol layer: the client obtains a token from an authorization server, and the API verifies it and reads the scopes. The API never issues its own tokens, it trusts the issuer.

The API key is the simpler counterweight, used for server-to-server calls where a whole OAuth dance is overkill. The two coexist in the same filter chain: the JWT for users, the API key for the machine clients, and each has its own identity and its own allowed paths.

## Common Mistakes

**One `if (token != null)`.** Presence of a token is not authentication. Verify the signature and the expiry, or any valid-looking header is an entry pass.

**The role from the body.** A `role` field sent by the client and trusted is how a caller sets its own permissions. The role comes from the verified token, never from the request.

**`401` and `403` swapped.** Returning `403` for "not logged in" or `401` for "not allowed" breaks the client's branch and hides the security state. Keep the two distinct.

## Interview Perspective

Interviewers probe security to test the two-check model and the trust boundary. A weak answer says "we send a token in the header." The strong answer says authentication proves who, authorization decides what, the JWT is verified by signature, and the two live in a filter before the controller, each with its own rejection code.

The follow-up that sorts people is "a caller is authenticated but not allowed to do the thing. What do they get?" The strong answer is `403`, because the identity is proven, the permission is not. The candidate who returns `401` has fused the two checks. The second follow-up is "where does the role come from?", and the strong answer is the verified token, never the body.

Common follow-ups:

- "What is the difference between a `401` and a `403`, and when do you use each?"
- "How does a JWT prove identity without a database lookup?"

## Knowledge Check

1. A request has a valid token for user 91 but is trying to modify an order that belongs to user 92. Which check fails, and what status does it return?
2. An endpoint checks that a token exists and then reads the role from the request body. Name the two violations of the trust model.
3. Why does the security check live in a filter chain and not inside the controller methods, and what would drift if it lived there?

## Key Takeaways

- Authenticate first, authorize second, and keep the `401` and `403` distinct.
- The JWT is verified by signature, expiry, and issuer; the claims become the identity.
- The checks belong in the security filter, so the controller only runs on verified requests.
- The permission model is least privilege, expressed as scopes and roles checked per endpoint.

## What's Next

The API can now know who is calling and what they may do. The final article of the chapter pulls back to the biggest design decision of all, who the API is for. Designing internal vs external APIs covers the fork in the road: the contract you give your own services versus the one you give the world, and why the two should be different shapes entirely.

---

This article explains API security as two ordered checks, authentication who is calling, then authorization what they may do. It argues that a verified token is the only source of identity, not a role the request body claims.