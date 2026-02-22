# Week 4 Test Gap Analysis

**Generated**: 2026-02-16
**Project**: Discuz! 6.1F → PHP 8.3 Migration
**Scope**: Week 4 Core Services Test Coverage Analysis

---

## Executive Summary

### Current Test Coverage Status

| Service | Lines of Code | Existing Tests | Coverage % | Status |
|---------|---------------|----------------|------------|--------|
| ForumPermissionService | 702 | 368 lines (~22 tests) | ~50% | PARTIAL |
| ContentValidator | 553 | 0 lines | 0% | MISSING |
| PostEditService | 486 | 0 lines | 0% | MISSING |
| ThreadReadStatusService | 486 | 0 lines | 0% | MISSING |
| SessionService | 519 | 0 lines | 0% | MISSING |
| **TOTAL** | **2,746** | **368 lines** | **~13%** | **CRITICAL** |

### Critical Findings

1. **ForumPermissionService** has existing tests but lacks coverage for:
   - Moderator permission checks
   - Extended group IDs (extgroupids)
   - User group permission validation
   - Credit-based permission checks
   - Edit/delete time limits

2. **4 out of 5 services have ZERO test coverage** - this is a critical gap

3. **High-risk services** (ContentValidator, PostEditService) are completely untested

4. **Complex business logic** (permissions, validation, edit restrictions) has no test protection

---

## Detailed Service Analysis

### 1. ForumPermissionService (702 lines)

**Location**: `/app/Forum/Services/ForumPermissionService.php`

**Current Test Status**: PARTIAL (22 existing tests)
**Test File**: `/tests/Unit/Forum/Services/ForumPermissionServiceTest.php` (368 lines)

#### Public Methods (16 total)

##### ✅ Already Tested (8 methods)
1. `canViewForum()` - Basic user-specific and forum-level permissions
2. `canPostThread()` - Basic post permissions
3. `canReplyThread()` - Basic reply permissions
4. `canGetAttachment()` - Attachment download permissions
5. `canPostAttachment()` - Attachment upload permissions
6. `clearCache()` - Cache clearing
7. `forumHasPassword()` - Password protection check
8. `verifyForumPassword()` - Password verification

##### ❌ Missing Tests (8 methods)
9. `canEditPost()` - **HIGH PRIORITY**
   - Moderator bypass
   - Time limit enforcement
   - User group permissions
   - User-specific access override

10. `canDeletePost()` - **HIGH PRIORITY**
    - Moderator bypass
    - Ownership validation
    - Forum-level restrictions

11. `isModerator()` - **HIGH PRIORITY**
    - Database query to cdb_moderators
    - Cache behavior
    - Edge cases (multiple forums, user not found)

12. `getUserGroupPermissionsDirect()` - **MEDIUM PRIORITY**
    - All 13 permission fields
    - Missing group IDs (returns empty array)
    - Caching behavior

13. `getForumFieldsDirect()` - **MEDIUM PRIORITY**
    - All forum field values
    - Empty forum handling
    - Caching behavior

14. `getUserAccess()` - **MEDIUM PRIORITY** (private method, tested indirectly)
    - User-specific access from cdb_access
    - Null handling (no user access record)
    - Caching

15. `checkForumPermission()` - **LOW PRIORITY** (private method)
    - Empty permission string (allows all)
    - Comma-separated group ID parsing
    - Tab-separated extgroupids parsing

16. `forumperm()` - **LOW PRIORITY** (private method)
    - Legacy regex pattern matching
    - Tab-separated group ID parsing
    - Extended group IDs support

#### Missing Test Scenarios

**Moderator Tests** (HIGH PRIORITY):
- Moderator can edit any post
- Moderator can delete any post
- Moderator bypasses credit checks
- Moderator bypasses time limits
- Non-moderator respects all restrictions

**Extended Group IDs** (HIGH PRIORITY):
- User with multiple extended groups
- Tab-separated parsing (e.g., "4\t5\t6")
- Empty extgroupids string
- Invalid extgroupids format
- Extended group permissions override

**Permission Priority** (HIGH PRIORITY):
- User-specific > Forum-level > User group level
- Null user access falls back to forum-level
- Empty forum permission allows all groups
- Deny at any level denies access

**Credit-Based Permissions** (MEDIUM PRIORITY):
- Post credits required
- Reply credits required
- Insufficient credits denied
- Credit checking for special thread types

