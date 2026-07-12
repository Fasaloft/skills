# Security Review Guide

Reminder sheet for security-focused PR review (OWASP-style). Signals to watch for in a diff and what to flag — not a tutorial.

## Authentication & Authorization

- Passwords: anything other than bcrypt/argon2 (MD5, SHA-*, homegrown) → flag. Missing lockout, weak reset flow, no MFA on sensitive ops.
- Session tokens must be cryptographically random; sessions must time out.
- Authorization check missing on any new/changed endpoint → flag. Look for IDOR: object fetched by user-supplied ID without ownership check.
- Privilege escalation paths, roles broader than needed (least privilege), unprotected admin/internal endpoints.
- JWT: hardcoded/weak secret, no `expiresIn`, `jwt.decode()` instead of `verify()`, or verify without pinning the algorithm:

```typescript
jwt.verify(token, secret, { algorithms: ['HS256'], issuer, audience }); // pin alg + claims
```

## Input Validation & Injection

- SQL: string-built queries (f-strings, concatenation, template literals) → parameterized queries or ORM.
- XSS: `innerHTML`/`dangerouslySetInnerHTML`/`v-html` with user input → `textContent` or DOMPurify. Plain JSX interpolation is safe.
- Command injection: `os.system`/`exec`/shell=True with interpolated input → `subprocess.run([...])` list args, or `shlex.quote`.
- Path traversal: user input joined into a filesystem path → resolve both sides and compare absolute paths:

```typescript
const p = path.resolve(baseDir, userInput);
if (!p.startsWith(baseDir + path.sep)) throw new Error('Invalid path');
```

- SSRF: fetch/request to a user-supplied URL → require https + hostname allowlist; block internal ranges (127/8, 10/8, 172.16/12, 192.168/16) and cloud metadata (169.254.169.254, metadata.google.internal); re-validate redirects.
- Deserialization: `pickle.loads`, Java `ObjectInputStream`, PHP `unserialize`, `yaml.load` (full loader) on untrusted input → plain JSON + schema validation (pydantic etc.).

## Data Protection

- Secrets in source/config/diff (passwords, API keys, tokens) → env vars or secret manager. Check YAML/JSON config files, not just code.
- Sensitive data: encrypt at rest and in transit; PII handled per GDPR etc.; secure deletion where required.
- Error responses leaking stack traces, SQL, or internals → generic message to client, detail to internal logs.

## API Security

- Rate limiting on public endpoints; stricter on auth endpoints; per-user and per-IP; graceful 429s.
- CSRF: state-changing endpoint with cookie auth and no CSRF token → flag. Cookies need `SameSite=Lax`/`Strict` + `HttpOnly` + `Secure`. Pure token-in-header APIs (no cookies) are exempt.
- CORS: `origin: '*'` (especially with `credentials: true`) or origin reflected from request → explicit origin list.
- Security headers: missing CSP, HSTS, `X-Content-Type-Options: nosniff`, frameguard (helmet or equivalent). Do not recommend `X-XSS-Protection` — deprecated; CSP is the XSS control.

## Cryptography

- Custom/homegrown crypto → flag; use established algorithms (AES-256, RSA-2048+).
- `Math.random()` for tokens/IDs/secrets → `crypto.randomBytes` / `secrets` module.
- MD5/SHA1 for passwords → bcrypt/argon2 (see Auth).
- Keys: hardcoded, unrotated, or outside HSM/KMS → flag.

## Dependencies

- New dependency: trusted source? known CVEs (npm audit, pip-audit, cargo audit, snyk)? actually needed?
- Lock files must be committed and updated consistently with manifest changes.
- License compliance for new packages.

## Logging & Monitoring

- Sensitive data (passwords, tokens, PII) in log statements → flag; log structured metadata instead.
- Log injection: raw user input interpolated into log lines.
- Security events (logins, permission changes) should be logged; logs tamper-protected with sane retention.

## Severity Labels

Use the skill's standard labels (see [assets/review-checklist.md](../assets/review-checklist.md)). Traditional mapping: Critical/High → 🔴, Medium → 🟡, Low/Info → 🟢.

| Label | Security meaning | Action |
|-------|------------------|--------|
| 🔴 `[blocking]` | Exploitable vulnerability (injection, auth bypass, secret exposure) or one that needs only specific conditions | Block merge, fix before release |
| 🟡 `[important]` | Moderate risk, missing defense in depth | Should fix; discuss if disagree |
| 🟢 `[nit]` | Hardening or best-practice improvement | Non-blocking |

## Review Checklist

- [ ] Auth: strong password hashing, lockout, secure reset, MFA on sensitive ops, random session tokens, session timeout
- [ ] Authz: check on every endpoint, least privilege, no IDOR, no privilege escalation
- [ ] JWT: strong secret from env, expiry set, `verify()` with pinned algorithm and claims
- [ ] No SQL/command injection (parameterized queries, list-arg subprocess)
- [ ] No XSS sinks with unsanitized input
- [ ] Path traversal: resolved-path containment check
- [ ] SSRF: allowlist, internal/metadata IPs blocked, redirects re-validated
- [ ] No native deserialization of untrusted input
- [ ] No secrets in code/config/logs; secret manager or env vars
- [ ] Sensitive data encrypted at rest and in transit; PII compliant
- [ ] Errors return generic messages; details logged internally
- [ ] Rate limiting on public and auth endpoints
- [ ] CSRF token + SameSite/HttpOnly/Secure cookies on cookie-auth state changes
- [ ] CORS restricted to explicit origins
- [ ] Security headers set (CSP, HSTS, nosniff, frameguard)
- [ ] Crypto: established algorithms, CSPRNG, proper key management
- [ ] Dependencies audited, minimal, lock files committed
- [ ] Security events logged; no log injection
