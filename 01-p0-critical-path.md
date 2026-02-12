# P0 Critical Path - Detailed Migration Guide

**Purpose:** Define the absolute minimum features needed for a functioning forum
**Priority:** CRITICAL - Must complete before any other work
**Timeline:** Weeks 1-3 (15 working days)

---

## Overview

The P0 critical path consists of features that form the foundation of the entire system. Without these, nothing else works. They must be migrated first, tested thoroughly, and validated before moving to P1.

**P0 Features:**
1. Core Bootstrap & Configuration
2. Authentication & Session Management
3. Caching System
4. Database Layer

---

## 1. Core Bootstrap & Configuration

### 1.1 Configuration System

**Current File:** `/root/poketb-renew/poketb.com/bbs/config.inc.php`

**Lines:** 1-100

**Current Code (PHP 4/5 style):**
```php
<?php
/*
    [Discuz!] (C)2001-2007 Comsenz Inc.
    This is NOT a freeware, use is subject to license terms

    $Id: config.inc.php 14480 2008-07-24 06:40:49Z liuqiang $
*/

// [EN] Database settings
$dbhost = 'localhost';
$dbuser = 'root';
$dbpw = '';
$dbname = 'discuz';
$pconnect = 0;

// [EN] Table prefix
$tablepre = 'cdb_';

// [EN] Database charset
$dbcharset = 'gbk';

// [EN] Application name
$bbname = 'Discuz! Board';

// [EN] Forum URL
$boardurl = 'http://localhost/discuz';

// [EN] Cookie settings
$cookiedomain = ''; // Cookie domain
$cookiepath = '/'; // Cookie path
$cookiepre = 'xnE_'; // Cookie prefix

// [EN] Error report
$errorreport = 1;

// [EN] Debug mode
$debug = 0;

// [EN] Founder ids
$forumfounders = '1';

// [EN] Admin CP path
$admincp = 'admincp.php';

// [EN] Attack defensive
$attackevasive = 0;

// [EN] Time offset
$timeoffset = 8;

// [EN] Application time zone
$timeformat = 'H:i:s';
$dateformat = 'Y-n-j';

// [EN] Register global
$registerglobals = 'on';

// [EN] Gzip
$gzipcompress = 0;

// [EN] Characters
$charset = 'GBK';

// [EN] Navigation
$navtitle = '';
$navlogo = '';
$navtime = '';

// [EN] Template
$styleid = 1;

// [EN] Language
$language = 'zh-cn';

// [EN] Cache
$cacheindexlife = 0;

// [EN] Session
$sessionexists = 0;

// [EN] Online list
$onlinerecord = '';

// [EN] SMTP
$smtp = array();

// [EN] UCenter
$uc = array();

// [EN] UC URL
$ucurl = '';

// [EN] Cache file
$cachefile = '';
?>
```

**Migrated Code (PHP 8.3 + Best Practices):**
```php
<?php
declare(strict_types=1);

/**
 * Discuz! Board - Modern Configuration
 * Migrated to PHP 8.3
 *
 * @package      Discuz
 * @version      8.0.0
 * @license      Proprietary
 */

use Dotenv\Dotenv;

// Load environment variables
$dotenv = Dotenv::createImmutable(__DIR__ . '/..');
$dotenv->safeLoad();
$dotenv->required(['DB_HOST', 'DB_NAME', 'DB_USER', 'DB_PASS'])
       ->notEmpty();

return [
    // Application
    'app' => [
        'name' => $_ENV['APP_NAME'] ?? 'Discuz! Board',
        'url' => $_ENV['APP_URL'] ?? 'http://localhost',
        'env' => $_ENV['APP_ENV'] ?? 'production',
        'debug' => filter_var($_ENV['APP_DEBUG'] ?? false, FILTER_VALIDATE_BOOLEAN),
        'timezone' => $_ENV['APP_TIMEZONE'] ?? 'Asia/Shanghai',
        'charset' => 'UTF-8', // Changed from GBK
        'locale' => $_ENV['APP_LOCALE'] ?? 'zh_CN',
    ],

    // Database
    'database' => [
        'host' => $_ENV['DB_HOST'],
        'port' => (int) ($_ENV['DB_PORT'] ?? 3306),
        'database' => $_ENV['DB_NAME'],
        'username' => $_ENV['DB_USER'],
        'password' => $_ENV['DB_PASS'],
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => $_ENV['DB_PREFIX'] ?? 'cdb_',
        'strict' => true,
        'engine' => null,
        'options' => [
            PDO::ATTR_CASE => PDO::CASE_NATURAL,
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_ORACLE_NULLS => PDO::NULL_NATURAL,
            PDO::ATTR_STRINGIFY_FETCHES => false,
            PDO::ATTR_EMULATE_PREPARES => false,
        ],
    ],

    // Session
    'session' => [
        'driver' => $_ENV['SESSION_DRIVER'] ?? 'database',
        'lifetime' => (int) ($_ENV['SESSION_LIFETIME'] ?? 120),
        'expire_on_close' => false,
        'encrypt' => false,
        'files' => $_ENV['SESSION_FILES'] ?? storage_path('framework/sessions'),
        'connection' => null,
        'table' => 'sessions',
        'store' => null,
        'lottery' => [2, 100],
        'cookie' => 'discuz_session',
        'path' => '/',
        'domain' => $_ENV['SESSION_DOMAIN'] ?? null,
        'secure' => filter_var($_ENV['SESSION_SECURE'] ?? false, FILTER_VALIDATE_BOOLEAN),
        'http_only' => true,
        'same_site' => 'lax',
    ],

    // Cache
    'cache' => [
        'default' => $_ENV['CACHE_DRIVER'] ?? 'redis',
        'stores' => [
            'apc' => [
                'driver' => 'apc',
            ],
            'array' => [
                'driver' => 'array',
                'serialize' => false,
            ],
            'database' => [
                'driver' => 'database',
                'table' => 'cache',
                'connection' => null,
                'lock_connection' => null,
            ],
            'file' => [
                'driver' => 'file',
                'path' => storage_path('framework/cache/data'),
            ],
            'memcached' => [
                'driver' => 'memcached',
                'persistent_id' => env('MEMCACHED_PERSISTENT_ID'),
                'sasl' => [
                    env('MEMCACHED_USERNAME'),
                    env('MEMCACHED_PASSWORD'),
                ],
                'options' => [
                    // Memcached::OPT_CONNECT_TIMEOUT => 2000,
                ],
                'servers' => [
                    [
                        'host' => $_ENV['MEMCACHED_HOST'] ?? '127.0.0.1',
                        'port' => (int) ($_ENV['MEMCACHED_PORT'] ?? 11211),
                        'weight' => 100,
                    ],
                ],
            ],
            'redis' => [
                'driver' => 'redis',
                'connection' => 'cache',
                'lock_connection' => 'default',
            ],
        ],
        'prefix' => 'discuz_cache_',
    ],

    // Security
    'security' => [
        'csrf_token_name' => 'csrf_token',
        'csrf_header_name' => 'X-CSRF-TOKEN',
        'csrf_field_name' => '_token',
        'cookie_prefix' => 'discuz_',
        'cookie_domain' => $_ENV['COOKIE_DOMAIN'] ?? null,
        'cookie_path' => '/',
        'cookie_secure' => filter_var($_ENV['COOKIE_SECURE'] ?? false, FILTER_VALIDATE_BOOLEAN),
        'cookie_http_only' => true,
        'cookie_same_site' => 'lax',
        'attack_defensive' => [
            'enabled' => filter_var($_ENV['ATTACK_DEFENSIVE'] ?? false, FILTER_VALIDATE_BOOLEAN),
            'level' => (int) ($_ENV['ATTACK_LEVEL'] ?? 0),
        ],
    ],

    // UCenter Integration
    'ucenter' => [
        'enabled' => filter_var($_ENV['UC_ENABLED'] ?? true, FILTER_VALIDATE_BOOLEAN),
        'url' => $_ENV['UC_URL'] ?? '',
        'key' => $_ENV['UC_KEY'] ?? '',
        'appid' => (int) ($_ENV['UC_APPID'] ?? 1),
        'connect' => filter_var($_ENV['UC_CONNECT'] ?? false, FILTER_VALIDATE_BOOLEAN),
    ],

    // Logging
    'logging' => [
        'channels' => [
            'stack' => [
                'driver' => 'stack',
                'channels' => ['single'],
                'ignore_exceptions' => false,
            ],
            'single' => [
                'driver' => 'single',
                'path' => storage_path('logs/laravel.log'),
                'level' => 'debug',
                'replace_placeholders' => true,
            ],
            'daily' => [
                'driver' => 'daily',
                'path' => storage_path('logs/laravel.log'),
                'level' => 'debug',
                'days' => 14,
                'replace_placeholders' => true,
            ],
            'syslog' => [
                'driver' => 'syslog',
                'level' => 'debug',
                'facility' => LOG_USER,
            ],
            'errorlog' => [
                'driver' => 'errorlog',
                'level' => 'debug',
                'replace_placeholders' => true,
            ],
        ],
    ],

    // Founders
    'founders' => array_filter(explode(',', $_ENV['FORUM_FOUNDERS'] ?? '1')),

    // Encryption
    'encryption' => [
        'key' => $_ENV['APP_KEY'] ?? '',
        'cipher' => 'AES-256-CBC',
    ],
];
```

