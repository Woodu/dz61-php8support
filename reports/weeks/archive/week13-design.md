# Week 13 Design Document: Quality Assurance & Production Readiness

**Date**: 2026-02-18
**Team**: Week 13 Implementation Team
**Status**: Design Phase
**Duration**: 6 days (2026-02-18 to 2026-02-23)

---

## Executive Summary

Week 13 focuses on **Quality Assurance & Production Readiness Enhancement**, aiming to increase production readiness from 95% to 100%. Based on the comprehensive audit report (AUDIT-SUMMARY.md), we will address two critical gaps:

1. **Controller Test Coverage** (75% → 90%)
2. **End-to-End Testing** (20% → 100%)

### Success Criteria

- ✅ All Controller tests ≥90% coverage
- ✅ All E2E tests for critical user journeys
- ✅ 100% test pass rate

---

## Current State Analysis

### Test Coverage Status

Based on audit report (2026-02-18):

| Component | Current | Target | Gap |
|-----------|---------|--------|-----|
| Controller Tests | 75% | 90% | -15% |
| E2E Tests | 20% | 100% | -80% |
| Unit Tests | 95% | 95% | ✅ |
| Integration Tests | 85% | 85% | ✅ |

### Controllers Requiring Tests

**Priority 1 (Critical)**:
1. `AuthController.php` - Login/logout flows
2. `RegistrationController.php` - User registration flow
3. `ForumController.php` - Forum navigation
4. `ThreadViewController.php` - Thread viewing
5. `ProfileController.php` - User profiles

**Priority 2 (Important)**:
6. `ModerationController.php` - Moderator CP
7. `ModerationApiController.php` - Moderation API
8. `AttachmentController.php` - Attachment upload/download

### Critical User Journeys (E2E)

**Identified 5 critical journeys**:

1. **User Registration Journey**
   - Access registration page → Fill form → Submit → Verify email → Login
   - Expected time: 3-5 minutes
   - Critical for user acquisition

2. **User Login Journey**
   - Access login page → Enter credentials → Submit → Verify session → Access protected page
   - Expected time: 1-2 minutes
   - Critical for user retention

3. **Post Creation Journey**
   - Login → Navigate forum → Click new thread → Fill content → Submit → Verify display
   - Expected time: 2-3 minutes
   - Core forum functionality

4. **Moderator Management Journey**
   - Moderator login → Access ModCP → View pending → Review post → Verify log
   - Expected time: 2-4 minutes
   - Critical for content moderation

5. **Attachment Upload Journey**
   - Login → Post page → Upload attachment → Submit → Verify display → Download
   - Expected time: 3-5 minutes
   - Important for user engagement

---

## Week 13 Work Plan

### Phase 1: Design (Day 1 - Morning)

**Objectives**:
- Analyze audit report recommendations
- Review existing test infrastructure
- Design test data strategy
- Plan E2E test scenarios

**Deliverables**:
- This design document
- Test data fixture plan
- E2E scenario specifications

### Phase 2: Controller Tests (Days 1-3)

**Day 1 Afternoon - AuthController Tests**
- File: `tests/Feature/Http/AuthControllerTest.php`
- Tests: 15-20 test cases
- Coverage: Login, logout, CSRF, session management

**Day 2 Morning - RegistrationController Tests**
- File: `tests/Feature/Http/RegistrationControllerTest.php`
- Tests: 12-15 test cases
- Coverage: Registration validation, email verification, duplicate handling

**Day 2 Afternoon - ForumController Tests**
- File: `tests/Feature/Forum/ForumControllerTest.php`
- Tests: 10-12 test cases
- Coverage: Forum listing, pagination, permissions

**Day 3 Morning - ThreadViewController Tests**
- File: `tests/Feature/Thread/ThreadViewControllerTest.php`
- Tests: 15-18 test cases
- Coverage: Thread viewing, pagination, BBCode rendering

