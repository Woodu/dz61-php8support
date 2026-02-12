# File-Level Test Plan - Detailed Testing Roadmap

**Purpose:** File-by-file testing requirements and validation checklists
**Coverage Goal:** Every migrated file has corresponding tests
**Test Types:** Unit, Integration, Regression, Performance

---

## Executive Summary

**Why File-Level Testing Matters:**

When migrating 964 PHP files from PHP 4/5 to PHP 8.3, each file change requires:

1. **Unit Tests** - Verify the migrated code works correctly
2. **Integration Tests** - Verify it works with other components
3. **Regression Tests** - Verify existing features still work
4. **Data Validation** - Verify GBK→UTF-8 data integrity
5. **Performance Tests** - Verify no performance degradation

**This document provides:**
- ✅ File → Test mapping (which files need which tests)
- ✅ Test case templates for common scenarios
- ✅ Sprint-by-sprint testing checklist
- ✅ Regression test suite
- ✅ Test data preparation guide

---

## Part 1: Test Mapping Framework

### File Priority → Test Coverage Matrix

| Priority | Files | Unit Tests | Integration Tests | Regression Tests | Performance Tests |
|-----------|--------|-------------|-------------------|-------------------|-------------------|
| **P0** | 9 files | 100% required | 100% required | 100% required | 80% required |
| **P1** | 22 files | 100% required | 100% required | 100% required | 50% required |
| **P2** | 50 files | 90% required | 70% required | 80% required | 30% required |
| **P3** | 883 files | 70% required | 30% required | 50% required | 10% required |

---

## Part 2: Critical Files - Test Requirements

### 2.1 P0 Files - Complete Test Coverage

#### File: `config.inc.php` → `config/app.php`

**Migration Changes:**
- Array-based config
- Environment variable support
- UTF-8 charset declaration

**Required Tests:**

```php
<?php
// tests/Unit/Config/ConfigTest.php

declare(strict_types=1);

namespace Tests\Unit\Config;

use PHPUnit\Framework\TestCase;

class ConfigTest extends TestCase
{
    public function testConfigLoadsFromArray()
    {
        // Arrange
        $configArray = require __DIR__ . '/config/test.php';

        // Act
        $config = new Config($configArray);

        // Assert
        $this->assertEquals('utf8mb4', $config->get('database.charset'));
        $this->assertEquals('UTF-8', $config->get('app.charset'));
    }

    public function testEnvironmentVariablesOverrideConfig()
    {
        // Arrange
        putenv('DB_HOST=override-host');
        $config = new Config();

        // Act & Assert
        $this->assertEquals('override-host', $config->get('database.host'));
    }

    public function testUtf8CharsetIsSet()
    {
        // Arrange & Act
        $config = new Config();

        // Assert
        $this->assertEquals('UTF-8', $config->get('app.charset'));
    }

    public function testConfigThrowsOnMissingRequiredKey()
    {
        // Arrange
        $config = new Config([]);

        // Assert
        $this->expectException(\RuntimeException::class);
        $config->get('database.host'); // Required key
    }
}
```

**Integration Tests:**
```php
<?php
// tests/Integration/Config/DatabaseConnectionTest.php

class DatabaseConnectionTest extends TestCase
{
    public function testConfigConnectsToDatabase()
    {
        // Use actual config to connect
        $config = new Config(require __DIR__ . '/../../config/app.php');
        $db = new Database($config);

        // Assert connection works
        $this->assertNotNull($db->getConnection());
        $this->assertEquals('utf8mb4', $db->getCharset());
    }
}
```

---

#### File: `include/common.inc.php` → `bootstrap/app.php`

**Migration Changes:**
- Composer autoloader
- Error handling
- UTF-8 initialization
- Session start

**Required Tests:**

```php
<?php
// tests/Integration/Bootstrap/AppTest.php

class AppTest extends TestCase
{
    public function testBootstrapLoadsWithoutErrors()
    {
        // Act
        ob_start();
        $loaded = require __DIR__ . '/../../bootstrap/app.php';
        ob_end_clean();

        // Assert
        $this->assertTrue($loaded);
        $this->assertEquals('UTF-8', mb_internal_encoding());
    }

    public function testAutoloaderWorks()
    {
        // Assert class can be autoloaded
        $this->assertTrue(class_exists(\Discuz\Database\Database::class));
    }

    public function testErrorHandlersRegistered()
    {
        // Assert error handlers are set
        $this->assertTrue(set_error_handler(function() {}) !== null);
    }

    public function testSessionStarts()
    {
        // Act
        session_start();

        // Assert
        $this->assertEquals(PHP_SESSION_ACTIVE, session_status());
    }
}
```