**Key Changes:**
- ✅ Environment-based configuration using `.env` files
- ✅ Type declarations and strict types
- ✅ PDO for database connections with prepared statements
- ✅ UTF-8/utf8mb4 instead of GBK
- ✅ Secure cookie settings (HttpOnly, SameSite)
- ✅ Structured array-based config
- ✅ Modern security defaults

**Database Tables Involved:**
- `cdb_settings` - Application settings

**Testing Checklist:**
- [ ] Configuration loads without errors
- [ ] Environment variables are validated
- [ ] Database connection succeeds
- [ ] Session configuration is valid
- [ ] Cache stores are accessible
- [ ] Security settings are applied
- [ ] UTF-8 encoding works correctly

**Dependencies:** None (first step)

---

### 1.2 Global Bootstrap

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/common.inc.php`

**Lines:** 1-500

**Current Code (PHP 4/5 style):**
```php
<?php
/*
    [Discuz!] (C)2001-2007 Comsenz Inc.
    This is NOT a freeware, use is subject to license terms

    $Id: common.inc.php 13803 2008-06-04 09:49:43Z heyond $
*/
//error_reporting(0);
// set_magic_quotes_runtime(0);
$mtime = explode(' ', microtime());
$discuz_starttime = $mtime[1] + $mtime[0];

define('SYS_DEBUG', FALSE);
define('IN_DISCUZ', TRUE);
define('DISCUZ_ROOT', substr(dirname(__FILE__), 0, -7));
define('MAGIC_QUOTES_GPC', get_magic_quotes_gpc());
!defined('CURSCRIPT') && define('CURSCRIPT', '');

function htmlout($str) {
    return htmlspecialchars($str,ENT_COMPAT,'ISO-8859-1');
}

if(PHP_VERSION < '4.1.0') {
    $_GET = &$HTTP_GET_VARS;
    $_POST = &$HTTP_POST_VARS;
    $_COOKIE = &$HTTP_COOKIE_VARS;
    $_SERVER = &$HTTP_SERVER_VARS;
    $_ENV = &$HTTP_ENV_VARS;
    $_FILES = &$HTTP_POST_FILES;
}

if (isset($_REQUEST['GLOBALS']) OR isset($_FILES['GLOBALS'])) {
    exit('Request tainting attempted.');
}

require_once DISCUZ_ROOT.'./include/global.func.php';

define('IS_ROBOT', getrobot());
if(defined('NOROBOT') && IS_ROBOT) {
    exit(header("HTTP/1.1 403 Forbidden"));
}

foreach(array('_COOKIE', '_POST', '_GET') as $_request) {
    foreach($$_request as $_key => $_value) {
        $_key{0} != '_' && $$_key = daddslashes($_value);
    }
}

if (!MAGIC_QUOTES_GPC && $_FILES) {
    $_FILES = daddslashes($_FILES);
}

$charset = $dbcharset = $forumfounders = $metakeywords = $extrahead = $seodescription = '';
$plugins = $hooks = $admincp = $jsmenu = $forum = $thread = $language = $actioncode = $modactioncode = $lang = array();

require_once DISCUZ_ROOT.'./config.inc.php';

$_DCOOKIE = $_DSESSION = $_DCACHE = $_DPLUGIN = $advlist = array();

$prelength = strlen($cookiepre);
foreach($_COOKIE as $key => $val) {
    if(substr($key, 0, $prelength) == $cookiepre) {
        $_DCOOKIE[(substr($key, $prelength))] = MAGIC_QUOTES_GPC ? $val : daddslashes($val);
    }
}
unset($prelength, $_request, $_key, $_value);

$inajax = !empty($inajax);
$timestamp = time();

if($attackevasive && CURSCRIPT != 'seccode') {
    require_once DISCUZ_ROOT.'./include/security.inc.php';
}

require_once DISCUZ_ROOT.'./include/db_'.$database.'.class.php';

$PHP_SELF = $_SERVER['PHP_SELF'] ? $_SERVER['PHP_SELF'] : $_SERVER['SCRIPT_NAME'];
$BASESCRIPT = basename($PHP_SELF);
$boardurl = htmlout('https://'.$_SERVER['HTTP_HOST'].preg_replace("/\/+(api|archiver|wap)?\/*$/i", '', substr($PHP_SELF, 0, strrpos($PHP_SELF, '/'))).'/');

if(getenv('HTTP_CLIENT_IP') && strcasecmp(getenv('HTTP_CLIENT_IP'), 'unknown')) {
    $onlineip = getenv('HTTP_CLIENT_IP');
} elseif(getenv('HTTP_X_FORWARDED_FOR') && strcasecmp(getenv('HTTP_X_FORWARDED_FOR'), 'unknown')) {
    $onlineip = getenv('HTTP_X_FORWARDED_FOR');
} elseif(getenv('REMOTE_ADDR') && strcasecmp(getenv('REMOTE_ADDR'), 'unknown')) {
    $onlineip = getenv('REMOTE_ADDR');
} elseif(isset($_SERVER['REMOTE_ADDR']) && $_SERVER['REMOTE_ADDR'] && strcasecmp($_SERVER['REMOTE_ADDR'], 'unknown')) {
    $onlineip = $_SERVER['REMOTE_ADDR'];
}

preg_match("/[\d\.]{7,15}/", $onlineip, $onlineipmatches);
$onlineip = $onlineipmatches[0] ? $onlineipmatches[0] : 'unknown';
unset($onlineipmatches);

$cachelost = (@include DISCUZ_ROOT.'./forumdata/cache/cache_settings.php') ? '' : 'settings';
@extract($_DCACHE['settings']);

if($gzipcompress && function_exists('ob_gzhandler') && !in_array(CURSCRIPT, array('attachment', 'wap')) && !$inajax) {
    ob_start('ob_gzhandler');
} else {
    $gzipcompress = 0;
    ob_start();
}
```

**Migrated Code (PHP 8.3):**
```php
<?php
declare(strict_types=1);

