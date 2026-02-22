# User Registration System - Quick Reference

## Quick Start

### 1. Register a New User

```php
use Discuz\User\Services\UserService;
use Discuz\User\DTOs\UserRegistration;

// Create service
$userService = new UserService(
    $userRepository,
    $usernameValidator,
    $passwordStrengthChecker
);

// Register user
$registration = new UserRegistration(
    username: 'testuser',
    email: 'test@example.com',
    password: 'Password123!',
    groupId: 10  // Optional: default 10
);

try {
    $user = $userService->register($registration);
    echo "User registered: ID = " . $user->getUserId();
} catch (UserValidationException $e) {
    echo "Validation failed: " . $e->getFirstError();
}
```

### 2. Validate Username

```php
use Discuz\User\Services\UsernameValidator;
use Discuz\User\Exceptions\UserValidationException;

$validator = new UsernameValidator();

try {
    $validator->validate('testuser');
    echo "Username valid!";
} catch (UserValidationException $e) {
    echo "Error: " . $e->getFirstError();
}

// Check validity without throwing
if ($validator->isValid('testuser')) {
    echo "Username is valid";
}
```

### 3. Hash and Verify Password

```php
use Discuz\User\Services\PasswordStrengthChecker;

$checker = new PasswordStrengthChecker();

// Hash password
$hash = $checker->hash('Password123!');
echo "Hash: $hash"; // $2y$12$...

// Verify password
if ($checker->verify('Password123!', $hash)) {
    echo "Password correct!";
}

// Get strength score
$score = $checker->getStrengthScore('Password123!');
$label = $checker->getStrengthLabel('Password123!');
echo "Strength: $score/4 ($label)";
```

### 4. Check User Existence

```php
use Discuz\User\Repositories\UserRepository;

$repository = new UserRepository($db);

// Check username
if ($repository->existsUsername('testuser')) {
    echo "Username taken!";
}

// Check email
if ($repository->existsEmail('test@example.com')) {
    echo "Email already registered!";
}

// Find user
$user = $repository->findByUsername('testuser');
if ($user !== null) {
    echo "Found: " . $user->getUsername();
}
```

## Validation Rules

### Username (UsernameValidator)
- **Length:** 3-15 characters
- **Characters:** Letters, numbers, underscore (not at start/end)
- **No consecutive underscores:** `test__user` ❌
- **No all numbers:** `12345678` ❌
- **No reserved names:** admin, root, system, etc. ❌
- **UTF-8 supported:** `紫鸢` ✅

### Email (UserService)
- **Format:** Valid email address
- **MX Record:** Domain must exist
- **Max length:** 255 characters

### Password (PasswordStrengthChecker)
- **Min length:** 8 characters
- **Max length:** 72 characters (bcrypt limit)
- **Hashing:** Bcrypt with cost=12 (4096 iterations)
- **Output:** 60 characters starting with `$2y$`

## User Entity Methods

### Getters
```php
$user->getUserId();        // int
$user->getUsername();      // string
$user->getEmail();         // string
$user->getPassword();      // string (bcrypt hash)
$user->getSalt();         // string|null (NULL for new users)
$user->getGroupId();      // int
$user->getCreatedAt();     // int (Unix timestamp)
$user->getUpdatedAt();     // int|null
$user->getLastLoginAt();  // int|null
$user->getStatus();       // int (0=active, 1=pending, 2=suspended, 3=banned)
```

### Status Methods
```php
$user->isActive();      // bool (status === 0)
$user->isPending();     // bool (status === 1)
$user->isSuspended();   // bool (status === 2)
$user->isBanned();      // bool (status === 3)
$user->isMigrated();    // bool (salt !== null)
```

### Serialization
```php
// Convert to array
$data = $user->toArray();

// Convert to JSON (excludes password, salt)
$json = json_encode($user);
```

## User Service Methods

### Registration
```php
$userService->register($registration);  // Returns User entity
```

### User Lookup
```php
$userService->getUserById($userId);         // User|null
$userService->getUserByUsername($username); // User|null
$userService->getUserByEmail($email);       // User|null
```

### User Management
```php
$userService->recordLogin($userId);            // Update last_login_at
$userService->changePassword($userId, $newPassword);
$userService->changeEmail($userId, $newEmail);
$userService->banUser($userId);              // status = 3
$userService->unbanUser($userId);            // status = 0
$userService->suspendUser($userId);          // status = 2
$userService->activateUser($userId);          // status = 0
$userService->deleteUser($userId);            // Delete user
```

### Pagination
```php
// Get all users (page 1, 100 per page)
$users = $userService->getAllUsers(1, 100);

// Get active users (page 1, 50 per page)
$activeUsers = $userService->getActiveUsers(1, 50);

// Get total count
$total = $userService->getTotalUserCount();
```

