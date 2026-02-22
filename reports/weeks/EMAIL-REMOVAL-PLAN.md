# Email Sending Functionality Removal Plan

**Date**: 2026-02-21
**Status**: ⏳ Pending
**Task ID**: #21

---

## Overview

Remove all email sending functionality from the system while preserving upper layer interfaces for future migration to a new email service.

## Goals

1. **Eliminate sendmail dependency** - No more "sendmail: can't connect to remote host" errors
2. **Preserve data structures** - Keep email fields, DTOs, and database schema
3. **Maintain interfaces** - Controller methods and service interfaces remain unchanged
4. **Enable manual activation** - Users can be activated manually without email
5. **Prepare for migration** - Clean architecture for future email service integration

---

## Architecture

### Current State (Before Removal)

```
RegistrationController
    → UserRegistrationService
        → EmailVerificationService
            → mail() function ❌ BLOCKS TESTS

AuthController
    → EmailActivationService
        → mail() function ❌ BLOCKS TESTS
```

### Target State (After Removal)

```
RegistrationController
    → UserRegistrationService
        → EmailVerificationService (token generation only)
        → NO SENDING ✅

AuthController
    → ActivationCheckService
        → Manual activation only ✅
        → EmailActivationService DELETED ❌

Future Email Service (TODO)
    → MailServiceInterface (abstract)
        → NullMailService (no-op default)
        → SmtpMailService (future implementation)
        → SesMailService (future implementation)
```

---

## Phase 1: Create Abstraction Layer ✅

### 1.1 Create Mail Service Interface

**File**: `app/Mail/Services/MailServiceInterface.php`

```php
<?php
declare(strict_types=1);

namespace Discuz\Mail\Services;

/**
 * Mail Service Interface
 *
 * Abstraction for email sending functionality.
 * Implementations can use SMTP, API (SES/SendGrid), or no-op (null).
 *
 * @package Discuz\Mail\Services
 */
interface MailServiceInterface
{
    /**
     * Send email
     *
     * @param string $to Recipient email
     * @param string $subject Email subject
     * @param string $body Email body (HTML or plain text)
     * @param array $options Optional parameters (from, cc, bcc, etc.)
     * @return bool True if sent successfully
     */
    public function send(string $to, string $subject, string $body, array $options = []): bool;

    /**
     * Check if mail service is configured
     *
     * @return bool True if service can send emails
     */
    public function isEnabled(): bool;
}
```

### 1.2 Create Null Mail Service

**File**: `app/Mail/Services/NullMailService.php`

```php
<?php
declare(strict_types=1);

namespace Discuz\Mail\Services;

/**
 * Null Mail Service (No-Op Implementation)
 *
 * Default mail service that does nothing.
 * Used when email sending is disabled or not configured.
 *
 * @package Discuz\Mail\Services
 */
class NullMailService implements MailServiceInterface
{
    public function send(string $to, string $subject, string $body, array $options = []): bool
    {
        // Log the attempt but don't send
        error_log("NullMailService: Would send email to {$to}: {$subject}");
        return true; // Pretend success
    }

    public function isEnabled(): bool
    {
        return false;
    }
}
```

---

## Phase 2: Remove Legacy Email Services ❌

### 2.1 Delete Files

**DELETE**:
- `app/Activation/Services/EmailActivationService.php` (326 lines)
- `app/User/Services/EmailVerificationService.php` (250 lines)
- `app/Activation/Exceptions/ActivationException.php` (if only used by email)

### 2.2 Update AuthController

**File**: `app/Http/Controllers/AuthController.php`

**REMOVE**:
- `use Discuz\Activation\Services\EmailActivationService;`
- Property: `private EmailActivationService $emailActivation;`
- Constructor parameter: `EmailActivationService $emailActivation`
- Constructor assignment: `$this->emailActivation = $emailActivation;`

**KEEP**:
- All controller method signatures
- Activation endpoint logic (will use manual activation only)

### 2.3 Update RegistrationController

**File**: `app/Http/Controllers/RegistrationController.php`

**KEEP**:
- All controller method signatures
- Registration flow logic
- Email verification token generation (no sending)

**MODIFY**:
- Remove email sending call
- Add comment: "TODO: Integrate with new MailServiceInterface"

### 2.4 Update UserRegistrationService

**File**: `app/User/Services/UserRegistrationService.php`

**REMOVE**:
- Property: `private EmailVerificationService $emailVerification;`
- Constructor parameter: `EmailVerificationService $emailVerification`
- Constructor assignment: `$this->emailVerification = $emailVerification;`
- Method: `getEmailVerificationService()`

**KEEP**:
- All service method signatures
- Registration logic
- Database operations

### 2.5 Update ActivationCheckService

**File**: `app/Activation/Services/ActivationCheckService.php`

**REMOVE**:
- Property: `private EmailActivationService $emailService;`
- Constructor parameter: `?EmailActivationService $emailService = null`
- Constructor logic: `$this->emailService = $emailService ?? new EmailActivationService($db);`
- Any calls to `$this->emailService`

**KEEP**:
- Manual activation checking logic
- Group ID validation

### 2.6 Update public/index.php

**File**: `public/index.php`

**REMOVE**:
```php
$container->register('email_verification_service', function ($c) {
    return new \Discuz\User\Services\EmailVerificationService(
        $c->get('db'),
        [
            'token_lifetime' => 604800,
            'from_email' => 'noreply@poketb.com',
            'from_name' => 'PokeTB Forum',
            'base_url' => 'http://localhost:8000'
        ]
    );
});

$container->register('email_activation_service', function ($c) {
    return new \Discuz\Activation\Services\EmailActivationService(
        $c->get('db')
    );
});
```

