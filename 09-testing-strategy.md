# Testing Strategy - Comprehensive Quality Assurance Plan

**Purpose:** Ensure quality throughout migration process
**Coverage Goal:** >80% code coverage
**Testing Levels:** Unit, Integration, E2E, Performance

---

## Testing Pyramid

```
         /\
        /  \        E2E Tests (10%)
       /____\       - Critical user flows
      /      \      - Full stack
     /        \     - Slow, fragile
    /__________\
   /            \    Integration Tests (30%)
  /              \   - Feature interactions
 /                \  - Database, API
/__________________\
/                    \ Unit Tests (60%)
- Individual functions
- Fast, isolated
- High coverage
```

---

## Phase 1: Unit Testing (Continuous)

### What to Unit Test

**Database Layer:**
- Connection establishment
- Query execution
- Transaction handling
- Prepared statements
- Error handling

**Authentication:**
- Password hashing
- Token generation
- Session management
- Permission checks

**Utilities:**
- String functions
- Date functions
- Validation functions
- Formatting functions

**Example Unit Test:**

```php
<?php
declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use Discuz\Auth\AuthService;
use Discuz\Database\Database;

class AuthServiceTest extends TestCase
{
    private AuthService $auth;
    private Database $db;

    protected function setUp(): void
    {
        parent::setUp();

        // Use test database
        $this->db = $this->createTestDatabaseConnection();
        $this->auth = new AuthService($this->db, ...);
    }

    public function testLoginWithValidCredentials()
    {
        // Arrange
        $username = 'testuser';
        $password = 'password123';
        $this->createTestUser($username, $password);

        // Act
        $user = $this->auth->attempt($username, $password);

        // Assert
        $this->assertNotNull($user);
        $this->assertEquals($username, $user->username);
    }

    public function testLoginWithInvalidCredentials()
    {
        // Arrange
        $username = 'nonexistent';

        // Act
        $user = $this->auth->attempt($username, 'wrongpass');

        // Assert
        $this->assertNull($user);
    }

    public function testPasswordHashing()
    {
        // Arrange
        $password = 'password123';

        // Act
        $hash = password_hash($password, PASSWORD_BCRYPT);

        // Assert
        $this->assertTrue(password_verify($password, $hash));
        $this->assertNotEquals($password, $hash);
    }

    public function testSessionCreation()
    {
        // Act
        $sessionId = $this->auth->createSession(1);

        // Assert
        $this->assertNotEmpty($sessionId);
        $this->assertEquals(40, strlen($sessionId)); // 40-char hex
    }
}
```

---

## Phase 2: Integration Testing (Each Sprint)

### What to Integration Test

**Feature Interactions:**
- Forum → Thread → Post chain
- User → Authentication → Permission
- Post → Attachment → Download
- Search → Index → Results

**Database Integration:**
- Multi-table transactions
- Foreign key constraints
- Cascade operations
- Data consistency

**API Integration:**
- Request/response flow
- Error handling
- Authentication
- Validation

**Example Integration Test:**

```php
<?php
declare(strict_types=1);

namespace Tests\Integration;

use PHPUnit\Framework\TestCase;
use Discuz\Controllers\ForumDisplayController;
use Discuz\Http\Request;

class ForumIntegrationTest extends TestCase
{
    public function testForumDisplayLoadsThreads()
    {
        // Arrange
        $forum = $this->createTestForum();
        $thread = $this->createTestThread($forum->id);
        $controller = app(ForumDisplayController::class);
        $request = Request::create("/forum-{$forum->id}");

        // Act
        $response = $controller->show($request, $forum->id);

        // Assert
        $this->assertEquals(200, $response->getStatusCode());
        $this->assertStringContainsString($thread->title, $response->getContent());
    }

    public function testNewThreadCreatesPost()
    {
        // Arrange
        $user = $this->createTestUser();
        $forum = $this->createTestForum();
        $request = Request::create('/post', 'POST', [
            'title' => 'Test Thread',
            'content' => 'Test content',
        ]);
        $request->setUserResolver(function () use ($user) {
            return $user;
        });

        // Act
        $controller = app(PostController::class);
        $response = $controller->newThread($request);

        // Assert
        $this->assertEquals(302, $response->getStatusCode()); // Redirect
        $thread = DB::table('threads')->where('title', 'Test Thread')->first();
        $this->assertNotNull($thread);
        $post = DB::table('posts')->where('tid', $thread->tid)->first();
        $this->assertNotNull($post);
    }
}
```

---

## Phase 3: E2E Testing (Each Phase)

### What to E2E Test

**Critical User Journeys:**
1. User registers account
2. User logs in
3. User views forum index
4. User views forum
5. User reads thread
6. User posts reply
7. User edits profile
8. User searches content
9. Moderator closes thread
10. Admin bans user