**Edge Cases** (LOW PRIORITY):
- Database errors throw ForumException
- Missing user group (returns empty array)
- Missing forum fields (returns empty array)
- Cache hit/miss behavior

#### Estimated Additional Tests
- **New test methods**: 25-30
- **Estimated lines**: 600-800
- **Priority**: HIGH (security-critical service)

---

### 2. ContentValidator (553 lines)

**Location**: `/app/Thread/Services/ContentValidator.php`

**Current Test Status**: NO TESTS
**Test File**: `/tests/Unit/Thread/Services/ContentValidatorTest.php` (TO BE CREATED)

#### Public Methods (8 total)

##### 1. `validateSubject()` - **CRITICAL**

**Dependencies**:
- `ForumPermissionService::canPostThread()`
- `ForumPermissionService::getUserGroupPermissionsDirect()`
- `CreditService::getUserCredits()` (via validatePostCredits)

**Test Cases**:
- ✅ Valid subject passes
- ✅ Subject too short (< minSubjectLength)
- ✅ Subject too long (> maxSubjectLength)
- ✅ Empty subject (after trim)
- ✅ Permission denied (canPostThread = false)
- ✅ Bypass for admins (disablepostctrl = 1)
- ✅ Bypass for admins (disableperiodctrl = 1)
- ✅ Insufficient credits for posting

**Estimated Tests**: 15-20 test methods

##### 2. `validateMessage()` - **CRITICAL**

**Dependencies**:
- `ForumPermissionService::canReplyThread()` or `canPostThread()`
- `ForumPermissionService::getUserGroupPermissionsDirect()`
- `CreditService::getUserCredits()`

**Test Cases**:
- ✅ Valid message passes
- ✅ Message too short (< minMessageLength)
- ✅ Message too long (> maxMessageLength)
- ✅ Empty message (after trim)
- ✅ Permission denied for reply
- ✅ Permission denied for new thread
- ✅ Admin bypass (disablepostctrl)
- ✅ Insufficient credits for reply
- ✅ BBCode validation called
- ✅ HTML sanitization called

**Estimated Tests**: 20-25 test methods

##### 3. `validateMessageContent()` - **HIGH PRIORITY**

**Test Cases**:
- ✅ Too many images (> maxImages)
- ✅ Too many links (> maxLinks)
- ✅ Code nesting too deep (> maxCodeNesting)
- ✅ Quote nesting too deep (> maxQuoteNesting)
- ✅ Mismatched code tags
- ✅ Mismatched quote tags
- ✅ Mismatched list tags
- ✅ Valid content passes

**Estimated Tests**: 15-20 test methods

##### 4. `validateHtmlContent()` - **HIGH PRIORITY** (Security)

**Test Cases**:
- ✅ `<script>` tag detection
- ✅ `javascript:` protocol detection
- ✅ Event handler detection (onclick, onload, etc.)
- ✅ `<iframe>` tag detection
- ✅ `<object>` tag detection
- ✅ `<embed>` tag detection
- ✅ PHP tag detection (`<?php`)
- ✅ Valid HTML passes

**Estimated Tests**: 10-15 test methods

##### 5. `validateSpecialType()` - **HIGH PRIORITY**

**Dependencies**:
- `ForumPermissionService::getUserGroupPermissionsDirect()`
- `CreditService::getUserCredits()` (for reward threads)

**Test Cases**:
- ✅ Special type 0 (normal thread) passes
- ✅ Invalid special type (< 1 or > 5)
- ✅ Special type without data throws
- ✅ Poll validation (type 1)
- ✅ Reward validation (type 2)
- ✅ Debate validation (type 3)
- ✅ Trade validation (type 4)
- ✅ Activity validation (type 5)
- ✅ Permission denied for each special type

**Estimated Tests**: 20-25 test methods

##### 6. `sanitizeMessage()` - **MEDIUM PRIORITY**

**Test Cases**:
- ✅ Trims whitespace
- ✅ Converts line endings
- ✅ Strips dangerous HTML tags
- ✅ Converts HTML entities
- ✅ Preserves safe HTML tags (b, i, u, strong, em, a, br, p, div, etc.)

**Estimated Tests**: 8-10 test methods

##### 7. `sanitizeSubject()` - **LOW PRIORITY**

**Test Cases**:
- ✅ Trims whitespace
- ✅ Normalizes multiple spaces
- ✅ Strips HTML tags
- ✅ Converts HTML entities

