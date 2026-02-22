# PostEditService Bug Report

**Date**: 2026-02-15
**File**: app/Post/Services/PostEditService.php (285 lines)
**Status**: 🔴 **CRITICAL BUGS FOUND**

---

## Summary

PostEditService handles post editing and deletion but has **critical integration issues** with ContentValidator and ForumPermissionService. It uses outdated method signatures and lacks proper permission checking.

---

## Critical Bugs

### Bug 1: ContentValidator Method Signature Mismatch 🔴

**Severity**: 🔴 CRITICAL (Fatal error - method signature changed)

**Problem**: `validateContent()` calls ContentValidator using **old method signatures**.

**Current Implementation**:
```php
private function validateContent(PostEdit $edit): void
{
    if ($edit->getSubject() !== null) {
        $this->contentValidator->validateSubject($edit->getSubject());  // ❌ WRONG!
        //       Missing: forumId, userId, groupId parameters
    }
    $this->contentValidator->validateMessage($edit->getMessage());  // ❌ WRONG!
    //       Missing: forumId, userId, groupId, isReply parameters
}
```

**Expected (after ContentValidator fix)**:
```php
public function validateSubject(int $forumId, int $userId, int $groupId, string $subject): void
public function validateMessage(int $forumId, int $userId, int $groupId, string $message, bool $isReply = false): void
```

**Impact**: This will cause a **fatal error** when called because method signatures don't match.

---

### Bug 2: Missing ForumId Parameter 🔴

**Severity**: 🔴 CRITICAL (Logic error)

**Problem**: `updatePost()` method doesn't receive `forumId` but needs to pass it to ContentValidator.

**Current Implementation**:
```php
public function updatePost(PostEdit $edit, int $userId, int $groupId, string $userIp): Post
{
    $postId = $edit->getPostId();
    $post = $this->postRepository->findById($postId);

    // ... validation ...

    $this->validateContent($edit);  // ❌ Can't pass forumId to ContentValidator
}
```

**Problem**: Need to extract `$forumId` from `$post` to pass to ContentValidator.

---

### Bug 3: Not Using ForumPermissionService 🔴

**Severity**: 🔴 CRITICAL (Duplicated logic, missing features)

**Problem**: PostEditService implements its own permission checking instead of using ForumPermissionService.

**Current Implementation**:
```php
public function canEditPost(Post $post, int $userId, int $groupId): bool
{
    if ($post->getAuthorId() !== $userId) {
        return false;
    }

    if (in_array($groupId, $this->editUnlimitedGroups, true)) {
        return true;
    }

    $timeLimit = $this->editTimeLimits[$groupId] ?? 600;
    // ❌ Duplicates ForumPermissionService logic!
    // ❌ Doesn't check forum-level permissions
    // ❌ Doesn't check moderator status

    $elapsed = time() - $post->getDateline();
    return $elapsed <= $timeLimit;
}
```

**Legacy Behavior** (editpost.inc.php):
```php
// Check if user is moderator
if ($ismoderator) {
    $allowedit = 1;
}

// Check forum-level edit time
$edittime = $forum['edittime'] ? $forum['edittime'] : $edittime;
```

**Impact**:
- Duplicated logic
- Missing moderator checks
- Missing forum-level permissions
- Inconsistent with ForumPermissionService

---

### Bug 4: Hard-coded Edit Time Limits 🔴

**Severity**: 🔴 HIGH (Wrong data source)

**Problem**: Edit time limits are hard-coded in constructor instead of reading from database.

**Current Implementation**:
```php
public function __construct(
    // ... dependencies ...
    ?array $editTimeLimits = null  // ❌ Hard-coded!
    ?array $editUnlimitedGroups = null
) {
    // ...
    $this->editTimeLimits = $editTimeLimits ?? [
        1 => 600,   // ❌ Should read from cdb_forumfields.edittime
        2 => 600,
        3 => 3600,
        4 => 7200,
        5 => 0,
        6 => 0,
        7 => 0,
        8 => 0,
    ];
}
```

**Legacy Behavior**:
```php
// Read from cdb_forumfields
$edittime = $forum['edittime'] ? $forum['edittime'] : $edittime;
```

**Impact**: Each forum can have different edit time limits, but this uses global defaults.

---

### Bug 5: Missing Moderator Check 🔴

**Severity**: 🔴 CRITICAL (Feature gap)

**Problem**: Moderators should be able to edit any post regardless of ownership or time limits.

**Current Implementation**:
```php
public function canEditPost(Post $post, int $userId, int $groupId): bool
{
    if ($post->getAuthorId() !== $userId) {
        return false;  // ❌ Moderators should bypass this
    }
    // ...
}
```

**Legacy Behavior** (editpost.inc.php):
```php
// Moderators can edit any post
if ($ismoderator) {
    $allowedit = 1;
}
```

**Impact**: Moderators cannot edit posts they don't own.

---