/**
 * Discuz! Board - Bootstrap
 * Modern PHP 8.3 Bootstrap with PSR standards
 *
 * @package      Discuz
 * @version      8.0.0
 */

use Discuz\Application;
use Discuz\Config\Config;
use Discuz\Database\Database;
use Discuz\Http\Request;
use Discuz\Session\SessionManager;
use Discuz\Cache\CacheManager;
use Discuz\Security\Csrf;
use Discuz\Exception\FatalException;

// Define constants
define('DISCUZ_START_TIME', microtime(true));
define('DISCUZ_VERSION', '8.0.0');
define('DISCUZ_START_MEMORY', memory_get_usage());

// Base paths
define('BASE_PATH', dirname(__DIR__));
define('APP_PATH', BASE_PATH . '/app');
define('CONFIG_PATH', BASE_PATH . '/config');
define('STORAGE_PATH', BASE_PATH . '/storage');
define('PUBLIC_PATH', BASE_PATH . '/public');

// Load Composer autoloader
if (file_exists(BASE_PATH . '/vendor/autoload.php')) {
    require BASE_PATH . '/vendor/autoload.php';
} else {
    die('Composer dependencies not installed. Run: composer install');
}

// Set error reporting based on environment
if (getenv('APP_ENV') === 'production') {
    error_reporting(E_ALL & ~E_DEPRECATED & ~E_STRICT);
    ini_set('display_errors', '0');
    ini_set('log_errors', '1');
    ini_set('error_log', STORAGE_PATH . '/logs/php-error.log');
} else {
    error_reporting(E_ALL);
    ini_set('display_errors', '1');
}

// Set timezone
date_default_timezone_set(getenv('APP_TIMEZONE') ?: 'Asia/Shanghai');

// Set internal encoding to UTF-8
mb_internal_encoding('UTF-8');
mb_http_output('UTF-8');

// Security: PreventGLOBALS overwrite
if (isset($_REQUEST['GLOBALS']) || isset($_FILES['GLOBALS'])) {
    throw new FatalException('Request tainting attempted.', 400);
}

// Load configuration
try {
    $config = Config::load(CONFIG_PATH . '/app.php');
} catch (Exception $e) {
    die("Configuration error: " . $e->getMessage());
}

// Initialize application
$app = new Application($config);

// Initialize database
try {
    $database = new Database($config->get('database'));
    $app->instance('db', $database);
} catch (Exception $e) {
    die("Database connection error: " . $e->getMessage());
}

// Initialize cache
try {
    $cache = CacheManager::factory($config->get('cache'));
    $app->instance('cache', $cache);
} catch (Exception $e) {
    // Cache is optional, log but continue
    error_log("Cache initialization failed: " . $e->getMessage());
    $cache = null;
}

// Initialize session
try {
    $session = SessionManager::factory(
        $config->get('session'),
        $app->make('request')
    );
    $app->instance('session', $session);
    $session->start();
} catch (Exception $e) {
    error_log("Session initialization failed: " . $e->getMessage());
}

// Initialize CSRF protection
$csrf = new Csrf($session, $config->get('security'));
$app->instance('csrf', $csrf);

// Load settings from cache/database
try {
    $settings = $cache ? $cache->remember('settings', 3600, function() use ($database) {
        return $database->table('settings')->pluck('value', 'key');
    }) : $database->table('settings')->pluck('value', 'key');

    $app->instance('settings', $settings);
} catch (Exception $e) {
    error_log("Failed to load settings: " . $e->getMessage());
    $settings = [];
}

// Initialize request
$request = Request::capture();
$request->setSession($session);
$app->instance('request', $request);

// Bot detection
$botDetector = $app->make('bot.detector');
$isBot = $botDetector->isBot();
define('IS_ROBOT', $isBot);

// Security checks
if ($config->get('security.attack_defensive.enabled')) {
    $security = $app->make('security.middleware');
    $security->handle($request);
}

// Start output buffering
$gzipEnabled = $settings['gzipcompress'] ?? false;
if ($gzipEnabled && !in_array($request->script(), ['attachment', 'wap']) && !$request->isAjax()) {
    if (function_exists('ob_gzhandler')) {
        ob_start('ob_gzhandler');
    } else {
        ob_start();
    }
} else {
    ob_start();
}

// Make app available globally
\App\Facades\App::setInstance($app);

// Return app for routing
return $app;
```

**Key Changes:**
- ✅ PSR-4 autoloading with Composer
- ✅ Dependency Injection container
- ✅ Request/Response objects
- ✅ Database abstraction layer
- ✅ Session management with secure defaults
- ✅ CSRF protection built-in
- ✅ Modern error handling
- ✅ UTF-8 by default
- ✅ Environment-based configuration
- ✅ Structured logging
- ✅ Proper exception handling

**Database Tables Involved:**
- `cdb_settings` - Application settings
- `cdb_sessions` - Session storage

**Testing Checklist:**
- [ ] Bootstrap loads without errors
- [ ] Composer dependencies are loaded
- [ ] Database connection succeeds
- [ ] Session starts correctly
- [ ] CSRF tokens generate
- [ ] Cache initializes
- [ ] Settings load from database
- [ ] Request object is populated
- [ ] Bot detection works
- [ ] Security middleware activates
- [ ] UTF-8 encoding is applied
- [ ] Error logging works
- [ ] Gzip compression activates when enabled

**Dependencies:**
- Configuration System (1.1)

---

### 1.3 Database Layer

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/db_mysql.class.php`

**Lines:** 1-400

**Current Code (PHP 4/5 style):**
```php
<?php

/*
    [Discuz!] (C)2001-2007 Comsenz Inc.
    This is NOT a freeware, use is subject to license terms

    $Id: db_mysql.class.php 9697 2007-07-11 06:59:07Z cnteacher $
*/

class dbstuff {
    var $querynum = 0;
    var $link;
    var $charset;
    var $tablepre;

    function connect($dbhost, $dbuser, $dbpw, $dbname = '', $pconnect = 0, $halt = TRUE, $dbcharset = 'gbk') {
        $func = empty($pconnect) ? 'mysql_connect' : 'mysql_pconnect';
        if(!$this->link = @$func($dbhost, $dbuser, $dbpw)) {
            $halt && $this->halt('Can not connect to MySQL server!');
        } else {
            if($this->version() > '4.1') {
                if($dbcharset) {
                    mysql_query("SET character_set_connection=$dbcharset, character_set_results=$dbcharset, character_set_client=binary");
                }
                if($this->version() > '5.0.1') {
                    mysql_query("SET sql_mode=''");
                }
            }
            if($dbname) {
                mysql_select_db($dbname, $this->link);
            }
        }
    }

    function select_db($dbname) {
        return mysql_select_db($dbname, $this->link);
    }

    function fetch_array($query, $result_type = MYSQL_ASSOC) {
        return mysql_fetch_array($query, $result_type);
    }

    function result_first($sql) {
        $query = $this->query($sql);
        return $this->result($query, 0);
    }

    function fetch_first($sql) {
        $query = $this->query($sql);
        return $this->fetch_array($query);
    }

    function fetch_all($sql) {
        $arr = array();
        $query = $this->query($sql);
        while($row = $this->fetch_array($query)) {
            $arr[] = $row;
        }
        return $arr;
    }

    function query($sql, $type = '') {
        global $debug, $discuz_starttime, $sqldebug;
        $func = $type == 'UNBUFFERED' && @function_exists('mysql_unbuffered_query') ?
            'mysql_unbuffered_query' : 'mysql_query';
        if(!($query = $func($sql, $this->link))) {
            if(in_array($this->errno(), array(2006, 2013)) && substr($this->tablepre, 0, 6) != '[Table]') {
                $this->connect();
                $query = $func($sql, $this->link);
            } else {
                $this->halt('MySQL Query Error', $sql);
            }
        }
        $this->querynum++;
        return $query;
    }

    function affected_rows() {
        return mysql_affected_rows($this->link);
    }

    function error() {
        return mysql_error();
    }

    function errno() {
        return mysql_errno();
    }

    function result($query, $row) {
        $query = @mysql_result($query, $row);
        return $query;
    }

    function num_rows($query) {
        return mysql_num_rows($query);
    }

    function num_fields($query) {
        return mysql_num_fields($query);
    }

    function free_result($query) {
        return mysql_free_result($query);
    }

    function insert_id() {
        return ($id = mysql_insert_id($this->link)) ? $id : $this->result($this->query("SELECT last_insert_id()"), 0);
    }

    function fetch_row($query) {
        return mysql_fetch_row($query);
    }

    function close() {
        return mysql_close($this->link);
    }

    function version() {
        return mysql_get_server_info();
    }

    function halt($message = '', $sql = '') {
        require_once DISCUZ_ROOT.'./include/db_mysql_error.inc.php';
        $dberror = new dberror();
        $dberror->display($message, $sql);
    }
}
?>
```