**Estimated Tests**: 5-8 test methods

##### 8. `hasMismatchedTags()` - **MEDIUM PRIORITY** (private, tested via validateMessageContent)

**Test Cases**:
- ✅ Matching code tags
- ✅ Mismatched code tags
- ✅ Matching quote tags
- ✅ Mismatched quote tags
- ✅ All tag types checked

**Estimated Tests**: 10-12 test methods

#### Estimated Total Tests
- **Total test methods**: 100-120
- **Estimated lines**: 1,500-2,000
- **Priority**: CRITICAL (security and validation)

---

### 3. PostEditService (486 lines)

**Location**: `/app/Post/Services/PostEditService.php`

**Current Test Status**: NO TESTS
**Test File**: `/tests/Unit/Post/Services/PostEditServiceTest.php` (TO BE CREATED)

#### Public Methods (9 total)

##### 1. `updatePost()` - **CRITICAL**

**Dependencies**:
- `PostRepository::findById()`
- `ContentValidator::validateSubject()`
- `ContentValidator::validateMessage()`
- `ForumPermissionService::canEditPost()`
- `Connection` (transaction management)
- `Cache` (cache invalidation)

**Test Cases**:
- ✅ Successful update (message only)
- ✅ Successful update (message + subject)
- ✅ First post updates thread subject
- ✅ Reply post doesn't update thread subject
- ✅ Post not found throws exception
- ✅ Permission denied throws exception
- ✅ Validation error throws exception
- ✅ Database error rolls back transaction
- ✅ Cache cleared on success
- ✅ Returns updated post entity

**Estimated Tests**: 15-20 test methods

##### 2. `deletePost()` - **CRITICAL**

**Dependencies**:
- `PostRepository::findById()`
- `ForumPermissionService::canDeletePost()`
- `PostRepository::softDelete()`
- `Connection` (transaction management)
- `Cache` (cache invalidation)

**Test Cases**:
- ✅ Successful deletion
- ✅ Post not found throws exception
- ✅ Permission denied throws exception
- ✅ Cannot delete first post (throws exception)
- ✅ Database error rolls back transaction
- ✅ Thread reply count decremented
- ✅ Cache cleared (thread, forum, post)
- ✅ Returns true on success

**Estimated Tests**: 12-15 test methods

##### 3. `canEditPost()` - **HIGH PRIORITY**

**Dependencies**:
- `PostRepository::findById()`
- `ForumPermissionService::canEditPost()`

**Test Cases**:
- ✅ User can edit own post
- ✅ User cannot edit others' posts
- ✅ Moderator can edit any post
- ✅ Post not found returns false
- ✅ Permission granted
- ✅ Permission denied

**Estimated Tests**: 8-10 test methods

##### 4. `canDeletePost()` - **HIGH PRIORITY**

**Dependencies**:
- `PostRepository::findById()`
- `ForumPermissionService::canDeletePost()`

**Test Cases**:
- ✅ User can delete own reply
- ✅ User cannot delete first post
- ✅ User cannot delete others' posts
- ✅ Moderator can delete any post
- ✅ Post not found returns false
- ✅ Permission granted
- ✅ Permission denied

**Estimated Tests**: 8-10 test methods

##### 5. `getEditTimeLimit()` - **MEDIUM PRIORITY**

**Dependencies**:
- `ForumPermissionService::getEditTimeLimit()`

**Test Cases**:
- ✅ Group 1 gets 600 seconds
- ✅ Group 2 gets 600 seconds
- ✅ Group 3 gets 3600 seconds
- ✅ Group 4 gets 7200 seconds
- ✅ Group 5+ gets 0 (unlimited)
- ✅ Unknown group gets 0

**Estimated Tests**: 6-8 test methods

##### 6. `getRemainingEditTime()` - **MEDIUM PRIORITY**

**Dependencies**:
- `PostRepository::findById()`
- `getEditTimeLimit()`

**Test Cases**:
- ✅ Unlimited edit time returns -1
- ✅ Time remaining positive
- ✅ Time expired returns 0
- ✅ Post not found throws exception
- ✅ Calculates correctly based on post dateline

**Estimated Tests**: 6-8 test methods

##### 7. `validateEditPermission()` - **MEDIUM PRIORITY** (private, tested via updatePost)