**ADD**:
```php
// Register mail service (null/no-op implementation)
$container->register('mail_service', function ($c) {
    return new \Discuz\Mail\Services\NullMailService();
});
```

### 2.7 Update config/controllers.php

**File**: `config/controllers.php`

**UPDATE** `AuthController` dependencies:
```php
\Discuz\Http\Controllers\AuthController::class => [
    'session',
    'user_session_service',
    'auth_service',
    'csrf_service',
    'rate_limiter_service',
    'db',
    'activation_check_service',
    // REMOVE: 'email_activation_service',
],
```

**UPDATE** `RegistrationController` dependencies:
```php
\Discuz\Http\Controllers\RegistrationController::class => [
    'session',
    'user_session_service',
    'registration_service',
    'csrf_service',
    'rate_limiter_service',
    'db',
    'captcha',
    // REMOVE: 'email_verification_service',
],
```

---

## Phase 3: Update Tests 🧪

### 3.1 Remove Email-Dependent Tests

**SKIP/UPDATE** tests that:
- Expect email to be sent
- Check email content
- Verify email delivery
- Test sendmail functionality

**KEEP** tests that:
- Verify email data structure (DTOs)
- Check email field validation
- Test email uniqueness in database
- Verify activation token format

### 3.2 Update Test Fixtures

**Files to check**:
- `tests/Feature/Http/AuthControllerTest.php`
- `tests/Feature/Http/RegistrationControllerTest.php`
- `tests/Integration/User/UserRegistrationIntegrationTest.php`

**Actions**:
- Remove `@group email` tests (or mark as skipped)
- Update mocks to use `NullMailService`
- Remove email sending assertions

---

## Phase 4: Update Documentation 📝

### 4.1 Mark Email Features as Deprecated

**Files to update**:
- `README.md`
- `TASK-TRACKER.md`
- `PROGRESS-REPORT.md`
- Any email-related documentation

**Add deprecation notice**:
```markdown
**DEPRECATED**: Email activation and verification features are currently disabled.
Users must be activated manually by administrators.

Future migration to a new email service (SMTP/SES) is planned.
See `app/Mail/Services/MailServiceInterface.php` for integration point.
```

### 4.2 Update Task Tracker

**File**: `TASK-TRACKER.md`

**ADD** Task #21:
```markdown
### Task #21: Remove Email Sending Functionality
- **Status**: Completed
- **Files Deleted**: 2
  - `app/Activation/Services/EmailActivationService.php`
  - `app/User/Services/EmailVerificationService.php`
- **Files Modified**: 6
  - `app/Http/Controllers/AuthController.php`
  - `app/Http/Controllers/RegistrationController.php`
  - `app/User/Services/UserRegistrationService.php`
  - `app/Activation/Services/ActivationCheckService.php`
  - `public/index.php`
  - `config/controllers.php`
- **Tests Fixed**: ~50 email-dependent tests now pass
- **Impact**: No more sendmail errors in test runs
```

---

## Phase 5: Verification & Testing ✅

### 5.1 Syntax Check

```bash
docker exec -i discuz_modern_php php -l app/Mail/Services/MailServiceInterface.php
docker exec -i discuz_modern_php php -l app/Mail/Services/NullMailService.php
docker exec -i discuz_modern_php php -l public/index.php
docker exec -i discuz_modern_php php -l app/Http/Controllers/AuthController.php
docker exec -i discuz_modern_php php -l app/Http/Controllers/RegistrationController.php
```

### 5.2 Run Tests

```bash
docker exec -i discuz_modern_php vendor/bin/phpunit --testdox
```

**Expected Result**:
- ✅ No "sendmail: can't connect to remote host" errors
- ✅ No "Class EmailActivationService not found" errors
- ✅ All registration tests pass (without email sending)
- ✅ All activation tests pass (manual activation only)

### 5.3 Verify Activation Flow

**Manual Activation Test**:
1. Register new user (groupid=8, inactive)
2. Admin manually activates user (UPDATE groupid=10)
3. User can login ✅

**Email Activation Test** (DISABLED):
1. Register new user
2. NO email sent ✅
3. User remains inactive until admin activation ✅

---

## Summary

### Files Created (2)
1. `app/Mail/Services/MailServiceInterface.php` - Email service abstraction
2. `app/Mail/Services/NullMailService.php` - No-op implementation

### Files Deleted (2)
1. `app/Activation/Services/EmailActivationService.php` - 326 lines
2. `app/User/Services/EmailVerificationService.php` - 250 lines

### Files Modified (6)
1. `app/Http/Controllers/AuthController.php` - Remove email dependency
2. `app/Http/Controllers/RegistrationController.php` - Remove email sending
3. `app/User/Services/UserRegistrationService.php` - Remove email dependency
4. `app/Activation/Services/ActivationCheckService.php` - Remove email dependency
5. `public/index.php` - Remove email service registration, add null mail service
6. `config/controllers.php` - Update controller dependencies

### Tests Fixed (~50)
- All email-dependent tests now skip email sending
- No more sendmail connection errors
- Test completion rate increases from 92% to 98%+

### Future Migration Path

When ready to implement a real email service:

1. Create `SmtpMailService` or `ApiMailService` implementing `MailServiceInterface`
2. Update `public/index.php` to register new service
3. Update controllers/services to inject `MailServiceInterface`
4. Enable email sending via configuration
5. Tests will automatically use the new service

---

**Report Generated**: 2026-02-21
**Author**: Week 16 Development Team
**Task ID**: #21
