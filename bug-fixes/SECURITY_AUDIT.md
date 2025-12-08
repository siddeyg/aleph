# Aleph Security Audit Report

**Date:** 2025-12-08
**Auditor:** Senior Security Code Review
**Scope:** Full codebase analysis focusing on authentication, authorization, and data handling
**Repository:** `/media/Daten1/projects/aleph`

---

## Executive Summary

This security audit identified **7 issues** ranging from CRITICAL to LOW severity:
- **1 CRITICAL** - Missing SECRET_KEY validation
- **2 HIGH** - Insecure default configurations (CSP, CORS)
- **1 MEDIUM** - Deprecated security-related API usage
- **3 LOW** - Code quality and potential runtime issues

All issues have patches provided in the `bug-fixes/patches/` directory.

---

## Critical Issues

### [CRITICAL-001] Missing SECRET_KEY Validation

**File:** `aleph/settings.py:76`
**Severity:** CRITICAL
**CWE:** CWE-798 (Use of Hard-coded Credentials)

**Description:**
The `SECRET_KEY` configuration can be `None`, which would cause cryptographic operations to fail or use predictable values. This key is used for:
- Session encryption
- JWT token signing
- CSRF protection
- Password reset tokens

**Code:**
```python
# Line 76
self.SECRET_KEY = env.get("ALEPH_SECRET_KEY")
```

**Impact:**
- Application crashes on first authentication attempt
- Potential for session hijacking if a default fallback is used
- JWT tokens could be forged if secret is predictable

**Recommendation:**
Add validation at application startup:
```python
self.SECRET_KEY = env.get("ALEPH_SECRET_KEY")
if not self.SECRET_KEY:
    raise RuntimeError(
        "ALEPH_SECRET_KEY environment variable must be set. "
        "Generate one with: openssl rand -hex 32"
    )
```

**Patch:** See `patches/001-secret-key-validation.patch`

---

## High Severity Issues

### [HIGH-001] Insecure Content Security Policy

**File:** `aleph/settings.py:64-67`
**Severity:** HIGH
**CWE:** CWE-1021 (Improper Restriction of Rendered UI Layers)

**Description:**
The default Content Security Policy allows `unsafe-inline` and `unsafe-eval`, which defeats the purpose of CSP and enables XSS attacks.

**Code:**
```python
# Line 64-67
self.CONTENT_POLICY = env.get(
    "ALEPH_CONTENT_POLICY",
    "default-src: 'self' 'unsafe-inline' 'unsafe-eval' data: *",
)
```

**Impact:**
- Cross-Site Scripting (XSS) attacks are not prevented
- Malicious scripts can execute inline JavaScript
- `eval()` and similar dangerous functions can be used by attackers

**Recommendation:**
Remove `unsafe-inline` and `unsafe-eval`. If required for specific libraries, use nonces or hashes:
```python
self.CONTENT_POLICY = env.get(
    "ALEPH_CONTENT_POLICY",
    "default-src 'self'; "
    "script-src 'self'; "
    "style-src 'self'; "
    "img-src 'self' data:; "
    "font-src 'self'; "
    "connect-src 'self'; "
    "frame-ancestors 'none';",
)
```

**Note:** This may require frontend code changes to remove inline scripts/styles.

**Patch:** See `patches/002-secure-csp.patch`

---

### [HIGH-002] Permissive CORS Configuration

**File:** `aleph/settings.py:70`
**Severity:** HIGH
**CWE:** CWE-346 (Origin Validation Error)

**Description:**
CORS is configured to allow all origins (`["*"]`) by default, which enables any website to make authenticated requests to the Aleph API.

**Code:**
```python
# Line 70
self.CORS_ORIGINS = env.to_list("ALEPH_CORS_ORIGINS", ["*"], separator="|")
```

**Impact:**
- Cross-Origin attacks from malicious websites
- Session cookies can be sent from any origin
- Sensitive data exposure through cross-origin requests

**Recommendation:**
Default to same-origin only:
```python
self.CORS_ORIGINS = env.to_list(
    "ALEPH_CORS_ORIGINS",
    [self.APP_UI_URL],  # Only allow the configured UI URL
    separator="|"
)
```

**Deployment Note:** Operators should explicitly configure `ALEPH_CORS_ORIGINS` with trusted domains.

**Patch:** See `patches/003-restrict-cors.patch`

---

## Medium Severity Issues

### [MEDIUM-001] Deprecated JWT Decode Parameter

**File:** `aleph/logic/util.py:59`
**Severity:** MEDIUM
**CWE:** CWE-477 (Use of Obsolete Function)

**Description:**
The `jwt.decode()` call uses the deprecated `verify=True` parameter. In PyJWT 2.x, this parameter was removed and replaced with `options` parameter.

**Code:**
```python
# Line 59
token = jwt.decode(token, key=SETTINGS.SECRET_KEY, algorithms=DECODE, verify=True)
```

**Impact:**
- Code will break when PyJWT is upgraded to 2.x
- Security: `verify=True` might be ignored in newer versions, leading to unverified tokens

**Recommendation:**
Update to modern PyJWT API:
```python
token = jwt.decode(
    token,
    key=SETTINGS.SECRET_KEY,
    algorithms=DECODE,
    options={"verify_signature": True, "verify_exp": True}
)
```

**Patch:** See `patches/004-jwt-api-update.patch`

---

## Low Severity Issues

### [LOW-001] Potential Query Exhaustion in API Key Notifications

**File:** `aleph/logic/api_keys.py:96`
**Severity:** LOW
**CWE:** CWE-400 (Uncontrolled Resource Consumption)

**Description:**
The query uses `yield_per(1000)` to create a server-side cursor, but then calls `query.update()` which may not work correctly with an exhausted iterator.