---

#### File: `include/db_mysql.class.php` → `Database/Connection.php`

**Migration Changes:**
- PDO instead of mysql_*
- Prepared statements
- UTF-8 charset
- Exception handling

**Required Tests:**

```php
<?php
// tests/Unit/Database/ConnectionTest.php

class ConnectionTest extends TestCase
{
    private Database $db;

    protected function setUp(): void
    {
        parent::setUp();
        $this->db = $this->createTestDatabaseConnection();
    }

    public function testConnectsToUtf8Database()
    {
        // Assert
        $info = $this->db->getConnection()->getAttribute(PDO::ATTR_SERVER_INFO);
        $this->assertStringContainsString('utf8mb4', $info);
    }

    public function testPreparedStatementsWork()
    {
        // Arrange
        $userId = 1;
        $username = 'test用户';

        // Act
        $stmt = $this->db->prepare('SELECT * FROM cdb_members WHERE uid = ? AND username = ?');
        $stmt->execute([$userId, $username]);
        $result = $stmt->fetch();

        // Assert
        $this->assertNotNull($result);
        $this->assertEquals('test用户', $result['username']);
    }

    public function testTransactionsWork()
    {
        // Act
        try {
            $this->db->beginTransaction();

            $this->db->exec('INSERT INTO cdb_members (username) VALUES ("test1")');
            $this->db->exec('INSERT INTO cdb_members (username) VALUES ("test2")');

            $this->db->commit();
            $committed = true;
        } catch (\Exception $e) {
            $this->db->rollBack();
            $committed = false;
        }

        // Assert
        $this->assertTrue($committed);

        $count = $this->db->query('SELECT COUNT(*) FROM cdb_members WHERE username IN ("test1", "test2")')->fetchColumn();
        $this->assertEquals(2, $count);
    }

    public function testUtf8CharactersStoredCorrectly()
    {
        // Arrange
        $chinese = '测试中文字符串';
        $emoji = '😀🎉🚀';

        // Act
        $stmt = $this->db->prepare('INSERT INTO cdb_posts (message) VALUES (?)');
        $stmt->execute([$chinese . $emoji]);

        // Assert
        $saved = $this->db->query('SELECT message FROM cdb_posts WHERE tid = ' . $this->db->lastInsertId())->fetch();
        $this->assertEquals($chinese . $emoji, $saved['message']);

        // Verify UTF-8 validity
        $this->assertTrue(mb_check_encoding($saved['message'], 'UTF-8'));
    }

    public function testSqlInjectionPrevented()
    {
        // Arrange
        $malicious = "admin' OR '1'='1";
        $password = 'password';

        // Act
        $stmt = $this->db->prepare('SELECT * FROM cdb_members WHERE username = ? AND password = ?');
        $stmt->execute([$malicious, $password]);
        $result = $stmt->fetch();

        // Assert - should return null, not admin user
        $this->assertNull($result);
    }
}
```

---

#### File: `include/global.func.php` → `Utils/GlobalFunctions.php`

**Migration Changes:**
- UTF-8 string functions
- Type hints
- Namespacing

**Required Tests:**

