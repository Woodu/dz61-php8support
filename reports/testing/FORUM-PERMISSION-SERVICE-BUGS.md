# ForumPermissionService.php - Critical Bugs Found

**Date**: 2026-02-15
**Status**: 🔴 CRITICAL BUGS FOUND
**File**: app/Forum/Services/ForumPermissionService.php (611 lines)

---

## Bug 1: Syntax Error - Missing Asterisk (Line 179)

**Location**: Line 179
**Severity**: 🔴 CRITICAL (Parse Error)

```php
// ❌ BROKEN - Line 178-180
/**
 * Parse forum permission field (comma-separated group IDs)
 * @param string $permValue Permission field value
 *return bool True if user has permission  // ← MISSING @ before "return"
```

**Fix Required**:
```php
// ✅ FIXED
/**
 * Parse forum permission field (comma-separated group IDs)
 * @param string $permValue Permission field value
 * @return bool True if user has permission  // ← Added @
```

---

## Bug 2: Type Checking Bug in checkForumPermission (Line 192)

**Location**: Line 192
**Severity**: 🔴 CRITICAL (Permission Check Always Fails)

```php
// ❌ BROKEN - Line 191-193
$allowedGroups = array_map('intval', explode(',', $permValue));
return in_array((string)$groupidArray, $allowedGroups, true);
//                                         ^^^^^^^^^^^^^
//                                         This is an ARRAY, not a string!
```

**Problem**:
- `$groupidArray` is an array parameter (declared on line 182)
- Casting an array to string returns `"Array"` (literal string)
- in_array() will compare `"Array"` string against integers
- **Result**: Permission check ALWAYS returns false!

**Correct Logic**:
- Should check if ANY of user's group IDs match allowed groups
- User has multiple groups: primary group + extended groups

**Fix Required**:
```php
// ✅ FIXED
private function checkForumPermission(string $permValue, array $groupidArray): bool
{
    // Empty permission means all groups allowed
    if ($permValue === '' || $permValue === null) {
        return true;
    }

    // Parse comma-separated group IDs
    $allowedGroups = array_map('intval', explode(',', $permValue));

    // Check if ANY of user's groups is in allowed groups
    foreach ($groupidArray as $gid) {
        if (in_array((int)$gid, $allowedGroups, true)) {
            return true;
        }
    }

    return false;
}
```

---

## Bug 3: canEditPost References Undefined Property (Line 484)

**Location**: Line 484
**Severity**: 🔴 CRITICAL (Fatal Error)

```php
// ❌ BROKEN - Line 482-490
public function canEditPost(int $forumId, int $userId, int $groupId): bool
{
    // ... permission checks ...
    $editTimeLimit = $this->getEditTimeLimit($groupId, $forumId);
    $post = $this->postRepository->findById($postId);
    //       ^^^^^^^^^^^^^^^
    //       This property DOES NOT EXIST in ForumPermissionService!
```

**Problem**:
1. `$this->postRepository` - Property not defined in __construct
2. `$postId` - Parameter not passed to method
3. This method signature doesn't match what's needed

**Fix Required**:
This method should NOT be in ForumPermissionService at all! It belongs in PostEditService.
The correct implementation for ForumPermissionService should only check permissions, not retrieve posts.

```php
// ✅ FIXED - ForumPermissionService version
public function canEditPost(int $forumId, int $userId, int $groupId): bool
{
    // Get user group permissions
    $groupPerms = $this->getUserGroupPermissions($groupId);

    // Check user group permission (cdb_usergroups.alloweditpost)
    if (empty($groupPerms) || !$groupPerms['alloweditpost']) {
        return false;
    }

    // Check if is moderator (moderators can always edit)
    if ($this->isModerator($forumId, $userId)) {
        return true;
    }

    // 1. Check user-specific permission (highest priority)
    $userAccess = $this->getUserAccess($forumId, $userId);
    if ($userAccess !== null) {
        return (bool) $userAccess['allowpost'];
    }

    // 2. Check forum-level permission
    $forum = $this->getForumFields($forumId);
    return $this->checkForumPermission($forum['postperm'] ?? '', [$groupId]);
}
```

