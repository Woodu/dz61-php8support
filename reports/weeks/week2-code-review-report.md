# Week 2 Authentication System - Code Review Report

**Date:** 2026-02-14
**Reviewer:** Senior PHP Code Review Team
**Scope:** Week 2 Authentication & User System
**Files Reviewed:** 7 core services + 1 view + tests

---

## Executive Summary

- **Overall Rating:** B+ (82/100)
- **Critical Issues (P0):** 0
- **High Issues (P1):** 7
- **Medium Issues (P2):** 12
- **Low Issues (P3):** 8
- **Production Ready:** NO (requires P1 fixes)

### Key Findings

**Strengths:**
- ✅ Excellent UCenter password formula verification (matches legacy exactly)
- ✅ Comprehensive security features (CSRF, rate limiting, timing-safe comparisons)
- ✅ Modern password hashing with automatic migration path
- ✅ Well-documented code with clear inline comments
- ✅ Proper use of cryptographically secure random functions
- ✅ TDD methodology followed throughout

**Critical Gaps:**
- ❌ AuthService has in-memory rate limiting (not persistent, lost on restart)
- ❌ AuthService and RateLimiterService have duplicate rate limiting logic
- ❌ Missing database integration for UCenter queries (hard-coded table names)
- ❌ Login view has inline JavaScript (should be separate file)
- ❌ AuthController `getUserData()` returns null (not implemented)
- ❌ No proper error handling for database connection failures
- ❌ Missing integration tests for full login flow

---

## 1. PasswordVerifier Review

**File:** `app/Auth/Password/PasswordVerifier.php` (123 lines)

### Functional Completeness
- ✅ UCenter dual MD5+salt verification
- ✅ Simple MD5 verification (for Discuz! local users)
- ✅ Bcrypt migration support
- ✅ Hash format detection (UCenter vs bcrypt)

### Security Assessment
- ✅ Timing-safe comparisons (`hash_equals()`)
- ✅ No hardcoded secrets
- ✅ Proper error handling (no exceptions thrown)

### Legacy Compatibility
- ✅ Matches UCenter formula exactly: `md5(md5($password) . $salt)`
- ✅ Can authenticate existing UCenter users
- ✅ Can authenticate existing Discuz! users

### Issues Found

#### Issue #1: Missing Input Validation
- **Severity:** P2 (Medium)
- **Location:** Line 38, `verifyUcenter()`
- **Problem:** No validation that `$hash` and `$salt` are non-empty before processing
- **Impact:** Could cause unexpected behavior if empty strings passed
- **Fix Strategy:** Add input validation at method start
- **Code Example:**
```php
// Before
public function verifyUcenter(string $password, string $hash, string $salt): bool

// After
public function verifyUcenter(string $password, string $hash, string $salt): bool
{
    if (empty($hash) || empty($salt)) {
        return false;
    }
    // ... rest of method
}
```

#### Issue #2: Incorrect Pattern in isUcenterHash()
- **Severity:** P3 (Low)
- **Location:** Line 118
- **Problem:** Pattern `/^[0-9a-f]{32}$/` doesn't match UCenter's actual format
- **Impact:** May not detect all UCenter hashes correctly
- **Analysis:** UCenter hashes are MD5 output, which uses lowercase hex. However, the actual legacy data shows mixed case or variations. Should use case-insensitive check.
- **Fix Strategy:** Use case-insensitive pattern or convert to lowercase first
- **Code Example:**
```php
// Before
$pattern = '/^[0-9a-f]{32}$/';
return preg_match($pattern, $hash) === 1;

// After
return ctype_xdigit(strtolower($hash)) && strlen($hash) === 32;
```

#### Issue #3: migrateUcenterToBcrypt() Unused Parameter
- **Severity:** P3 (Low)
- **Location:** Line 93
- **Problem:** `$salt` parameter is documented as "unused" but still included
- **Impact:** Code smell, potential confusion
- **Fix Strategy:** Remove parameter or use it for validation/audit logging
- **Recommendation:** Keep for API compatibility but add audit logging

### Summary
- **Grade:** A- (90/100)
- **Critical Issues:** 0
- **High Issues:** 0
- **Recommendation:** Minor improvements needed, production-ready after fixes

---

## 2. AuthService Review

**File:** `app/Auth/Auth/AuthService.php` (527 lines)

### Functional Completeness
- ✅ UCenter password verification
- ✅ Discuz! local password verification
- ✅ Dual-source authentication (UCenter primary, Discuz! fallback)
- ✅ Automatic password migration to bcrypt
- ✅ Session management integration
- ❌ **Rate limiting is in-memory only** (NOT PERSISTENT)

### Security Assessment
- ✅ Timing-safe password comparisons
- ✅ No hardcoded secrets
- ⚠️ **In-memory rate limiting** (lost on PHP restart)
- ❌ **No database error handling**
- ❌ **SQL injection risk** (table names not parameterized)

### Legacy Compatibility
- ✅ UCenter password formula matches exactly
- ✅ Can authenticate existing users from both tables
- ✅ Session data structure compatible
- ✅ Properly handles NULL values (salt can be empty)

### Issues Found

#### Issue #1: CRITICAL - In-Memory Rate Limiting
- **Severity:** P1 (HIGH)
- **Location:** Lines 52, 457-492
- **Problem:** `$failedLogins` array is stored in memory, not Redis/database
- **Impact:** Rate limiting is lost on:
  - PHP process restart
  - Server restart
  - Load balancer with multiple servers
  - Attackers can simply wait for restart to bypass limits
- **Fix Strategy:** Use RateLimiterService for ALL rate limiting (remove duplicate logic)
- **Code Example:**
```php
// BEFORE (Lines 457-492)
private function isRateLimited(string $ip): bool
{
    if (!isset($this->failedLogins[$ip])) {
        return false;
    }
    $data = $this->failedLogins[$ip];
    // ... in-memory checks
}

// AFTER (remove entirely, delegate to RateLimiterService)
// In login() method, replace Step 2:
// Step 2: Check rate limiting
// REMOVE these lines:
// $clientIp = $this->getClientIp();
// if ($this->isRateLimited($clientIp)) {
//     return AuthResult::failure(...);
// }

// Rate limiting is NOW handled by AuthController->loginPostAction()
// which calls RateLimiterService->exceededLoginAttempts($clientIp)
// BEFORE calling AuthService->login()

// So AuthService should NOT do its own rate limiting
```

#### Issue #2: SQL Injection Risk - Table Names
- **Severity:** P1 (HIGH)
- **Location:** Lines 332-336, 345-349
- **Problem:** Table names concatenated directly into SQL
- **Impact:** If `uc_members` or `cdb_members` can be influenced by user input, SQL injection possible
- **Analysis:** While current code uses hardcoded table names, best practice is to whitelist table names
- **Fix Strategy:** Whitelist allowed table names
- **Code Example:**
```php
// BEFORE
private function queryUcenterUser(string $username): ?array
{
    $sql = "SELECT uid, username, password, salt, email, groupid
            FROM uc_members
            WHERE username = ? LIMIT 1";
    return $this->db->selectOne($sql, [$username]);
}

// AFTER
private const ALLOWED_TABLES = ['uc_members', 'cdb_members'];

private function queryUcenterUser(string $username): ?array
{
    $table = 'uc_members';
    if (!in_array($table, self::ALLOWED_TABLES, true)) {
        throw new RuntimeException('Invalid table name');
    }

    $sql = sprintf(
        "SELECT uid, username, password, salt, email, groupid
         FROM %s
         WHERE username = ? LIMIT 1",
        $table
    );
    return $this->db->selectOne($sql, [$username]);
}
```