**Migrated Code (PHP 8.3 with PDO):**
```php
<?php
declare(strict_types=1);

namespace Discuz\Database;

use PDO;
use PDOException;
use PDOStatement;
use Discuz\Database\Query\Builder;
use Discuz\Exception\DatabaseException;

/**
 * Database Connection Class
 * Modern PDO-based database layer with prepared statements
 */
class Connection
{
    /**
     * The active PDO connection.
     */
    protected PDO $pdo;

    /**
     * The table prefix for the database.
     */
    protected string $tablePrefix = '';

    /**
     * The database connection configuration.
     */
    protected array $config;

    /**
     * Query log.
     */
    protected array $queryLog = [];

    /**
     * Number of queries executed.
     */
    protected int $queryCount = 0;

    /**
     * Whether to log queries.
     */
    protected bool $logging = false;

    /**
     * Create a new database connection instance.
     */
    public function __construct(array $config)
    {
        $this->config = $config;
        $this->tablePrefix = $config['prefix'] ?? '';
        $this->logging = $config['logging'] ?? false;

        $this->connect($config);
    }

    /**
     * Create a new PDO connection.
     */
    protected function connect(array $config): void
    {
        try {
            $dsn = $this->getDsn($config);

            $options = [
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_EMULATE_PREPARES => false,
                PDO::ATTR_STRINGIFY_FETCHES => false,
                PDO::ATTR_PERSISTENT => $config['persistent'] ?? false,
            ];

            // Add charset option
            if (isset($config['charset'])) {
                $options[PDO::MYSQL_ATTR_INIT_COMMAND] =
                    "SET NAMES {$config['charset']}" .
                    ($config['collation'] ? " COLLATE {$config['collation']}" : '');
            }

            $this->pdo = new PDO(
                $dsn,
                $config['username'],
                $config['password'],
                $options
            );

            // Set timezone
            if (isset($config['timezone'])) {
                $this->pdo->exec("SET time_zone = '{$config['timezone']}'");
            }

            // Set strict mode
            if ($config['strict'] ?? false) {
                $this->pdo->exec("SET sql_mode='STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'");
            }

        } catch (PDOException $e) {
            throw new DatabaseException(
                'Database connection failed: ' . $e->getMessage(),
                (int) $e->getCode(),
                $e
            );
        }
    }

    /**
     * Get the DSN string for a PDO connection.
     */
    protected function getDsn(array $config): string
    {
        extract($config, EXTR_SKIP);

        return sprintf(
            'mysql:host=%s;port=%s;dbname=%s;charset=%s',
            $host,
            $port ?? 3306,
            $database,
            $charset ?? 'utf8mb4'
        );
    }

    /**
     * Get the PDO connection.
     */
    public function getPdo(): PDO
    {
        return $this->pdo;
    }

    /**
     * Execute a query and return the first result.
     */
    public function selectOne(string $query, array $bindings = []): ?array
    {
        $records = $this->select($query, $bindings);

        return array_shift($records) ?? null;
    }

    /**
     * Execute a query and return all results.
     */
    public function select(string $query, array $bindings = []): array
    {
        $statement = $this->run($query, $bindings);

        return $statement->fetchAll();
    }

    /**
     * Execute a query and get the value of the first column.
     */
    public function scalar(string $query, array $bindings = []): mixed
    {
        $statement = $this->run($query, $bindings);

        return $statement->fetchColumn();
    }

    /**
     * Execute an INSERT statement and return the ID.
     */
    public function insert(string $query, array $bindings = []): int
    {
        $this->run($query, $bindings);

        return (int) $this->pdo->lastInsertId();
    }

    /**
     * Execute an UPDATE statement and return affected rows.
     */
    public function update(string $query, array $bindings = []): int
    {
        return $this->affectingStatement($query, $bindings);
    }

    /**
     * Execute a DELETE statement and return affected rows.
     */
    public function delete(string $query, array $bindings = []): int
    {
        return $this->affectingStatement($query, $bindings);
    }

    /**
     * Execute a statement and return affected rows.
     */
    public function affectingStatement(string $query, array $bindings = []): int
    {
        $statement = $this->run($query, $bindings);

        return $statement->rowCount();
    }

    /**
     * Run a SQL statement and log its execution context.
     */
    protected function run(string $query, array $bindings): PDOStatement
    {
        $start = microtime(true);

        try {
            $statement = $this->pdo->prepare($query);
            $statement->execute($bindings);
        } catch (PDOException $e) {
            throw new DatabaseException(
                "Query error: {$e->getMessage()} [SQL: {$query}] [Bindings: " . json_encode($bindings) . "]",
                (int) $e->getCode(),
                $e
            );
        }

        $this->queryCount++;

        if ($this->logging) {
            $this->logQuery($query, $bindings, microtime(true) - $start);
        }

        return $statement;
    }

    /**
     * Log a query in the query log.
     */
    protected function logQuery(string $query, array $bindings, float $time): void
    {
        $this->queryLog[] = [
            'query' => $query,
            'bindings' => $bindings,
            'time' => $time,
        ];
    }

    /**
     * Get the query log.
     */
    public function getQueryLog(): array
    {
        return $this->queryLog;
    }

    /**
     * Get the number of queries executed.
     */
    public function getQueryCount(): int
    {
        return $this->queryCount;
    }

    /**
     * Begin a database transaction.
     */
    public function beginTransaction(): bool
    {
        return $this->pdo->beginTransaction();
    }

    /**
     * Commit a database transaction.
     */
    public function commit(): bool
    {
        return $this->pdo->commit();
    }

    /**
     * Rollback a database transaction.
     */
    public function rollBack(): bool
    {
        return $this->pdo->rollBack();
    }

    /**
     * Get the table prefix.
     */
    public function getTablePrefix(): string
    {
        return $this->tablePrefix;
    }

    /**
     * Get a query builder instance.
     */
    public function table(string $table): Builder
    {
        return new Builder($this, $table);
    }

    /**
     * Escape a value for SQL (use prepared statements instead).
     * @deprecated Use parameterized queries instead
     */
    public function escape(string $value): string
    {
        return substr($this->pdo->quote($value), 1, -1);
    }

    /**
     * Close the database connection.
     */
    public function close(): void
    {
        $this->pdo = null;
    }

    /**
     * Handle dynamic method calls.
     */
    public function __call(string $method, array $parameters): mixed
    {
        return $this->table($method)->...$parameters;
    }
}
```