**Tool:** Playwright or Cypress

**Example E2E Test (Playwright):**

```javascript
// tests/e2e/user-journey.spec.js
import { test, expect } from '@playwright/test';

test.describe('User Journey', () => {
  test('user can register, login, and post', async ({ page }) => {
    // 1. Register
    await page.goto('/register');
    await page.fill('input[name="username"]', 'testuser');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.fill('input[name="password_confirm"]', 'password123');
    await page.click('button[type="submit"]');

    // 2. Login
    await page.goto('/login');
    await page.fill('input[name="username"]', 'testuser');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    // Expect redirect to home
    await expect(page).toHaveURL('/');

    // 3. View forum
    await page.click('text=General Discussion');
    await expect(page).toHaveURL(/forum-\d+/);

    // 4. Create thread
    await page.click('text=New Thread');
    await page.fill('input[name="subject"]', 'Test Thread');
    await page.fill('textarea[name="message"]', 'This is a test thread');
    await page.click('button[type="submit"]');

    // 5. Verify thread created
    await expect(page.locator('h1')).toContainText('Test Thread');
  });

  test('moderator can close thread', async ({ page }) => {
    // Login as moderator
    await page.goto('/login');
    await page.fill('input[name="username"]', 'moderator');
    await page.fill('input[name="password"]', 'password');
    await page.click('button[type="submit"]');

    // Navigate to thread
    await page.goto('/thread-1');

    // Close thread
    await page.click('text=Moderate');
    await page.click('text=Close Thread');
    await page.click('button:has-text("Confirm")');

    // Verify thread is closed
    await expect(page.locator('.thread-status')).toContainText('Closed');
  });
});
```

---

## Phase 4: Performance Testing (End of Each Phase)

### Metrics to Track

**Response Time:**
- Homepage: <500ms
- Forum page: <300ms
- Thread page: <400ms
- Search: <1000ms

**Database Queries:**
- Homepage: <50 queries
- Forum page: <20 queries
- Thread page: <30 queries

**Concurrency:**
- Support 100 concurrent users
- <5% error rate under load

**Performance Test Script:**

```php
<?php
declare(strict_types=1);

use GuzzleHttp\Client;
use GuzzleHttp\Promise;

class PerformanceTest
{
    private Client $client;
    private array $results = [];

    public function __construct()
    {
        $this->client = new Client(['base_uri' => 'http://localhost:8000']);
    }

    public function testHomepageLoadTime(): void
    {
        $times = [];

        for ($i = 0; $i < 100; $i++) {
            $start = microtime(true);
            $response = $this->client->get('/');
            $end = microtime(true);

            $times[] = ($end - $start) * 1000; // Convert to ms
        }

        $average = array_sum($times) / count($times);
        $max = max($times);
        $min = min($times);

        echo "Homepage Load Time:\n";
        echo "  Average: {$average}ms\n";
        echo "  Max: {$max}ms\n";
        echo "  Min: {$min}ms\n";

        $this->results['homepage'] = [
            'average' => $average,
            'max' => $max,
            'min' => $min,
        ];

        $this->assertLessThan(500, $average, 'Homepage should load in <500ms');
    }

    public function testConcurrentUsers(): void
    {
        $concurrency = 100;
        $requests = [];

        for ($i = 0; $i < $concurrency; $i++) {
            $requests[] = $this->client->getAsync('/');
        }

        $start = microtime(true);
        $responses = Promise\Utils::settle($requests)->wait();
        $end = microtime(true);

        $totalTime = ($end - $start) * 1000;
        $successCount = count(array_filter($responses, fn($r) => $r['state'] === 'fulfilled'));

        echo "Concurrency Test:\n";
        echo "  Concurrent Users: $concurrency\n";
        echo "  Total Time: {$totalTime}ms\n";
        echo "  Success Rate: " . ($successCount / $concurrency * 100) . "%\n";

        $this->assertGreaterThan(95, $successCount / $concurrency * 100, 'Success rate should be >95%');
    }
}
```

---

## Phase 5: Security Testing (End of Migration)

### Security Checklist

**SQL Injection:**
- [ ] All queries use prepared statements
- [ ] No direct variable interpolation in SQL
- [ ] Input validation on all user data

**XSS (Cross-Site Scripting):**
- [ ] All output is escaped
- [ ] Content Security Policy implemented
- [ ] No unsafe HTML/JS injection

**CSRF (Cross-Site Request Forgery):**
- [ ] CSRF tokens on all POST requests
- [ ] Token validation
- [ ] SameSite cookie attribute

**Authentication:**
- [ ] Passwords hashed with bcrypt/argon2
- [ ] Secure session management
- [ ] Login rate limiting
- [ ] Secure password reset

**Authorization:**
- [ ] Permission checks on all protected routes
- [ ] No direct object references
- [ ] Role-based access control