**Day 3 Afternoon - ProfileController Tests**
- File: `tests/Feature/Http/ProfileControllerTest.php`
- Tests: 10-12 test cases
- Coverage: Profile viewing, editing, permissions

### Phase 3: E2E Tests (Days 4-5)

**Day 4 - Critical User Journeys 1-3**

**Journey 1: User Registration**
- File: `tests/Feature/E2E/UserRegistrationJourneyTest.php`
- Tests: 8-10 scenarios
- Database: Test database (discuz_utf8_test)
- Fixtures: RegistrationTestData

**Journey 2: User Login**
- File: `tests/Feature/E2E/UserLoginJourneyTest.php`
- Tests: 6-8 scenarios
- Database: Test database
- Fixtures: LoginTestData

**Journey 3: Post Creation**
- File: `tests/Feature/E2E/PostCreationJourneyTest.php`
- Tests: 10-12 scenarios
- Database: Test database
- Fixtures: PostTestData

**Day 5 - Critical User Journeys 4-5**

**Journey 4: Moderator Management**
- File: `tests/Feature/E2E/ModeratorManagementJourneyTest.php`
- Tests: 12-15 scenarios
- Database: Test database
- Fixtures: ModerationTestData

**Journey 5: Attachment Upload**
- File: `tests/Feature/E2E/AttachmentUploadJourneyTest.php`
- Tests: 8-10 scenarios
- Database: Test database
- Fixtures: AttachmentTestData

---

## Test Data Strategy

### Fixtures Design

**Location**: `tests/Fixture/`

**Existing Fixtures**:
- `ModerationTestData.php` - Moderation test data
- `UserTestData.php` - User test data

**New Fixtures to Create**:
- `RegistrationTestData.php` - Registration scenarios
- `LoginTestData.php` - Login scenarios
- `PostTestData.php` - Post/thread data
- `AttachmentTestData.php` - Attachment files

### Database Isolation

**Approach**: Use test database with transaction rollback

**Configuration**:
```php
// tests/bootstrap.php
putenv('DB_NAME=discuz_utf8_test');
```

**Cleanup Strategy**:
- Each test: Transaction rollback
- Each test suite: TRUNCATE tables
- E2E tests: Full database reset

---

## Testing Infrastructure

### Test Base Classes

**HttpTest Base** (`tests/Feature/HttpTest.php`):
```php
abstract class HttpTest extends TestCase {
    protected function createTestClient(): HttpClient;
    protected function authenticateAs(int $uid): string;
    protected function assertCsrfProtection(string $route);
    protected function assertPermissionDenied(int $statusCode);
}
```

**E2ETest Base** (`tests/Feature/E2ETest.php`):
```php
abstract class E2ETest extends TestCase {
    protected function navigateTo(string $url): Response;
    protected function submitForm(array $data): Response;
    protected function followRedirect(): Response;
    protected function assertDatabaseHas(string $table, array $data);
}
```

### Mock Strategy

**Services to Mock**:
- `CacheService` - Redis/File cache
- `EmailService` - Email sending
- `RateLimiterService` - Rate limiting (optional)
- `SessionService` - Session storage

**Repositories to Mock**:
- None (use real database with fixtures)

**Services NOT to Mock**:
- `AuthService` - Critical path
- `PermissionService` - Security critical
- `ModerationService` - Business logic

---

---

## Risk Assessment

### Technical Risks

**Risk 1: Test Data Contamination**
- Probability: Medium
- Impact: High
- Mitigation: Transaction rollback, database isolation

**Risk 2: E2E Test Flakiness**
- Probability: High
- Impact: Medium
- Mitigation: Explicit waits, retry logic, proper fixtures

### Schedule Risks

**Risk 1: Test Implementation Complexity**
- Probability: Medium
- Impact: High
- Mitigation: Start with simpler tests, build complexity gradually

---

## Quality Gates

### Definition of Done

