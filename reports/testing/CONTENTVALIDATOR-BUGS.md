# ContentValidator Bug Report

**Date**: 2026-02-15
**File**: app/Thread/Services/ContentValidator.php (232 lines)
**Status**: 🔴 **CRITICAL BUGS FOUND**

---

## Summary

ContentValidator performs basic content validation but is missing critical integration with the permission system. It validates content but doesn't check if the user has permission to post specific types of content.

---

## Critical Bugs

### Bug 1: Missing User Group Permission Checks 🔴

**Severity**: 🔴 CRITICAL (Security issue)

**Problem**: ContentValidator doesn't check `cdb_usergroups` permissions before validating content.

**Legacy Behavior** (newthread.inc.php):
```php
// Legacy checks user group permissions first
if (!$allowpost) {
    showmessage('post_nopermission');
}

// For special thread types
if ($special == 1 && !$allowpostpoll) {
    showmessage('group_nopermission');
}
```

**Modern Implementation**:
```php
// ❌ No permission checks at all!
public function validateSubject(string $subject): void
{
    // Just validates length, doesn't check allowpost
}
```

**Impact**: Users without posting permissions can still pass validation.

---

### Bug 2: Missing Special Type Permission Checks 🔴

**Severity**: 🔴 CRITICAL (Security issue)

**Problem**: `validateSpecialType()` doesn't check if user has permission to create special thread types.

**Legacy Behavior** (newthread.inc.php):
```php
// Check special type permissions
if ($special == 1 && !$allowpostpoll) {
    showmessage('group_nopermission');
}
if ($special == 2 && !$allowpostreward) {
    showmessage('group_nopermission');
}
if ($special == 3 && !$allowpostdebate) {
    showmessage('group_nopermission');
}
```

**Modern Implementation**:
```php
public function validateSpecialType(int $special, ?array $specialData): void
{
    // ❌ Only validates data structure, doesn't check permissions!
    match ($special) {
        1 => $this->validatePollData($data),    // No allowpostpoll check
        2 => $this->validateRewardData($data),  // No allowpostreward check
        3 => $this->validateDebateData($data),  // No allowpostdebate check
        4 => $this->validateTradeData($data),
        5 => $this->validateActivityData($data),
    };
}
```

**Impact**: Users without special permissions can create poll/reward/debate threads.

---

### Bug 3: Missing disablepostctrl Check 🟡

**Severity**: 🟡 MEDIUM (Feature gap)

**Problem**: ContentValidator doesn't respect `cdb_usergroups.disablepostctrl` setting.

**Legacy Behavior** (newthread.inc.php):
```php
// Skip validation for admins
if ($disablepostctrl) {
    $subject = '';
    $message = '';
}
```

**Modern Implementation**:
```php
// ❌ Always validates, even for admins who should bypass validation
public function validateSubject(string $subject): void
{
    if (trim($subject) === '') {
        throw ThreadException::invalidSubject("Subject cannot be empty");
    }
}
```

---

### Bug 4: Incomplete BBCode Validation 🟡

**Severity**: 🟡 MEDIUM (Feature gap)

**Problem**: Only validates `[img]` and `[url]` tags, missing many other BBCode tags.

**Current Implementation**:
```php
public function validateMessageContent(string $message): void
{
    $imageCount = preg_match_all('/\[img\]|\[img=.*?\]/i', $message);
    if ($imageCount > $this->maxImages) {
        throw ThreadException::invalidMessage("Too many images");
    }

    $linkCount = preg_match_all('/\[url\]|https?:\/\//i', $message);
    if ($linkCount > $this->maxLinks) {
        throw ThreadException::invalidMessage("Too many links");
    }

    // ❌ Missing checks for:
    // - [code] tags (should have limited nesting)
    // - [quote] tags (should have limited nesting)
    // - [list] tags
    // - [table] tags
    // - [size] tags (font size limits)
    // - [color] tags
    // - [b], [i], [u] tags
    // - [align] tags
    // - [free], [flash] tags (if enabled)
}
```

**Legacy Implementation** Validates:
- `[img]` with nesting limits
- `[code]` with nesting limits
- `[quote]` with nesting limits
- `[list]` tags
- File size limits for attachments

---

### Bug 5: Missing HTML Sanitization 🟡