## Repository Methods

### CRUD
```php
$repository->create($data);              // Returns user ID
$repository->findById($userId);          // Returns User|null
$repository->findByUsername($username); // Returns User|null
$repository->findByEmail($email);       // Returns User|null
$repository->delete($userId);           // Returns bool
```

### Uniqueness Checks
```php
$repository->existsUsername($username); // Returns bool
$repository->existsEmail($email);      // Returns bool
```

### Updates
```php
$repository->updateLastLogin($userId, $timestamp); // bool
$repository->updatePassword($userId, $hash);       // bool
$repository->updateEmail($userId, $email);         // bool
$repository->updateStatus($userId, $status);       // bool
```

### Queries
```php
$repository->getAll($limit, $offset);    // Returns User[]
$repository->getActive($limit, $offset); // Returns User[]
$repository->count();                     // Returns int
```

## Error Handling

### UserValidationException
```php
try {
    $userService->register($registration);
} catch (UserValidationException $e) {
    // Get first error message
    $error = $e->getFirstError();

    // Get all errors
    $errors = $e->getErrors();

    // Get errors for specific field
    $usernameErrors = $e->getErrorsForField('username');

    // Check if field has errors
    if ($e->hasErrorsForField('email')) {
        // Handle email error
    }
}
```

### DatabaseException
```php
use Discuz\Exception\DatabaseException;

try {
    $repository->create($data);
} catch (DatabaseException $e) {
    // Log detailed error (SQL, bindings)
    error_log($e->getDetailedMessage());

    // Show safe error to user
    echo $e->getSafeMessage();
}
```

## Testing

### Run Tests
```bash
# Run all User tests
docker compose exec -T php vendor/bin/phpunit tests/Unit/User/

# Run specific test
docker compose exec -T php vendor/bin/phpunit tests/Unit/User/UserServiceTest.php

# Run with coverage
docker compose exec -T php vendor/bin/phpunit --coverage-html coverage tests/Unit/User/
```

### Test Count
- UserServiceTest: 31 tests
- UsernameValidatorTest: 30+ tests
- PasswordStrengthCheckerTest: 30+ tests
- UserRepositoryNewTest: 25+ tests
- **Total: 120+ tests**

## Database Schema

### Table: users
```sql
CREATE TABLE users (
    user_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL COLLATE utf8mb4_bin,
    email VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    salt CHAR(6) DEFAULT NULL,
    group_id SMALLINT UNSIGNED NOT NULL DEFAULT 10,
    created_at INT UNSIGNED NOT NULL,
    updated_at INT UNSIGNED DEFAULT NULL,
    last_login_at INT UNSIGNED DEFAULT NULL,
    status TINYINT UNSIGNED NOT NULL DEFAULT 0,
    UNIQUE KEY uniq_username (username),
    KEY idx_email (email),
    KEY idx_created_at (created_at),
    KEY idx_status (status),
    KEY idx_group_id (group_id),
    KEY idx_status_created (status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Security Best Practices

### 1. Password Storage
✅ Hash passwords with bcrypt (cost=12)
✅ Never store plain text passwords
✅ Use timing-safe verification (`password_verify()`)
✅ Auto-generated salt (bcrypt internal)

### 2. SQL Injection Prevention
✅ Always use prepared statements
✅ Parameterized queries via PDO
✅ Never concatenate user input into SQL

### 3. Validation
✅ Validate username format (length, characters)
✅ Validate email format + MX record
✅ Check uniqueness (username, email)
✅ Enforce password strength

### 4. Error Handling
✅ Log technical errors (SQL, bindings)
✅ Show safe errors to users (no sensitive data)
✅ Use structured exceptions
✅ Never leak internal details

## Migration Notes

### From uc_members (UCenter)
```php
// Old: UCenter dual MD5+salt
$ucHash = md5(md5($password) . $salt);

// New: Bcrypt
$bcryptHash = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);
```

### From cdb_members (Discuz!)
```php
// Old: Simple MD5
$discuzHash = md5($password);

// New: Bcrypt
$bcryptHash = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);
```

### Migration Strategy
1. Export users from uc_members (authoritative)
2. Export users from cdb_members (fallback)
3. Merge duplicates (UCenter priority)
4. Migrate passwords (verify first, then hash)
5. Import into `users` table (preserve user_id)
6. Update foreign keys in related tables

## Next Steps

1. **Integration Tests:** End-to-end registration flow
2. **API Layer:** REST endpoints for registration
3. **Email Verification:** Send verification emails
4. **Password Reset:** Forgot password workflow
5. **Frontend:** Registration form with validation
