# Configuration System - Quick Reference

## Overview

Modern PHP 8.3 configuration system replacing legacy `config.inc.php`.

## Usage

### Loading Configuration

```php
use Discuz\Config\Config;
use Dotenv\Dotenv;

// Load .env file
$dotenv = Dotenv::createImmutable(__DIR__ . '/..');
$dotenv->safeLoad();

// Load config
$configArray = require __DIR__ . '/../config/app.php';
$config = new Config($configArray);
```

### Accessing Values

```php
// Get a value
$debug = $config->get('app.debug'); // true
$host = $config->get('database.host'); // 'mysql'

// Get with default
$value = $config->get('nonexistent.key', 'default'); // 'default'

// Check if key exists
if ($config->has('app.debug')) { ... }

// Set runtime value (doesn't persist)
$config->set('app.debug', false);

// Get all config
$all = $config->all();
```

## Configuration Structure

```
config/
└── app.php                 # Main configuration file
    ├── app                 # Application settings
    ├── database            # Database connection
    ├── session            # Session configuration
    ├── cache              # Cache configuration
    ├── security           # Security settings
    ├── ucenter            # UCenter integration
    └── logging            # Logging configuration
```

## Key Configuration Items

### Application

```php
$config->get('app.name');           // 'Discuz! Board'
$config->get('app.charset');         // 'UTF-8' (enforced)
$config->get('app.env');            // 'development' | 'production'
$config->get('app.debug');          // true | false
$config->get('app.founders');       // [1, 2, 3, 18]
$config->get('app.admin_email');    // 'admin@poketb.cn'
```

### Database

```php
$config->get('database.host');        // 'mysql'
$config->get('database.port');        // 3306
$config->get('database.database');    // 'discuz_utf8'
$config->get('database.username');    // 'root'
$config->get('database.password');    // from .env
$config->get('database.charset');     // 'utf8mb4' (enforced)
$config->get('database.prefix');     // 'cdb_'
```

### Session

```php
$config->get('session.driver');           // 'file' | 'database'
$config->get('session.lifetime');        // 120 (minutes)
$config->get('session.cookie');           // 'discuz_session'
$config->get('session.cookie_prefix');    // 'discuz_'
$config->get('session.cookie_path');      // '/'
$config->get('session.cookie_domain');    // null
$config->get('session.cookie_http_only'); // true (security)
$config->get('session.cookie_same_site'); // 'lax' (security)
```

### Security

```php
$config->get('security.attack_evasive');  // 0-7 (attack defense level)
$config->get('security.cookie_prefix');  // 'discuz_'
$config->get('security.csrf_token_name'); // 'csrf_token'
```

## Environment Variables

### Required (.env)

```bash
DB_HOST=mysql
DB_NAME=discuz_utf8
DB_USER=root
DB_PASS=discuz_root_pass
```

### Optional (.env)

```bash
# Application
APP_NAME="Discuz! Board"
APP_ENV=development
APP_DEBUG=true

# Database
DB_CHARSET=utf8mb4
DB_PREFIX=cdb_

# Session
SESSION_DRIVER=database
SESSION_LIFETIME=120

# Security
ATTACK_DEFENSIVE=true
ATTACK_LEVEL=1

# Founders
FORUM_FOUNDERS=1,2,3,18
```

## Validation

### Required Keys

Config enforces these top-level keys:
- `app`
- `database`
- `session`
- `security`

### Database Validation

```php
// Required: host, database, username
// Enforced: charset = 'utf8mb4'
```

### Application Validation

```php
// Enforced: charset = 'UTF-8'
```

## Migration: Legacy → Modern

| Legacy | Modern | Notes |
|--------|--------|-------|
| `$dbhost` | `database.host` | String |
| `$charset` | `app.charset` | 'UTF-8' (was 'GBK') |
| `$dbcharset` | `database.charset` | 'utf8mb4' (was '') |
| `$forumfounders` | `app.founders` | Array (was string) |
| `$cookiepre` | `session.cookie_prefix` | String |
| `$attackevasive` | `security.attack_evasive` | Integer 0-7 |

