# Week 13 Day 2: Controller Tests Design Document

**Date**: 2026-02-18
**Phase**: Week 13 - QA & Production Readiness
**Day**: Day 2 - Controller Tests Implementation
**Status**: Design Complete

## Overview

This document outlines the comprehensive test strategy for 7 core controllers in the Discuz! migration project. The tests will ensure all controllers meet production-ready standards with proper error handling, security, and edge case coverage.

## Target Controllers

### 1. ProfileController
**Path**: `app/Http/Controllers/ProfileController.php`
**Test File**: `tests/Feature/Http/ProfileControllerTest.php`
**Test Count**: 12 tests (all new)
**Complexity**: Simple

### 2. ForumController
**Path**: `app/Http/Forum/ForumController.php`
**Test File**: `tests/Feature/Forum/ForumControllerTest.php`
**Test Count**: 11 existing + 1 new = 12 tests
**Complexity**: Simple

### 3. ThreadViewController
**Path**: `app/Http/Thread/ThreadViewController.php`
**Test File**: `tests/Feature/Thread/ThreadViewControllerTest.php`
**Test Count**: 4 existing + 14 new = 18 tests
**Complexity**: Simple

### 4. RegistrationController
**Path**: `app/Http/Controllers/RegistrationController.php`
**Test File**: `tests/Feature/Http/RegistrationControllerTest.php`
**Test Count**: 14 tests (all new)
**Complexity**: Moderate

### 5. ModerationController
**Path**: `app/Http/Controllers/ModerationController.php`
**Test File**: `tests/Feature/Http/ModerationControllerTest.php`
**Test Count**: 15 tests (all new)
**Complexity**: Moderate

### 6. ModerationApiController
**Path**: `app/Http/Controllers/ModerationApiController.php`
**Test File**: `tests/Feature/Http/ModerationApiControllerTest.php`
**Test Count**: 20 tests (all new)
**Complexity**: Complex

### 7. AttachmentController
**Path**: `app/Http/Controllers/AttachmentController.php`
**Test File**: `tests/Feature/Http/AttachmentControllerTest.php`
**Test Count**: 16 tests (all new)
**Complexity**: Complex

**Total**: 89 tests (18 existing, 71 new)

---

## Test Coverage Matrix

### 1. ProfileController (12 tests)

| ID | Test Method | Scenario | Type | Priority |
|----|-------------|----------|------|----------|
| P1 | `test_view_action_returns_profile_for_own_profile` | User views own profile | ✓ Happy path | P0 |
| P2 | `test_view_action_returns_permission_denied_for_restricted_profile` | User views restricted profile | ✗ Sad path | P0 |
| P3 | `test_view_action_returns_not_found_for_nonexistent_user` | User views non-existent profile | ✗ Sad path | P0 |
| P4 | `test_view_html_action_displays_profile_page` | HTML profile page render | ✓ Happy path | P0 |
| P5 | `test_view_html_action_shows_recent_posts` | Display recent posts | ✓ Happy path | P1 |
| P6 | `test_view_html_action_shows_recent_threads` | Display recent threads | ✓ Happy path | P1 |
| P7 | `test_view_html_action_checks_email_visibility` | Email visibility privacy | ◦ Edge case | P1 |
| P8 | `test_view_html_action_allows_editing_own_profile` | Edit own profile | ✓ Happy path | P0 |
| P9 | `test_view_html_action_prevents_editing_others_profile` | Cannot edit others' profile | 🔒 Security | P0 |
| P10 | `test_edit_action_returns_profile_with_csrf` | Edit form with CSRF token | ✓ Happy path | P0 |
| P11 | `test_update_action_updates_profile_successfully` | Update profile success | ✓ Happy path | P0 |
| P12 | `test_update_action_fails_with_invalid_csrf` | CSRF validation failure | 🔒 Security | P0 |

**Key Dependencies**:
- `UserRepository`
- `CsrfService`
- `SessionService`