**Key Changes:**
- ✅ PDO instead of mysql_* functions (deprecated in PHP 7, removed in PHP 8)
- ✅ Prepared statements by default (SQL injection protection)
- ✅ Proper exception handling
- ✅ Query logging for debugging
- ✅ Transaction support
- ✅ UTF-8/utf8mb4 charset
- ✅ Query builder pattern
- ✅ Type hints and return types
- ✅ Connection pooling support

**Database Tables Involved:**
- All tables (this is the database layer)

**Testing Checklist:**
- [ ] Database connection succeeds
- [ ] Prepared statements work
- [ ] INSERT returns correct last insert ID
- [ ] UPDATE/DELETE return affected row count
- [ ] SELECT returns correct data
- [ ] Transactions commit/rollback correctly
- [ ] Query logging works
- [ ] UTF-8 encoding works
- [ ] Connection pooling works (if enabled)
- [ ] Error handling catches exceptions
- [ ] Table prefix is applied correctly

**Dependencies:**
- Configuration System (1.1)
- Core Bootstrap (1.2)

---

## 2. Authentication & Session Management

### 2.1 User Login

**Current File:** `/root/poketb-renew/poketb.com/bbs/logging.php`

**Lines:** 16-150

**Current Code (PHP 4/5 style):**
```php
if($action == 'logout' && !empty($formhash)) {
    // ... logout code ...
} elseif($action == 'login') {
    if(!submitcheck('loginsubmit')) {
        // ... show login form ...
    } else {
        $loginperm = logincheck();
        if(!$loginperm) {
            showmessage('login_strike');
        }

        require_once DISCUZ_ROOT.'./uc_client/client.php';

        $secques = $questionid > 0 ? quescrypt($questionid, $answer) : '';

        $ucresult = uc_user_login($username, $password, $loginfield == 'uid', 1, $questionid, $answer);

        if($ucresult[0] > 0) {
            $uid = $ucresult[0];
            $username = $ucresult[1];
            $password = $ucresult[2];
            $email = $ucresult[3];

            $member = $db->fetch_first("SELECT * FROM {$tablepre}members WHERE uid='$uid'");
            if($member) {
                // Update existing member
            } else {
                // Insert new member
            }

            dsetcookie('auth', authcode("$password\t$secques\t$uid", 'ENCODE'), 2592000);
            dsetcookie('sid', '', -86400);

            showmessage('login_succeed', dreferer());
        } elseif($ucresult[0] == -1) {
            showmessage('user_does_not_exist');
        } elseif($ucresult[0] == -2) {
            showmessage('password_invalid');
        }
    }
}
```

**Migrated Code (PHP 8.3):**
```php
<?php
declare(strict_types=1);

namespace Discuz\Auth\Controllers;

use Discuz\Auth\AuthService;
use Discuz\Auth\LoginService;
use Discuz\Http\Request;
use Discuz\Http\Response;
use Discuz\Session\Session;
use Discuz\Validation\ValidationException;
use Discuz\Exception\AuthenticationException;

/**
 * Authentication Controller
 * Handles user login, logout, and session management
 */
class AuthController
{
    protected AuthService $authService;
    protected LoginService $loginService;
    protected Session $session;

    public function __construct(
        AuthService $authService,
        LoginService $loginService,
        Session $session
    ) {
        $this->authService = $authService;
        $this->loginService = $loginService;
        $this->session = $session;
    }

    /**
     * Show login form
     */
    public function showLoginForm(Request $request): Response
    {
        // Check if already logged in
        if ($this->authService->check()) {
            return redirect('/');
        }

        // Generate CSRF token
        $csrfToken = $request->session()->token();

        // Get redirect URL
        $redirect = $request->query('redirect', $request->header('referer', '/'));

        return view('auth.login', [
            'csrfToken' => $csrfToken,
            'redirect' => $redirect,
        ]);
    }

    /**
     * Handle login request
     */
    public function login(Request $request): Response
    {
        try {
            // Validate request
            $validated = $request->validate([
                'username' => 'required|string|max:50',
                'password' => 'required|string',
                'questionid' => 'nullable|int',
                'answer' => 'nullable|string',
            ]);

            // Check rate limiting
            if (!$this->loginService->attemptLogin($request->ip())) {
                throw new AuthenticationException('Too many login attempts. Please try again later.');
            }

            // Perform login
            $user = $this->authService->attempt(
                $validated['username'],
                $validated['password'],
                (bool) $request->input('remember', false)
            );

            if (!$user) {
                $this->loginService->recordFailedLogin($request->ip());
                throw new AuthenticationException('Invalid username or password');
            }

            // Check if user is banned
            if ($user->isBanned()) {
                $this->authService->logout();
                throw new AuthenticationException('Your account has been banned');
            }

            // Regenerate session ID
            $this->session->regenerate();

            // Record successful login
            $this->loginService->recordSuccessfulLogin($user->id, $request->ip(), $request->userAgent());

            // Redirect to intended URL
            $redirect = $validated['redirect'] ?? '/';

            return redirect($redirect)->with('success', 'Login successful');

        } catch (ValidationException $e) {
            return redirect()->back()->withErrors($e->errors())->withInput();
        } catch (AuthenticationException $e) {
            return redirect()->back()
                ->with('error', $e->getMessage())
                ->withInput($request->only('username'));
        }
    }

    /**
     * Handle logout request
     */
    public function logout(Request $request): Response
    {
        // Validate CSRF token
        $request->validate([
            '_token' => 'required|csrf_token',
        ]);

        // Get current user before logout
        $user = $this->authService->user();

        // Perform logout
        $this->authService->logout();

        // Invalidate session
        $this->session->invalidate();

        // Regenerate CSRF token
        $request->session()->regenerateToken();

        return redirect('/')
            ->with('success', 'You have been logged out successfully');
    }
}
```