**Note**: The actual post retrieval and time limit checking should be in PostEditService, not here.

---

## Bug 4: getEditTimeLimit Uses Undefined $forumId (Line 421)

**Location**: Line 421
**Severity**: 🔴 HIGH (Wrong Forum Data)

```php
// ❌ BROKEN - Line 418-427
private function getEditTimeLimit(int $groupId): int
{
    // Try to get from forum settings (if user is editing in a specific forum)
    $forumId = 0; // ← This should be a parameter!
    $sql = "
        SELECT ff.edittime
        FROM cdb_forumfields ff
        WHERE ff.fid = :fid
        LIMIT 1
    ";
    // Always queries fid=0, which probably doesn't exist!
```

**Problem**:
- Method only takes $groupId but needs $forumId
- Hardcodes $forumId = 0
- Will always return wrong data (or null) for actual forums

**Fix Required**:
```php
// ✅ FIXED
private function getEditTimeLimit(int $groupId, int $forumId): int
{
    $sql = "
        SELECT ff.edittime
        FROM cdb_forumfields ff
        WHERE ff.fid = :fid
        LIMIT 1
    ";

    try {
        $row = $this->db->selectOne($sql, ['fid' => $forumId]);

        if ($row !== null && !empty($row['edittime'])) {
            return (int) $row['edittime'];
        }
    } catch (\Exception $e) {
        error_log('ForumPermissionService: Failed to get edit time: ' . $e->getMessage());
    }

    // Default values (same as original hardcoded)
    return match ($groupId) {
        1 => 600,   // Admin
        2 => 600,   // Admin
        3 => 3600,  // Moderator
        4 => 7200,  // Super Moderator
        5 => 0,      // Regular User (no limit)
        6 => 0,      // Regular User (no limit)
        7 => 0,      // Regular User (no limit)
        8 => 0,      // Regular User (no limit)
    };
}
```

---

## Bug 5: canGetAttachment References Wrong Array Key (Line 530)

**Location**: Line 530
**Severity**: 🟡 MEDIUM (Fatal Error)

```php
// ❌ BROKEN - Line 527-531
$userAccess = $this->getUserAccess($forumId, $userId);
if ($userAccess !== null) {
    return (bool) $userAccess['allowgetattach'];
    //       ^^^^^^^^^^^^^^^^^^^
    //       cdb_access table does NOT have allowgetattach field!
}
```

**Problem**:
- `cdb_access` table only has: allowview, allowpost, allowreply
- `allowgetattach` doesn't exist in that table
- Will throw undefined array key notice

**Fix Required**:
```php
// ✅ FIXED
public function canGetAttachment(int $forumId, int $userId, int $groupId): bool
{
    // Get user group permissions
    $groupPerms = $this->getUserGroupPermissions($groupId);

    // Check user group permission
    if (empty($groupPerms) || !$groupPerms['allowgetattach']) {
        return false;
    }

    // cdb_access doesn't have allowgetattach, skip user-specific check
    // 2. Check forum-level permission
    $forum = $this->getForumFields($forumId);
    return $this->checkForumPermission($forum['getattachperm'] ?? '', [$groupId]);
}
```

---

## Bug 6: canPostAttachment References Wrong Array Key (Line 560)

**Location**: Line 560
**Severity**: 🟡 MEDIUM (Fatal Error)

```php
// ❌ BROKEN - Line 557-561
$userAccess = $this->getUserAccess($forumId, $userId);
if ($userAccess !== null) {
    return (bool) $userAccess['allowpostattach'];
    //       ^^^^^^^^^^^^^^^^^^^^
    //       cdb_access table does NOT have allowpostattach field!
}
```