**Security Concerns**:
- CSRF token validation
- Email visibility privacy
- Permission checks (own vs others' profiles)

---

### 2. ForumController (12 tests)

| ID | Test Method | Scenario | Type | Priority |
|----|-------------|----------|------|----------|
| F1 | `test_index_action_returns_forum_list_json` | JSON forum list | ✓ Happy path | P0 |
| F2 | `test_index_action_filters_forums_by_permission` | Permission-based filtering | ✓ Happy path | P0 |
| F3 | `test_index_action_returns_empty_for_no_forums` | No forums available | ◦ Edge case | P1 |
| F4 | `test_index_html_action_renders_forum_index_page` | HTML forum index | ✓ Happy path | P0 |
| F5 | `test_display_action_returns_forum_details_json` | JSON forum details | ✓ Happy path | P0 |
| F6 | `test_display_action_returns_not_found_for_invalid_forum` | Invalid forum ID | ✗ Sad path | P0 |
| F7 | `test_display_action_filters_threads_by_permission` | Thread permission filter | ✓ Happy path | P0 |
| F8 | `test_display_action_handles_pagination` | Thread pagination | ✓ Happy path | P1 |
| F9 | `test_display_html_action_renders_forum_thread_list` | HTML forum display | ✓ Happy path | P0 |
| F10 | `test_display_action_sorts_threads_by_sticky` | Sticky threads first | ✓ Happy path | P1 |
| F11 | `test_display_action_filters_closed_forums` | Closed forum access | ✗ Sad path | P0 |
| F12 | `test_display_action_handles_forum_password` | Password-protected forums | ◦ Edge case | P2 |

**Key Dependencies**:
- `ForumRepository`
- `ThreadRepository`
- `PermissionService`

**Security Concerns**:
- Password-protected forum access
- Permission-based filtering
- Closed forum visibility

---

### 3. ThreadViewController (18 tests)

| ID | Test Method | Scenario | Type | Priority |
|----|-------------|----------|------|----------|
| T1 | `test_view_action_returns_thread_json` | JSON thread view | ✓ Happy path | P0 |
| T2 | `test_view_action_returns_not_found_for_invalid_thread` | Invalid thread ID | ✗ Sad path | P0 |
| T3 | `test_view_action_includes_post_count` | Post count in response | ✓ Happy path | P1 |
| T4 | `test_view_html_action_renders_thread_page` | HTML thread page | ✓ Happy path | P0 |
| T5 | `test_view_html_action_includes_author_info` | Author information | ✓ Happy path | P0 |
| T6 | `test_view_html_action_handles_closed_thread` | Closed thread display | ◦ Edge case | P0 |
| T7 | `test_view_html_action_handles_digest_thread` | Digest thread badge | ✓ Happy path | P1 |
| T8 | `test_view_html_action_calculates_floor_numbers` | Floor number calculation | ✓ Happy path | P0 |
| T9 | `test_view_html_action_displays_attachments` | Attachment display | ✓ Happy path | P0 |
| T10 | `test_view_html_action_handles_nonexistent_thread` | 404 for missing thread | ✗ Sad path | P0 |
| T11 | `test_view_html_action_check_reply_permission` | Reply permission check | ✓ Happy path | P0 |
| T12 | `test_view_html_action_displays_pagination` | Post pagination | ✓ Happy path | P1 |
| T13 | `test_view_html_action_handles_special_thread_types` | Poll/debate threads | ◦ Edge case | P2 |
| T14 | `test_view_html_action_displays_signature` | User signature display | ✓ Happy path | P1 |
| T15 | `test_view_html_action_handles_edited_posts` | Edited post marker | ✓ Happy path | P1 |
| T16 | `test_view_html_action_displays_forum_breadcrumb` | Navigation breadcrumb | ✓ Happy path | P1 |
| T17 | `test_view_action_increments_view_count` | View count increment | ✓ Happy path | P1 |
| T18 | `test_view_action_prevents_increment_for_author` | No self-view increment | ◦ Edge case | P2 |

**Key Dependencies**:
- `ThreadRepository`
- `PostRepository`
- `ForumRepository`
- `UserService`

**Security Concerns**:
- Permission-based visibility
- Closed thread access control
- Reply permission checks

---

### 4. RegistrationController (14 tests)

| ID | Test Method | Scenario | Type | Priority |
|----|-------------|----------|------|----------|
| R1 | `test_index_action_displays_registration_form` | Render registration form | ✓ Happy path | P0 |
| R2 | `test_index_action_includes_csrf_token` | CSRF token in form | 🔒 Security | P0 |
| R3 | `test_register_action_creates_user_successfully` | Successful registration | ✓ Happy path | P0 |
| R4 | `test_register_action_fails_with_duplicate_username` | Duplicate username | ✗ Sad path | P0 |
| R5 | `test_register_action_fails_with_duplicate_email` | Duplicate email | ✗ Sad path | P0 |
| R6 | `test_register_action_fails_with_weak_password` | Weak password validation | ✗ Sad path | P0 |
| R7 | `test_register_action_fails_with_invalid_email` | Invalid email format | ✗ Sad path | P0 |
| R8 | `test_register_action_fails_with_csrf_mismatch` | CSRF validation failure | 🔒 Security | P0 |
| R9 | `test_register_action_applies_rate_limiting` | Rate limiting enforcement | 🔒 Security | P0 |
| R10 | `test_register_action_validates_username_format` | Username format validation | ✗ Sad path | P0 |
| R11 | `test_register_action_handles_email_verification_required` | Email verification flow | ✓ Happy path | P1 |
| R12 | `test_verify_action_activates_account_with_valid_token` | Email verification success | ✓ Happy path | P0 |
| R13 | `test_verify_action_rejects_invalid_token` | Invalid verification token | ✗ Sad path | P0 |
| R14 | `test_resend_action_sends_new_verification_email` | Resend verification email | ✓ Happy path | P1 |

**Key Dependencies**:
- `UserService`
- `RegistrationService`
- `EmailService`
- `CsrfService`
- `RateLimiterService`

**Security Concerns**:
- CSRF protection
- Rate limiting (prevent brute force)
- Email verification
- Password strength validation
- Bot detection (honeypot field)

---

### 5. ModerationController (15 tests)

| ID | Test Method | Scenario | Type | Priority |
|----|-------------|----------|------|----------|
| M1 | `test_index_action_requires_moderator_permission` | Dashboard access control | 🔒 Security | P0 |
| M2 | `test_index_action_displays_moderation_stats` | Stats display | ✓ Happy path | P1 |
| M3 | `test_deleteThread_action_deletes_thread_successfully` | Delete thread | ✓ Happy path | P0 |
| M4 | `test_deleteThread_action_fails_without_permission` | Delete permission check | 🔒 Security | P0 |
| M5 | `test_moveThread_action_moves_thread_successfully` | Move thread | ✓ Happy path | P0 |
| M6 | `test_moveThread_action_validates_target_forum` | Target forum validation | ✗ Sad path | P0 |
| M7 | `test_closeThread_action_closes_thread_successfully` | Close thread | ✓ Happy path | P0 |
| M8 | `test_closeThread_action_reopens_closed_thread` | Reopen thread | ✓ Happy path | P0 |
| M9 | `test_pinThread_action_pins_thread_successfully` | Pin thread | ✓ Happy path | P0 |
| M10 | `test_pinThread_action_unpins_pinned_thread` | Unpin thread | ✓ Happy path | P0 |
| M11 | `test_deletePost_action_deletes_post_successfully` | Delete post | ✓ Happy path | P0 |
| M12 | `test_deletePost_action_updates_thread_lastpost` | Update thread metadata | ✓ Happy path | P1 |
| M13 | `test_banUser_action_bans_user_successfully` | Ban user | ✓ Happy path | P0 |
| M14 | `test_banUser_action_prevents_banning_admins` | Cannot ban admins | 🔒 Security | P0 |
| M15 | `test_batchDelete_action_processes_batch_operation` | Batch delete | ✓ Happy path | P1 |

**Key Dependencies**:
- `ModerationService`
- `ThreadService`
- `PostService`
- `UserService`
- `PermissionService`

**Security Concerns**:
- Moderator permission checks
- Admin protection (cannot ban admins)
- CSRF protection for all actions
- Audit logging

---

### 6. ModerationApiController (20 tests)

**API Endpoints**: 14 endpoints

| ID | Test Method | Endpoint | Scenario | Priority |
|----|-------------|----------|----------|----------|
| A1 | `test_getThreads_returns_json_thread_list` | GET /api/moderation/threads | JSON response | P0 |
| A2 | `test_getThreads_returns_401_unauthorized` | GET /api/moderation/threads | Auth required | P0 |
| A3 | `test_getThreads_returns_403_forbidden` | GET /api/moderation/threads | Moderator only | P0 |
| A4 | `test_getThread_returns_json_thread_details` | GET /api/moderation/threads/{id} | Thread details | P0 |
| A5 | `test_deleteThread_returns_204_no_content` | DELETE /api/moderation/threads/{id} | Delete success | P0 |
| A6 | `test_deleteThread_returns_404_for_missing_thread` | DELETE /api/moderation/threads/{id} | Thread not found | P0 |
| A7 | `test_moveThread_returns_200_success` | POST /api/moderation/threads/{id}/move | Move success | P0 |
| A8 | `test_moveThread_returns_400_for_invalid_forum` | POST /api/moderation/threads/{id}/move | Invalid target | P0 |
| A9 | `test_pinThread_returns_200_success` | PUT /api/moderation/threads/{id}/pin | Pin success | P0 |
| A10 | `test_closeThread_returns_200_success` | PUT /api/moderation/threads/{id}/close | Close success | P0 |
| A11 | `test_getPosts_returns_json_post_list` | GET /api/moderation/posts | Post list | P0 |
| A12 | `test_deletePost_returns_204_no_content` | DELETE /api/moderation/posts/{id} | Delete success | P0 |
| A13 | `test_getUsers_returns_json_user_list` | GET /api/moderation/users | User list | P0 |
| A14 | `test_banUser_returns_200_success` | POST /api/moderation/users/{id}/ban | Ban success | P0 |
| A15 | `test_unbanUser_returns_200_success` | POST /api/moderation/users/{id}/unban | Unban success | P0 |
| A16 | `test_getReports_returns_json_report_list` | GET /api/moderation/reports | Report list | P1 |
| A17 | `test_all_responses_have_content_type_json` | All endpoints | Content-Type header | P0 |
| A18 | `test_error_responses_follow_standard_format` | All error endpoints | Error JSON structure | P0 |
| A19 | `test_pagination_parameters_work_correctly` | GET endpoints | Pagination (page, limit) | P1 |
| A20 | `test_api_requires_bearer_token_auth` | All endpoints | Authentication format | P0 |

**API Standards**:
- **Content-Type**: `application/json`
- **Authentication**: Bearer token in Authorization header
- **Success Responses**: 200 OK, 204 No Content
- **Error Responses**: 400, 401, 403, 404, 500 with JSON body
- **Error Format**: `{ "success": false, "error": { "code": "ERROR_CODE", "message": "Description" } }`

**Key Dependencies**:
- `ModerationApiController`
- JWT/Token authentication
- `ModerationService`

**Security Concerns**:
- Bearer token validation
- Moderator role check
- Input validation for all endpoints
- Rate limiting on API

---

### 7. AttachmentController (16 tests)

| ID | Test Method | Scenario | Type | Priority |
|----|-------------|----------|------|----------|
| AT1 | `test_upload_action_uploads_file_successfully` | File upload success | ✓ Happy path | P0 |
| AT2 | `test_upload_action_fails_with_invalid_file_type` | Invalid file type | ✗ Sad path | P0 |
| AT3 | `test_upload_action_fails_with_file_too_large` | File size exceeded | ✗ Sad path | P0 |
| AT4 | `test_upload_action_deducts_credits_successfully` | Credit deduction | ✓ Happy path | P0 |
| AT5 | `test_upload_action_prevents_upload_without_credits` | Insufficient credits | ✗ Sad path | P0 |
| AT6 | `test_upload_action_requires_login` | Authentication check | 🔒 Security | P0 |
| AT7 | `test_download_action_sends_file_for_valid_attachment` | File download | ✓ Happy path | P0 |
| AT8 | `test_download_action_checks_attachment_permissions` | Download permission | 🔒 Security | P0 |
| AT9 | `test_download_action_returns_not_found_for_missing_attachment` | File not found | ✗ Sad path | P0 |
| AT10 | `test_thumbnail_action_generates_thumbnail_successfully` | Thumbnail generation | ✓ Happy path | P0 |
| AT11 | `test_thumbnail_action_returns_original_for_non_image` | Non-image thumbnail | ◦ Edge case | P1 |
| AT12 | `test_delete_action_deletes_attachment_successfully` | Delete attachment | ✓ Happy path | P0 |
| AT13 | `test_delete_action_allows_only_owner_or_moderator` | Delete permission | 🔒 Security | P0 |
| AT14 | `test_listAction_returns_user_attachments_json` | Attachment list | ✓ Happy path | P1 |
| AT15 | `test_listAction_paginates_results_correctly` | List pagination | ✓ Happy path | P1 |
| AT16 | `test_all_actions_validate_csrf_tokens` | CSRF protection | 🔒 Security | P0 |

**Key Dependencies**:
- `AttachmentService`
- `StorageInterface` (file system)
- `CreditService`
- `UserService`

**Security Concerns**:
- File type validation (MIME spoofing prevention)
- File size limits
- Credit deduction
- CSRF protection
- Permission checks (owner/moderator only)
- File path traversal prevention

**File Validation Rules**:
- **Allowed types**: Images (jpg, png, gif), Documents (pdf, doc, docx, txt)
- **Max size**: 5MB for images, 10MB for documents
- **Filenames**: Sanitized to prevent path traversal
- **Virus scanning**: (Future enhancement)

---

## Test Infrastructure

### 1. ControllerTestHelpers Trait

**Path**: `tests/Traits/ControllerTestHelpers.php`

**Purpose**: Reusable test helper methods for all controller tests

**Methods**:
```php
<?php
declare(strict_types=1);

namespace Tests\Traits;

use PHPUnit\Framework\TestCase;
use Discuz\Database\Database;
use Discuz\Database\DatabaseException;

trait ControllerTestHelpers
{
    /**
     * Create a mock user in database
     *
     * @param int $uid User ID
     * @param string $username Username
     * @param int $groupId User group ID
     * @param array $extraFields Additional fields
     * @return array User data
     */
    protected function createMockUser(
        int $uid,
        string $username,
        int $groupId = 10,
        array $extraFields = []
    ): array {
        $db = Database::getInstance();

        $data = array_merge([
            'uid' => $uid,
            'username' => $username,
            'groupid' => $groupId,
            'email' => "{$username}@example.com",
            'password' => password_hash('password123', PASSWORD_BCRYPT),
            'regdate' => time(),
            'lastvisit' => time(),
            'lastactivity' => time(),
            'posts' => 0,
            'credits' => 0,
            'regip' => '127.0.0.1',
        ], $extraFields);

        $sql = "INSERT INTO cdb_members (uid, username, groupid, email, password, regdate, lastvisit, lastactivity, posts, credits, regip)
                VALUES (:uid, :username, :groupid, :email, :password, :regdate, :lastvisit, :lastactivity, :posts, :credits, :regip)
                ON DUPLICATE KEY UPDATE username = :username";

        $db->prepare($sql)->execute($data);

        return $data;
    }

    /**
     * Create a mock forum in database
     *
     * @param int $fid Forum ID
     * @param string $name Forum name
     * @param string $type Forum type (group, forum, sub)
     * @param array $extraFields Additional fields
     * @return object Forum data
     */
    protected function createMockForum(
        int $fid,
        string $name,
        string $type = 'forum',
        array $extraFields = []
    ): object {
        $db = Database::getInstance();

        $data = array_merge([
            'fid' => $fid,
            'fup' => 0,
            'type' => $type,
            'name' => $name,
            'status' => 1,
            'displayorder' => 0,
            'threads' => 0,
            'posts' => 0,
            'todayposts' => 0,
            'lastpost' => '',
        ], $extraFields);

        $sql = "INSERT INTO cdb_forums (fid, fup, type, name, status, displayorder, threads, posts, todayposts, lastpost)
                VALUES (:fid, :fup, :type, :name, :status, :displayorder, :threads, :posts, :todayposts, :lastpost)
                ON DUPLICATE KEY UPDATE name = :name";

        $db->prepare($sql)->execute($data);

        return (object) $data;
    }

    /**
     * Create a mock thread in database
     *
     * @param int $tid Thread ID
     * @param int $fid Forum ID
     * @param int $authorId Author user ID
     * @param string $subject Thread subject
     * @param array $extraFields Additional fields
     * @return object Thread data
     */
    protected function createMockThread(
        int $tid,
        int $fid,
        int $authorId,
        string $subject,
        array $extraFields = []
    ): object {
        $db = Database::getInstance();

        $data = array_merge([
            'tid' => $tid,
            'fid' => $fid,
            'authorid' => $authorId,
            'subject' => $subject,
            'dateline' => time(),
            'lastpost' => time(),
            'lastposter' => 'test_user',
            'views' => 0,
            'replies' => 0,
            'displayorder' => 0,
            'highlight' => 0,
            'digest' => 0,
            'rate' => 0,
            'special' => 0,
            'status' => 0,
            'closed' => 0,
        ], $extraFields);

        $sql = "INSERT INTO cdb_threads (tid, fid, authorid, subject, dateline, lastpost, lastposter, views, replies, displayorder, highlight, digest, rate, special, status, closed)
                VALUES (:tid, :fid, :authorid, :subject, :dateline, :lastpost, :lastposter, :views, :replies, :displayorder, :highlight, :digest, :rate, :special, :status, :closed)
                ON DUPLICATE KEY UPDATE subject = :subject";

        $db->prepare($sql)->execute($data);

        return (object) $data;
    }

    /**
     * Assert JSON response structure
     *
     * @param array $response Response data
     * @param bool $success Expected success status
     * @param int $expectedCode Expected HTTP code
     * @return void
     */
    protected function assertJsonResponse(array $response, bool $success, int $expectedCode = 200): void
    {
        $this->assertIsArray($response);
        $this->assertArrayHasKey('success', $response);
        $this->assertEquals($success, $response['success']);

        if (!$success) {
            $this->assertArrayHasKey('error', $response);
            $this->assertArrayHasKey('code', $response['error']);
            $this->assertArrayHasKey('message', $response['error']);
        }
    }

    /**
     * Clean up test data from database
     *
     * @param array $tables Tables to clean (table => id_field)
     * @param array $ids IDs to delete
     * @return void
     */
    protected function cleanupTestData(array $tables, array $ids): void
    {
        $db = Database::getInstance();

        foreach ($tables as $table => $idField) {
            if (!empty($ids[$table])) {
                $placeholders = implode(',', array_fill(0, count($ids[$table]), '?'));
                $sql = "DELETE FROM {$table} WHERE {$idField} IN ({$placeholders})";
                $db->prepare($sql)->execute($ids[$table]);
            }
        }
    }

    /**
     * Mock authenticated session
     *
     * @param int $uid User ID
     * @param string $username Username
     * @param int $groupId User group ID
     * @return void
     */
    protected function mockAuthSession(int $uid, string $username, int $groupId = 10): void
    {
        $_SESSION['uid'] = $uid;
        $_SESSION['username'] = $username;
        $_SESSION['groupid'] = $groupId;
    }

    /**
     * Clear authenticated session
     *
     * @return void
     */
    protected function clearAuthSession(): void
    {
        unset($_SESSION['uid'], $_SESSION['username'], $_SESSION['groupid']);
    }

    /**
     * Create a valid CSRF token
     *
     * @return string CSRF token
     */
    protected function createValidCsrfToken(): string
    {
        return 'valid_csrf_token_' . bin2hex(random_bytes(16));
    }

    /**
     * Mock file upload
     *
     * @param string $filename Filename
     * @param string $mimeType MIME type
     * @param int $size File size in bytes
     * @param string $tmpPath Temporary path
     * @return array File upload data
     */
    protected function mockFileUpload(
        string $filename,
        string $mimeType,
        int $size,
        string $tmpPath = '/tmp/test_upload'
    ): array {
        return [
            'name' => $filename,
            'type' => $mimeType,
            'size' => $size,
            'tmp_name' => $tmpPath,
            'error' => UPLOAD_ERR_OK,
        ];
    }
}
```

### 2. ControllerTestFixturetures Class

**Path**: `tests/Fixtures/ControllerTestData.php`

**Purpose**: Test data fixtures for controller tests

**Methods**:
```php
<?php
declare(strict_types=1);

namespace Tests\Fixtures;

class ControllerTestData
{
    /**
     * Valid user registration data
     *
     * @return array
     */
    public static function validUserData(): array
    {
        return [
            'username' => 'testuser_' . time(),
            'email' => 'testuser_' . time() . '@example.com',
            'password' => 'SecurePass123!',
            'password_confirm' => 'SecurePass123!',
            'csrf_token' => 'valid_token',
        ];
    }

    /**
     * Invalid username list for validation testing
     *
     * @return array
     */
    public static function invalidUsernames(): array
    {
        return [
            'too_short' => ['ab', 'Username must be at least 3 characters'],
            'too_long' => [str_repeat('a', 20), 'Username must be less than 15 characters'],
            'invalid_chars' => ['user@name', 'Username contains invalid characters'],
            'starts_with_number' => ['123user', 'Username must start with a letter'],
            'contains_spaces' => ['user name', 'Username cannot contain spaces'],
        ];
    }

    /**
     * Weak password list for validation testing
     *
     * @return array
     */
    public static function weakPasswords(): array
    {
        return [
            'too_short' => ['12345', 'Password must be at least 6 characters'],
            'only_numbers' => ['12345678', 'Password must contain letters'],
            'only_letters' => ['abcdefgh', 'Password must contain numbers'],
            'common_password' => ['password', 'Password is too common'],
            'no_special_chars' => ['Password123', 'Password should contain special characters'],
        ];
    }

    /**
     * Valid file data for upload testing
     *
     * @return array
     */
    public static function validFileData(): array
    {
        return [
            'image_jpg' => [
                'name' => 'test_image.jpg',
                'type' => 'image/jpeg',
                'size' => 1024 * 500, // 500KB
                'tmp_path' => '/tmp/test_image.jpg',
            ],
            'image_png' => [
                'name' => 'test_image.png',
                'type' => 'image/png',
                'size' => 1024 * 300, // 300KB
                'tmp_path' => '/tmp/test_image.png',
            ],
            'document_pdf' => [
                'name' => 'test_document.pdf',
                'type' => 'application/pdf',
                'size' => 1024 * 1024 * 2, // 2MB
                'tmp_path' => '/tmp/test_document.pdf',
            ],
        ];
    }

    /**
     * Invalid file types for validation testing
     *
     * @return array
     */
    public static function invalidFileTypes(): array
    {
        return [
            'executable' => [
                'name' => 'malware.exe',
                'type' => 'application/x-executable',
                'size' => 1024,
            ],
            'script' => [
                'name' => 'script.js',
                'type' => 'application/javascript',
                'size' => 1024,
            ],
            'oversized_image' => [
                'name' => 'huge.jpg',
                'type' => 'image/jpeg',
                'size' => 1024 * 1024 * 10, // 10MB (exceeds 5MB limit)
            ],
        ];
    }

    /**
     * Thread moderation actions
     *
     * @return array
     */
    public static function moderationActions(): array
    {
        return [
            'delete' => ['action' => 'delete', 'reason' => 'Spam'],
            'move' => ['action' => 'move', 'target_fid' => 2],
            'close' => ['action' => 'close', 'reason' => 'Rule violation'],
            'pin' => ['action' => 'pin', 'duration' => 1], // 1 day
            'highlight' => ['action' => 'highlight', 'color' => 'red'],
        ];
    }

    /**
     * User group data
     *
     * @return array
     */
    public static function userGroups(): array
    {
        return [
            'guest' => ['groupid' => 7, 'grouptitle' => 'Guest'],
            'member' => ['groupid' => 10, 'grouptitle' => 'Member'],
            'moderator' => ['groupid' => 3, 'grouptitle' => 'Moderator'],
            'super_moderator' => ['groupid' => 4, 'grouptitle' => 'Super Moderator'],
            'admin' => ['groupid' => 1, 'grouptitle' => 'Administrator'],
        ];
    }
}
```

---

## Implementation Plan

### Phase 1: Simple Controllers (2 hours estimated)

**Order**: From simplest to most complex

#### 1. ProfileControllerTest (45 minutes)
- **Why start here**: Single resource controller, straightforward CRUD
- **Dependencies**: UserRepository, CsrfService
- **Complexity**: Low (no external services, simple permission checks)
- **Implementation Steps**:
  1. Read `app/Http/Controllers/ProfileController.php`
  2. Identify all public methods and their responsibilities
  3. Create test file with setUp() for mocks
  4. Implement 12 tests following the matrix
  5. Run tests and fix failures

#### 2. ForumControllerTest (30 minutes)
- **Why second**: 11 tests already exist, only 1 new test needed
- **Dependencies**: ForumRepository, ThreadRepository
- **Complexity**: Low (mostly read operations)
- **Implementation Steps**:
  1. Read existing test file
  2. Identify missing test: `test_index_html_action_renders_forum_index_page`
  3. Add missing HTML rendering test
  4. Run all tests to ensure no regressions

#### 3. ThreadViewControllerTest (45 minutes)
- **Why third**: 4 tests already exist, clear pattern to follow
- **Dependencies**: ThreadRepository, PostRepository
- **Complexity**: Low-Medium (more data relationships)
- **Implementation Steps**:
  1. Read existing test file
  2. Review controller methods for all HTML rendering scenarios
  3. Add 14 new tests for HTML rendering, pagination, permissions
  4. Ensure floor calculations and attachment display are tested

### Phase 2: Moderate Controllers (3 hours estimated)

#### 4. RegistrationControllerTest (90 minutes)
- **Why fourth**: Multiple validation scenarios, rate limiting
- **Dependencies**: RegistrationService, EmailService, RateLimiter
- **Complexity**: Medium (service coordination, security features)
- **Implementation Steps**:
  1. Read `app/Http/Controllers/RegistrationController.php`
  2. Create test file with comprehensive setUp()
  3. Mock all external services (Email, RateLimiter, CSRF)
  4. Use data providers for validation test cases
  5. Test rate limiting with time-based assertions
  6. Test email verification flow end-to-end

#### 5. ModerationControllerTest (90 minutes)
- **Why fifth**: Multiple action types, permission checks
- **Dependencies**: ModerationService, ThreadService, PostService
- **Complexity**: Medium (role-based access control, batch operations)
- **Implementation Steps**:
  1. Read `app/Http/Controllers/ModerationController.php`
  2. Create test file
  3. Test each moderation action (delete, move, close, pin)
  4. Test permission checks for different user roles
  5. Test admin protection (cannot ban admins)
  6. Test batch operations

### Phase 3: Complex Controllers (3 hours estimated)

#### 6. ModerationApiControllerTest (120 minutes)
- **Why sixth**: REST API with 14 endpoints, JSON responses
- **Dependencies**: JWT/Token auth, ModerationService
- **Complexity**: High (API standards, HTTP status codes, JSON format)
- **Implementation Steps**:
  1. Read `app/Http/Controllers/ModerationApiController.php`
  2. Create test file
  3. Test all 14 API endpoints
  4. Verify HTTP status codes (200, 204, 400, 401, 403, 404, 500)
  5. Verify JSON response format for success and error cases
  6. Verify Content-Type headers
  7. Test Bearer token authentication
  8. Test pagination parameters

#### 7. AttachmentControllerTest (60 minutes)
- **Why last**: File system mocking, credit deduction
- **Dependencies**: StorageInterface, CreditService, AttachmentService
- **Complexity**: High (file handling, storage abstraction, credits)
- **Implementation Steps**:
  1. Read `app/Http/Controllers/AttachmentController.php`
  2. Create test file
  3. Mock StorageInterface for file operations
  4. Test file upload (success, invalid type, size exceeded)
  5. Test credit deduction integration
  6. Test file download with permission checks
  7. Test thumbnail generation
  8. Test file deletion (owner/moderator only)

---

## Test Execution Strategy

### Running Tests

```bash
# Single test file
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/Feature/Http/ProfileControllerTest.php

# All controller tests
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/Feature/

# Verbose output
docker exec -i discuz_modern_php php vendor/bin/phpunit --verbose tests/Feature/

# With coverage
docker exec -i discuz_modern_php php vendor/bin/phpunit --coverage-text tests/Feature/
```

### Test Organization

```
tests/
├── Feature/
│   ├── Http/
│   │   ├── ProfileControllerTest.php (NEW)
│   │   ├── RegistrationControllerTest.php (NEW)
│   │   ├── ModerationControllerTest.php (NEW)
│   │   ├── ModerationApiControllerTest.php (NEW)
│   │   └── AttachmentControllerTest.php (NEW)
│   ├── Forum/
│   │   └── ForumControllerTest.php (UPDATE)
│   └── Thread/
│       └── ThreadViewControllerTest.php (UPDATE)
├── Traits/
│   └── ControllerTestHelpers.php (NEW)
└── Fixtures/
    └── ControllerTestData.php (NEW)
```

### Coverage Targets

- **Line Coverage**: ≥90%
- **Method Coverage**: 100%
- **Branch Coverage**: ≥85%

---

## Test Quality Standards

### Naming Conventions

**Test Methods**: `test_{methodName}_{scenario}_{expectedResult}`

Examples:
- `test_view_action_returns_profile_for_own_profile`
- `test_register_action_fails_with_duplicate_username`
- `test_deleteThread_action_requires_moderator_permission`

### Test Structure

Each test should follow this pattern:

```php
public function test_{scenario}_{expected_result(): void
{
    // Arrange - Set up test data and mocks
    $userId = 1001;
    $this->createMockUser($userId, 'testuser');

    // Act - Execute the method being tested
    $response = $this->controller->view($userId);

    // Assert - Verify expected outcome
    $this->assertJsonResponse($response, true);
    $this->assertEquals('testuser', $response['data']['username']);
}
```

### Mocking Strategy

**DO Mock**:
- External services (EmailService, StorageInterface)
- Service dependencies (UserService, ModerationService)
- Session/Cookie handling

**DON'T Mock**:
- Database (use test database with cleanup)
- Simple value objects (DTOs, Entities)
- Controller dependencies that are being tested

### Data Providers

Use data providers for multiple test cases with same logic:

```php/**
 * @dataProvider invalidUsernameProvider
 */
public function test_register_action_fails_with_invalid_username(string $username, string $expectedError): void
{
    // Test implementation
}

public function invalidUsernameProvider(): array
{
    return ControllerTestData::invalidUsernames();
}
```

---

## Success Criteria

### Code Quality
- [ ] All tests pass (100% pass rate)
- [ ] PSR-12 compliance (run `composer lint:fix`)
- [ ] PHPDoc comments on all test methods
- [ ] No PHPUnit warnings or deprecations

### Coverage Metrics
- [ ] ≥90% line coverage
- [ ] 100% method coverage
- [ ] ≥85% branch coverage

### Test Quality
- [ ] Tests are independent (no interdependencies)
- [ ] Tests clean up after themselves
- [ ] Tests run in random order without failures
- [ ] Clear test names describing what is being tested

### Documentation
- [ ] Test file headers with purpose
- [ ] Complex test logic has inline comments
- [ ] Data providers are documented

---

## Estimated Timeline

| Phase | Tasks | Time | Cumulative |
|-------|-------|------|------------|
| Phase 1 | Simple Controllers (Profile, Forum, ThreadView) | 2h | 2h |
| Phase 2 | Moderate Controllers (Registration, Moderation) | 3h | 5h |
| Phase 3 | Complex Controllers (ModerationApi, Attachment) | 3h | 8h |
| **Total** | **All Controllers** | **8h** | **8h** |

**Buffer Time**: 1-2 hours for debugging and refinements

**Total Estimated**: 8-10 hours

---

## Next Steps

1. **Implement**: Follow the implementation plan phase by phase
2. **Execute Tests**: Run tests after each phase
3. **Fix Failures**: Debug and fix any failing tests
4. **Refactor**: Improve test quality and reduce duplication
5. **Report**: Generate implementation report with results

---

## Appendix: Reference Materials

### Existing Test Example
- **File**: `tests/Feature/Http/AuthControllerTest.php`
- **Purpose**: Reference for test structure, mocking, assertions

### Controller Files to Test
- `app/Http/Controllers/ProfileController.php`
- `app/Http/Forum/ForumController.php`
- `app/Http/Thread/ThreadViewController.php`
- `app/Http/Controllers/RegistrationController.php`
- `app/Http/Controllers/ModerationController.php`
- `app/Http/Controllers/ModerationApiController.php`
- `app/Http/Controllers/AttachmentController.php`

### Service Dependencies
- `app/User/Services/UserService.php`
- `app/Security/Services/RegistrationService.php`
- `app/Security/Services/ModerationService.php`
- `app/Attachment/Services/AttachmentService.php`
- `app/Cache/Services/CacheService.php`

---

**End of Design Document**