**Code:**
```python
# Line 69-96
query = Role.all_users()
query = query.yield_per(1000)
query = query.where(...)

for role in query:
    # ... send emails ...

query.update({Role.api_key_expiration_notification_sent: days})
db.session.commit()
```

**Impact:**
- `query.update()` might not update any rows because the iterator was consumed
- API key expiration notifications could be sent repeatedly
- Database inefficiency

**Recommendation:**
Store role IDs and update explicitly:
```python
role_ids = []
for role in query:
    # ... send emails ...
    role_ids.append(role.id)

Role.query.filter(Role.id.in_(role_ids)).update(
    {Role.api_key_expiration_notification_sent: days},
    synchronize_session=False
)
db.session.commit()
```

**Patch:** See `patches/005-fix-query-exhaustion.patch`

---

### [LOW-002] Typo in Log Message

**File:** `aleph/logic/api_keys.py:134`
**Severity:** LOW
**CWE:** N/A (Code Quality)

**Description:**
Spelling error: "Comitting" should be "Committing"

**Code:**
```python
# Line 134
log.info(f"Comitting partition {index}")
```

**Recommendation:**
```python
log.info(f"Committing partition {index}")
```

**Patch:** See `patches/006-fix-typo.patch`

---

### [LOW-003] Weak Database Type Check

**File:** `aleph/core.py:70`
**Severity:** LOW
**CWE:** N/A (Code Quality)

**Description:**
The PostgreSQL check uses substring matching (`"postgres" not in`) which could match false positives like "postgrest" or be bypassed.

**Code:**
```python
# Line 70
if "postgres" not in SETTINGS.DATABASE_URI:
    raise RuntimeError("aleph database must be PostgreSQL!")
```

**Recommendation:**
Use a more specific check:
```python
if not SETTINGS.DATABASE_URI.startswith(('postgresql://', 'postgresql+psycopg2://')):
    raise RuntimeError(
        "Aleph requires PostgreSQL. DATABASE_URI must start with "
        "'postgresql://' or 'postgresql+psycopg2://'"
    )
```

**Patch:** See `patches/007-improve-db-check.patch`

---

## Additional Security Observations

### Positive Findings

1. **Password Hashing:** Uses `werkzeug.security.generate_password_hash()` which is secure
2. **API Key Storage:** API keys are hashed with SHA-256 before storage
3. **SQL Injection:** SQLAlchemy ORM is used consistently, preventing SQL injection
4. **Authorization:** Comprehensive `Authz` class with proper permission checking
5. **Session Management:** Redis-backed sessions with configurable expiration

### Recommendations for Further Hardening

1. **Rate Limiting:** Already implemented (`API_RATE_LIMIT`) - ensure it's enabled in production
2. **HTTPS Enforcement:** `FORCE_HTTPS` should be mandatory in production deployments
3. **API Key Rotation:** 90-day expiration is good, but consider enforcing rotation
4. **Audit Logging:** Event model exists - ensure comprehensive audit trails
5. **Input Validation:** Review file upload endpoints for malicious file detection
6. **Dependency Scanning:** Run `safety check` and `pip-audit` regularly

---

## Testing Recommendations

### Security Testing

1. **Penetration Testing:**
   - Test authentication bypass attempts
   - XSS injection in search queries
   - CSRF protection validation
   - OAuth token manipulation

2. **Automated Scanning:**
   ```bash
   # Install security scanners
   pip install bandit safety pip-audit

   # Run static analysis
   bandit -r aleph/ -ll

   # Check dependencies
   safety check
   pip-audit
   ```

3. **Manual Testing:**
   - Verify SECRET_KEY is set in all environments
   - Test CORS configuration with cross-origin requests
   - Validate CSP headers with browser dev tools
   - Test API key expiration and rotation

---

## Patch Application Instructions

All patches are located in `bug-fixes/patches/` directory.

### Apply All Patches

```bash
cd /media/Daten1/projects/aleph

# Apply patches in order
for patch in bug-fixes/patches/*.patch; do
    git apply "$patch" || echo "Failed to apply $patch"
done

# Review changes
git diff

# Test thoroughly before committing
make test

# Commit if all tests pass
git add -A
git commit -m "Security: Apply security audit fixes

Applies patches from security audit:
- CRITICAL-001: Add SECRET_KEY validation
- HIGH-001: Secure Content Security Policy
- HIGH-002: Restrict CORS to configured origins
- MEDIUM-001: Update JWT API to non-deprecated version
- LOW-001: Fix query exhaustion in API key notifications
- LOW-002: Fix typo in log message
- LOW-003: Improve database type checking

See bug-fixes/SECURITY_AUDIT.md for details."
```

### Apply Individual Patches

```bash
# Apply specific patch
git apply bug-fixes/patches/001-secret-key-validation.patch

# Check what changed
git diff aleph/settings.py
```

---

## Verification Checklist

After applying patches:

- [ ] All unit tests pass (`make test`)
- [ ] Application starts without errors
- [ ] SECRET_KEY environment variable is set
- [ ] CORS_ORIGINS is explicitly configured
- [ ] CSP headers are present in browser (check DevTools)
- [ ] JWT token validation still works
- [ ] API key expiration notifications are sent correctly
- [ ] Database connection works with updated check
- [ ] Integration tests pass

---

## References

- **CWE:** Common Weakness Enumeration (cwe.mitre.org)
- **OWASP Top 10:** https://owasp.org/www-project-top-ten/
- **Flask Security:** https://flask.palletsprojects.com/en/2.3.x/security/
- **PyJWT Documentation:** https://pyjwt.readthedocs.io/

---

**Report Generated:** 2025-12-08
**Audit Tool:** Manual code review + static analysis
**Reviewer:** Claude Code Security Analysis
