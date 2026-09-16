---
title: "Decoding a JWT: Reading and Verifying a Real Token"
date: 2026-09-07 14:00:00 +0000
categories: [Security, Identity]
tags: [jwt, oidc, tokens, cryptography, authentication]     # TAG names should always be lowercase
description: We take a real JSON Web Token apart piece by piece, decode it by hand, and walk through how an app checks that it's genuine.
mermaid: true
---

## Where we left off

In the [OIDC post]({% post_url 2026-09-07-what-is-openid-connect %}) we said the Identity Provider hands your app a signed **ID Token** — a **JWT**. We treated it as a magic "signed note." Now let's open the envelope and see what's actually inside, and how an app knows it wasn't forged.

> **JWT** is pronounced *"jot."* It stands for **JSON Web Token**.
{: .prompt-tip }

## A JWT is just three chunks joined by dots

Here's a real (tiny, example) JWT. Notice the **two dots** splitting it into three parts:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFkYSBMb3ZlbGFjZSIsImlhdCI6MTUxNjIzOTAyMn0.nmm0qepYRjhNji3O-jq8Pvcr7l_me1_bomw4EJucQCU
```

Colour-coded, the three parts are:

```text
HEADER . PAYLOAD . SIGNATURE
```

- **Header** — *what kind* of token this is and *how* it was signed.
- **Payload** — the actual data (the claims about you).
- **Signature** — the tamper-proof seal.

```mermaid
graph LR
    J["xxxxx.yyyyy.zzzzz"] --> H["Header<br/>(algorithm & type)"]
    J --> P["Payload<br/>(the claims)"]
    J --> S["Signature<br/>(the seal)"]
```

## "Encoded" is not "encrypted"

Each part is **Base64URL-encoded**, *not* encrypted. Base64 just makes bytes safe to travel in a URL — **anyone can decode it**. So a JWT is like a **glass envelope**: everyone can *read* what's inside, but the *signature* proves nobody *changed* it.

> **Never put secrets in a JWT payload.** Passwords, credit-card numbers, private data — all readable by anyone holding the token.
{: .prompt-warning }

## Part 1 — the Header

Take the first chunk and Base64URL-decode it:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg` — the **signing algorithm** (here `HS256`). This tells the app how to check the signature.
- `typ` — the token type, `JWT`.

## Part 2 — the Payload (the claims)

Decode the middle chunk:

```json
{
  "sub": "1234567890",
  "name": "Ada Lovelace",
  "iat": 1516239022
}
```

Those key/value pairs are **claims**. There are three flavours:

| Type | Meaning | Examples |
|------|---------|----------|
| **Registered** | Standard, reserved keys | `iss`, `sub`, `aud`, `exp`, `iat` |
| **Public** | Agreed-on names, e.g. from OIDC | `email`, `name` |
| **Private** | Custom to your app | `role`, `tenant_id` |

The short registered names, again:

- `iss` — **issuer** (who made it)
- `sub` — **subject** (the user's unique ID)
- `aud` — **audience** (who it's for)
- `exp` — **expiry** timestamp
- `iat` — **issued-at** timestamp

## Part 3 — the Signature (the important bit)

This is what makes a JWT trustworthy. The signature is created from the **header + payload + a key**:

```text
signature = HMACSHA256(
    base64url(header) + "." + base64url(payload),
    secret
)
```

In plain words: the issuer takes the visible parts, mixes in a **key only it knows**, and produces the seal. Change even one character of the payload and the seal no longer matches.

## Signing comes in two styles

This is the single most important concept, so let's be explicit:

### Symmetric — `HS256`
- **One shared secret** signs *and* verifies.
- Simple, but everyone who can *verify* can also *forge*. Only good when the same party does both.

### Asymmetric — `RS256` / `ES256` (what OIDC uses)
- A **key pair**: a **private key** signs; a **public key** verifies.
- The IdP keeps the private key secret. It publishes the **public key** for the whole world.
- Your app verifies with the public key but *cannot* forge tokens. This is why real "Sign in with Google" uses `RS256`, not `HS256`.

```mermaid
graph TD
    subgraph IdP [Identity Provider]
        PK[Private Key<br/>signs the token]
    end
    subgraph App [Your App]
        PUB[Public Key<br/>verifies the token]
    end
    PK -->|creates signature| TOKEN[JWT]
    TOKEN --> PUB
    PUB -->|match?| OK[Trusted ✅]
```

## How an app verifies a token — the checklist

Getting a signature match is necessary but **not enough**. A correct verifier does *all* of these:

1. **Split** the token into its three parts.
2. **Recompute the signature** over `header.payload` and confirm it matches — using the IdP's **public key**.
3. **Check `exp`** — is it expired?
4. **Check `iss`** — did it come from the IdP you expect?
5. **Check `aud`** — was it issued *for your app*?
6. **Check `nbf`** (not-before), if present — is it valid yet?

> A signature can be perfectly valid and the token still be **wrong for you** — e.g. a real Google token meant for a *different* app. Always verify `aud` and `iss`, not just the signature.
{: .prompt-warning }

## Where does the public key come from?

You don't hardcode it. The IdP publishes its keys at a **JWKS** endpoint (JSON Web Key Set), found via the discovery document from the last post:

```text
https://accounts.google.com/.well-known/openid-configuration
        └─ points to →  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs"
```

The header's **`kid`** (key ID) claim tells your app *which* key in that set to use — handy because IdPs rotate keys regularly.

## Try it yourself

- Paste any JWT into [jwt.io](https://jwt.io) to see the decoded header and payload live.
- Or decode a part by hand in a terminal:

```bash
echo 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9' | base64 -d
# → {"alg":"HS256","typ":"JWT"}
```

> **Security note:** don't paste *real* production tokens into public websites — they may still be valid and grant access. Use test tokens.
{: .prompt-danger }

## The one-paragraph takeaway

A **JWT** is three Base64URL parts — **header . payload . signature** — joined by dots. The header and payload are *readable by anyone* (encoded, not encrypted), so they never hold secrets. The **signature**, made with the issuer's key, is what makes the token trustworthy. Real OIDC uses **asymmetric signing (`RS256`)**: the IdP signs with a private key and your app verifies with a published **public key** fetched from the **JWKS** endpoint. And verifying means more than a signature match — you also check `exp`, `iss`, and `aud`.

> **Next up:** we'll wire this into a real login flow and watch the tokens move end to end.
{: .prompt-tip }