**Test Cases**:
- ✅ Moderator bypasses all checks
- ✅ Owner with permission passes
- ✅ Non-owner fails
- ✅ Permission denied fails
- ✅ Edit time expired throws exception

**Estimated Tests**: 8-10 test methods

##### 8. `validateDeletePermission()` - **MEDIUM PRIORITY** (private, tested via deletePost)

**Test Cases**:
- ✅ Moderator bypasses all checks
- ✅ Owner with permission passes
- ✅ Non-owner fails
- ✅ Permission denied fails
- ✅ Edit time expired throws exception

**Estimated Tests**: 8-10 test methods

##### 9. `validateContent()` - **MEDIUM PRIORITY** (private, tested via updatePost)

**Dependencies**:
- `ContentValidator::validateSubject()`
- `ContentValidator::validateMessage()`

**Test Cases**:
- ✅ Validates subject when provided
- ✅ Validates message
- ✅ Passes isReply flag for replies
- ✅ Converts ThreadException to PostEditException

**Estimated Tests**: 6-8 test methods

#### Private Helper Methods (Test Indirectly)

- `updateThreadSubject()` - Tested via updatePost()
- `deleteThread()` - Not currently used
- `decrementThreadReplyCount()` - Tested via deletePost()

#### Estimated Total Tests
- **Total test methods**: 80-100
- **Estimated lines**: 1,200-1,600
- **Priority**: CRITICAL (data integrity)

---

### 4. ThreadReadStatusService (486 lines)

**Location**: `/app/Thread/Services/ThreadReadStatusService.php`

**Current Test Status**: NO TESTS
**Test File**: `/tests/Unit/Thread/Services/ThreadReadStatusServiceTest.php` (TO BE CREATED)

#### Public Methods (13 total)

##### 1. `markThreadAsRead()` - **HIGH PRIORITY**

**Dependencies**:
- `Redis::zadd()` (sorted set)
- `Redis::expire()` (TTL)
- `Redis::zremrangebyrank()` (trim)

**Test Cases**:
- ✅ Marks single thread as read
- ✅ Uses current timestamp by default
- ✅ Uses custom timestamp when provided
- ✅ Sets TTL (30 days)
- ✅ Trims to MAX_READ_THREADS (10000)
- ✅ Redis error returns false
- ✅ Success returns true

**Estimated Tests**: 8-10 test methods

##### 2. `markThreadsAsRead()` - **HIGH PRIORITY**

**Dependencies**:
- `Redis::zadd()` (bulk)

**Test Cases**:
- ✅ Marks multiple threads as read
- ✅ Thread IDs with timestamps array format
- ✅ Sets TTL
- ✅ Trims to max threads
- ✅ Redis error returns false
- ✅ Empty array returns true

**Estimated Tests**: 6-8 test methods

##### 3. `isThreadRead()` - **HIGH PRIORITY**

**Dependencies**:
- `Redis::zscore()` (check membership)

**Test Cases**:
- ✅ Returns true for read thread
- ✅ Returns false for unread thread
- ✅ Returns false on Redis error
- ✅ Handles null score
- ✅ Handles false score

**Estimated Tests**: 6-8 test methods

##### 4. `getThreadReadTimestamp()` - **MEDIUM PRIORITY**

**Dependencies**:
- `Redis::zscore()`

**Test Cases**:
- ✅ Returns timestamp for read thread
- ✅ Returns null for unread thread
- ✅ Returns null on Redis error
- ✅ Converts score to integer

**Estimated Tests**: 5-6 test methods

##### 5. `markForumAsRead()` - **MEDIUM PRIORITY**

**Dependencies**:
- `Redis::set()` (simple key)

**Test Cases**:
- ✅ Marks forum as read
- ✅ Uses current timestamp by default
- ✅ Uses custom timestamp when provided
- ✅ Sets TTL
- ✅ Redis error returns false
- ✅ Success returns true

**Estimated Tests**: 6-8 test methods

##### 6. `getForumReadTimestamp()` - **MEDIUM PRIORITY**

**Dependencies**:
- `Redis::get()`

**Test Cases**:
- ✅ Returns timestamp for read forum
- ✅ Returns 0 for unread forum
- ✅ Returns 0 on Redis error
- ✅ Converts value to integer

**Estimated Tests**: 5-6 test methods

##### 7. `getUnreadThreadCount()` - **LOW PRIORITY** (incomplete implementation)

**Dependencies**:
- `Redis::zcount()`
- Database (not yet implemented)