```php
<?php
// tests/Unit/Utils/StringFunctionsTest.php

class StringFunctionsTest extends TestCase
{
    // UTF-8 String Functions
    public function testMbStringLength()
    {
        $chinese = '中文测试';
        $emoji = '😀😁😂';

        $this->assertEquals(4, mb_strlen($chinese, 'UTF-8'));
        $this->assertEquals(3, mb_strlen($emoji, 'UTF-8'));
    }

    public function testMbSubstringDoesNotCutCharacters()
    {
        $string = '测试UTF-8字符';

        // Take first 2 characters
        $substring = mb_substr($string, 0, 2, 'UTF-8');

        $this->assertEquals('测试', $substring);
        $this->assertTrue(mb_check_encoding($substring, 'UTF-8'));
    }

    public function testMbStrposFindsChineseCharacters()
    {
        $string = '搜索关键词';
        $needle = '关键';

        $pos = mb_strpos($string, $needle, 0, 'UTF-8');

        $this->assertEquals(2, $pos);
    }

    public function testMbLowercaseHandlesChinese()
    {
        $mixed = '中文ABC测试';

        $lower = mb_strtolower($mixed, 'UTF-8');

        $this->assertEquals('中文abc测试', $lower);
    }

    public function testStringHelperContains()
    {
        $haystack = '这是一段测试文本';
        $needle = '测试';

        $result = StringHelper::contains($haystack, $needle);

        $this->assertTrue($result);
    }
}
```

---

#### File: `logging.php` → `Auth/AuthService.php`

**Migration Changes:**
- bcrypt password hashing
- JWT tokens
- CSRF protection
- Session management

**Required Tests:**

```php
<?php
// tests/Unit/Auth/AuthServiceTest.php

class AuthServiceTest extends TestCase
{
    private AuthService $auth;
    private Database $db;

    protected function setUp(): void
    {
        parent::setUp();
        $this->db = $this->createTestDatabaseConnection();
        $this->auth = new AuthService($this->db, new Session($this->db));
    }

    public function testLoginWithValidCredentials()
    {
        // Arrange
        $user = $this->createTestUser('testuser', 'password123');
        $user->password = password_hash('password123', PASSWORD_BCRYPT);
        $this->db->save($user);

        // Act
        $result = $this->auth->attempt('testuser', 'password123');

        // Assert
        $this->assertTrue($result->success);
        $this->assertEquals('testuser', $result->user->username);
    }

    public function testLoginWithInvalidCredentials()
    {
        // Act
        $result = $this->auth->attempt('nonexistent', 'wrongpass');

        // Assert
        $this->assertFalse($result->success);
        $this->assertNull($result->user);
    }

    public function testPasswordHashingUsesBcrypt()
    {
        // Arrange
        $password = 'testpassword';

        // Act
        $hash = password_hash($password, PASSWORD_BCRYPT);

        // Assert
        $this->assertTrue(password_verify($password, $hash));
        $this->assertStringStartsWith('$2y$', $hash); // Bcrypt identifier
        $this->assertEquals(60, strlen($hash)); // Bcrypt hash length
    }

    public function testCsrfTokenGenerated()
    {
        // Act
        $token = $this->auth->generateCsrfToken();

        // Assert
        $this->assertNotEmpty($token);
        $this->assertEquals(40, strlen($token)); // SHA1 hex length
    }

    public function testCsrfTokenValidation()
    {
        // Arrange
        $token = $this->auth->generateCsrfToken();

        // Act
        $valid = $this->auth->validateCsrfToken($token);

        // Assert
        $this->assertTrue($valid);
    }

    public function testSessionPersistsAfterLogin()
    {
        // Arrange
        $user = $this->createTestUser('testuser', 'password');
        $this->auth->attempt('testuser', 'password');

        // Act
        $sessionUser = $_SESSION['user'] ?? null;

        // Assert
        $this->assertNotNull($sessionUser);
        $this->assertEquals('testuser', $sessionUser['username']);
    }
}
```

---

### 2.2 P1 Files - Core Feature Tests

#### File: `index.php` → `Controllers/ForumIndexController.php`

**Test Cases:**

```php
<?php
// tests/Integration/Forum/ForumIndexTest.php

class ForumIndexTest extends TestCase
{
    public function testIndexPageLoads()
    {
        // Act
        $response = $this->get('/');

        // Assert
        $this->assertEquals(200, $response->getStatusCode());
        $this->assertStringContainsString('<title>', $response->getContent());
    }

    public function testForumCategoriesDisplayed()
    {
        // Arrange
        $category = $this->createForumCategory('技术讨论', 'discuss');

        // Act
        $response = $this->get('/');

        // Assert
        $this->assertStringContainsString('技术讨论', $response->getContent());
    }

    public function testChineseCharactersDisplayCorrectly()
    {
        // Arrange - Create forum with Chinese name
        $forum = $this->createForum('宝可梦讨论', 'pokemon');

        // Act
        $response = $this->get('/');

        // Assert
        $content = $response->getContent();
        $this->assertStringContainsString('宝可梦讨论', $content);
        $this->assertTrue(mb_check_encoding($content, 'UTF-8'));
    }

    public function testForumStatisticsShown()
    {
        // Arrange
        $this->createForumWithStats('测试版块', 100, 500);

        // Act
        $response = $this->get('/');

        // Assert
        $this->assertStringContainsString('100', $response->getContent()); // Threads
        $this->assertStringContainsString('500', $response->getContent()); // Posts
    }
}
```