**Fix Required**: Same as Bug 5, skip user-specific check for postattach

---

## Bug 7: canDeletePost Incorrectly Calls canEditPost (Line 505)

**Location**: Line 505
**Severity**: 🟡 MEDIUM (Logic Error)

```php
// ❌ BROKEN - Line 502-506
public function canDeletePost(int $forumId, int $userId, int $groupId): bool
{
    // Same logic as edit post for now
    return $this->canEditPost($forumId, $userId, $groupId);
    //   ^^^^^^^^^^^^^^^^
    //   This method signature doesn't match! Missing $postId parameter!
}
```

**Problem**:
- canEditPost() needs $postId for time limit checking
- But canDeletePost() doesn't pass it
- Method will crash with missing parameter

**Fix Required**:
```php
// ✅ FIXED
public function canDeletePost(int $forumId, int $userId, int $groupId): bool
{
    // Get user group permissions
    $groupPerms = $this->getUserGroupPermissions($groupId);

    // Check user group permission (cdb_usergroups.alloweditpost)
    if (empty($groupPerms) || !$groupPerms['alloweditpost']) {
        return false;
    }

    // 1. Check user-specific permission (highest priority)
    $userAccess = $this->getUserAccess($forumId, $userId);
    if ($userAccess !== null) {
        return (bool) $userAccess['allowpost'];
    }

    // 2. Check if is moderator
    if ($this->isModerator($forumId, $userId)) {
        return true;
    }

    // 3. Check forum-level permission
    $forum = $this->getForumFields($forumId);
    return $this->checkForumPermission($forum['postperm'] ?? '', [$groupId]);
}
```

---

## Bug 8: Typo in PHPDoc Comment (Line 515, 545, 573, 587)

**Location**: Multiple lines
**Severity**: 🟢 LOW (Documentation)

```php
// ❌ Line 515, 545, 573, 587
* @throws ForumException OnDatabaseError  // ← Two words
* @throws ForumException On databaseError  // ← Mixed case

// ✅ Should be consistent:
* @throws ForumException On database error
```

---

## Summary of Bugs

| Bug | Severity | Type | Impact |
|-----|----------|-------|---------|
| 1 | 🔴 CRITICAL | Syntax Error | Parse error, file won't load |
| 2 | 🔴 CRITICAL | Logic Error | Permission checks always fail |
| 3 | 🔴 CRITICAL | Fatal Error | Undefined property |
| 4 | 🔴 HIGH | Wrong Data | Always queries forum ID 0 |
| 5 | 🟡 MEDIUM | Fatal Error | Undefined array key |
| 6 | 🟡 MEDIUM | Fatal Error | Undefined array key |
| 7 | 🟡 MEDIUM | Logic Error | Wrong method signature |
| 8 | 🟢 LOW | Documentation | Inconsistent comments |

---

## Required Fixes

**Priority 1 (Must Fix Immediately)**:
1. ✅ Fix Bug 1: Add missing `@` symbol on line 179
2. ✅ Fix Bug 2: Rewrite checkForumPermission() logic
3. ✅ Fix Bug 3: Fix canEditPost() method
4. ✅ Fix Bug 4: Add $forumId parameter to getEditTimeLimit()

**Priority 2 (Must Fix Soon)**:
5. ✅ Fix Bug 5: Remove invalid allowgetattach check
6. ✅ Fix Bug 6: Remove invalid allowpostattach check
7. ✅ Fix Bug 7: Implement proper canDeletePost() logic

**Priority 3 (Should Fix)**:
8. ✅ Fix Bug 8: Standardize PHPDoc comments

---

## Next Steps

1. ✅ Create this bug report document
2. ⏳ Apply all 8 fixes to ForumPermissionService.php
3. ⏳ Create unit tests for permission checking
4. ⏳ Verify all permission checks work correctly
5. ⏳ Test with real data from migrated database