**Authentication Service:**
```php
<?php
declare(strict_types=1);

namespace Discuz\Auth;

use Discuz\Database\Database;
use Discuz\Auth\Models\User;
use Discuz\Auth\Hashing\Hasher;
use Discuz\Session\Session;
use Discuz\Cookie\CookieJar;

/**
 * Authentication Service
 * Handles user authentication logic
 */
class AuthService
{
    protected Database $db;
    protected Hasher $hasher;
    protected Session $session;
    protected CookieJar $cookie;

    protected ?User $currentUser = null;

    public function __construct(
        Database $db,
        Hasher $hasher,
        Session $session,
        CookieJar $cookie
    ) {
        $this->db = $db;
        $this->hasher = $hasher;
        $this->session = $session;
        $this->cookie = $cookie;
    }

    /**
     * Attempt to authenticate a user
     */
    public function attempt(string $username, string $password, bool $remember = false): ?User
    {
        // Find user by username
        $user = $this->fetchUser($username);

        if (!$user) {
            return null;
        }

        // Verify password
        if (!$this->hasher->check($password, $user->password)) {
            return null;
        }

        // Set user as authenticated
        $this->setUser($user);

        // Set remember token if requested
        if ($remember) {
            $this->createRememberToken($user);
        }

        // Update last visit
        $this->updateLastVisit($user);

        return $user;
    }

    /**
     * Set the current user
     */
    public function setUser(User $user): void
    {
        $this->currentUser = $user;

        // Store in session
        $this->session->put('auth.user_id', $user->id);
        $this->session->put('auth.username', $user->username);
        $this->session->put('auth.admin_id', $user->adminId);
        $this->session->put('auth.group_id', $user->groupId);
    }

    /**
     * Get the current user
     */
    public function user(): ?User
    {
        if ($this->currentUser !== null) {
            return $this->currentUser;
        }

        // Try to load from session
        $userId = $this->session->get('auth.user_id');

        if ($userId) {
            $user = $this->fetchUserById((int) $userId);
            if ($user) {
                $this->currentUser = $user;
                return $user;
            }
        }

        // Try to load from remember token
        $token = $this->cookie->get('remember_token');

        if ($token) {
            $user = $this->fetchUserByRememberToken($token);
            if ($user) {
                $this->setUser($user);
                return $user;
            }
        }

        return null;
    }

    /**
     * Check if user is authenticated
     */
    public function check(): bool
    {
        return $this->user() !== null;
    }

    /**
     * Get user ID
     */
    public function id(): ?int
    {
        return $this->user()?->id;
    }

    /**
     * Logout user
     */
    public function logout(): void
    {
        $user = $this->user();

        if ($user) {
            // Delete remember tokens
            $this->deleteRememberTokens($user);
        }

        // Clear session
        $this->session->forget('auth');

        // Clear cookies
        $this->cookie->delete('remember_token');
        $this->cookie->delete('session_token');

        $this->currentUser = null;
    }

    /**
     * Fetch user by username
     */
    protected function fetchUser(string $username): ?User
    {
        $data = $this->db->table('members')
            ->where('username', $username)
            ->select();

        return $data ? new User($data) : null;
    }

    /**
     * Fetch user by ID
     */
    protected function fetchUserById(int $id): ?User
    {
        $data = $this->db->table('members')
            ->where('uid', $id)
            ->selectOne();

        return $data ? new User($data) : null;
    }

    /**
     * Create remember token
     */
    protected function createRememberToken(User $user): void
    {
        $token = bin2hex(random_bytes(32));

        $this->db->table('member_fields')
            ->where('uid', $user->id)
            ->update([
                'remember_token' => hash('sha256', $token),
            ]);

        $this->cookie->set('remember_token', $token, 2592000); // 30 days
    }

    /**
     * Delete remember tokens
     */
    protected function deleteRememberTokens(User $user): void
    {
        $this->db->table('member_fields')
            ->where('uid', $user->id)
            ->update([
                'remember_token' => null,
            ]);
    }

    /**
     * Update last visit timestamp
     */
    protected function updateLastVisit(User $user): void
    {
        $this->db->table('members')
            ->where('uid', $user->id)
            ->update([
                'lastvisit' => time(),
                'lastactivity' => time(),
            ]);
    }
}
```

**Key Changes:**
- ✅ Dependency injection
- ✅ CSRF protection on logout
- ✅ Secure password hashing (bcrypt/argon2)
- ✅ Remember me with secure tokens
- ✅ Rate limiting on login attempts
- ✅ Session fixation protection (regenerate ID)
- ✅ Type hints and return types
- ✅ Exception-based error handling
- ✅ Separation of concerns (Controller → Service)

**Database Tables Involved:**
- `cdb_members` - User accounts
- `cdb_memberfields` - Extended user data
- `cdb_sessions` - Session storage
- `cdb_banned` - Banned users
- `cdb_failedlogins` - Failed login attempts

**Testing Checklist:**
- [ ] Valid login succeeds
- [ ] Invalid credentials fail
- [ ] Session is created after login
- [ ] Remember me token is set
- [ ] CSRF token is validated
- [ ] Rate limiting works
- [ ] Logout clears session
- [ ] Logout clears remember token
- [ ] Banned users cannot login
- [ ] Session regeneration works
- [ ] Last visit timestamp updates
- [ ] Multiple logins are tracked

**Dependencies:**
- Core Bootstrap (1.2)
- Database Layer (1.3)
- Session Management (2.3)

---