**Severity**: 🟡 MEDIUM (Security issue)

**Problem**: No HTML sanitization in `sanitizeMessage()` or `sanitizeSubject()`.

**Current Implementation**:
```php
public function sanitizeMessage(string $message): string
{
    $message = trim($message);
    $message = preg_replace('/\r\n/', "\n", $message);
    $message = preg_replace('/\s+$/', '', $message);

    // ❌ No HTML sanitization!
    return $message;
}
```

**Security Risk**: Users can inject malicious HTML/JavaScript.

---

### Bug 6: Hard-coded Validation Limits 🟢

**Severity**: 🟢 LOW (Enhancement)

**Problem**: Validation limits are hard-coded in constructor, not read from database.

**Legacy Behavior**:
- Read limits from `cdb_usergroups`:
  - `maxsubjectsize` - Max subject length
  - `maxmessagesize` - Max message length
  - `maxbiosize` - Max bio size

**Modern Implementation**:
```php
public function __construct(
    int $minSubjectLength = 1,
    int $maxSubjectLength = 80,        // ❌ Hard-coded
    int $minMessageLength = 1,
    int $maxMessageLength = 50000,     // ❌ Hard-coded
    int $maxImages = 20,               // ❌ Hard-coded
    int $maxLinks = 50                 // ❌ Hard-coded
) {
    // ...
}
```

**Impact**: Cannot customize limits per user group.

---

### Bug 7: Missing postcredits Check 🔴

**Severity**: 🔴 CRITICAL (Business logic)

**Problem**: ContentValidator doesn't check if user has enough credits to post.

**Legacy Behavior** (newthread.inc.php):
```php
// Check post credits
if ($forum['postcredits'] && $credits < $forum['postcredits']) {
    showmessage('postcredits_insufficient');
}

// Check reply credits
if ($forum['replycredits'] && $credits < $forum['replycredits']) {
    showmessage('replycredits_insufficient');
}
```

**Modern Implementation**:
```php
// ❌ No credit checks at all!
public function validateMessage(string $message): void
{
    $length = mb_strlen($message, 'UTF-8');
    if ($length > $this->maxMessageLength) {
        throw ThreadException::invalidMessage("Message too long");
    }
}
```

**Impact**: Users can post without having required credits.

---

### Bug 8: Missing Forum Rules Check 🟢

**Severity**: 🟢 LOW (Feature gap)

**Problem**: ContentValidator doesn't check if content violates forum rules.

**Legacy Behavior** (newthread.inc.php):
```php
// Check forum rules
if ($forum['rules']) {
    // Display rules and require acceptance
}
```

**Modern Implementation**:
```php
// ❌ No forum rules check
```

---

## Summary of Bugs

| Bug | Severity | Type | Impact |
|-----|----------|-------|---------|
| 1 | 🔴 CRITICAL | Security | No permission check for posting |
| 2 | 🔴 CRITICAL | Security | No permission check for special types |
| 3 | 🟡 MEDIUM | Feature | No disablepostctrl support |
| 4 | 🟡 MEDIUM | Feature | Incomplete BBCode validation |
| 5 | 🟡 MEDIUM | Security | No HTML sanitization |
| 6 | 🟢 LOW | Enhancement | Hard-coded validation limits |
| 7 | 🔴 CRITICAL | Business logic | No credit checking |
| 8 | 🟢 LOW | Feature | No forum rules check |

---

## Required Fixes

### Priority 1 (Must Fix Immediately)
1. ✅ Add user group permission checks (Bug 1)
2. ✅ Add special type permission checks (Bug 2)
3. ✅ Add postcredits checking (Bug 7)

### Priority 2 (Should Fix Soon)
4. ✅ Add disablepostctrl support (Bug 3)
5. ✅ Add HTML sanitization (Bug 5)
6. ✅ Complete BBCode validation (Bug 4)

### Priority 3 (Should Fix)
7. ⏳ Read validation limits from database (Bug 6)
8. ⏳ Add forum rules checking (Bug 8)

---

## Next Steps

1. ✅ Create this bug report document
2. ⏳ Fix ContentValidator with all P0 and P1 bugs
3. ⏳ Create unit tests for permission checks
4. ⏳ Verify all validation works correctly
5. ⏳ Test with real data from migrated database
