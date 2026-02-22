# P0 Security Fix - EXPLORATION COMPLETE

## Summary

Comprehensive exploration of the Discuz! migration codebase has been completed. All 3 P0 security vulnerabilities have been **CONFIRMED** and documented.

---

## Confirmed Vulnerabilities

### P0-1: API Routes Lack Authentication - ✅ CONFIRMED

**Evidence**:
- File: `config/routes.php` line 34
- All `/api/v1/*` routes declare `middleware => ['api']`
- File: `config/middleware.php` line 51-54
- 'api' group defined as [AuthenticationMiddleware, CsrfMiddleware]
- **PROBLEM**: Router.php doesn't resolve middleware groups
- **RESULT**: AuthenticationMiddleware NEVER runs

**Impact**:
- 50+ API endpoints publicly accessible
- Anyone can call `POST /api/v1/pm/send` without login
- Private messages, posts, user data exposed

---

### P0-2: Controllers Lack Permission Checks - ✅ CONFIRMED

**Evidence**:
- File: `app/Security/Services/PermissionService.php`
- Comprehensive permission system (60+ permissions) exists
- **PROBLEM**: Controllers don't use PermissionService
- **ROOT CAUSE**: Router.php line 339 creates controllers WITHOUT dependencies
- **RESULT**: Controllers CAN'T check permissions even if they wanted to

**Impact**:
- Any user can delete any thread
- Any user can ban any user
- Non-moderators can moderate
- Non-admins can access admin functions
- Complete privilege escalation

---

### P0-3: CSRF Protection Missing - ✅ CONFIRMED (PARTIALLY)

**Evidence**:
- File: `app/Http/Middleware/CsrfMiddleware.php`
- CSRF middleware fully implemented
- File: `app/Bootstrap/Bootstrap.php` line 122
- CsrfMiddleware applied globally
- **PROBLEM**: CSRF assumes authentication works
- **RESULT**: With P0-1 unfixed, attackers bypass CSRF by calling APIs anonymously

**Impact**:
- Cross-site request forgery possible
- But authentication bypass is higher priority

---

## Root Cause Analysis

### 1. Router Doesn't Resolve Middleware Groups

**File**: `app/Http/Router.php`
**Lines**: 251-258

```php
// Current code (BROKEN)
$routeMiddleware = $route->getMiddleware();
foreach ($routeMiddleware as $middleware) {
    $pipeline->pipe($middleware);  // Just pipes 'api' string!
}
```

**Expected behavior**:
1. Load config/middleware.php
2. Resolve 'api' → [AuthenticationMiddleware::class, CsrfMiddleware::class]
3. Instantiate middleware with dependencies
4. Apply to pipeline

**Actual behavior**:
- Tries to instantiate string 'api' as class
- Fails silently or treats as no-op

---

### 2. Controllers Instantiated Without Dependencies

**File**: `app/Http/Router.php`
**Lines**: 335-340

```php
// Current code (BROKEN)
protected function resolveController(string $controllerClass): object
{
    // TODO: Implement proper DI container
    // For now, just instantiate without constructor arguments
    return new $controllerClass();
}
```

**Problem**:
- Controllers require dependencies (UserService, PermissionService, etc.)
- Created with NO constructor arguments
- Controllers crash or can't function

**Evidence from actual controllers**:
```php
// PostController.php line 92-100
public function __construct(
    ThreadCreationService $threadCreation,
    PostReplyService $postReply,
    ThreadRepository $threadRepository,
    ForumRepository $forumRepository,
    UserRepository $userRepository,
    UserSessionService $userSession,
    CsrfTokenService $csrf
) {
    // ... stores dependencies
}
```

**Result**: Controller can't be created, or created without required services

---

### 3. Manual Authentication in Controllers

**File**: `app/Http/Controllers/PostController.php`
**Lines**: 125-132

```php
// Current code (INADEQUATE)
public function newThreadFormAction(array $params = []): array
{
    // Check authentication
    $userId = $this->userSession->getUserId();
    if ($userId === 0) {
        return ['error' => 'Authentication required'];
    }
    // ... continues
}
```