**Test Cases**:
- ✅ Returns 0 when forum never read
- ✅ Returns 0 on Redis error
- ✅ Future: Returns actual count when DB integrated

**Estimated Tests**: 3-5 test methods

##### 8. `isThreadUnread()` - **HIGH PRIORITY**

**Dependencies**:
- `getThreadReadTimestamp()`

**Test Cases**:
- ✅ Returns true for never-read thread
- ✅ Returns true for thread with new posts
- ✅ Returns false for up-to-date thread
- ✅ Compares timestamps correctly

**Estimated Tests**: 5-6 test methods

##### 9. `getReadThreadIds()` - **MEDIUM PRIORITY**

**Dependencies**:
- `Redis::zrevrange()`

**Test Cases**:
- ✅ Returns array of thread IDs
- ✅ Respects limit parameter
- ✅ Returns most recent threads first
- ✅ Returns empty array on error
- ✅ Converts all values to integers

**Estimated Tests**: 6-8 test methods

##### 10. `clearThreadReadStatus()` - **MEDIUM PRIORITY**

**Dependencies**:
- `Redis::zrem()`

**Test Cases**:
- ✅ Removes thread from read set
- ✅ Returns true on success
- ✅ Returns false on Redis error
- ✅ Idempotent (can clear non-existent thread)

**Estimated Tests**: 5-6 test methods

##### 11. `clearAllReadStatus()` - **MEDIUM PRIORITY**

**Dependencies**:
- `Redis::del()`

**Test Cases**:
- ✅ Removes entire read set
- ✅ Returns true on success
- ✅ Returns false on Redis error
- ✅ Idempotent (can clear empty set)

**Estimated Tests**: 5-6 test methods

##### 12. `cleanOldReadRecords()` - **LOW PRIORITY** (stub implementation)

**Test Cases**:
- ✅ Returns 0 (stub)
- ✅ Future: Returns count of removed records

**Estimated Tests**: 2-3 test methods

##### 13. `importFromLegacyCookie()` - **MEDIUM PRIORITY**

**Dependencies**:
- `markThreadAsRead()` (recursive)

**Test Cases**:
- ✅ Parses legacy format "D123D456D789D"
- ✅ Handles empty cookie
- ✅ Handles cookie with only "D"
- ✅ Imports multiple thread IDs
- ✅ Returns true on success
- ✅ Returns false on error
- ✅ Uses current timestamp for all threads

**Estimated Tests**: 8-10 test methods

##### 14. `exportToLegacyCookie()` - **MEDIUM PRIORITY**

**Dependencies**:
- `getReadThreadIds()`

**Test Cases**:
- ✅ Exports to "D123D456D" format
- ✅ Respects max size (3072 bytes)
- ✅ Returns "D" for no threads
- ✅ Returns "D" on error
- ✅ Handles partial export (truncates at limit)

**Estimated Tests**: 6-8 test methods

##### 15. `getReadStatusStatistics()` - **LOW PRIORITY**

**Dependencies**:
- `Redis::zcard()`

**Test Cases**:
- ✅ Returns total read count
- ✅ Returns user ID in stats
- ✅ Returns 0 on Redis error
- ✅ Returns correct structure

**Estimated Tests**: 4-5 test methods

#### Estimated Total Tests
- **Total test methods**: 80-100
- **Estimated lines**: 1,200-1,500
- **Priority**: MEDIUM (feature completeness)

---

### 5. SessionService (519 lines)

**Location**: `/app/Session/Services/SessionService.php`

**Current Test Status**: NO TESTS
**Test File**: `/tests/Unit/Session/Services/SessionServiceTest.php` (TO BE CREATED)

**Note**: UserSessionService already has tests. SessionService is a wrapper.

#### Public Methods (40 total)

##### User Data Methods (Delegated to UserSessionService)

Most methods delegate to `UserSessionService`, which is already tested:
- `setUserId()`, `getUserId()`, `isLoggedIn()`
- `setUsername()`, `getUsername()`
- `setGroupId()`, `getGroupId()`
- `setGroupExpiry()`, `getGroupExpiry()`, `isGroupExpired()`
- `setAdminId()`, `getAdminId()`, `isAdmin()`
- `setPermissions()`, `getPermissions()`, `hasPermission()`
- `setUserData()`, `getUserData()`, `clearUserData()`
- `setLastActivity()`, `getLastActivity()`
- `refreshUserData()`, `updateField()`, `getField()`