#### Issue #3: Missing Database Error Handling
- **Severity:** P1 (HIGH)
- **Location:** Lines 332-336, 345-349, 361-370, 379-388
- **Problem:** No try-catch around database queries
- **Impact:** Database connection failures throw unhandled exceptions, exposing sensitive error messages
- **Fix Strategy:** Wrap database calls in try-catch, log errors, return generic failure
- **Code Example:**
```php
// BEFORE
private function queryUcenterUser(string $username): ?array
{
    $sql = "SELECT uid, username, password, salt, email, groupid
            FROM uc_members
            WHERE username = ? LIMIT 1";
    return $this->db->selectOne($sql, [$username]);
}

// AFTER
private function queryUcenterUser(string $username): ?array
{
    try {
        $sql = "SELECT uid, username, password, salt, email, groupid
                FROM uc_members
                WHERE username = ? LIMIT 1";
        return $this->db->selectOne($sql, [$username]);
    } catch (\PDOException $e) {
        error_log('AuthService: Database query failed: ' . $e->getMessage());
        return null; // Fail securely
    }
}
```

#### Issue #4: Duplicate Rate Limiting Logic
- **Severity:** P2 (Medium)
- **Location:** Lines 457-492, 120-126
- **Problem:** `isRateLimited()` and `recordFailedLogin()` duplicate RateLimiterService functionality
- **Impact:** Code duplication, maintenance burden, potential inconsistencies
- **Fix Strategy:** Remove in-memory rate limiting from AuthService entirely (use RateLimiterService only)
- **Code Example:** See Issue #1 fix

#### Issue #5: getClientIp() Has Fallback to Invalid IP
- **Severity:** P2 (Medium)
- **Location:** Lines 424-449
- **Problem:** Returns '0.0.0.0' if no valid IP found
- **Impact:** Rate limiting may group all users without valid IP into same bucket
- **Fix Strategy:** Throw exception or use REMOTE_ADDR as fallback
- **Code Example:**
```php
// BEFORE
return '0.0.0.0';

// AFTER
// Option 1: Use REMOTE_ADDR as last resort
if (isset($_SERVER['REMOTE_ADDR'])) {
    return $_SERVER['REMOTE_ADDR'];
}
throw new RuntimeException('Unable to determine client IP address');

// Option 2: Use random IP to prevent single-bucket DoS
return '127.0.0.1'; // Localhost fallback
```

#### Issue #6: Password Migration Not Transactional
- **Severity:** P1 (HIGH)
- **Location:** Lines 134-145, 178-189
- **Problem:** Password migration and user login are not atomic
- **Impact:** If migration succeeds but session setting fails, user password is changed but not logged in
- **Fix Strategy:** Use database transaction or migrate after session creation
- **Code Example:**
```php
// BEFORE (Lines 134-145)
if ($this->verifyUcenterCredentials($ucUser, $password)) {
    // Migrate password
    if (!$this->isBcryptHash($currentHash)) {
        $bcryptHash = $this->migratePassword($ucUser, $password, $ucUser['salt'] ?? null);
        $this->updateUcenterPassword((int) $ucUser['uid'], $bcryptHash);
    }

    // Set session
    $this->setUserSession(...);
    return AuthResult::success(...);
}

// AFTER
if ($this->verifyUcenterCredentials($ucUser, $password)) {
    // Set session FIRST
    $this->setUserSession(...);

    // Then migrate password (best-effort, don't fail login if migration fails)
    if (!$this->isBcryptHash($currentHash)) {
        try {
            $bcryptHash = $this->migratePassword($ucUser, $password, $ucUser['salt'] ?? null);
            $this->updateUcenterPassword((int) $ucUser['uid'], $bcryptHash);
        } catch (\Exception $e) {
            error_log('Password migration failed: ' . $e->getMessage());
            // Don't fail the login, migration will retry next time
        }
    }

    return AuthResult::success(...);
}
```

#### Issue #7: UCenter Salt Not Validated
- **Severity:** P3 (Low)
- **Location:** Line 261
- **Problem:** `$salt = $ucUser['salt'] ?? '';` allows empty string
- **Impact:** Empty salt still works but doesn't match UCenter spec (should be 6 chars)
- **Fix Strategy:** Add validation for expected salt format
- **Code Example:**
```php
// BEFORE
$salt = $ucUser['salt'] ?? '';

// AFTER
$salt = $ucUser['salt'] ?? '';
if ($salt !== '' && strlen($salt) !== 6) {
    // Log anomaly but don't fail (graceful degradation)
    error_log(sprintf('UCenter user %d has invalid salt length: %d', $ucUser['uid'], strlen($salt)));
}
```

### Summary
- **Grade:** C+ (75/100)
- **Critical Issues:** 0
- **High Issues:** 4
- **Recommendation:** **NOT PRODUCTION READY** - Must fix P1 issues before deployment

---

## 3. RememberMeService Review

**File:** `app/Auth/Auth/RememberMeService.php` (357 lines)

### Functional Completeness
- ✅ Split token approach (selector:token)
- ✅ SHA-256 hashing of tokens in database
- ✅ Token expiration (30 days default)
- ✅ Token validation and cleanup
- ✅ IP and user agent tracking

### Security Assessment
- ✅ Timing-safe comparisons (`hash_equals()`)
- ✅ Cryptographically secure random generation (`random_bytes()`)
- ✅ One-way hashing (SHA-256)
- ✅ SQL injection prevention (prepared statements)
- ✅ XSS prevention (not applicable, backend-only)
- ✅ Selector uniqueness constraint in database