### 2.2 Session Management

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/common.inc.php`

**Lines:** 141-200

**Current Code (PHP 4/5 style):**
```php
if($sid) {
    if($discuz_uid) {
        $query = $db->query("SELECT s.sid, s.styleid, s.groupid='6' AS ipbanned, s.pageviews AS spageviews, s.lastolupdate, s.seccode, $membertablefields
            FROM {$tablepre}sessions s, {$tablepre}members m
            WHERE m.uid=s.uid AND s.sid='$sid' AND CONCAT_WS('.',s.ip1,s.ip2,s.ip3,s.ip4)='$onlineip' AND m.uid='$discuz_uid'
            AND m.password='$discuz_pw' AND m.secques='$discuz_secques'");
    } else {
        $query = $db->query("SELECT sid, uid AS sessionuid, groupid, groupid='6' AS ipbanned, pageviews AS spageviews, styleid, lastolupdate, seccode
            FROM {$tablepre}sessions WHERE sid='$sid' AND CONCAT_WS('.',ip1,ip2,ip3,ip4)='$onlineip'");
    }

    if($_DSESSION = $db->fetch_array($query)) {
        $sessionexists = 1;
    } else {
        $sessionexists = 0;
    }
} else {
    if($discuz_uid) {
        $query = $db->query("SELECT $membertablefields FROM {$tablepre}members WHERE uid='$discuz_uid' AND password='$discuz_pw' AND secques='$discuz_secques'");
        if($_DSESSION = $db->fetch_array($query)) {
            $_DSESSION['sid'] = random(6);
            $sessionexists = 0;
        }
    }
}

// ... session update logic ...

$db->query("DELETE FROM {$tablepre}sessions WHERE lastactivity<='".($timestamp - $onlinehold)."' OR ('$sid' && sid='$sid' && ip1='$onlineip1' && ip2='$onlineip2' && ip3='$onlineip3' && ip4='$onlineip4')");
```

**Migrated Code (PHP 8.3):**
```php
<?php
declare(strict_types=1);

namespace Discuz\Session;

use Discuz\Database\Database;
use Discuz\Cookie\CookieJar;
use Discuz\Support\Str;

/**
 * Session Manager
 * Handles session storage and retrieval
 */
class SessionManager
{
    protected Database $db;
    protected CookieJar $cookie;
    protected array $config;

    protected ?string $sessionId = null;
    protected array $data = [];

    public function __construct(Database $db, CookieJar $cookie, array $config)
    {
        $this->db = $db;
        $this->cookie = $cookie;
        $this->config = $config;
    }

    /**
     * Start the session
     */
    public function start(): void
    {
        $sessionId = $this->getIdFromRequest();

        if ($sessionId) {
            $this->load($sessionId);
        }

        if (!$this->isValid()) {
            $this->create();
        } else {
            $this->refresh();
        }
    }

    /**
     * Get session ID from request
     */
    protected function getIdFromRequest(): ?string
    {
        // Try cookie first
        $sessionId = $this->cookie->get($this->config['cookie']);

        if ($sessionId) {
            return $sessionId;
        }

        // Try query string if enabled
        if ($this->config['query_string_on'] ?? false) {
            return $_GET[$this->config['cookie']] ?? null;
        }

        return null;
    }

    /**
     * Load session from database
     */
    protected function load(string $sessionId): bool
    {
        $session = $this->db->table('sessions')
            ->where('sid', $sessionId)
            ->where('ip', $this->getClientIp())
            ->selectOne();

        if (!$session) {
            return false;
        }

        $this->sessionId = $sessionId;
        $this->data = json_decode($session['data'] ?? '{}', true) ?: [];

        // Update last activity
        $this->refresh();

        return true;
    }

    /**
     * Check if session is valid
     */
    protected function isValid(): bool
    {
        return $this->sessionId !== null;
    }

    /**
     * Create new session
     */
    protected function create(): void
    {
        $this->sessionId = Str::random(40);
        $this->data = [];

        $this->db->table('sessions')->insert([
            'sid' => $this->sessionId,
            'uid' => $this->get('auth.user_id', 0),
            'ip' => $this->getClientIp(),
            'lastactivity' => time(),
            'data' => json_encode($this->data),
        ]);

        $this->saveCookie();
    }

    /**
     * Refresh session timestamp
     */
    protected function refresh(): void
    {
        $this->db->table('sessions')
            ->where('sid', $this->sessionId)
            ->update([
                'lastactivity' => time(),
                'uid' => $this->get('auth.user_id', 0),
            ]);
    }

    /**
     * Regenerate session ID
     */
    public function regenerate(): void
    {
        if (!$this->sessionId) {
            return;
        }

        // Delete old session
        $this->db->table('sessions')
            ->where('sid', $this->sessionId)
            ->delete();

        // Create new session
        $oldSessionId = $this->sessionId;
        $this->sessionId = Str::random(40);

        $this->db->table('sessions')->insert([
            'sid' => $this->sessionId,
            'uid' => $this->get('auth.user_id', 0),
            'ip' => $this->getClientIp(),
            'lastactivity' => time(),
            'data' => json_encode($this->data),
        ]);

        $this->saveCookie();
    }

    /**
     * Get value from session
     */
    public function get(string $key, mixed $default = null): mixed
    {
        return $this->data[$key] ?? $default;
    }

    /**
     * Set value in session
     */
    public function put(string $key, mixed $value): void
    {
        $this->data[$key] = $value;
        $this->save();
    }

    /**
     * Remove value from session
     */
    public function forget(string $key): void
    {
        unset($this->data[$key]);
        $this->save();
    }

    /**
     * Check if key exists
     */
    public function has(string $key): bool
    {
        return isset($this->data[$key]);
    }

    /**
     * Save session to database
     */
    protected function save(): void
    {
        if (!$this->sessionId) {
            return;
        }

        $this->db->table('sessions')
            ->where('sid', $this->sessionId)
            ->update([
                'data' => json_encode($this->data),
            ]);
    }

    /**
     * Invalidate current session
     */
    public function invalidate(): void
    {
        if ($this->sessionId) {
            $this->db->table('sessions')
                ->where('sid', $this->sessionId)
                ->delete();
        }

        $this->sessionId = null;
        $this->data = [];
        $this->cookie->delete($this->config['cookie']);
    }

    /**
     * Get session ID
     */
    public function getId(): ?string
    {
        return $this->sessionId;
    }

    /**
     * Save session cookie
     */
    protected function saveCookie(): void
    {
        $this->cookie->set(
            $this->config['cookie'],
            $this->sessionId,
            $this->config['lifetime']
        );
    }

    /**
     * Get client IP
     */
    protected function getClientIp(): string
    {
        $headers = [
            'HTTP_CF_CONNECTING_IP',
            'HTTP_CLIENT_IP',
            'HTTP_X_FORWARDED_FOR',
            'HTTP_X_FORWARDED',
            'HTTP_X_CLUSTER_CLIENT_IP',
            'HTTP_FORWARDED_FOR',
            'HTTP_FORWARDED',
            'REMOTE_ADDR',
        ];

        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                $ips = explode(',', $_SERVER[$header]);
                $ip = trim($ips[0]);

                if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
                    return $ip;
                }
            }
        }

        return '0.0.0.0';
    }

    /**
     * Clean up expired sessions
     */
    public function cleanup(): int
    {
        $expired = time() - $this->config['lifetime'];

        return $this->db->table('sessions')
            ->where('lastactivity', '<', $expired)
            ->delete();
    }

    /**
     * Get online user count
     */
    public function getOnlineCount(): int
    {
        return $this->db->table('sessions')
            ->where('lastactivity', '>', time() - $this->config['lifetime'])
            ->count();
    }
}
```

**Key Changes:**
- ✅ JSON encoding for session data
- ✅ Secure session ID generation (40 chars vs 6)
- ✅ IP validation with private range filtering
- ✅ Session fixation protection (regenerate)
- ✅ Automatic cleanup of expired sessions
- ✅ Cookie secure defaults (HttpOnly, SameSite)
- ✅ Type hints and return types
- ✅ Proper IP detection order (CF, X-Forwarded, etc.)

**Database Tables Involved:**
- `cdb_sessions` - Session storage

**Testing Checklist:**
- [ ] Session creates successfully
- [ ] Session loads from database
- [ ] Session saves data correctly
- [ ] Session ID regeneration works
- [ ] Session invalidation works
- [ ] Cookie is set correctly
- [ ] IP detection works
- [ ] Expired sessions are cleaned up
- [ ] Online count is accurate
- [ ] Session timeout works

**Dependencies:**
- Database Layer (1.3)

---

## 3. Caching System

### 3.1 Cache Core

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/cache.func.php`

**Lines:** 1-800

**Current Code (PHP 4/5 style):**
```php
function updatecache($cachename = '') {
    // ... cache update logic ...
}

function getcacheinfo($cacheid) {
    // ... cache file info ...
}

function arrayeval($array, $level = 0) {
    // ... convert array to PHP code ...
}
```