**Controller Tests**:
- [ ] All tests pass (100%)
- [ ] Code coverage ≥90%
- [ ] PSR-12 compliant
- [ ] PHPDoc complete
- [ ] No TODOs in tests

**E2E Tests**:
- [ ] All 5 journeys implemented
- [ ] All tests pass (100%)
- [ ] Each journey has ≥8 scenarios
- [ ] Database cleanup verified
- [ ] Screenshots on failure (if applicable)

---

## Execution Timeline

| Day | Morning | Afternoon |
|-----|---------|-----------|
| **Day 1** | Design + fixtures | AuthController tests |
| **Day 2** | RegistrationController tests | ForumController tests |
| **Day 3** | ThreadViewController tests | ProfileController tests |
| **Day 4** | E2E: Registration journey | E2E: Login journey |
| **Day 5** | E2E: Post creation | E2E: Moderator + Attachments |

---

## Success Metrics

### Quantitative Metrics

- **Test Coverage**: Controller ≥90% (from 75%)
- **E2E Scenarios**: ≥50 scenarios (5 journeys × 10 scenarios)
- **Test Pass Rate**: 100%

### Qualitative Metrics

- All critical user journeys verified
- Production readiness increased to 100%

---

## Next Steps

1. ✅ Review and approve this design document
2. ✅ Set up test database fixtures
3. ✅ Create base test classes
4. ✅ Begin AuthController tests
5. ✅ Execute Week 13 plan

---

## Appendix A: Test Case Checklist

### AuthController Test Cases

- [ ] Login with valid credentials
- [ ] Login with invalid username
- [ ] Login with invalid password
- [ ] Login with CSRF token missing
- [ ] Login with CSRF token invalid
- [ ] Login with rate limit exceeded
- [ ] Logout destroys session
- [ ] Logout regenerates CSRF token
- [ ] Session fixation prevention
- [ ] Remember me handling (if applicable)
- [ ] Redirect after login
- [ ] Redirect after logout
- [ ] Login with UC credentials
- [ ] Login with local credentials
- [ ] Concurrent login handling

### RegistrationController Test Cases

- [ ] Register with valid data
- [ ] Register with duplicate username
- [ ] Register with duplicate email
- [ ] Register with weak password
- [ ] Register with invalid email format
- [ ] Register with CSRF token missing
- [ ] Register with agree terms unchecked
- [ ] Email verification required
- [ ] Email verification code valid
- [ ] Email verification code invalid
- [ ] Email verification code expired
- [ ] Registration rate limiting

### ForumController Test Cases

- [ ] View forum index
- [ ] View forum by ID
- [ ] View forum with pagination
- [ ] View forum without permission
- [ ] View subforums
- [ ] View thread list
- [ ] Filter by announcement
- [ ] Filter by sticky
- [ ] Search forum
- [ ] Access non-existent forum
- [ ] Forum RSS feed (if applicable)
- [ ] Forum statistics

### ThreadViewController Test Cases

- [ ] View thread by ID
- [ ] View thread with pagination
- [ ] View thread without permission
- [ ] View thread with [free] tags (paid)
- [ ] View thread with poll
- [ ] View thread with attachments
- [ ] View non-existent thread
- [ ] View deleted thread
- [ ] View closed thread
- [ ] BBCode rendering
- [ ] Smiley rendering
- [ ] Auto-link rendering
- [ ] Quote nesting
- [ ] Code block formatting
- [ ] Thread rating (if applicable)

### ProfileController Test Cases

- [ ] View own profile
- [ ] View other user profile
- [ ] Edit own profile
- [ ] Edit other user profile (permission denied)
- [ ] Update avatar
- [ ] Update signature
- [ ] Update bio
- [ ] View profile with privacy settings
- [ ] View user posts
- [ ] View user threads
- [ ] Profile not found
- [ ] Profile with banned user

---

## Appendix B: E2E Test Scenarios