---

#### File: `forumdisplay.php` → `Controllers/ForumDisplayController.php`

**Test Cases:**

```php
<?php
// tests/Integration/Forum/ForumDisplayTest.php

class ForumDisplayTest extends TestCase
{
    public function testForumPageLoads()
    {
        // Arrange
        $forum = $this->createForum('测试版块');

        // Act
        $response = $this->get("/forum-{$forum->fid}");

        // Assert
        $this->assertEquals(200, $response->getStatusCode());
    }

    public function testThreadsListedWithPagination()
    {
        // Arrange
        $forum = $this->createForum('分页测试');
        $this->createThreads($forum, 25); // 2 pages

        // Act
        $response = $this->get("/forum-{$forum->fid}");

        // Assert
        $this->assertStringContainsString('Page 1 of 2', $response->getContent());
    }

    public function testStickyThreadsShownFirst()
    {
        // Arrange
        $forum = $this->createForum('置顶测试');
        $normal = $this->createThread($forum, '普通主题');
        $sticky = $this->createThread($forum, '置顶主题', ['sticky' => true]);

        // Act
        $response = $this->get("/forum-{$forum->fid}");
        $content = $response->getContent();

        // Assert - sticky should appear before normal
        $stickyPos = strpos($content, '置顶主题');
        $normalPos = strpos($content, '普通主题');
        $this->assertLessThan($normalPos, $stickyPos);
    }

    public function testPermissionCheckWorks()
    {
        // Arrange
        $forum = $this->createForum('私密版块', ['private' => true]);
        $user = $this->createUser('regularuser');

        // Act
        $response = $this->actingAs($user)->get("/forum-{$forum->fid}");

        // Assert
        $this->assertEquals(403, $response->getStatusCode());
    }
}
```

---

## Part 3: Data Migration Test Plan

### 3.1 GBK→UTF-8 Conversion Tests

**Critical Validation:**

```php
<?php
// tests/Migration/Utf8ConversionTest.php

class Utf8ConversionTest extends TestCase
{
    private Database $oldDb; // GBK database
    private Database $newDb; // UTF-8 database

    public function testChineseCharactersConvertCorrectly()
    {
        // Arrange - GBK data
        $gbkText = '这是GBK编码的文本';

        // Act - Convert
        $utf8Text = mb_convert_encoding($gbkText, 'UTF-8', 'GBK');

        // Assert
        $this->assertEquals('这是GBK编码的文本', $utf8Text);
        $this->assertTrue(mb_check_encoding($utf8Text, 'UTF-8'));
    }

    public function testDatabaseStoresUtf8Correctly()
    {
        // Arrange
        $chinese = '测试中文字符';
        $emoji = '😀🎉';

        // Act
        $this->newDb->insert('cdb_posts', [
            'message' => $chinese . $emoji
        ]);

        // Assert
        $saved = $this->newDb->query('SELECT message FROM cdb_posts WHERE tid = 1')->fetch();
        $this->assertEquals($chinese . $emoji, $saved['message']);
        $this->assertTrue(mb_check_encoding($saved['message'], 'UTF-8'));
    }

    public function testNoDataLossDuringConversion()
    {
        // Arrange - Get old GBK data count
        $oldCount = $this->oldDb->query('SELECT COUNT(*) FROM cdb_members')->fetchColumn();

        // Act - Convert database
        $this->convertDatabaseToUtf8();

        // Assert - Count should be same
        $newCount = $this->newDb->query('SELECT COUNT(*) FROM cdb_members')->fetchColumn();
        $this->assertEquals($oldCount, $newCount, 'Member count mismatch after conversion');
    }

    public function testAllTablesAreUtf8mb4()
    {
        // Act
        $tables = $this->newDb->query('SHOW TABLES')->fetchAll();

        foreach ($tables as $table) {
            $tableName = array_shift($table);
            $status = $this->newDb->query("SHOW TABLE STATUS LIKE '$tableName'")->fetch();

            // Assert
            $this->assertStringContainsString('utf8mb4', $status['Collation'],
                "Table $tableName is not utf8mb4");
        }
    }

    public function testEmojiSupport()
    {
        // Arrange
        $emoji = ['😀', '😎', '🎮', '🚀', '💻'];

        // Act
        foreach ($emoji as $e) {
            $this->newDb->insert('cdb_posts', ['message' => $e]);
        }

        // Assert
        $results = $this->newDb->query('SELECT message FROM cdb_posts WHERE tid >= 1')->fetchAll();
        $this->assertCount(5, $results);

        foreach ($results as $row) {
            $this->assertTrue(mb_check_encoding($row['message'], 'UTF-8'));
            $this->assertNotEmpty($row['message']);
        }
    }
}
```