## Helper Functions

```php
use function Discuz\Support\storage_path;
use function Discuz\Support\config_path;
use function Discuz\Support\app_path;

storage_path('logs/app.log');    // /path/to/storage/logs/app.log
config_path('app.php');          // /path/to/config/app.php
app_path('Config/Config.php');   // /path/to/app/Config/Config.php
```

## Testing

### Run Unit Tests

```bash
# Using PHP container
docker run --rm \
  -v /root/poketb-renew/modern-php-migration-code:/app \
  -w /app \
  php:8.3-cli \
  /app/vendor/bin/phpunit tests/Unit/Config/ConfigTest.php
```

### Run Database Connection Test

```bash
# Using PHP container with PDO MySQL
docker run --rm \
  -v /root/poketb-renew/modern-php-migration-code:/app \
  -w /app \
  --network modern-php-migration-code_discuz_modern \
  php:8.3-cli \
  sh -c "docker-php-ext-install pdo pdo_mysql && php tests/database-connection-test.php"
```

## Best Practices

1. **Always use Config class** - Don't access `$_ENV` directly
2. **Use dot notation** - `database.host` not nested array access
3. **Provide defaults** - `get('key', 'default')` for optional values
4. **Never hardcode credentials** - Use .env file
5. **Validate on load** - Config validates required keys on construction
6. **Type-safe access** - Use proper types (bool, int, array)

## Security Considerations

1. **UTF-8 Enforcement**
   - App charset: `UTF-8`
   - Database charset: `utf8mb4`
   - Validated on load

2. **Cookie Security**
   - `http_only`: true (prevents XSS)
   - `same_site`: 'lax' (CSRF protection)
   - `secure`: false (dev) / true (prod)

3. **Attack Evasion**
   - Levels 0-7 (0 = disabled, 7 = maximum)
   - Configurable via `ATTACK_DEFENSIVE` and `ATTACK_LEVEL`

4. **Credentials**
   - Never in code
   - Always in .env
   - .env never committed to git

## Troubleshooting

### Config throws "Missing required configuration"

**Cause**: Required top-level key missing
**Fix**: Ensure `app`, `database`, `session`, `security` keys exist in config/app.php

### Config throws "Database charset must be 'utf8mb4'"

**Cause**: Database charset is not utf8mb4
**Fix**: Set `DB_CHARSET=utf8mb4` in .env

### Config throws "Application charset must be 'UTF-8'"

**Cause**: App charset is not UTF-8
**Fix**: Do not change app.charset (enforced for migration)

### Database connection fails

**Cause**: Wrong host credentials
**Fix**: Verify DB_HOST, DB_NAME, DB_USER, DB_PASS in .env

### UTF-8 data shows garbled

**Cause**: Database table not utf8mb4
**Fix**: Run migration: `ALTER TABLE cdb_members CONVERT TO CHARACTER SET utf8mb4`

## Examples

### Example 1: Connect to Database

```php
$config = new Config(require __DIR__ . '/config/app.php');

$pdo = new PDO(
    sprintf(
        'mysql:host=%s;port=%d;dbname=%s;charset=%s',
        $config->get('database.host'),
        $config->get('database.port'),
        $config->get('database.database'),
        $config->get('database.charset')
    ),
    $config->get('database.username'),
    $config->get('database.password'),
    [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8mb4",
    ]
);
```

### Example 2: Check if User is Founder

```php
$userId = 4; // Current user ID
$founders = $config->get('app.founders');

if (in_array($userId, $founders)) {
    // User is founder - grant full access
}
```

### Example 3: Get Session Cookie Name

```php
session_name(
    $config->get('session.cookie_prefix') .
    $config->get('session.cookie')
);
```

## See Also

- `/docs/legacy-analysis/config-analysis.md` - Legacy config analysis
- `/DAY2-SUMMARY.md` - Day 2 migration summary
- `/app/Config/Config.php` - Config class source
- `/tests/Unit/Config/ConfigTest.php` - Unit tests
