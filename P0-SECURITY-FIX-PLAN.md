# P0 Security Vulnerabilities - Fix Plan

## Executive Summary

This document outlines the STRICT TDD approach to fix ALL P0 security vulnerabilities in the Discuz! migration project.

**Status**: EXPLORATION COMPLETE - Ready for implementation

**Vulnerabilities Confirmed**: 3 CRITICAL P0 issues

---

## 1. EXPLORATION FINDINGS

### 1.1 Current Architecture

**Existing Security Components**:
- ✅ `AuthenticationMiddleware` - Fully implemented (app/Http/Middleware/AuthenticationMiddleware.php)
- ✅ `CsrfMiddleware` - Fully implemented (app/Http/Middleware/CsrfMiddleware.php)
- ✅ `PermissionService` - Comprehensive (60+ permissions) (app/Security/Services/PermissionService.php)
- ✅ Middleware configuration - Well-defined (config/middleware.php)

**Route Configuration** (config/routes.php):
- ✅ All `/api/v1/*` routes declare `middleware => ['api']`
- ✅ `api` group defined as [AuthenticationMiddleware, CsrfMiddleware]

**Middleware Application** (Bootstrap.php):
- ✅ CsrfMiddleware added globally (line 122)
- ✅ RateLimiterMiddleware added globally (line 125)
- ❌ AuthenticationMiddleware NOT added globally

### 1.2 ROOT CAUSE ANALYSIS

**Vulnerability #1: Router Doesn't Resolve Middleware Groups**

File: `app/Http/Router.php`

**Problem**:
- Line 133-169: `group()` method exists but doesn't resolve middleware groups
- Line 251-258: Route middleware extracted but NOT resolved from config
- Routes have `middleware => ['api']` but Router treats it as literal string

**Evidence**:
```php
// Router.php line 251-258
$routeMiddleware = $route->getMiddleware();
foreach ($routeMiddleware as $middleware) {
    $pipeline->pipe($middleware);  // Just pipes 'api' string!
}
```

**Result**: AuthenticationMiddleware NEVER runs because Router doesn't know 'api' = [AuthenticationMiddleware, CsrfMiddleware]

**Vulnerability #2: Controllers Instantiated Without Dependencies**

File: `app/Http/Router.php` line 335-340

**Problem**:
```php
protected function resolveController(string $controllerClass): object
{
    // TODO: Implement proper DI container
    // For now, just instantiate without constructor arguments
    return new $controllerClass();
}
```

**Impact**:
- Controllers require: UserService, PermissionService, etc.
- Created with NO constructor arguments
- Controllers CAN'T check permissions even if code exists!

**Vulnerability #3: Manual Authentication in Controllers**

File: `app/Http/Controllers/PostController.php` line 125-132

**Problem**:
```php
public function newThreadFormAction(array $params = []): array
{
    // Check authentication
    $userId = $this->userSession->getUserId();
    if ($userId === 0) {
        return [
            'success' => false,
            'error' => 'Authentication required',
            'error_code' => 'auth_required'
        ];
    }
    // ... continues execution
}
```