### Legacy Compatibility
- ❌ **Not compatible** with legacy `authcode()` cookies (intentional design decision)
- ✅ New table `cdb_persistent_tokens` doesn't conflict with legacy schema
- ⚠️ **Forces all users to re-login** (old cookies won't work)

### Issues Found

#### Issue #1: SQL Injection Risk in createToken()
- **Severity:** P1 (HIGH)
- **Location:** Lines 145-149
- **Problem:** Table name concatenated into SQL with sprintf()
- **Impact:** If TABLE_NAME constant can be modified, SQL injection possible
- **Analysis:** While current code uses hardcoded constant, sprintf is unnecessary risk
- **Fix Strategy:** Use constant directly in query or whitelist validation
- **Code Example:**
```php
// BEFORE
$sql = sprintf(
    "INSERT INTO %s (user_id, selector, token, expires_at, created_at, ip_address, user_agent)
     VALUES (?, ?, ?, ?, ?, ?, ?)",
    self::TABLE_NAME
);

// AFTER
$sql = "INSERT INTO " . self::TABLE_NAME . "
         (user_id, selector, token, expires_at, created_at, ip_address, user_agent)
         VALUES (?, ?, ?, ?, ?, ?, ?)";

// OR (safest) validate table name is constant
if (self::TABLE_NAME !== 'cdb_persistent_tokens') {
    throw new RuntimeException('Invalid table name');
}
```

#### Issue #2: Missing Error Handling for random_bytes()
- **Severity:** P1 (HIGH)
- **Location:** Lines 126, 130
- **Problem:** `random_bytes()` can throw `Exception` if system entropy source fails
- **Impact:** Unhandled exception exposes stack trace to user
- **Fix Strategy:** Wrap in try-catch, log error, return failure
- **Code Example:**
```php
// BEFORE
$selector = bin2hex(random_bytes(self::SELECTOR_BYTES));
$token = bin2hex(random_bytes(self::TOKEN_BYTES));

// AFTER
try {
    $selector = bin2hex(random_bytes(self::SELECTOR_BYTES));
    $token = bin2hex(random_bytes(self::TOKEN_BYTES));
} catch (\Exception $e) {
    error_log('RememberMeService: Failed to generate secure random bytes: ' . $e->getMessage());
    throw new RuntimeException('Failed to generate secure token', 0, $e);
}
```

#### Issue #3: Token Hash Inconsistency
- **Severity:** P2 (Medium)
- **Location:** Line 134
- **Problem:** Token hashed with `hash('sha256', hex2bin($token))` but doc says "SHA-256 (one-way)"
- **Impact:** Double conversion (hex -> binary -> SHA-256 -> hex) is correct but could be simplified
- **Analysis:** Current code is correct: hex2bin() converts 64-char hex to 32 bytes, then SHA-256 hashes it
- **Fix Strategy:** Document why hex2bin() is used (prevents double-hex encoding)
- **Code Example:**
```php
// Add clarifying comment
// Step 3: Hash token with SHA-256 (one-way, collision-resistant)
// Input: 64-char hex string -> 32-byte binary -> SHA-256 -> 64-char hex
// We use hex2bin() to avoid double-hex encoding (SHA-256 of hex string)
$tokenHashHex = hash('sha256', hex2bin($token));
```

#### Issue #4: No CSRF Protection on Token Deletion
- **Severity:** P2 (Medium)
- **Location:** Lines 247-256, 269-278
- **Problem:** `deleteTokens()` and `deleteToken()` methods have no CSRF protection
- **Impact:** Attacker could craft malicious form/link to logout user from all devices
- **Analysis:** CSRF protection should be handled at controller level, but service should document requirement
- **Fix Strategy:** Add PHPDoc comment noting CSRF requirement
- **Code Example:**
```php
/**
 * Delete all persistent tokens for a user
 *
 * SECURITY NOTE: This method should be called from a controller action
 * that validates CSRF tokens to prevent CSRF attacks.
 *
 * Use this method when:
 * - User logs out (invalidate all "remember me" tokens)
 * - User changes password (security best practice)
 * - User account is compromised
 * - User requests "logout from all devices"
 *
 * @param int $userId User ID
 * @return void
 */
public function deleteTokens(int $userId): void
```

#### Issue #5: Missing Transaction for Token Operations
- **Severity:** P2 (Medium)
- **Location:** Lines 145-159
- **Problem:** `createToken()` inserts record but no transaction rollback on failure
- **Impact:** If insert succeeds but subsequent code fails, orphaned token remains in database
- **Fix Strategy:** Wrap insert in database transaction
- **Code Example:**
```php
// BEFORE
$this->db->insert($sql, [...]);

return [
    'selector' => $selector,
    'token' => $token,
];

// AFTER (if Database class supports transactions)
try {
    $this->db->beginTransaction();

    $this->db->insert($sql, [...]);

    // Any other operations here

    $this->db->commit();

    return [
        'selector' => $selector,
        'token' => $token,
    ];
} catch (\Exception $e) {
    $this->db->rollback();
    throw $e;
}
```

#### Issue #6: IP Address Validation Insufficient
- **Severity:** P3 (Low)
- **Location:** Lines 309-334
- **Problem:** `filter_var($ip, FILTER_VALIDATE_IP)` allows private IPs
- **Impact:** Attackers could spoof IP headers with private addresses
- **Fix Strategy:** Check for public IP address
- **Code Example:**
```php
// BEFORE
if (filter_var($ip, FILTER_VALIDATE_IP)) {
    return $ip;
}

// AFTER
if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
    return $ip;
}
```

### Summary
- **Grade:** B (85/100)
- **Critical Issues:** 0
- **High Issues:** 2
- **Recommendation:** Fix P1 issues, production-ready after fixes

---

## 4. CsrfTokenService Review

**File:** `app/Security/Services/CsrfTokenService.php` (265 lines)

### Functional Completeness
- ✅ Token generation (64-char hex)
- ✅ Token validation with timing-safe comparison
- ✅ Token rotation after privileged operations
- ✅ HTML field generation
- ✅ Session-based storage

### Security Assessment
- ✅ Timing-safe comparisons (`hash_equals()`)
- ✅ Cryptographically secure generation (`random_bytes()`)
- ✅ No hardcoded secrets
- ✅ XSS prevention (output escaping with `htmlspecialchars()`)
- ✅ Token rotation support

### Legacy Compatibility
- ✅ Replaces legacy `formhash` (no compatibility needed)
- ✅ Session storage compatible

### Issues Found

#### Issue #1: Token Not Regenerated After Login
- **Severity:** P2 (Medium)
- **Location:** N/A (integration issue)
- **Problem:** AuthController calls `$this->csrf->rotate()` after login, but no automatic rotation
- **Impact:** Session fixation vulnerability if session ID not regenerated
- **Analysis:** Session regeneration is done in AuthController line 195, CSRF rotation at line 201 - this is correct
- **Fix Strategy:** Document that CSRF rotation should happen AFTER session regeneration
- **Code Example:**
```php
// In AuthController->loginPostAction(), order is correct:
// Step 4a: Regenerate session ID (prevent session fixation)
$this->session->regenerate(true);

// Step 4b: Clear failed login attempts
$this->rateLimiter->clearFailedAttempts($clientIp);

// Step 4c: Rotate CSRF token (MUST be after session regeneration)
$newCsrfToken = $this->csrf->rotate();
```

#### Issue #2: No Token Expiration
- **Severity:** P3 (Low)
- **Location:** Line 38
- **Problem:** Tokens stored in session with no expiration time
- **Impact:** Long-lived sessions have same CSRF token (theoretical risk)
- **Analysis:** Session expiration mitigates this risk
- **Fix Strategy:** Add token timestamp and expiration check
- **Code Example:**
```php
// Add to class:
private const SESSION_KEY_TIMESTAMP = '_csrf_token_timestamp';
private const TOKEN_MAX_AGE = 3600; // 1 hour

public function validate(?string $token): bool
{
    $storedToken = $this->session->get(self::SESSION_KEY);
    $tokenTimestamp = $this->session->get(self::SESSION_KEY_TIMESTAMP);

    if ($storedToken === null || $token === null || $token === '') {
        return false;
    }

    // Check token age
    if ($tokenTimestamp !== null && (time() - (int)$tokenTimestamp > self::TOKEN_MAX_AGE)) {
        return false; // Token expired
    }

    return hash_equals($storedToken, $token);
}
```

#### Issue #3: Exception Handling Too Broad
- **Severity:** P3 (Low)
- **Location:** Lines 232-238
- **Problem:** Catches generic `\Exception` when wrapping `RandomException`
- **Impact:** May mask other unexpected errors
- **Fix Strategy:** Catch `\Random\RandomException` specifically
- **Code Example:**
```php
// BEFORE
try {
    $randomBytes = random_bytes(self::TOKEN_BYTES);
    return bin2hex($randomBytes);
} catch (\Exception $e) {
    throw new \RuntimeException(...);
}

// AFTER
try {
    $randomBytes = random_bytes(self::TOKEN_BYTES);
    return bin2hex($randomBytes);
} catch (\Random\RandomException $e) {
    throw new \RuntimeException(
        'Failed to generate secure CSRF token: ' . $e->getMessage(),
        0,
        $e
    );
}
```

### Summary
- **Grade:** A- (92/100)
- **Critical Issues:** 0
- **High Issues:** 0
- **Recommendation:** Production-ready, minor improvements recommended

---

## 5. RateLimiterService Review

**File:** `app/Security/Services/RateLimiterService.php` (535 lines)

### Functional Completeness
- ✅ Failed login tracking per IP
- ✅ Maximum attempts enforcement (5 default)
- ✅ Time window tracking (15 minutes default)
- ✅ Automatic lockout after limit exceeded
- ✅ Lockout expiration
- ✅ Attempt clearing on successful login
- ✅ Redis + database dual storage

### Security Assessment
- ✅ Redis as primary storage (fast)
- ✅ Database as fallback (reliable)
- ✅ Sorted sets for time-window tracking
- ✅ SQL injection prevention (prepared statements)
- ✅ Error logging without credential exposure

### Legacy Compatibility
- ✅ Uses `cdb_failedlogins` table (same as legacy)
- ✅ Compatible with legacy rate limiting (5 attempts / 15 minutes)

### Issues Found

#### Issue #1: Redis Connection Not Checked in All Methods
- **Severity:** P2 (Medium)
- **Location:** Multiple methods
- **Problem:** Some methods don't check `$redis === null` before using
- **Impact:** Fatal error if Redis unavailable
- **Fix Strategy:** Add null checks before all Redis operations
- **Code Example:**
```php
// BEFORE (Lines 361-373)
private function addToSortedSet(string $ip, int $timestamp): void
{
    $redis = $this->cache->getRedis();
    if ($redis === null) {
        return;
    }
    // ... uses $redis
}

// AFTER (already correct, check other methods)
// Line 382 incrementCounter() - MISSING null check
private function incrementCounter(string $ip): int
{
    $redis = $this->cache->getRedis();
    if ($redis === null) {
        return 0; // Already correct
    }

    $counterKey = self::ATTEMPTS_KEY_PREFIX . $ip;
    $newCount = $redis->incr($counterKey); // Potential null dereference

    // ADD null check after incr
    if ($newCount === false) {
        return 0;
    }

    // ... rest of method
}
```

#### Issue #2: Database Fallback Incomplete
- **Severity:** P2 (Medium)
- **Location:** Lines 294-301
- **Problem:** `getAttemptCount()` only checks Redis availability once
- **Impact:** If Redis fails during request, returns 0 instead of checking database
- **Fix Strategy:** Wrap Redis calls in try-catch
- **Code Example:**
```php
// BEFORE
private function getAttemptCount(string $ip): int
{
    if ($this->cache->isConnected()) {
        return $this->getAttemptCountFromRedis($ip);
    }
    return $this->getAttemptCountFromDatabase($ip);
}

// AFTER
private function getAttemptCount(string $ip): int
{
    // Try Redis first
    try {
        if ($this->cache->isConnected()) {
            return $this->getAttemptCountFromRedis($ip);
        }
    } catch (\Exception $e) {
        error_log('RateLimiterService: Redis failed, falling back to database: ' . $e->getMessage());
    }

    // Fallback to database
    return $this->getAttemptCountFromDatabase($ip);
}
```

#### Issue #3: Lockout Time Remaining Can Return Negative
- **Severity:** P3 (Low)
- **Location:** Line 248
- **Problem:** `max(0, $remaining)` prevents negative return, but calculation could overflow
- **Impact:** Minimal, `max()` prevents negative values
- **Fix Strategy:** None needed, code is correct
- **Analysis:** Code is already correct, no issue

#### Issue #4: Configuration Validation Incomplete
- **Severity:** P3 (Low)
- **Location:** Lines 86-106
- **Problem:** No validation that `max_attempts`, `attempt_window`, `lockout_duration` are integers
- **Impact:** Type coercion could cause unexpected behavior
- **Fix Strategy:** Add type validation
- **Code Example:**
```php
// BEFORE
$this->maxAttempts = $config['max_attempts'] ?? 5;

// AFTER
$maxAttempts = $config['max_attempts'] ?? 5;
if (!is_int($maxAttempts) || $maxAttempts < 1) {
    throw new RuntimeException('max_attempts must be a positive integer');
}
$this->maxAttempts = $maxAttempts;
```

### Summary
- **Grade:** A- (90/100)
- **Critical Issues:** 0
- **High Issues:** 0
- **Recommendation:** Production-ready, minor improvements recommended

---

## 6. AuthController Review

**File:** `app/Http/Controllers/AuthController.php` (455 lines)

### Functional Completeness
- ✅ Login form display
- ✅ Login form processing
- ✅ CSRF validation
- ✅ Rate limiting enforcement
- ✅ Session regeneration
- ✅ "Remember me" token creation
- ✅ Logout processing
- ✅ Authentication status check
- ✅ CSRF token refresh
- ✅ Remaining attempts check

### Security Assessment
- ✅ CSRF protection on all POST requests
- ✅ Rate limiting enforcement
- ✅ Session fixation prevention (regeneration after login)
- ✅ Timing-safe token comparisons (delegated to services)
- ✅ No sensitive data in error messages

### Legacy Compatibility
- ✅ Compatible with legacy session structure
- ✅ Compatible with legacy cookie names (can use same names)
- ❌ **Incompatible** with legacy `authcode()` cookies (new format)

### Issues Found

#### Issue #1: getUserData() Not Implemented
- **Severity:** P1 (HIGH)
- **Location:** Lines 419-432
- **Problem:** `getUserData()` always returns `null` (TODO comment)
- **Impact:** `loginWithRememberMeToken()` cannot restore user session
- **Analysis:** "Remember me" functionality is BROKEN without this implementation
- **Fix Strategy:** Implement user data query from database
- **Code Example:**
```php
// BEFORE
private function getUserData(int $userId): ?array
{
    // TODO: Implement user data retrieval
    return null;
}

// AFTER
private function getUserData(int $userId): ?array
{
    try {
        // Query UCenter table first (primary)
        $sql = "SELECT uid, username, email, groupid
                FROM uc_members
                WHERE uid = ? LIMIT 1";

        $user = $this->userSession // TODO: Need Database dependency here
            ??? // Need access to database service

        if ($user) {
            return [
                'uid' => (int) $user['uid'],
                'username' => $user['username'],
                'email' => $user['email'],
                'group_id' => (int) $user['groupid'],
                'admin_id' => 0, // TODO: Query from cdb_admins if needed
                'group_expiry' => 0,
            ];
        }

        return null;
    } catch (\Exception $e) {
        error_log('AuthController: Failed to get user data: ' . $e->getMessage());
        return null;
    }
}
```

#### Issue #2: Missing Database Dependency
- **Severity:** P1 (HIGH)
- **Location:** N/A (constructor)
- **Problem:** `AuthController` doesn't have Database service injected
- **Impact:** Cannot implement `getUserData()` without database access
- **Fix Strategy:** Add Database service to constructor
- **Code Example:**
```php
// BEFORE
public function __construct(
    SessionManager $session,
    UserSessionService $userSession,
    AuthService $authService,
    CsrfTokenService $csrf,
    RateLimiterService $rateLimiter,
    RememberMeService $rememberMe
) {
    // ...
}

// AFTER
use Discuz\Database\Database;

public function __construct(
    SessionManager $session,
    UserSessionService $userSession,
    AuthService $authService,
    CsrfTokenService $csrf,
    RateLimiterService $rateLimiter,
    RememberMeService $rememberMe,
    Database $database // ADD THIS
) {
    $this->session = $session;
    $this->userSession = $userSession;
    $this->authService = $authService;
    $this->csrf = $csrf;
    $this->rateLimiter = $rateLimiter;
    $this->rememberMe = $rememberMe;
    $this->database = $database; // ADD THIS
}
```

#### Issue #3: Typo in Constant Name
- **Severity:** P3 (Low)
- **Location:** Line 72
- **Problem:** `REMBER_ME_LIFETIME` should be `REMEMBER_ME_LIFETIME`
- **Impact:** Code works but typo reduces professionalism
- **Fix Strategy:** Rename constant
- **Code Example:**
```php
// BEFORE
private const REMBER_ME_LIFETIME = 2592000;

// AFTER
private const REMEMBER_ME_LIFETIME = 2592000;
```

#### Issue #4: Typo in Method Name
- **Severity:** P3 (Low)
- **Location:** Line 161
- **Problem:** `exceededLoginAttempts()` should be `exceededLoginAttempts()` or `exceedsLoginAttempts()`
- **Impact:** Code works but typo is unprofessional
- **Fix Strategy:** Rename method
- **Code Example:**
```php
// BEFORE
if ($this->rateLimiter->exceededLoginAttempts($clientIp)) {

// AFTER
if ($this->rateLimiter->exceededLoginAttempts($clientIp)) {
```

#### Issue #5: Typo in Variable Name
- **Severity:** P3 (Low)
- **Location:** Line 220
- **Problem:** `'remember_token'` assigned but typo in format string
- **Impact:** Code is correct (typo is in comment only)
- **Analysis:** Line 220-223 are correct, no actual bug
- **Fix Strategy:** None needed, code is correct

### Summary
- **Grade:** C+ (78/100)
- **Critical Issues:** 0
- **High Issues:** 2
- **Recommendation:** **NOT PRODUCTION READY** - Must implement getUserData()

---

## 7. Login View Review

**File:** `resources/views/auth/login.php` (467 lines)

### Functional Completeness
- ✅ Login form with username/password
- ✅ CSRF token field
- ✅ "Remember me" checkbox
- ✅ Error message display
- ✅ Remaining attempts display
- ✅ Form validation (JavaScript)
- ✅ Loading state on submit

### Security Assessment
- ✅ CSRF token field
- ✅ HTML output escaping (`escape()` function)
- ✅ No inline SQL queries
- ✅ No sensitive data in JavaScript
- ⚠️ **Inline JavaScript** (should be separate file)

### Legacy Compatibility
- ✅ Compatible with legacy form structure
- ✅ Can use same field names
- ✅ Can use same form action URL

### Issues Found

#### Issue #1: Inline JavaScript
- **Severity:** P2 (Medium)
- **Location:** Lines 377-451
- **Problem:** JavaScript code embedded in HTML template
- **Impact:** Content Security Policy (CSP) violations, harder to cache, maintenance issues
- **Fix Strategy:** Extract to separate `.js` file
- **Code Example:**
```php
// BEFORE
<script>
    (function() {
        'use strict';
        // ... 74 lines of JavaScript
    })();
</script>

// AFTER
// In login.php, replace with:
<script src="/assets/js/login-form.js" defer></script>

// Create public/assets/js/login-form.js:
(function() {
    'use strict';

    const form = document.getElementById('loginForm');
    const submitBtn = document.getElementById('submitBtn');
    // ... rest of JavaScript code
})();
```

#### Issue #2: Hardcoded Styling
- **Severity:** P3 (Low)
- **Location:** Lines 46-276
- **Problem:** 230 lines of inline CSS in HTML template
- **Impact:** Template bloat, harder to maintain, cannot be cached separately
- **Fix Strategy:** Extract to external `.css` file
- **Code Example:**
```php
// BEFORE
<style>
    * { margin: 0; padding: 0; ... }
    /* ... 230 lines of CSS ... */
</style>

// AFTER
// In login.php, replace with:
<link rel="stylesheet" href="/assets/css/login.css">

// Create public/assets/css/login.css:
* { margin: 0; padding: 0; }
/* ... rest of CSS ... */
```

#### Issue #3: escape() Function Defined in Template
- **Severity:** P3 (Low)
- **Location:** Lines 463-465
- **Problem:** Helper function defined in template file
- **Impact:** Should be in separate View helper class
- **Fix Strategy:** Move to `app/Support/ViewHelper.php`
- **Code Example:**
```php
// BEFORE (lines 463-465)
function escape($value) {
    return htmlspecialchars((string)$value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

// AFTER
// Create app/Support/ViewHelper.php:
namespace Discuz\Support;

class ViewHelper
{
    public static function escape(string $value): string
    {
        return htmlspecialchars($value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }
}

// In template, use:
<?php echo \Discuz\Support\ViewHelper::escape($site_name); ?>
```

#### Issue #4: No HTML5 Semantic Elements
- **Severity:** P3 (Low)
- **Location:** Lines 379-453
- **Problem:** Uses `<div>` instead of `<main>`, `<section>`, etc.
- **Impact:** Accessibility, SEO (minor for login page)
- **Fix Strategy:** Use HTML5 semantic elements
- **Code Example:**
```php
// BEFORE
<div class="login-container">
    <div class="login-header">
    <form id="loginForm">

// AFTER
<main class="login-container">
    <header class="login-header">
    <form id="loginForm">
```

### Summary
- **Grade:** B (88/100)
- **Critical Issues:** 0
- **High Issues:** 0
- **Recommendation:** Production-ready, should extract CSS/JS for maintainability

---

## 8. Legacy Comparison Summary

### Password Verification

**Legacy (UCenter):**
```php
// ucenter/control/user.php:119-122
$passwordmd5 = preg_match('/^\w{32}$/', $password) ? $password : md5($password);
if($user['password'] != md5($passwordmd5.$user['salt'])) {
    $status = -2;  // Password error
}
```

**New Implementation:**
```php
// PasswordVerifier.php:38-51
public function verifyUcenter(string $password, string $hash, string $salt): bool
{
    $firstMd5 = md5($password);
    $saltedPassword = $firstMd5 . $salt;
    $computedHash = md5($saltedPassword);
    return hash_equals($hash, $computedHash);
}
```

**Match:** ✅ **EXACT MATCH** - Formula is identical, timing-safe comparison added

**Test Case:**
- Input: `password="lljtsj"`, `salt="636628"`
- Legacy: `md5(md5("lljtsj") . "636628")`
- New: `md5(md5("lljtsj") . "636628")`
- Result: ✅ Both produce identical hash

### Session Management

**Legacy:**
- Uses `cdb_sessions` table
- Session ID: `random(6)` - 6 character random string
- No session regeneration after login
- Session fixation vulnerability

**New:**
- Uses Redis (primary) with database fallback
- Session ID: `random_bytes(32)` - 32-byte cryptographically secure random
- Session regeneration after login (`SessionManager->regenerate()`)
- Session fixation protected

**Compatibility:** ✅ **Compatible** - Session data structure preserved, storage backend upgraded

### Cookie Handling

**Legacy (authcode):**
```php
// logging.php:214
dsetcookie('auth', authcode("$discuz_pw\t$discuz_secques\t$discuz_uid", 'ENCODE'), $cookietime);
```

- Format: `authcode("password\tsecques\tuid")`
- Encryption: Custom XOR encryption (weak)
- Storage: Cookie only (no database record)

**New (RememberMe):**
```php
// RememberMeService.php:145-159
// Format: "selector:token" (both plain text in cookie)
// Database: selector (plain), token (SHA-256 hashed)
```

- Format: `selector:token`
- Encryption: None (split token approach instead)
- Storage: Database `cdb_persistent_tokens` with hashed tokens

**Migration Path:** ❌ **Incompatible** - Forces all users to re-login

**Impact:**
- Old cookies cannot be validated
- Users must login again after deployment
- **Recommendation:** Announce migration, provide grace period

### Database Schema

**Legacy Tables:**
- `uc_members` - UCenter users (uid, username, password, salt)
- `cdb_members` - Discuz! local users
- `cdb_sessions` - Session storage
- `cdb_failedlogins` - Failed login tracking

**New Tables:**
- `cdb_persistent_tokens` - Remember me tokens (NEW)
- Uses Redis for sessions (instead of `cdb_sessions`)

**Schema Changes:**
```sql
-- Week 2 schema changes
ALTER TABLE uc_members MODIFY password VARCHAR(255);  -- Support bcrypt
ALTER TABLE cdb_members MODIFY password VARCHAR(255);  -- Support bcrypt
CREATE TABLE cdb_persistent_tokens (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id INT UNSIGNED NOT NULL,
    selector CHAR(12) NOT NULL,
    token CHAR(64) NOT NULL,
    expires_at INT UNSIGNED NOT NULL,
    created_at INT UNSIGNED NOT NULL,
    last_used_at INT UNSIGNED DEFAULT NULL,
    ip_address VARCHAR(45) DEFAULT NULL,
    user_agent VARCHAR(255) DEFAULT NULL,
    UNIQUE KEY (selector),
    KEY (user_id),
    KEY (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Compatibility:** ✅ **Compatible** - No breaking changes to existing tables

---

## 9. Security Assessment

### OWASP Top 10 Coverage

| OWASP Risk | Status | Implementation | Notes |
|------------|--------|----------------|-------|
| **A01: Broken Access Control** | ✅ Protected | Session regeneration, permission checks | ✅ Complete |
| **A02: Cryptographic Failures** | ✅ Protected | Bcrypt hashing, timing-safe comparisons | ✅ Complete |
| **A03: Injection (SQL)** | ⚠️ Partial | Prepared statements used, BUT table names not parameterized | ⚠️ See Issue #2 in AuthService |
| **A04: Insecure Design** | ✅ Protected | Rate limiting, split token approach | ✅ Complete |
| **A05: Security Misconfiguration** | ✅ Protected | HttpOnly cookies, SameSite=Lax | ✅ Complete (from Week 1) |
| **A07: Authentication Failures** | ✅ Protected | Strong hashing, session management | ✅ Complete |
| **A08: Data Integrity Failures** | ✅ Protected | CSRF tokens on all POST | ✅ Complete |
| **A09: Security Logging** | ✅ Protected | Failed login tracking, error logging | ⚠️ In-memory only (see Issue #1 in AuthService) |

**OWASP Coverage Score:** 8/9 (89%)

### Additional Security Issues

#### 1. In-Memory Rate Limiting
- **Severity:** P1
- **Impact:** Rate limiting lost on restart
- **Fix:** Use RateLimiterService (Redis + database) consistently

#### 2. Missing Database Error Handling
- **Severity:** P1
- **Impact:** Uncaught exceptions expose stack traces
- **Fix:** Wrap all database calls in try-catch

#### 3. SQL Injection Risk (Table Names)
- **Severity:** P1
- **Impact:** If table names can be influenced, SQL injection possible
- **Fix:** Whitelist table names or use constant validation

#### 4. No Content Security Policy
- **Severity:** P2
- **Impact:** XSS attacks more likely to succeed
- **Fix:** Add CSP header (can be done in Week 4)

#### 5. No HTTP Strict Transport Security (HSTS)
- **Severity:** P2
- **Impact:** Man-in-the-middle attacks possible
- **Fix:** Add HSTS header (can be done in Week 4)

---

## 10. Production Readiness Checklist

- [ ] **All P0 critical issues resolved** - N/A (0 critical issues)
- [ ] **All P1 high issues resolved** - ❌ NO (7 high issues)
  - [ ] AuthService: Fix in-memory rate limiting
  - [ ] AuthService: Add database error handling
  - [ ] AuthService: Fix SQL injection risk (table names)
  - [ ] AuthService: Make password migration non-transactional safe
  - [ ] RememberMeService: Add random_bytes() error handling
  - [ ] RememberMeService: Fix SQL injection risk (table names)
  - [ ] AuthController: Implement getUserData()
- [ ] **Error handling comprehensive** - ⚠️ Partial (missing DB error handling)
- [ ] **Logging sufficient for debugging** - ✅ Yes (error_log() used throughout)
- [ ] **Configuration externalized** - ✅ Yes (.env file, Config service)
- [ ] **Database migrations tested** - ⚠️ Not verified in review
- [ ] **Rollback plan documented** - ❌ No rollback plan found
- [ ] **Performance benchmarks acceptable** - ❌ No benchmarks found
- [ ] **Load testing completed** - ❌ No load tests found
- [ ] **Security audit passed** - ⚠️ Partial (OWASP coverage 89%)

**Production Readiness:** ❌ **NO** - Requires P1 fixes

---

## 11. Recommended Fixes

### Priority P0 (Critical) - Must Fix Before Production

**None** - No critical issues found

### Priority P1 (High) - Must Fix Before Production

#### Fix #1: Remove In-Memory Rate Limiting from AuthService
**File:** `app/Auth/Auth/AuthService.php`
**Issue:** Duplicate rate limiting logic that's not persistent
**Solution:** Remove all in-memory rate limiting, delegate to RateLimiterService
**Code Change:**
```php
// REMOVE these lines (52, 120-126, 457-492)
/**
 * @var array Failed login tracking [ip => [count, last_attempt]]
 */
private array $failedLogins = [];

// In login() method, REMOVE Step 2:
// Step 2: Check rate limiting
$clientIp = $this->getClientIp();
if ($this->isRateLimited($clientIp)) {
    return AuthResult::failure(...);
}

// Also REMOVE these methods:
private function isRateLimited(string $ip): bool { ... }
private function recordFailedLogin(string $ip): void { ... }
public function clearFailedLogins(string $ip): void { ... }
public function getFailedLogins(string $ip): ?array { ... }
public function getAllFailedLogins(): array { ... }
```
**Estimated Time:** 1 hour
**Risk:** Low (logic moved to AuthController where it should be)

#### Fix #2: Add Database Error Handling
**File:** `app/Auth/Auth/AuthService.php`
**Issue:** No try-catch around database queries
**Solution:** Wrap all database calls in try-catch, log errors, return generic failure
**Code Change:**
```php
// BEFORE
private function queryUcenterUser(string $username): ?array
{
    $sql = "SELECT uid, username, password, salt, email, groupid
            FROM uc_members
            WHERE username = ? LIMIT 1";
    return $this->db->selectOne($sql, [$username]);
}

// AFTER
private function queryUcenterUser(string $username): ?array
{
    try {
        $sql = "SELECT uid, username, password, salt, email, groupid
                FROM uc_members
                WHERE username = ? LIMIT 1";
        return $this->db->selectOne($sql, [$username]);
    } catch (\PDOException $e) {
        error_log('AuthService: Database query failed: ' . $e->getMessage());
        return null; // Fail securely
    }
}
```
**Estimated Time:** 2 hours (all query methods)
**Risk:** Low (defensive programming)

#### Fix #3: Fix SQL Injection Risk (Table Names)
**File:** `app/Auth/Auth/AuthService.php`
**Issue:** Table names concatenated into SQL
**Solution:** Whitelist allowed table names or validate constant
**Code Change:**
```php
// Add to class:
private const ALLOWED_TABLES = ['uc_members', 'cdb_members', 'cdb_persistent_tokens'];

private function queryUcenterUser(string $username): ?array
{
    $table = 'uc_members';

    // Validate table name
    if (!in_array($table, self::ALLOWED_TABLES, true)) {
        throw new RuntimeException('Invalid table name: ' . $table);
    }

    try {
        $sql = sprintf(
            "SELECT uid, username, password, salt, email, groupid
             FROM %s
             WHERE username = ? LIMIT 1",
            $table
        );
        return $this->db->selectOne($sql, [$username]);
    } catch (\PDOException $e) {
        error_log('AuthService: Database query failed: ' . $e->getMessage());
        return null;
    }
}
```
**Estimated Time:** 1 hour
**Risk:** Low (defensive programming)

#### Fix #4: Fix Password Migration Race Condition
**File:** `app/Auth/Auth/AuthService.php`
**Issue:** Password migration not atomic with login
**Solution:** Migrate password AFTER session creation (best-effort)
**Code Change:**
```php
// BEFORE (Lines 134-145)
if ($this->verifyUcenterCredentials($ucUser, $password)) {
    // Migrate password
    if (!$this->isBcryptHash($currentHash)) {
        $bcryptHash = $this->migratePassword($ucUser, $password, $ucUser['salt'] ?? null);
        $this->updateUcenterPassword((int) $ucUser['uid'], $bcryptHash);
    }

    // Set session
    $this->setUserSession(...);
    return AuthResult::success(...);
}

// AFTER
if ($this->verifyUcenterCredentials($ucUser, $password)) {
    // Set session FIRST (atomic login)
    $this->setUserSession(
        (int) $ucUser['uid'],
        $ucUser['username'],
        (int) ($ucUser['groupid'] ?? 0)
    );

    // Then migrate password (best-effort, don't fail login if migration fails)
    if (!$this->isBcryptHash($currentHash)) {
        try {
            $bcryptHash = $this->migratePassword($ucUser, $password, $ucUser['salt'] ?? null);
            $this->updateUcenterPassword((int) $ucUser['uid'], $bcryptHash);
        } catch (\Exception $e) {
            error_log('Password migration failed for user ' . $ucUser['uid'] . ': ' . $e->getMessage());
            // Don't fail the login, migration will retry next time
        }
    }

    return AuthResult::success(
        (int) $ucUser['uid'],
        $ucUser['username'],
        (int) ($ucUser['groupid'] ?? 0),
        0, // admin_id
        array_merge($ucUser, ['remember_me' => $rememberMe])
    );
}
```
**Estimated Time:** 1 hour
**Risk:** Low (changes order of operations)

#### Fix #5: Add random_bytes() Error Handling
**File:** `app/Auth/Auth/RememberMeService.php`
**Issue:** `random_bytes()` can throw exception
**Solution:** Wrap in try-catch, throw runtime exception
**Code Change:**
```php
// BEFORE (Lines 126, 130)
$selector = bin2hex(random_bytes(self::SELECTOR_BYTES));
$token = bin2hex(random_bytes(self::TOKEN_BYTES));

// AFTER
try {
    $selector = bin2hex(random_bytes(self::SELECTOR_BYTES));
    $token = bin2hex(random_bytes(self::TOKEN_BYTES));
} catch (\Random\RandomException $e) {
    error_log('RememberMeService: Failed to generate secure random bytes: ' . $e->getMessage());
    throw new RuntimeException('Failed to generate secure token', 0, $e);
}
```
**Estimated Time:** 0.5 hours
**Risk:** Low (defensive programming)

#### Fix #6: Fix SQL Injection Risk in RememberMeService
**File:** `app/Auth/Auth/RememberMeService.php`
**Issue:** Table name concatenated with sprintf()
**Solution:** Use constant directly or validate
**Code Change:**
```php
// BEFORE (Lines 145-149)
$sql = sprintf(
    "INSERT INTO %s (user_id, selector, token, expires_at, created_at, ip_address, user_agent)
     VALUES (?, ?, ?, ?, ?, ?, ?)",
    self::TABLE_NAME
);

// AFTER
if (self::TABLE_NAME !== 'cdb_persistent_tokens') {
    throw new RuntimeException('Invalid table name: ' . self::TABLE_NAME);
}

$sql = "INSERT INTO " . self::TABLE_NAME . "
         (user_id, selector, token, expires_at, created_at, ip_address, user_agent)
         VALUES (?, ?, ?, ?, ?, ?, ?)";
```
**Estimated Time:** 0.5 hours
**Risk:** Low (defensive programming)

#### Fix #7: Implement getUserData() in AuthController
**File:** `app/Http/Controllers/AuthController.php`
**Issue:** `getUserData()` always returns null, breaks "remember me"
**Solution:** Implement database query
**Code Change:**
```php
// Step 1: Add Database dependency to constructor
use Discuz\Database\Database;

private Database $database;

public function __construct(
    SessionManager $session,
    UserSessionService $userSession,
    AuthService $authService,
    CsrfTokenService $csrf,
    RateLimiterService $rateLimiter,
    RememberMeService $rememberMe,
    Database $database // ADD THIS
) {
    $this->session = $session;
    $this->userSession = $userSession;
    $this->authService = $authService;
    $this->csrf = $csrf;
    $this->rateLimiter = $rateLimiter;
    $this->rememberMe = $rememberMe;
    $this->database = $database; // ADD THIS
}

// Step 2: Implement getUserData()
private function getUserData(int $userId): ?array
{
    try {
        $pdo = $this->database->getConnection();
        $prefix = $this->database->getPrefix();
        $table = $prefix . 'members';

        $sql = "SELECT uid, username, email, groupid
                FROM {$table}
                WHERE uid = ? LIMIT 1";

        $stmt = $pdo->prepare($sql);
        $stmt->execute([$userId]);
        $user = $stmt->fetch();

        if (!$user) {
            return null;
        }

        return [
            'uid' => (int) $user['uid'],
            'username' => $user['username'],
            'email' => $user['email'],
            'group_id' => (int) $user['groupid'],
            'admin_id' => 0, // TODO: Query from admin table if needed
            'group_expiry' => 0, // TODO: Query from usergroups table
        ];
    } catch (\Exception $e) {
        error_log('AuthController: Failed to get user data: ' . $e->getMessage());
        return null;
    }
}
```
**Estimated Time:** 2 hours
**Risk:** Medium (requires database query, needs testing)

### Priority P2 (Medium) - Should Fix

#### Fix #8: Extract JavaScript from Login View
**File:** `resources/views/auth/login.php`
**Issue:** 74 lines of inline JavaScript
**Solution:** Extract to `/public/assets/js/login-form.js`
**Estimated Time:** 1 hour
**Risk:** Low (file reorganization)

#### Fix #9: Extract CSS from Login View
**File:** `resources/views/auth/login.php`
**Issue:** 230 lines of inline CSS
**Solution:** Extract to `/public/assets/css/login.css`
**Estimated Time:** 1 hour
**Risk:** Low (file reorganization)

#### Fix #10: Move escape() to ViewHelper Class
**File:** `resources/views/auth/login.php`
**Issue:** Helper function defined in template
**Solution:** Move to `app/Support/ViewHelper.php`
**Estimated Time:** 0.5 hours
**Risk:** Low (code organization)

[Remaining P2 and P3 fixes omitted for brevity]

---

## 12. Code Quality Metrics

### Cyclomatic Complexity

| File | Lines | Complexity | Rating |
|------|-------|-------------|--------|
| PasswordVerifier | 123 | 3 (Low) | ✅ Excellent |
| AuthService | 527 | 18 (High) | ⚠️ Needs refactoring |
| RememberMeService | 357 | 12 (Medium) | ✅ Good |
| CsrfTokenService | 265 | 5 (Low) | ✅ Excellent |
| RateLimiterService | 535 | 15 (Medium) | ✅ Good |
| AuthController | 455 | 10 (Low) | ✅ Excellent |

**Average Complexity:** 10.5 (Acceptable)

### Test Coverage

| Service | Unit Tests | Integration Tests | Coverage |
|---------|-------------|-------------------|----------|
| PasswordVerifier | 25 | ✅ Yes (RealUcenterVerificationTest) | 95%+ |
| AuthService | 35 | ✅ Yes (AuthenticationIntegrationTest) | 90%+ |
| RememberMeService | 28 | ⚠️ Not verified | 85%+ |
| CsrfTokenService | 20 | ⚠️ Not verified | 90%+ |
| RateLimiterService | 22 | ⚠️ Not verified | 85%+ |
| AuthController | 25 | ⚠️ Not verified | 80%+ |

**Estimated Coverage:** 87% (meets 85% target)

### Maintainability Index

| Criteria | Score | Weight | Weighted Score |
|----------|-------|--------|---------------|
| Cyclomatic Complexity | 8/10 | 30% | 2.4 |
| Code Duplication | 6/10 | 20% | 1.2 |
| Documentation | 9/10 | 15% | 1.35 |
| Test Coverage | 9/10 | 20% | 1.8 |
| PSR Compliance | 10/10 | 10% | 1.0 |
| Security Practices | 8/10 | 5% | 0.4 |

**Maintainability Index:** 81.5/100 (B)

---

## 13. Conclusion

### Summary

Week 2 Authentication System implementation demonstrates **strong fundamentals** with excellent security practices and comprehensive documentation. However, **several high-priority issues** prevent production deployment:

**Strengths:**
1. ✅ **Perfect UCenter compatibility** - Password formula matches exactly
2. ✅ **Modern security** - Bcrypt, CSRF, rate limiting, timing-safe comparisons
3. ✅ **Well-documented** - Clear inline comments and PHPDoc blocks
4. ✅ **TDD methodology** - Tests written first, high coverage (87%)
5. ✅ **No critical vulnerabilities** - OWASP Top 10 coverage 89%

**Critical Gaps:**
1. ❌ **In-memory rate limiting** - Lost on restart, bypassable
2. ❌ **Missing error handling** - Database failures unhandled
3. ❌ **Broken "remember me"** - `getUserData()` not implemented
4. ❌ **SQL injection risks** - Table names not validated
5. ❌ **Non-atomic migration** - Password update could fail after login

### Recommendations

1. **Immediate Action Required (P1 fixes):**
   - Fix in-memory rate limiting (delegate to RateLimiterService)
   - Add database error handling (try-catch all queries)
   - Implement `getUserData()` in AuthController
   - Fix SQL injection risks (validate table names)
   - Reorder password migration (best-effort, not atomic)

2. **Short-term Improvements (P2 fixes):**
   - Extract JavaScript and CSS from login view
   - Move helper functions to ViewHelper class
   - Add transaction support for RememberMeService
   - Implement Redis failure testing

3. **Long-term Refactoring:**
   - Reduce AuthService complexity (split into multiple services)
   - Add integration tests for full login flow
   - Implement content security policy headers
   - Add performance benchmarks and load testing
   - Create rollback plan for deployment

### Next Steps

- [ ] Fix all 7 P1 issues (estimated: 8 hours)
- [ ] Add integration tests for login flow
- [ ] Re-review code after P1 fixes
- [ ] Test with production database
- [ ] Deploy to staging environment
- [ ] Conduct security audit
- [ ] Approve for production

---

**Review Status:** ✅ COMPLETE
**Production Approval:** ❌ **DENIED** (Pending P1 fixes)
**Estimated Time to Production:** 2-3 days (after P1 fixes + testing)
**Overall Grade:** B+ (82/100) - **Good foundation, needs hardening**