**Data Validation:**
- [ ] Input validation on all forms
- [ ] Type checking
- [ ] Length limits
- [ ] Format validation

**Security Test Examples:**

```php
<?php
declare(strict_types=1);

class SecurityTest extends TestCase
{
    public function testSqlInjectionPrevention()
    {
        // Arrange
        $maliciousInput = "admin' OR '1'='1";

        // Act
        $user = DB::table('members')
            ->where('username', $maliciousInput)
            ->first();

        // Assert - should return null, not admin
        $this->assertNull($user);
    }

    public function testXssPrevention()
    {
        // Arrange
        $xssPayload = '<script>alert("XSS")</script>';

        // Act
        $html = view('user.profile', ['bio' => $xssPayload])->render();

        // Assert - script should be escaped
        $this->assertStringNotContainsString('<script>', $html);
        $this->assertStringContainsString('&lt;script&gt;', $html);
    }

    public function testCsrfProtection()
    {
        // Arrange
        $client = $this->createClient();
        $client->request('POST', '/post', [
            'title' => 'Test',
            'content' => 'Test',
            // Missing CSRF token
        ]);

        // Assert - should be rejected
        $this->assertEquals(419, $client->getResponse()->getStatusCode());
    }

    public function testPasswordHashing()
    {
        // Arrange
        $password = 'password123';

        // Act
        $hash = password_hash($password, PASSWORD_BCRYPT);

        // Assert
        $this->assertNotEquals($password, $hash);
        $this->assertTrue(password_verify($password, $hash));
        $this->assertFalse(password_verify('wrongpassword', $hash));
    }
}
```

---

## Continuous Testing

### CI/CD Pipeline

**GitHub Actions Workflow:**

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: discuz_test
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo, pdo_mysql

      - name: Install dependencies
        run: composer install --no-progress

      - name: Run unit tests
        run: vendor/bin/phpunit --testsuite=Unit

      - name: Run integration tests
        run: vendor/bin/phpunit --testsuite=Integration
        env:
          DB_HOST: 127.0.0.1
          DB_NAME: discuz_test

      - name: Run E2E tests
        run: npx playwright test

      - name: Check coverage
        run: |
          vendor/bin/phpunit --coverage-html=coverage
          echo "Coverage report generated"
```

---

## Test Data Management

### Fixtures

**User Fixture:**

```php
<?php
// tests/fixtures/users.php

return [
    'admin' => [
        'username' => 'admin',
        'email' => 'admin@example.com',
        'password' => bcrypt('password'),
        'admin_id' => 1,
        'group_id' => 1,
    ],
    'moderator' => [
        'username' => 'moderator',
        'email' => 'mod@example.com',
        'password' => bcrypt('password'),
        'admin_id' => 2,
        'group_id' => 3,
    ],
    'user' => [
        'username' => 'testuser',
        'email' => 'user@example.com',
        'password' => bcrypt('password'),
        'admin_id' => 0,
        'group_id' => 10,
    ],
];
```

---

### Database Seeding

```php
<?php
declare(strict_types=1);

use Discuz\Database\Database;

class DatabaseSeeder
{
    public function run(Database $db): void
    {
        // Truncate existing data
        $db->table('members')->truncate();
        $db->table('forums')->truncate();

        // Seed users
        $users = require __DIR__ . '/../fixtures/users.php';
        foreach ($users as $user) {
            $db->table('members')->insert($user);
        }

        // Seed forums
        $forums = [
            ['name' => 'General Discussion', 'type' => 'forum'],
            ['name' => 'Off-Topic', 'type' => 'forum'],
        ];
        foreach ($forums as $forum) {
            $db->table('forums')->insert($forum);
        }

        echo "Database seeded successfully!\n";
    }
}
```

---

## Test Documentation

### Test Report Template

**Sprint Test Report:**

```
Sprint: X
Date: YYYY-MM-DD
Tester: Name

Summary:
- Total Tests: 100
- Passed: 95
- Failed: 5
- Skipped: 0

Coverage:
- Code Coverage: 85%
- Feature Coverage: 90%

Failed Tests:
1. testNewThreadWithAttachment - Attachment upload timing issue
2. testSearchChineseCharacters - Fulltext index issue
3. testAdminBanUser - Permission check bug
4. testUserLogout - Session not clearing
5. testConcurrentUsers - Race condition

Bugs Found:
- Critical: 0
- Major: 3
- Minor: 2

Recommendations:
- Fix attachment upload timeout
- Rebuild fulltext indexes
- Review permission system
```

---

## Success Criteria

✅ 80%+ code coverage
✅ All critical tests pass
✅ No critical bugs
✅ Performance targets met
✅ Security audit passed
✅ E2E tests cover all user journeys

---

**Document Status:** ✅ Complete
**Next Document:** 10-deployment-plan.md