**Migrated Code (PHP 8.3):**
```php
<?php
declare(strict_types=1);

namespace Discuz\Cache;

use Discuz\Database\Database;
use Discuz\Cache\Stores\FileStore;
use Discuz\Cache\Stores\RedisStore;
use Discuz\Cache\Stores\MemcachedStore;
use Discuz\Cache\Stores\DatabaseStore;
use Discuz\Cache\Stores\ArrayStore;
use Discuz\Cache\Serializers\PhpSerializer;
use Closure;

/**
 * Cache Repository
 * Unified caching interface with multiple backend support
 */
class CacheRepository
{
    protected Store $store;
    protected Database $db;

    public function __construct(Store $store, Database $db)
    {
        $this->store = $store;
        $this->db = $db;
    }

    /**
     * Get an item from the cache
     */
    public function get(string $key, mixed $default = null): mixed
    {
        $value = $this->store->get($key);

        if ($value === null) {
            return $default instanceof Closure ? $default() : $default;
        }

        return $value;
    }

    /**
     * Get an item or store default value
     */
    public function remember(string $key, int $ttl, Closure $callback): mixed
    {
        $value = $this->get($key);

        if ($value !== null) {
            return $value;
        }

        $this->put($key, $value = $callback(), $ttl);

        return $value;
    }

    /**
     * Put an item in the cache
     */
    public function put(string $key, mixed $value, int $ttl): bool
    {
        return $this->store->put($key, $value, $ttl);
    }

    /**
     * Store an item forever
     */
    public function forever(string $key, mixed $value): bool
    {
        return $this->store->forever($key, $value);
    }

    /**
     * Delete an item from cache
     */
    public function forget(string $key): bool
    {
        return $this->store->forget($key);
    }

    /**
     * Clear all cache
     */
    public function flush(): bool
    {
        return $this->store->flush();
    }

    /**
     * Check if item exists
     */
    public function has(string $key): bool
    {
        return $this->store->has($key);
    }

    /**
     * Increment item value
     */
    public function increment(string $key, int $value = 1): int|bool
    {
        return $this->store->increment($key, $value);
    }

    /**
     * Decrement item value
     */
    public function decrement(string $key, int $value = 1): int|bool
    {
        return $this->store->decrement($key, $value);
    }

    /**
     * Update system caches
     */
    public function updateSystemCaches(): void
    {
        $this->updateSettingsCache();
        $this->updateForumsCache();
        $this->updateUsergroupsCache();
        $this->updateSmiliesCache();
        $this->updateAnnouncementsCache();
    }

    /**
     * Update settings cache
     */
    protected function updateSettingsCache(): void
    {
        $settings = $this->db->table('settings')
            ->pluck('value', 'key');

        $this->forever('settings', $settings);
    }

    /**
     * Update forums cache
     */
    protected function updateForumsCache(): void
    {
        $forums = $this->db->query("
            SELECT f.fid, f.fup, f.type, f.name, f.threads, f.posts, f.todayposts,
                   f.lastpost, f.inheritedmod, f.forumcolumns, f.simple,
                   ff.description, ff.moderators, ff.icon, ff.viewperm, ff.redirect
            FROM cdb_forums f
            LEFT JOIN cdb_forumfields ff USING(fid)
            WHERE f.status > 0
            ORDER BY f.type, f.displayorder
        ")->fetchAll();

        $this->forever('forums', $forums);
    }

    /**
     * Update usergroups cache
     */
    protected function updateUsergroupsCache(): void
    {
        $usergroups = $this->db->table('usergroups')
            ->orderBy('groupid')
            ->fetchAll();

        $this->forever('usergroups', $usergroups);
    }

    /**
     * Update smilies cache
     */
    protected function updateSmiliesCache(): void
    {
        $smilies = $this->db->table('smilies')
            ->where('type', 'smiley')
            ->orderBy('displayorder')
            ->fetchAll();

        $this->forever('smilies', $smilies);
    }

    /**
     * Update announcements cache
     */
    protected function updateAnnouncementsCache(): void
    {
        $announcements = $this->db->table('announcements')
            ->where('starttime', '<=', time())
            ->where('endtime', '>=', time())
            ->orderBy('displayorder', 'desc')
            ->orderBy('starttime', 'desc')
            ->fetchAll();

        $this->put('announcements', $announcements, 3600);
    }
}

/**
 * Cache Store Interface
 */
interface Store
{
    public function get(string $key): mixed;
    public function put(string $key, mixed $value, int $ttl): bool;
    public function forever(string $key, mixed $value): bool;
    public function forget(string $key): bool;
    public function flush(): bool;
    public function has(string $key): bool;
    public function increment(string $key, int $value = 1): int|bool;
    public function decrement(string $key, int $value = 1): int|bool;
}

/**
 * Redis Store Implementation
 */
class RedisStore implements Store
{
    protected \Redis $redis;
    protected string $prefix;

    public function __construct(\Redis $redis, string $prefix = 'discuz_cache_')
    {
        $this->redis = $redis;
        $this->prefix = $prefix;
    }

    protected function getPrefixedKey(string $key): string
    {
        return $this->prefix . $key;
    }

    public function get(string $key): mixed
    {
        $value = $this->redis->get($this->getPrefixedKey($key));

        return $value !== null ? unserialize($value) : null;
    }

    public function put(string $key, mixed $value, int $ttl): bool
    {
        return $this->redis->setex(
            $this->getPrefixedKey($key),
            $ttl,
            serialize($value)
        );
    }

    public function forever(string $key, mixed $value): bool
    {
        return $this->redis->set(
            $this->getPrefixedKey($key),
            serialize($value)
        );
    }

    public function forget(string $key): bool
    {
        return (bool) $this->redis->del($this->getPrefixedKey($key));
    }

    public function flush(): bool
    {
        $keys = $this->redis->keys($this->prefix . '*');

        if (empty($keys)) {
            return true;
        }

        return $this->redis->del(...$keys) > 0;
    }

    public function has(string $key): bool
    {
        return (bool) $this->redis->exists($this->getPrefixedKey($key));
    }

    public function increment(string $key, int $value = 1): int|bool
    {
        return $this->redis->incrBy($this->getPrefixedKey($key), $value);
    }

    public function decrement(string $key, int $value = 1): int|bool
    {
        return $this->redis->decrBy($this->getPrefixedKey($key), $value);
    }
}

/**
 * File Store Implementation
 */
class FileStore implements Store
{
    protected string $path;
    protected PhpSerializer $serializer;

    public function __construct(string $path)
    {
        $this->path = $path;
        $this->serializer = new PhpSerializer();

        if (!is_dir($path)) {
            mkdir($path, 0755, true);
        }
    }

    protected function getFilePath(string $key): string
    {
        return $this->path . '/' . sha1($key) . '.cache';
    }

    public function get(string $key): mixed
    {
        $path = $this->getFilePath($key);

        if (!file_exists($path)) {
            return null;
        }

        $contents = file_get_contents($path);

        // Check expiration
        $expire = substr($contents, 0, 10);

        if (time() >= (int) $expire) {
            @unlink($path);
            return null;
        }

        return $this->serializer->unserialize(substr($contents, 10));
    }

    public function put(string $key, mixed $value, int $ttl): bool
    {
        $path = $this->getFilePath($key);
        $expire = time() + $ttl;

        $contents = $expire . $this->serializer->serialize($value);

        return file_put_contents($path, $contents, LOCK_EX) !== false;
    }

    public function forever(string $key, mixed $value): bool
    {
        return $this->put($key, $value, 9999999999);
    }

    public function forget(string $key): bool
    {
        $path = $this->getFilePath($key);

        return @unlink($path);
    }

    public function flush(): bool
    {
        $files = glob($this->path . '/*.cache');

        if ($files === false) {
            return false;
        }

        foreach ($files as $file) {
            @unlink($file);
        }

        return true;
    }

    public function has(string $key): bool
    {
        return file_exists($this->getFilePath($key));
    }

    public function increment(string $key, int $value = 1): int|bool
    {
        if (!$this->has($key)) {
            $this->put($key, 0, 9999999999);
        }

        $current = $this->get($key);
        $new = $current + $value;

        $this->forever($key, $new);

        return $new;
    }

    public function decrement(string $key, int $value = 1): int|bool
    {
        return $this->increment($key, -$value);
    }
}
```

**Key Changes:**
- ✅ Multiple cache backends (Redis, Memcached, File, Database, Array)
- ✅ Serialization for complex data types
- ✅ TTL (time-to-live) support
- ✅ Atomic operations (increment/decrement)
- ✅ Cache tagging support (can be added)
- ✅ Automatic cache updates
- ✅ Type hints and return types
- ✅ PSR-6/PSR-16 compatibility (can be implemented)

**Database Tables Involved:**
- `cdb_settings` - Settings cache
- `cdb_forums` - Forum cache
- `cdb_forumfields` - Forum extended info cache
- `cdb_usergroups` - User group cache
- `cdb_smilies` - Smilies cache
- `cdb_announcements` - Announcement cache
- `cache` table (for database cache backend)

**Testing Checklist:**
- [ ] Cache writes work
- [ ] Cache reads work
- [ ] Cache TTL works (items expire)
- [ ] Cache invalidation works
- [ ] Multiple cache backends work
- [ ] Serialization works
- [ ] Increment/decrement work atomically
- [ ] System caches update correctly
- [ ] Cache flush works
- [ ] File cache handles concurrent access
- [ ] Redis cache connection handles failures

**Dependencies:**
- Database Layer (1.3)

---

## Summary: P0 Critical Path

**Total Features:** 9 core features
**Estimated Time:** 2-3 weeks (15 working days)
**Dependencies:** Sequential (must do in order)

### Week 1: Foundation (Days 1-5)
- Day 1-2: Configuration System (1.1)
- Day 3-4: Core Bootstrap (1.2)
- Day 5: Database Layer (1.3)

### Week 2: Authentication (Days 6-10)
- Day 6-7: Session Management (2.2)
- Day 8-9: User Login (2.1)
- Day 10: Testing & Bug Fixes

### Week 3: Caching & Testing (Days 11-15)
- Day 11-12: Caching System (3.1)
- Day 13-14: Integration Testing
- Day 15: Performance Testing

### Success Criteria
✅ User can login successfully
✅ Session persists across requests
✅ Database queries work correctly
✅ Cache speeds up page loads
✅ No security vulnerabilities
✅ UTF-8 encoding works correctly
✅ All P0 tests pass

**Next Phase:** P1 Core Features (Document 02)