**Problems**:
- Manual auth checks in EVERY controller method
- Inconsistent across controllers
- Easily forgotten
- No centralized enforcement
- Relies on controllers having dependencies (but they don't!)

---

## Infrastructure Assessment

### Existing Security Components ✅

1. **AuthenticationMiddleware** - Fully implemented
   - Location: `app/Http/Middleware/AuthenticationMiddleware.php`
   - Features: Required/optional modes, user injection
   - Status: Ready to use, just needs to be applied

2. **CsrfMiddleware** - Fully implemented
   - Location: `app/Http/Middleware/CsrfMiddleware.php`
   - Features: Timing-safe comparison, token rotation
   - Status: Applied globally, works correctly

3. **PermissionService** - Comprehensive
   - Location: `app/Security/Services/PermissionService.php`
   - Features: 60+ permission checks, forum-specific, admin checks
   - Status: Complete, just needs to be used

4. **Middleware Configuration** - Well-defined
   - Location: `config/middleware.php`
   - Features: Groups, aliases, priorities
   - Status: Perfect, just needs to be used by Router

---

## Implementation Strategy

### STRICT TDD METHODOLOGY

For EACH phase:

1. **RED**: Write failing test first
   - Test must demonstrate the vulnerability
   - Test must fail before the fix
   - Test name clearly describes what it's testing

2. **GREEN**: Write minimal implementation
   - Make the test pass
   - No extra code
   - Simplest working solution

3. **REFACTOR**: Improve code quality
   - Clean up implementation
   - Follow PSR-12
   - Add documentation

---

## Implementation Phases

### Phase 1: Fix Router Middleware Resolution (Day 1)

**Files to modify**:
- `app/Http/Router.php`
- Create: `tests/Integration/Http/MiddlewareGroupResolutionTest.php`

**Changes**:
1. Load middleware config in Router constructor
2. Create `resolveMiddleware()` method
3. Update `dispatch()` to resolve middleware groups

**Tests**:
- Verify 'api' group resolves to [AuthenticationMiddleware, CsrfMiddleware]
- Verify middleware instantiated with dependencies
- Verify middleware applied in correct order

**Success**: Routes with `middleware => ['api']` get authentication + CSRF

---

### Phase 2: Fix Controller DI Container (Day 2)

**Files to modify**:
- `app/Http/Router.php` (resolveController method)
- `app/Bootstrap/Bootstrap.php` (add services to container)
- Create: `tests/Integration/Http/ControllerDependencyInjectionTest.php`

**Changes**:
1. Create simple DI container in Bootstrap
2. Register all controller dependencies
3. Update Router to resolve dependencies
4. Inject services into controllers

**Tests**:
- Verify controller instantiated with correct dependencies
- Verify PermissionService injected
- Verify UserService injected
- Verify all required services available

**Success**: Controllers created with all required dependencies

---

### Phase 3: Add Permission Checks (Day 3)

**Files to modify**:
- `app/Http/Controllers/PostController.php`
- `app/Http/Controllers/PrivateMessageController.php`
- `app/Http/Controllers/PollController.php`
- `app/Http/Controllers/PaymentController.php`
- `app/Http/Controllers/ProfileController.php`
- Create: `tests/Feature/Security/AuthorizationTest.php`

**Changes**:
1. Add permission checks before sensitive operations
2. Use PermissionService for all authorization
3. Return 403 when permissions denied

**Tests**:
- Regular user can't delete other's threads
- Non-moderator can't moderate
- Non-admin can't access admin functions
- Permission denials return 403

**Success**: All sensitive operations protected by permission checks

---

### Phase 4: Verify CSRF (Day 4)

**Files to modify**:
- No changes expected, just verification
- Create: `tests/Feature/Security/CsrfProtectionTest.php`

**Verification**:
1. Ensure AuthenticationMiddleware runs before CsrfMiddleware
2. Verify CSRF tokens validated
3. Verify tokens included in responses

**Tests**:
- POST without CSRF token returns 403
- POST with valid CSRF token succeeds
- CSRF tokens in response headers

**Success**: CSRF protection working correctly

---

## Test Coverage Requirements

### Minimum Coverage: 90%

**New test files to create**:

1. `tests/unit/Http/MiddlewareGroupResolverTest.php`
2. `tests/unit/Http/ControllerContainerTest.php`
3. `tests/unit/Security/PermissionCheckerTest.php`
4. `tests/Integration/Http/MiddlewarePipelineTest.php`
5. `tests/Integration/Http/ControllerResolutionTest.php`
6. `tests/Integration/Http/AuthenticationFlowTest.php`
7. `tests/Feature/Security/AuthenticationRequiredTest.php`
8. `tests/Feature/Security/AuthorizationEnforcedTest.php`
9. `tests/Feature/Security/CsrfProtectionTest.php`
10. `tests/Feature/Http/Controllers/*AuthTest.php` (all controllers)

---

## Success Criteria

### P0-1: Authentication ✅
- [ ] All /api/v1/* routes return 401 without auth
- [ ] Public routes work without auth
- [ ] AuthenticationMiddleware properly applied
- [ ] Tests pass: 100% authentication tests

### P0-2: Authorization ✅
- [ ] All controller methods check permissions
- [ ] Users can't delete others' threads
- [ ] Non-moderators can't moderate
- [ ] Non-admins can't access admin functions
- [ ] Tests pass: 100% authorization tests

### P0-3: CSRF ✅
- [ ] All state-changing routes require CSRF token
- [ ] Tokens validated on POST/PUT/DELETE
- [ ] Tokens generated in responses
- [ ] Tests pass: 100% CSRF tests

### Quality ✅
- [ ] All tests pass (100% pass rate)
- [ ] Test coverage >90% for new code
- [ ] Code follows PSR-12 standards
- [ ] No PHP warnings/errors
- [ ] All code runs in Docker

---

## Files Analyzed

### Routing Layer
- ✅ `public/index.php` - Entry point
- ✅ `app/Http/Router.php` - Route dispatcher
- ✅ `app/Http/Route.php` - Route entity
- ✅ `app/Http/MiddlewarePipeline.php` - Middleware pipeline
- ✅ `config/routes.php` - Route definitions (667 lines)
- ✅ `config/middleware.php` - Middleware configuration

### Middleware
- ✅ `app/Http/Middleware/AuthenticationMiddleware.php` - Auth middleware
- ✅ `app/Http/Middleware/CsrfMiddleware.php` - CSRF middleware
- ✅ `app/Http/Middleware/RateLimiterMiddleware.php` - Rate limiting

### Controllers (11 files)
- ✅ `app/Http/Controllers/AuthController.php` (374 lines)
- ✅ `app/Http/Controllers/PostController.php` (572 lines)
- ✅ `app/Http/Controllers/PrivateMessageController.php` (623 lines)
- ✅ `app/Http/Controllers/PollController.php` (359 lines)
- ✅ `app/Http/Controllers/PaymentController.php` (485 lines)
- ✅ `app/Http/Controllers/ProfileController.php` (333 lines)
- ✅ `app/Http/Controllers/SearchController.php` (354 lines)
- ✅ `app/Http/Controllers/CreditsController.php` (168 lines)
- ✅ `app/Http/Controllers/FriendsController.php` (130 lines)
- ✅ `app/Http/Controllers/RegistrationController.php` (619 lines)
- ✅ `app/Http/Controllers/AttachmentController.php` (681 lines)

### Security
- ✅ `app/Security/Services/PermissionService.php` (860 lines)
- ✅ `app/Security/Services/PermissionServiceInterface.php`
- ✅ `app/Security/Services/CsrfTokenService.php`

### Bootstrap
- ✅ `app/Bootstrap/Bootstrap.php` - Application bootstrap

### Tests (Existing)
- ✅ 20+ existing test files reviewed
- ✅ Test structure understood

**Total files analyzed**: 40+
**Total lines of code reviewed**: 10,000+

---

## Key Insights

### 1. Good News ✅
- All security components EXIST and are WELL-IMPLEMENTED
- Middleware architecture is sound
- Permission system is comprehensive
- Configuration is well-organized

### 2. Bad News ❌
- Router doesn't use the middleware config
- Controllers can't be instantiated with dependencies
- Security components not wired together

### 3. Root Cause 🎯
**Router is the bottleneck**:
- Doesn't resolve middleware groups
- Doesn't resolve controller dependencies
- Broken since initial implementation

### 4. Solution Path 🛤️
**Fix Router → Fix DI → Add Permissions → Verify CSRF**
- Sequential dependencies
- Each phase enables the next
- STRICT TDD prevents regressions

---

## Next Steps

### ✅ EXPLORATION COMPLETE

### ⏳ READY FOR IMPLEMENTATION

**Phase 1.1 (RED)**: Create first failing test
- File: `tests/Integration/Http/MiddlewareGroupResolutionTest.php`
- Test: Router resolves 'api' middleware group
- Expected: Test fails (Router doesn't resolve groups yet)

**Begin implementation now?** Yes/No

---

**Document Version**: 1.0
**Date**: 2026-02-18
**Status**: EXPLORATION COMPLETE
**Next Phase**: IMPLEMENTATION PHASE 1