**Issues**:
- Manual auth checks in EVERY controller method
- Inconsistent implementation across controllers
- Can be easily forgotten
- No centralized enforcement
- Relies on controllers having dependencies (but they don't!)

---

## 2. P0 SECURITY VULNERABILITIES

### P0-1: API Routes Lack Authentication - CRITICAL

**Severity**: CRITICAL
**Impact**: Unauthorized access to private messages, posts, user data

**Current State**:
- 50+ API endpoints publicly accessible
- Anyone can call `POST /api/v1/pm/send` without logging in
- Anyone can call `DELETE /api/v1/pm` without authentication
- Private message blacklist endpoints exposed

**Root Cause**:
1. Router doesn't resolve middleware groups ('api')
2. AuthenticationMiddleware not applied to routes
3. Controllers can't be instantiated with dependencies to check auth

**Required Fix**:
1. Fix Router to resolve middleware groups from config/middleware.php
2. Apply AuthenticationMiddleware to all /api/v1/* routes
3. Remove manual auth checks from controllers (middleware handles it)

**Success Criteria**:
- ✅ All /api/v1/* routes return 401 without authentication
- ✅ Public routes (login, register) explicitly marked as optional auth
- ✅ Tests verify 401 response when accessing protected routes anonymously

---

### P0-2: Controllers Lack Permission Checks - CRITICAL

**Severity**: CRITICAL
**Impact**: Privilege escalation, data destruction, unauthorized moderation

**Current State**:
- PermissionService exists but controllers don't use it
- Any authenticated user can delete any thread
- Any user can ban any user
- Non-moderators can access moderation functions
- Non-admins can access admin functions

**Root Cause**:
1. Controllers created without dependencies (Router.php line 339)
2. No PermissionService injection into controllers
3. No permission checks before destructive operations

**Required Fix**:
1. Implement proper DI container for controller instantiation
2. Inject PermissionService into all controllers
3. Add permission checks before sensitive operations:
   - canDeletePost() before deleting
   - canBanUser() before banning
   - canModerate() before moderation actions
   - isAdmin() before admin functions

**Success Criteria**:
- ✅ Users can't delete posts/threads they don't own
- ✅ Non-moderators can't perform moderation actions
- ✅ Non-admins can't access admin functions
- ✅ Tests verify permission denials

---

### P0-3: CSRF Protection Missing - CRITICAL

**Severity**: CRITICAL
**Impact**: Cross-site request forgery attacks

**Current State**:
- CsrfMiddleware exists and is applied globally
- CSRF tokens validated on state-changing requests
- BUT: Authentication bypass makes CSRF ineffective
- Attackers don't need CSRF if they can call APIs anonymously

**Root Cause**:
1. CSRF assumes authentication works
2. With P0-1 unfixed, CSRF is bypassable
3. AuthenticationMiddleware should run BEFORE CsrfMiddleware

**Required Fix**:
1. Ensure AuthenticationMiddleware runs before CsrfMiddleware
2. Verify CSRF tokens are properly validated
3. Ensure CSRF tokens are included in all responses

**Success Criteria**:
- ✅ All POST/PUT/DELETE routes require valid CSRF token
- ✅ CSRF rejection returns 403 when token missing/invalid
- ✅ CSRF acceptance when token valid
- ✅ Tests verify CSRF enforcement

---

## 3. IMPLEMENTATION PLAN (STRICT TDD)

### Phase 1: Fix Router Middleware Resolution

**Order of Implementation**:

#### Step 1.1: Write Test (RED)
Create test: `tests/Integration/Http/MiddlewareGroupResolutionTest.php`

Test should:
- Register a route with `middleware => ['api']`
- Verify middleware groups are resolved from config/middleware.php
- Verify AuthenticationMiddleware and CsrfMiddleware are applied
- **Expected to FAIL** (Router doesn't resolve groups yet)

#### Step 1.2: Implement Fix (GREEN)

File: `app/Http/Router.php`

Changes:
1. Load middleware config in constructor
2. Create `resolveMiddleware()` method
3. Update `group()` to resolve middleware aliases
4. Update `dispatch()` to resolve middleware before piping

```php
// In Router::dispatch()
$middlewareConfig = require CONFIG_PATH . '/middleware.php';

foreach ($routeMiddleware as $mw) {
    // Resolve middleware groups
    if (is_string($mw) && isset($middlewareConfig['groups'][$mw])) {
        foreach ($middlewareConfig['groups'][$mw] as $groupMw) {
            $pipeline->pipe($groupMw);
        }
    } else {
        $pipeline->pipe($mw);
    }
}
```

#### Step 1.3: Refactor
- Extract middleware resolution to separate method
- Add error handling for missing middleware groups
- Add logging for middleware resolution

---

### Phase 2: Fix Controller Instantiation with DI

#### Step 2.1: Write Test (RED)
Create test: `tests/Integration/Http/ControllerDependencyInjectionTest.php`

Test should:
- Define a controller with required dependencies
- Route to that controller
- Verify controller is instantiated with dependencies
- **Expected to FAIL** (Router creates controllers without deps)

#### Step 2.2: Implement Fix (GREEN)

File: `app/Http/Router.php`

Create simple DI container:

```php
protected function resolveController(string $controllerClass): object
{
    // Get service container from bootstrap
    global $app;  // Temporary solution

    // Define controller dependencies
    $dependencies = [
        PostController::class => [
            ThreadCreationService::class,
            PostReplyService::class,
            // ... etc
        ],
    ];

    if (!isset($dependencies[$controllerClass])) {
        throw new RuntimeException("Controller not configured: {$controllerClass}");
    }

    // Resolve dependencies from service container
    $args = [];
    foreach ($dependencies[$controllerClass] as $depClass) {
        if (isset($app[strtolower($depClass)]) ||
            isset($app['container'][$depClass])) {
            $args[] = $app[strtolower($depClass)] ?? $app['container'][$depClass];
        } else {
            throw new RuntimeException("Service not found: {$depClass}");
        }
    }

    return new $controllerClass(...$args);
}
```

File: `app/Bootstrap/Bootstrap.php`

Add services to container:
```php
self::$app['container'][PermissionService::class] = new PermissionService(
    self::$app['database'],
    $user,  // Need to inject current user
    self::$app['cache']
);
```

#### Step 2.3: Refactor
- Implement proper PSR-11 DI container
- Use autowiring for constructor parameters
- Add service provider pattern

---

### Phase 3: Add Permission Checks to Controllers

#### Step 3.1: Write Test (RED)
Create test: `tests/Feature/Security/AuthorizationTest.php`

Test should:
- Create two users: regular_user and moderator
- Login as regular_user
- Attempt to delete another user's thread
- **Expected to FAIL** (no permission checks yet)

#### Step 3.2: Implement Fix (GREEN)

File: `app/Http/Controllers/PostController.php`

Add permission checks:

```php
public function newThreadAction(array $params = []): array
{
    // Get user from request (injected by AuthenticationMiddleware)
    $user = $this->request->user;
    $permissionService = new PermissionService(
        $this->database,
        $user,
        $this->cache
    );

    // Check forum permission
    $forumId = (int)($params['forum_id'] ?? 0);
    if (!$permissionService->canPostInForum($forumId)) {
        return [
            'success' => false,
            'error' => 'Permission denied',
            'error_code' => 'permission_denied'
        ];
    }

    // ... continue with thread creation
}
```

Repeat for all sensitive operations in all controllers:
- PostController: canPostInForum(), canReplyInForum()
- PrivateMessageController: canTransferCredits() (for blacklist)
- PollController: canVote(), canPostPoll()
- PaymentController: Check user permissions
- ProfileController: canViewProfile(), canEditUser()

#### Step 3.3: Refactor
- Create base Controller class with permission checking methods
- Extract permission logic to helper methods
- Add authorization exceptions

---

### Phase 4: Verify CSRF Protection

#### Step 4.1: Write Test (RED)
Create test: `tests/Feature/Security/CsrfProtectionTest.php`

Test should:
- Login as user
- Attempt POST request without CSRF token
- **Expect to FAIL** (should get 403 but might not)

#### Step 4.2: Verify Fix (GREEN)

Ensure:
1. AuthenticationMiddleware runs before CsrfMiddleware
2. CsrfMiddleware properly validates tokens
3. CSRF tokens included in all responses

#### Step 4.3: Refactor
- Verify token rotation is working
- Add exception for safe methods (GET, HEAD, OPTIONS)
- Document CSRF workflow

---

## 4. ORDER OF IMPLEMENTATION

**Sequential Dependencies**:
1. Phase 1 (Router) → Unblocks Phase 2 (DI)
2. Phase 2 (DI) → Unblocks Phase 3 (Permissions)
3. Phase 3 (Permissions) → Enables Phase 4 (CSRF verification)

**Implementation Timeline**:
- Day 1: Phase 1 (Router middleware resolution)
- Day 2: Phase 2 (Controller DI container)
- Day 3: Phase 3 (Permission checks in controllers)
- Day 4: Phase 4 (CSRF verification) + Final testing

**Total Estimated Time**: 4 days

---

## 5. TEST COVERAGE REQUIREMENTS

### Minimum Coverage: 90%

#### Unit Tests (tests/unit/Security/):
- MiddlewareGroupResolverTest.php
- ControllerContainerTest.php
- PermissionCheckerTest.php

#### Integration Tests (tests/Integration/Http/):
- MiddlewarePipelineTest.php
- ControllerResolutionTest.php
- AuthenticationFlowTest.php

#### Feature Tests (tests/Feature/Security/):
- AuthenticationRequiredTest.php
- AuthorizationEnforcedTest.php
- CsrfProtectionTest.php

#### Controller Tests (tests/Feature/Http/Controllers/):
- PostControllerAuthTest.php
- PrivateMessageControllerAuthTest.php
- All controllers *AuthTest.php

---

## 6. SUCCESS CRITERIA

### P0-1: Authentication
- ✅ All /api/v1/* routes return 401 without auth
- ✅ Public routes work without auth
- ✅ AuthenticationMiddleware properly applied
- ✅ Tests pass: 100% authentication tests passing

### P0-2: Authorization
- ✅ All controller methods check permissions
- ✅ Users can't delete threads they don't own
- ✅ Non-moderators can't moderate
- ✅ Non-admins can't access admin functions
- ✅ Tests pass: 100% authorization tests passing

### P0-3: CSRF
- ✅ All state-changing routes require CSRF token
- ✅ Tokens validated on every POST/PUT/DELETE
- ✅ Tokens generated and included in responses
- ✅ Tests pass: 100% CSRF tests passing

### Quality
- ✅ All tests pass (100% pass rate)
- ✅ Test coverage >90% for new code
- ✅ Code follows PSR-12 standards
- ✅ No PHP warnings/errors
- ✅ All code runs in Docker container

---

## 7. RISK MITIGATION

### Potential Issues:
1. **Breaking existing functionality**: Comprehensive test suite prevents regressions
2. **Performance impact**: Middleware overhead is minimal (<5ms)
3. **Service container complexity**: Start with simple array-based container, upgrade to PSR-11 later
4. **Circular dependencies**: Clear dependency graph, lazy resolution where needed

### Rollback Plan:
- All changes in version control
- Each phase independently revertable
- Feature flags for enabling/disabling fixes
- Comprehensive backup before deployment

---

## 8. NEXT STEPS

1. ✅ Exploration complete
2. ✅ Implementation plan documented
3. ⏳ **BEGIN IMPLEMENTATION** → Start Phase 1

**First Action**: Create Phase 1.1 test file (RED phase)

---

**Document Version**: 1.0
**Last Updated**: 2025-02-18
**Status**: Ready for Implementation