**Estimated Tests**: 20-30 test methods (light delegation testing)

##### Session Metadata Methods (New)

##### 1. `setUserIp()` / `getUserIp()` - **LOW PRIORITY**

**Test Cases**:
- ✅ Sets IP address
- ✅ Gets IP address
- ✅ Returns empty string when not set

**Estimated Tests**: 3-4 test methods

##### 2. `setUserAgent()` / `getUserAgent()` - **LOW PRIORITY**

**Test Cases**:
- ✅ Sets user agent
- ✅ Gets user agent
- ✅ Returns empty string when not set

**Estimated Tests**: 3-4 test methods

##### 3. `setCsrfToken()` / `getCsrfToken()` - **MEDIUM PRIORITY**

**Test Cases**:
- ✅ Sets CSRF token
- ✅ Gets CSRF token
- ✅ Returns empty string when not set

**Estimated Tests**: 3-4 test methods

##### 4. `setFlash()` / `getFlash()` / `hasFlash()` - **MEDIUM PRIORITY**

**Test Cases**:
- ✅ Sets flash message
- ✅ Gets flash message
- ✅ Flash message removed after retrieval
- ✅ Returns default when not set
- ✅ Checks if flash exists
- ✅ Multiple flash messages

**Estimated Tests**: 8-10 test methods

##### 5. `getFlashMessages()` / `clearFlash()` - **LOW PRIORITY**

**Test Cases**:
- ✅ Gets all flash messages
- ✅ Returns empty array when none
- ✅ Clears all flash messages

**Estimated Tests**: 3-4 test methods

##### Session Manager Methods (Delegated)

##### 6. `getId()` - **LOW PRIORITY**

**Test Cases**:
- ✅ Returns session ID

**Estimated Tests**: 1-2 test methods

##### 7. `regenerate()` - **MEDIUM PRIORITY** (Security)

**Test Cases**:
- ✅ Regenerates session ID
- ✅ Returns true on success
- ✅ Returns false on failure

**Estimated Tests**: 2-3 test methods

##### 8. `destroy()` - **MEDIUM PRIORITY** (Security)

**Test Cases**:
- ✅ Destroys session
- ✅ Returns true on success
- ✅ Returns false on failure

**Estimated Tests**: 2-3 test methods

##### 9. `has()` / `get()` / `set()` / `remove()` / `clear()` - **LOW PRIORITY**

**Test Cases**:
- ✅ Checks if key exists
- ✅ Gets value with default
- ✅ Sets value
- ✅ Removes value
- ✅ Clears all session data

**Estimated Tests**: 8-10 test methods

#### Estimated Total Tests
- **Total test methods**: 60-80
- **Estimated lines**: 900-1,200
- **Priority**: MEDIUM (UserSessionService already tested, this is a wrapper)

---

## Mock Dependencies Summary

### Common Mocks Required

1. **Discuz\Database\Connection**
   - Methods: `selectOne()`, `update()`, `beginTransaction()`, `commit()`, `rollBack()`
   - Used by: ForumPermissionService, PostEditService

2. **Discuz\Forum\Services\ForumPermissionService**
   - Methods: All permission check methods
   - Used by: ContentValidator, PostEditService

3. **Discuz\Credits\Services\CreditService**
   - Methods: `getUserCredits()`
   - Used by: ContentValidator

4. **Discuz\Thread\Services\ContentValidator**
   - Methods: All validation methods
   - Used by: PostEditService

5. **Discuz\Post\Repositories\PostRepository**
   - Methods: `findById()`, `update()`, `softDelete()`
   - Used by: PostEditService

6. **Discuz\Thread\Repositories\ThreadRepository**
   - Methods: `findById()`, `update()`
   - Used by: PostEditService (indirectly)

7. **Discuz\Cache\Cache**
   - Methods: `deletePattern()`
   - Used by: PostEditService, ThreadReadStatusService

8. **Discuz\Redis\Redis**
   - Methods: `zadd()`, `zscore()`, `zcard()`, `zremrangebyrank()`, `zrevrange()`, `zrem()`, `zcount()`, `set()`, `get()`, `del()`, `expire()`
   - Used by: ThreadReadStatusService

9. **Discuz\Session\SessionManager**
   - Methods: `set()`, `get()`, `remove()`, `has()`, `clear()`, `getId()`, `regenerate()`, `destroy()`
   - Used by: SessionService