### Journey 1: User Registration

**Scenario 1.1**: Successful Registration
- Given: New user visits registration page
- When: Fill valid form data
- Then: Account created, email sent, redirect to verification page

**Scenario 1.2**: Registration with Duplicate Email
- Given: Existing email
- When: Attempt registration
- Then: Error message "Email already exists"

**Scenario 1.3**: Registration with Weak Password
- Given: Password < 6 characters
- When: Attempt registration
- Then: Error message "Password too weak"

**Scenario 1.4**: Email Verification Success
- Given: New registration pending
- When: Click valid verification link
- Then: Account activated, redirect to login

**Scenario 1.5**: Email Verification Invalid Code
- Given: New registration pending
- When: Submit invalid verification code
- Then: Error message "Invalid verification code"

**Scenario 1.6**: Email Verification Expired
- Given: Registration > 24 hours ago
- When: Submit verification code
- Then: Error message "Verification code expired"

**Scenario 1.7**: Registration Rate Limit
- Given: 5 registrations from same IP in 15 minutes
- When: Attempt 6th registration
- Then: Error message "Too many registration attempts"

**Scenario 1.8**: CSRF Token Missing
- Given: Registration page loaded
- When: Submit without CSRF token
- Then: Error message "CSRF token validation failed"

### Journey 2: User Login

**Scenario 2.1**: Successful Login
- Given: Valid credentials
- When: Submit login form
- Then: Session created, redirect to home

**Scenario 2.2**: Invalid Username
- Given: Non-existent username
- When: Submit login form
- Then: Error message "Invalid credentials"

**Scenario 2.3**: Invalid Password
- Given: Valid username, wrong password
- When: Submit login form
- Then: Error message "Invalid credentials"

**Scenario 2.4**: Login Rate Limit
- Given: 5 failed attempts from IP
- When: Attempt 6th login
- Then: Error message "Too many login attempts"

**Scenario 2.5**: Session After Login
- Given: Successfully logged in
- When: Access protected page
- Then: Page displays correctly

**Scenario 2.6**: Login with UC Credentials
- Given: UC user exists
- When: Login with UC password
- Then: Successful, password migrated to bcrypt

**Scenario 2.7**: Session Fixation Prevention
- Given: Login page loaded
- When: Log in
- Then: Session ID regenerated

**Scenario 2.8**: Logout
- Given: Logged in
- When: Click logout
- Then: Session destroyed, redirect to login

### Journey 3: Post Creation

**Scenario 3.1**: Create Simple Thread
- Given: Logged in user with permission
- When: Create thread with title and content
- Then: Thread created, redirect to thread view

**Scenario 3.2**: Create Thread with Poll
- Given: Logged in user
- When: Create thread with poll options
- Then: Thread created with poll

**Scenario 3.3**: Create Thread with Attachment
- Given: Logged in user
- When: Upload attachment and create thread
- Then: Thread created with attachment

**Scenario 3.4**: Create Thread with BBCode
- Given: Logged in user
- When: Use BBCode formatting
- Then: BBCode rendered correctly

**Scenario 3.5**: Create Thread without Permission
- Given: User without post permission
- When: Attempt to create thread
- Then: Error message "Permission denied"

**Scenario 3.6**: Create Thread in Closed Forum
- Given: Forum is closed
- When: Attempt to create thread
- Then: Error message "Forum is closed"

**Scenario 3.7**: Create Thread with Duplicate Title
- Given: Existing thread title
- When: Attempt duplicate
- Then: Thread created (allow duplicates)

**Scenario 3.8**: Create Paid Thread
- Given: User with credits
- When: Set thread price
- Then: Thread created with [free] tags

**Scenario 3.9**: Edit Own Thread
- Given: Own thread
- When: Edit content
- Then: Thread updated successfully

**Scenario 3.10**: Delete Own Thread
- Given: Own thread
- When: Delete thread
- Then: Thread deleted, redirect to forum