### 3.2 Data Integrity Tests

**Regression Tests:**

```php
<?php
// tests/Regression/DataIntegrityTest.php

class DataIntegrityTest extends TestCase
{
    public function testUserProfilesIntact()
    {
        // Arrange - Get data from old system
        $oldUser = $this->oldDb->query('SELECT * FROM cdb_members WHERE uid = 1')->fetch();

        // Act - Migrate
        $migrator = new UserMigrator();
        $migrator->migrate(1);

        // Assert - Compare with new system
        $newUser = $this->newDb->query('SELECT * FROM cdb_members WHERE uid = 1')->fetch();

        $this->assertEquals($oldUser['username'], $newUser['username']);
        $this->assertEquals($oldUser['email'], $newUser['email']);
        $this->assertEquals($oldUser['posts'], $newUser['posts']);
    }

    public function testThreadContentPreserved()
    {
        // Arrange
        $oldThread = $this->oldDb->query('SELECT * FROM cdb_threads WHERE tid = 1')->fetch();
        $oldPosts = $this->oldDb->query('SELECT * FROM cdb_posts WHERE tid = 1')->fetchAll();

        // Act - Migrate
        $migrator = new ThreadMigrator();
        $migrator->migrate(1);

        // Assert
        $newThread = $this->newDb->query('SELECT * FROM cdb_threads WHERE tid = 1')->fetch();
        $newPosts = $this->newDb->query('SELECT * FROM cdb_posts WHERE tid = 1')->fetchAll();

        $this->assertEquals($oldThread['subject'], $newThread['subject']);
        $this->assertCount(count($oldPosts), $newPosts);
    }
}
```

---

## Part 4: Sprint-by-Sprint Test Plan

### Sprint 1 Tests (Days 1-5)

**Configuration & UTF-8 Migration**

| File | Unit Tests | Integration Tests | Performance Tests |
|------|-------------|-------------------|-------------------|
| `config/app.php` | ✅ ConfigTest | ✅ DatabaseConnectionTest | ❌ N/A |
| `bootstrap/app.php` | ✅ AppTest | ✅ BootstrapTest | ⏱️ Load time |
| UTF-8 Conversion | ✅ Utf8ConversionTest | ✅ DataIntegrityTest | ⏱️ Conversion time |

**Sprint 1 Test Checklist:**

```bash
#!/bin/bash
# scripts/test-sprint1.sh

echo "=== Sprint 1 Test Suite ==="

# Unit tests
echo "Running unit tests..."
vendor/bin/phpunit tests/Unit/Config/
vendor/bin/phpunit tests/Unit/Bootstrap/
vendor/bin/phpunit tests/Unit/Database/
vendor/bin/phpunit tests/Unit/Migration/

# Integration tests
echo "Running integration tests..."
vendor/bin/phpunit tests/Integration/

# UTF-8 validation
echo "Validating UTF-8 conversion..."
php scripts/validate-utf8.php

# Performance test
echo "Running performance tests..."
ab -n 1000 -c 10 https://staging.example.com/

echo "=== Sprint 1 Tests Complete ==="
```

---

### Sprint 2 Tests (Days 6-10)