---

## Priority Matrix

### CRITICAL Priority (Must Complete First)
1. **ContentValidator** - Security and validation, no tests exist
2. **PostEditService** - Data integrity, no tests exist
3. **ForumPermissionService** - Security, partial tests but missing critical paths

**Rationale**:
- ContentValidator prevents XSS, injection, and invalid data
- PostEditService modifies user content (edit/delete)
- ForumPermissionService controls all access control

### HIGH Priority
4. **ThreadReadStatusService** - Feature completeness

**Rationale**:
- User-visible feature (read/unread threads)
- Complex Redis operations
- Legacy data migration (cookies)

### MEDIUM Priority
5. **SessionService** - Wrapper around tested UserSessionService

**Rationale**:
- Mostly delegation to already-tested UserSessionService
- Some new metadata methods need testing
- Lower risk (session management already tested)

---

## Estimated Workload

### By Service

| Service | Test Methods | Lines of Code | Days (8h/day) |
|---------|--------------|---------------|---------------|
| ContentValidator | 100-120 | 1,500-2,000 | 5-7 days |
| PostEditService | 80-100 | 1,200-1,600 | 4-6 days |
| ForumPermissionService (missing) | 25-30 | 600-800 | 2-3 days |
| ThreadReadStatusService | 80-100 | 1,200-1,500 | 4-5 days |
| SessionService | 60-80 | 900-1,200 | 3-4 days |
| **TOTAL** | **345-430** | **5,400-7,100** | **18-25 days** |

### By Priority

| Priority | Services | Days |
|----------|----------|------|
| CRITICAL | ContentValidator, PostEditService, ForumPermissionService | 11-16 days |
| HIGH | ThreadReadStatusService | 4-5 days |
| MEDIUM | SessionService | 3-4 days |

---

## Implementation Strategy

### Phase 1: Critical Security Services (Week 1-2)
**Goal**: Test all security-critical validation and permission logic

**Services**:
1. ContentValidator (5-7 days)
2. PostEditService (4-6 days)
3. ForumPermissionService gaps (2-3 days)

**Deliverables**:
- `/tests/Unit/Thread/Services/ContentValidatorTest.php` (1,500-2,000 lines)
- `/tests/Unit/Post/Services/PostEditServiceTest.php` (1,200-1,600 lines)
- `/tests/Unit/Forum/Services/ForumPermissionServiceTest.php` (add 600-800 lines)

**Acceptance Criteria**:
- All public methods tested
- All exception paths tested
- All security checks tested (XSS, SQL injection, permission bypass)
- Code coverage > 85% for these services

### Phase 2: Feature Services (Week 3)
**Goal**: Test user-visible features

**Services**:
1. ThreadReadStatusService (4-5 days)

**Deliverables**:
- `/tests/Unit/Thread/Services/ThreadReadStatusServiceTest.php` (1,200-1,500 lines)

**Acceptance Criteria**:
- All Redis operations tested
- Legacy cookie import/export tested
- Edge cases (empty data, errors) tested
- Code coverage > 80%

### Phase 3: Wrapper Services (Week 4)
**Goal**: Test remaining wrapper utilities

**Services**:
1. SessionService (3-4 days)

**Deliverables**:
- `/tests/Unit/Session/Services/SessionServiceTest.php` (900-1,200 lines)

**Acceptance Criteria**:
- All public methods tested
- Delegation to UserSessionService verified
- Flash messages tested
- Code coverage > 75%

---

## Test Naming Convention

### File Structure
```
tests/
└── Unit/
    ├── Forum/
    │   └── Services/
    │       └── ForumPermissionServiceTest.php ✅ (exists, expand)
    ├── Thread/
    │   └── Services/
    │       ├── ContentValidatorTest.php ❌ (create)
    │       └── ThreadReadStatusServiceTest.php ❌ (create)
    ├── Post/
    │   └── Services/
    │       └── PostEditServiceTest.php ❌ (create)
    └── Session/
        └── Services/
            └── SessionServiceTest.php ❌ (create)
```

### Test Method Naming
```php
// Format: test{MethodName}{Scenario}{ExpectedResult}

public function testValidateSubjectWithValidSubjectPasses(): void
public function testValidateSubjectWithEmptySubjectThrowsException(): void
public function testValidateSubjectWithPermissionDeniedThrowsException(): void

public function testUpdatePostWithValidDataSucceeds(): void
public function testUpdatePostWithPostNotFoundThrowsException(): void
public function testUpdatePostWithPermissionDeniedThrowsException(): void
```

