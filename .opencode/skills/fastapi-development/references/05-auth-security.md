# Authentication, Authorization & Security

## Decision framework: built-in JWT/OAuth2 vs third-party

| | Build it yourself (FastAPI security utils + JWT) | Third-party (Auth0, Clerk, Supabase Auth) |
|---|---|---|
| Best for | Full control, no per-user vendor cost, simple credential auth, internal/service-to-service auth | Consumer-facing apps needing social login, MFA, magic links, user management UI fast |
| Cost | Engineering time now, ongoing maintenance (password reset flows, MFA, session revocation) | Per-MAU/subscription pricing, less engineering time |
| Control/lock-in | Full control, no vendor lock-in | Vendor lock-in on user data/identity; migration later is real work |
| Time to ship | Slower for full-featured auth (MFA, social login, email verification) | Fast — these flows come built-in |

**Practical default**: for internal APIs, service-to-service auth, and microservices authenticating each other, build JWT/OAuth2 yourself — it's simple and avoids external dependency for critical-path auth. For a consumer-facing product needing social login/MFA/passwordless fast, a third-party provider is usually the better time-to-market tradeoff. Many real systems mix both: third-party provider issues the identity token, your API validates it and manages its own authorization (roles/permissions) layer on top.

## Building JWT/OAuth2 yourself

- **JWT library**: `PyJWT` is the leaner, actively-maintained current recommendation; `python-jose` is still what FastAPI's own docs historically reference and remains widely used — either is acceptable, but prefer PyJWT for new projects if you don't need `python-jose`'s broader crypto-algorithm surface.
- **Password hashing**: `passlib` with `bcrypt` or `argon2` — never roll your own hashing, never store plaintext, cost factor ≥ 12 for bcrypt.
- **Access + refresh token pattern**: short-lived access token (~15 min), longer-lived refresh token, secret sourced from environment/secret manager, never hardcoded.
- **Token revocation**: JWTs are stateless by design — the server can't invalidate one early without extra state. For logout/revocation, maintain a blacklist of revoked token IDs (`jti` claim) in Redis with a TTL matching the token's remaining lifetime, or move to short-enough access-token lifetimes that revocation matters less and rely on refresh-token revocation instead.
- **FastAPI's own primitives**: `OAuth2PasswordBearer`/`OAuth2AuthorizationCodeBearer` integrate authentication into the auto-generated OpenAPI docs ("Authorize" button) — use these rather than hand-rolling header parsing.

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: DbSession,
) -> User:
    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Could not validate credentials")
    user = await UserRepository(db).get_by_id(payload["sub"])
    if user is None:
        raise HTTPException(status_code=401, detail="Could not validate credentials")
    return user
```

## Third-party auth

- Auth0/Clerk/Supabase Auth issue tokens (JWT/OIDC) your API validates against the provider's public keys (JWKS) rather than a shared secret — implement a dependency that fetches/caches the JWKS and validates signature + claims, conceptually similar to the self-built flow above but validating against provider keys instead of your own secret.
- Still build your own authorization layer (roles/permissions/resource ownership checks) on top — third-party auth solves *authentication* (who is this), not *authorization* (what can they do), which stays your API's responsibility.

## Cross-cutting security

- **CORS**: explicit allowed origins per environment — never `allow_origins=["*"]` in production, especially if credentials/cookies are involved.
- **Rate limiting**: `slowapi` (Starlette-compatible middleware) for basic per-route/per-IP throttling; apply it explicitly to auth endpoints (login/register/password-reset) specifically — a generic app-wide rate limit does not automatically cover routes mounted by a separate auth library/router, so verify it's actually wired to every auth-sensitive route.
- **Secrets**: sourced from environment/secret manager only, never hardcoded, never logged. Rotate the JWT signing secret on a defined schedule for high-security contexts.
- **HTTPS/TLS**: terminate at a reverse proxy/load balancer (Nginx, cloud LB) in front of Uvicorn — Uvicorn itself doesn't need to handle TLS in a typical containerized deployment.
- **Input sanitization**: Pydantic validation covers structure/type; still parameterize any raw SQL (rare when using the ORM correctly) and validate file uploads (type/size limits) explicitly.

## Definition of done for this phase
- [ ] Auth approach (built-in vs third-party vs hybrid) chosen deliberately and documented, not defaulted to without consideration.
- [ ] Passwords hashed with bcrypt/argon2 via passlib; never stored/logged in plaintext.
- [ ] JWT secret sourced from environment/secret manager; access tokens short-lived, refresh flow and revocation strategy defined.
- [ ] CORS explicit per environment; rate limiting verified to actually cover every auth-sensitive route, including ones mounted by third-party auth libraries.