**Database Layer**

| File | Unit Tests | Integration Tests |
|------|-------------|-------------------|
| `Database/Connection.php` | ✅ ConnectionTest | ✅ DatabaseTest |
| `Database/QueryBuilder.php` | ✅ QueryBuilderTest | ✅ QueryTest |

**Sprint 2 Test Checklist:**

```bash
#!/bin/bash
# scripts/test-sprint2.sh

echo "=== Sprint 2 Test Suite ==="

# Unit tests
vendor/bin/phpunit tests/Unit/Database/

# Integration tests
vendor/bin/phpunit tests/Integration/Database/

# SQL injection tests
vendor/bin/phpunit tests/Security/SqlInjectionTest.php

# Transaction tests
vendor/bin/phpunit tests/Database/TransactionTest.php

echo "=== Sprint 2 Tests Complete ==="
```

---

## Part 5: Regression Test Suite

### Critical User Journey Tests

**Must pass after every sprint:**

```php
<?php
// tests/Regression/CriticalUserJourneysTest.php

class CriticalUserJourneysTest extends TestCase
{
    public function testUserCanRegister()
    {
        // Act
        $response = $this->post('/register', [
            'username' => 'newuser',
            'email' => 'test@example.com',
            'password' => 'password123'
        ]);

        // Assert
        $this->assertEquals(302, $response->getStatusCode()); // Redirect

        $user = DB::query('SELECT * FROM cdb_members WHERE username = "newuser"')->fetch();
        $this->assertNotNull($user);
    }

    public function testUserCanLogin()
    {
        // Arrange
        $this->createUser('testuser', 'password');

        // Act
        $response = $this->post('/login', [
            'username' => 'testuser',
            'password' => 'password'
        ]);

        // Assert
        $this->assertEquals(302, $response->getStatusCode());
        $this->assertNotNull($_SESSION['user']);
    }

    public function testUserCanViewForum()
    {
        // Arrange
        $forum = $this->createForum('测试版块');

        // Act
        $response = $this->get("/forum-{$forum->fid}");

        // Assert
        $this->assertEquals(200, $response->getStatusCode());
        $this->assertStringContainsString('测试版块', $response->getContent());
    }

    public function testUserCanReadThread()
    {
        // Arrange
        $thread = $this->createThread('测试主题', '这是测试内容');

        // Act
        $response = $this->get("/thread-{$thread->tid}");

        // Assert
        $this->assertEquals(200, $response->getStatusCode());
        $this->assertStringContainsString('测试内容', $response->getContent());
    }

    public function testUserCanPostReply()
    {
        // Arrange
        $user = $this->createUser('poster');
        $thread = $this->createThread('主题');

        // Act
        $response = $this->actingAs($user)->post('/reply', [
            'tid' => $thread->tid,
            'message' => '这是回复内容'
        ]);

        // Assert
        $this->assertEquals(302, $response->getStatusCode());

        $post = DB::query('SELECT * FROM cdb_posts WHERE tid = ' . $thread->tid)->fetch();
        $this->assertEquals('这是回复内容', $post['message']);
    }

    public function testChineseCharactersDisplayCorrectly()
    {
        // Arrange
        $thread = $this->createThread('中文标题', '中文内容😀');

        // Act
        $response = $this->get("/thread-{$thread->tid}");
        $content = $response->getContent();

        // Assert
        $this->assertStringContainsString('中文标题', $content);
        $this->assertStringContainsString('中文内容😀', $content);
        $this->assertTrue(mb_check_encoding($content, 'UTF-8'));
    }
}
```

---

## Part 6: Test Data Management

### Test Data Fixtures

**Location:** `tests/fixtures/`

```php
<?php
// tests/fixtures/users.php

return [
    'admin' => [
        'uid' => 1,
        'username' => 'admin',
        'password' => password_hash('admin123', PASSWORD_BCRYPT),
        'email' => 'admin@example.com',
        'group_id' => 1,
    ],
    'moderator' => [
        'uid' => 2,
        'username' => 'moderator',
        'password' => password_hash('mod123', PASSWORD_BCRYPT),
        'email' => 'mod@example.com',
        'group_id' => 3,
    ],
    'regular_user' => [
        'uid' => 3,
        'username' => 'testuser',
        'password' => password_hash('user123', PASSWORD_BCRYPT),
        'email' => 'user@example.com',
        'group_id' => 10,
    ],
];
```