### Bug 6: Not Distinguishing Post vs Reply 🟡

**Severity**: 🟡 MEDIUM (Missing feature)

**Problem**: `validateMessage()` call doesn't specify `$isReply` parameter.

**Current Implementation**:
```php
private function validateContent(PostEdit $edit): void
{
    // ...
    $this->contentValidator->validateMessage($edit->getMessage());
    // ❌ Missing: isReply parameter
}
```

**ContentValidator Signature**:
```php
public function validateMessage(
    int $forumId,
    int $userId,
    int $groupId,
    string $message,
    bool $isReply = false  // ← This parameter!
): void
```

**Impact**: Reply credit checking won't work properly.

---

### Bug 7: validateContent() Lacks Forum Context 🟡

**Severity**: 🟡 MEDIUM (Incomplete validation)

**Problem**: `validateContent()` doesn't have access to forum information needed for ContentValidator.

**Current Method**:
```php
private function validateContent(PostEdit $edit): void
{
    if ($edit->getSubject() !== null) {
        $this->contentValidator->validateSubject($edit->getSubject());
    }
    $this->contentValidator->validateMessage($edit->getMessage());
}
```

**Needed Parameters**:
```php
private function validateContent(PostEdit $edit, int $forumId, int $userId, int $groupId, bool $isReply): void
{
    if ($edit->getSubject() !== null) {
        $this->contentValidator->validateSubject($forumId, $userId, $groupId, $edit->getSubject());
    }
    $this->contentValidator->validateMessage($forumId, $userId, $groupId, $edit->getMessage(), $isReply);
}
```

---

### Bug 8: getEditTimeLimit() Implementation Duplication 🟢

**Severity**: 🟢 LOW (Code duplication)

**Problem**: This method duplicates `ForumPermissionService::getEditTimeLimit()`.

**Current Implementation**:
```php
public function getEditTimeLimit(int $groupId): int
{
    return $this->editTimeLimits[$groupId] ?? 600;  // ❌ Uses hard-coded array
}
```

**Should Be**:
```php
public function getEditTimeLimit(int $groupId, int $forumId): int
{
    return $this->permissionService->getEditTimeLimit($groupId, $forumId);
}
```

---

## Summary of Bugs

| Bug | Severity | Type | Impact |
|-----|----------|-------|---------|
| 1 | 🔴 CRITICAL | Fatal error | Method signature mismatch |
| 2 | 🔴 CRITICAL | Logic error | Missing forumId parameter |
| 3 | 🔴 CRITICAL | Architecture | Not using ForumPermissionService |
| 4 | 🔴 HIGH | Wrong data source | Hard-coded time limits |
| 5 | 🔴 CRITICAL | Feature gap | Missing moderator check |
| 6 | 🟡 MEDIUM | Missing feature | Not distinguishing post/reply |
| 7 | 🟡 MEDIUM | Incomplete validation | Lacks forum context |
| 8 | 🟢 LOW | Code duplication | Duplicated getEditTimeLimit() |

---

## Required Fixes

### Priority 1 (Must Fix Immediately)
1. ✅ Fix ContentValidator method signature calls (Bug 1)
2. ✅ Add forumId extraction and passing (Bug 2)
3. ✅ Use ForumPermissionService for permission checks (Bug 3)
4. ✅ Add moderator permission check (Bug 5)

### Priority 2 (Should Fix Soon)
5. ✅ Remove hard-coded edit time limits (Bug 4)
6. ✅ Add isReply parameter to validateContent() (Bug 6)

### Priority 3 (Should Fix)
7. ✅ Rewrite validateContent() with full context (Bug 7)
8. ⏳ Remove getEditTimeLimit() duplication (Bug 8) - Optional

---

## Integration Requirements

### New Dependencies

PostEditService needs to depend on:
- `ForumPermissionService` - For permission checks and edit time limits

### Constructor Changes

```php
// Current
public function __construct(
    PostRepository $postRepository,
    ThreadRepository $threadRepository,
    ContentValidator $contentValidator,
    Connection $db,
    Cache $cache,
    ?array $editTimeLimits = null,
    ?array $editUnlimitedGroups = null
) { }

// Should Be
public function __construct(
    private PostRepository $postRepository,
    private ThreadRepository $threadRepository,
    private ContentValidator $contentValidator,
    private ForumPermissionService $permissionService,  // ← NEW
    private Connection $db,
    private Cache $cache
) {
    // Remove hard-coded editTimeLimits and editUnlimitedGroups
}
```

---

## Next Steps

1. ✅ Create this bug report document
2. ⏳ Fix all P0 and P1 bugs in PostEditService
3. ⏳ Create unit tests for permission checks
4. ⏳ Verify all validation works correctly
5. ⏳ Test with real data from migrated database

---

## Time Estimate

- Bug analysis: 30 minutes (completed)
- Fix implementation: ~2 hours
- Testing: ~30 minutes
- **Total**: ~3 hours