**Scenario 3.11**: Reply to Thread
- Given: Existing thread
- When: Post reply
- Then: Reply added, thread updated

**Scenario 3.12**: Quote in Reply
- Given: Existing post
- When: Reply with quote
- Then: Quote rendered correctly

### Journey 4: Moderator Management

**Scenario 4.1**: Access ModCP
- Given: Moderator logged in
- When: Access ModCP
- Then: Dashboard displayed

**Scenario 4.2**: View Pending Posts
- Given: Moderator logged in
- When: Navigate to pending posts
- Then: List of pending posts displayed

**Scenario 4.3**: Approve Post
- Given: Pending post
- When: Approve post
- Then: Post visible, log created

**Scenario 4.4**: Delete Post
- Given: Post to delete
- When: Delete post
- Then: Post deleted, log created

**Scenario 4.5**: Close Thread
- Given: Open thread
- When: Close thread
- Then: Thread closed, log created

**Scenario 4.6**: Open Thread
- Given: Closed thread
- When: Open thread
- Then: Thread opened, log created

**Scenario 4.7**: Sticky Thread
- Given: Normal thread
- When: Make sticky
- Then: Thread sticky, log created

**Scenario 4.8**: Unsticky Thread
- Given: Sticky thread
- When: Remove sticky
- Then: Thread unsticky, log created

**Scenario 4.9**: Move Thread
- Given: Thread in Forum A
- When: Move to Forum B
- Then: Thread moved, log created

**Scenario 4.10**: Ban User
- Given: User to ban
- When: Ban user
- Then: User banned, log created

**Scenario 4.11**: Unban User
- Given: Banned user
- When: Unban user
- Then: User unbanned, log created

**Scenario 4.12**: View Moderation Logs
- Given: Moderator logged in
- When: View logs
- Then: Log history displayed

**Scenario 4.13**: Non-Moderator Access Denied
- Given: Regular member
- When: Access ModCP
- Then: Redirect to home / permission denied

**Scenario 4.14**: Moderator Actions Outside Forum
- Given: Moderator of Forum A
- When: Attempt action in Forum B
- Then: Permission denied

**Scenario 4.15**: Edit User Post
- Given: Moderator permission
- When: Edit user post
- Then: Post updated, log created

### Journey 5: Attachment Upload

**Scenario 5.1**: Upload Image Attachment
- Given: Logged in user
- When: Upload JPG image
- Then: Attachment uploaded, thumbnail created

**Scenario 5.2**: Upload PDF Attachment
- Given: Logged in user
- When: Upload PDF file
- Then: Attachment uploaded

**Scenario 5.3**: Upload Oversized File
- Given: File exceeds max size
- When: Attempt upload
- Then: Error message "File too large"

**Scenario 5.4**: Upload Invalid Format
- Given: .exe file
- When: Attempt upload
- Then: Error message "Invalid file type"

**Scenario 5.5**: Upload without Credits (Paid Attachment)
- Given: Paid attachment, insufficient credits
- When: Attempt upload
- Then: Error message "Insufficient credits"

**Scenario 5.6**: Download Own Attachment
- Given: Own attachment
- When: Download
- Then: File served correctly

**Scenario 5.7**: Download Free Attachment
- Given: Free attachment
- When: Download
- Then: File served correctly

**Scenario 5.8**: Download Paid Attachment (Payment)
- Given: Paid attachment, sufficient credits
- When: Download
- Then: Credits deducted, file served

**Scenario 5.9**: Hotlink Protection
- Given: Attachment URL
- When: Access from external site
- Then: 403 Forbidden

**Scenario 5.10**: Delete Attachment
- Given: Own attachment
- When: Delete
- Then: File and record deleted

---

**Document Version**: 1.0
**Last Updated**: 2026-02-18
**Next Review**: End of Day 1 (2026-02-18)

---

End of Design Document