---

## Coverage Goals

### Target Metrics
- **Line Coverage**: > 80% (minimum), > 85% (target)
- **Method Coverage**: > 90% (all public methods)
- **Branch Coverage**: > 75% (all conditionals)

### Critical Paths (100% Required)
- Permission checks (all deny scenarios)
- Validation failures (all invalid inputs)
- Security checks (XSS, injection, bypass)
- Transaction rollbacks (all error scenarios)

### Edge Cases (80% Required)
- Empty/null inputs
- Database errors
- Cache failures
- Redis errors
- Boundary conditions

---

## Risk Assessment

### High Risk (Untested)

1. **ContentValidator** - NO TESTS
   - **Risk**: XSS vulnerabilities, injection attacks, invalid data
   - **Impact**: Security breach, data corruption
   - **Mitigation**: Test ALL validation paths immediately

2. **PostEditService** - NO TESTS
   - **Risk**: Unauthorized edits, data loss, transaction failures
   - **Impact**: User data corruption, integrity issues
   - **Mitigation**: Test ALL edit/delete operations immediately

3. **ForumPermissionService Moderator Logic** - UNTESTED
   - **Risk**: Privilege escalation, permission bypass
   - **Impact**: Security breach, unauthorized access
   - **Mitigation**: Test moderator checks immediately

### Medium Risk (Partial Tests)

4. **ThreadReadStatusService** - NO TESTS
   - **Risk**: Incorrect read status, Redis errors
   - **Impact**: User experience issues, data inconsistency
   - **Mitigation**: Test within Phase 2

5. **SessionService** - NO TESTS
   - **Risk**: Session metadata incorrect
   - **Impact**: Minor UX issues, CSRF token issues
   - **Mitigation**: Test within Phase 3

---

## Recommended Testing Tools

### Required
- **PHPUnit 10.x** - Already in project
- **MockObject** - PHPUnit's built-in mocking

### Optional (Recommended)
- **PHPUnit Data Providers** - For multiple test cases
- **PHPUnit Exceptions** - For exception testing
- **Faker** - For test data generation (if needed)

### Code Coverage
- **PHPUnit --coverage-html** - Generate coverage reports
- **Xdebug** - Required for coverage analysis

---

## Next Steps

### Immediate Actions (This Week)
1. ✅ Review this document with team
2. ✅ Prioritize CRITICAL services (ContentValidator, PostEditService)
3. ✅ Set up test infrastructure (ensure PHPUnit, Xdebug working)
4. ✅ Create test file stubs for all missing services

### Week 1-2: Critical Security
1. Write ContentValidator tests (100-120 methods)
2. Write PostEditService tests (80-100 methods)
3. Expand ForumPermissionService tests (25-30 methods)
4. Achieve > 85% coverage for these 3 services

### Week 3: Feature Services
1. Write ThreadReadStatusService tests (80-100 methods)
2. Achieve > 80% coverage

### Week 4: Wrapper Services
1. Write SessionService tests (60-80 methods)
2. Achieve > 75% coverage

### Final Review
1. Generate coverage report for all Week 4 services
2. Verify all exception paths tested
3. Verify all security checks tested
4. Document any remaining gaps

---

## Conclusion

### Current State
- **13% test coverage** for Week 4 services
- **4 out of 5 services** have ZERO tests
- **Critical security and validation logic** is untested

### Target State
- **> 80% test coverage** for all Week 4 services
- **345-430 test methods** across 5 services
- **5,400-7,100 lines of test code**

### Effort Required
- **18-25 working days** (3.5-5 weeks)
- **1 developer** full-time
- **OR** 2-3 developers working in parallel

### Recommendations
1. **START IMMEDIATELY** with ContentValidator (highest security risk)
2. **ALLOCATE RESOURCES** for 2-3 weeks of dedicated test writing
3. **REVIEW ALL TESTS** before merging to main branch
4. **SET UP CI** to run tests automatically (if not already done)
5. **DOCUMENT EXCEPTIONS** for any code deemed too complex to test

---

**Document Version**: 1.0
**Last Updated**: 2026-02-16
**Author**: Claude Code Analysis
**Status**: READY FOR REVIEW