```php
<?php
// tests/fixtures/forums.php

return [
    'chinese_forum' => [
        'fid' => 1,
        'name' => '宝可梦讨论',
        'description' => '讨论宝可梦相关内容',
        'threads' => 100,
        'posts' => 1000,
    ],
    'technical_forum' => [
        'fid' => 2,
        'name' => '技术交流',
        'description' => 'PHP、MySQL等技术讨论',
        'threads' => 50,
        'posts' => 500,
    ],
];
```

### Test Data Seeding

```php
<?php
// scripts/seed-test-data.php

declare(strict_types=1);

require_once __DIR__ . '/../vendor/autoload.php';

use Discuz\Database\Database;

function seedTestDatabase(Database $db): void
{
    echo "Seeding test database...\n";

    // Truncate existing data
    $db->exec('TRUNCATE TABLE cdb_members');
    $db->exec('TRUNCATE TABLE cdb_forums');
    $db->exec('TRUNCATE TABLE cdb_threads');
    $db->exec('TRUNCATE TABLE cdb_posts');

    // Load fixtures
    $users = require __DIR__ . '/../tests/fixtures/users.php';
    $forums = require __DIR__ . '/../tests/fixtures/forums.php';

    // Insert users
    foreach ($users as $user) {
        $db->insert('cdb_members', $user);
        echo "✅ Created user: {$user['username']}\n";
    }

    // Insert forums
    foreach ($forums as $forum) {
        $db->insert('cdb_forums', $forum);
        echo "✅ Created forum: {$forum['name']}\n";
    }

    echo "Test database seeded successfully!\n";
}

// Usage
$db = new Database(require __DIR__ . '/../config/test.php');
seedTestDatabase($db);
```

---

## Part 7: Test Execution Automation

### Continuous Integration Script

```yaml
# .github/workflows/test.yml

name: Test Suite

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

      - name: Seed test database
        run: php scripts/seed-test-data.php

      - name: Run unit tests
        run: vendor/bin/phpunit --testsuite=Unit

      - name: Run integration tests
        run: vendor/bin/phpunit --testsuite=Integration

      - name: Run migration tests
        run: vendor/bin/phpunit --testsuite=Migration

      - name: Run regression tests
        run: vendor/bin/phpunit --testsuite=Regression

      - name: Check coverage
        run: |
          vendor/bin/phpunit --coverage-html=coverage
          echo "Coverage report generated"

      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: coverage/
```

---

## Part 8: Test Reporting

### Test Report Template

```markdown
# Sprint X Test Report

**Date:** YYYY-MM-DD
**Sprint:** X
**Tester:** Name

## Summary

| Metric | Value |
|--------|--------|
| Total Tests | 150 |
| Passed | 145 |
| Failed | 5 |
| Skipped | 0 |
| Coverage | 85% |

## Failed Tests

1. **testChineseCharactersDisplayCorrectly** (ForumIndexTest)
   - **Error:** Character encoding mismatch
   - **Fix:** Update HTML meta tag to UTF-8

2. **testSqlInjectionPrevented** (ConnectionTest)
   - **Error:** Prepared statement not working
   - **Fix:** Add emulated prepares = false

## Regression Tests

✅ All 20 critical user journeys passed

## Performance Tests

| Endpoint | Before (ms) | After (ms) | Target |
|----------|---------------|--------------|--------|
| Homepage | 450 | 420 | <500 |
| Forum Display | 300 | 280 | <300 |
| Thread View | 400 | 390 | <400 |

## Recommendations

- Fix failed tests before next sprint
- Monitor performance in production
- Add more edge case tests
```

---

## Success Criteria

✅ Every migrated file has corresponding tests
✅ All critical user journeys covered
✅ UTF-8 data integrity verified
✅ Performance benchmarks met
✅ Security tests pass
✅ 80%+ code coverage

---

**Document Status:** ✅ Complete
**Related Documents:**
- 09-testing-strategy.md (General testing approach)
- 06-execution-sprints.md (Sprint planning)
- 07-file-migration-checklist.md (File tracking)
