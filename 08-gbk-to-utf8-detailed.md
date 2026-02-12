# GBK to UTF-8 Migration - Detailed Technical Guide

**Purpose:** Step-by-step instructions for converting GBK encoding to UTF-8
**Scope:** All PHP files, HTML templates, and database data
**Risk Level:** HIGH - Data loss possible if done incorrectly
**Timeline:** Parallel with code migration (Weeks 1-20)

---

## Executive Summary

**Current State:**
- File encoding: GBK/GB2312
- Database encoding: latin1 (storing GBK bytes)
- HTML charset: GBK

**Target State:**
- File encoding: UTF-8
- Database encoding: utf8mb4
- HTML charset: UTF-8

**Strategy:**
1. Convert database first (highest risk)
2. Convert PHP files (medium risk)
3. Convert templates (low risk)
4. Validate continuously

---

## Phase 1: Database Conversion (Days 1-5)

### Pre-Conversion Checklist

**BACKUP CRITICAL:**
```bash
# Full database backup
mysqldump -u root -p --all-databases --single-transaction --quick --lock-tables=false > backup_before_utf8_$(date +%Y%m%d).sql

# Compress backup
gzip backup_before_utf8_$(date +%Y%m%d).sql

# Verify backup
zcat backup_before_utf8_$(date +%Y%m%d).sql.gz | head -n 5
```

**Test Database Setup:**
```bash
# Create test database
mysql -u root -p -e "CREATE DATABASE discuz_utf8_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Copy to test database
mysqldump -u root -p discuz | mysql -u root -p discuz_utf8_test
```

---

### Step 1: Convert Database Schema

**Current State:** latin1 with GBK data
**Target:** utf8mb4 with proper UTF-8 data

**Script:** `scripts/convert-database-utf8.php`

```php
<?php
declare(strict_types=1);

/**
 * Database UTF-8 Conversion Script
 * Converts GBK data in latin1 columns to UTF-8
 */

// Database connection
$db = new PDO('mysql:host=localhost;dbname=discuz', 'root', 'password');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

// Get all tables
$tables = $db->query("SHOW TABLES")->fetchAll(PDO::FETCH_COLUMN);

foreach ($tables as $table) {
    echo "Converting table: $table\n";

    // Get table columns
    $columns = $db->query("SHOW FULL COLUMNS FROM `$table`")->fetchAll();

    foreach ($columns as $column) {
        $fieldName = $column['Field'];
        $fieldType = $column['Type'];
        $fieldNull = $column['Null'] === 'YES' ? 'NULL' : 'NOT NULL';
        $fieldDefault = $column['Default'] ?? 'NULL';

        // Skip non-text columns
        if (!preg_match('/(char|text|blob)/i', $fieldType)) {
            continue;
        }

        echo "  Converting column: $fieldName ($fieldType)\n";

        // Strategy: Add temp column, convert data, drop old, rename new

        // 1. Add temp UTF-8 column
        $tempColumn = $fieldName . '_utf8';
        $db->query("ALTER TABLE `$table` ADD COLUMN `$tempColumn` $fieldType CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci $fieldNull");

        // 2. Convert data from GBK to UTF-8
        $db->query("UPDATE `$table` SET `$tempColumn` = CONVERT(CAST(CONVERT(`$fieldName` USING latin1) AS BINARY) USING utf8mb4)");

        // 3. Drop old column
        $db->query("ALTER TABLE `$table` DROP COLUMN `$fieldName`");

        // 4. Rename temp column
        $db->query("ALTER TABLE `$table` CHANGE COLUMN `$tempColumn` `$fieldName` $fieldType CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci $fieldNull");

        echo "  ✅ Converted: $fieldName\n";
    }

    // Convert table default charset
    $db->query("ALTER TABLE `$table` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");
    echo "✅ Table converted: $table\n\n";
}

echo "\n✅ All tables converted successfully!\n";
```

**Execution:**
```bash
php scripts/convert-database-utf8.php | tee conversion_log.txt
```

---

### Step 2: Validate Database Conversion

**Validation Script:** `scripts/validate-utf8.php`

```php
<?php
declare(strict_types=1);

$db = new PDO('mysql:host=localhost;dbname=discuz', 'root', 'password');

echo "Validating UTF-8 conversion...\n\n";

// Check table charset
$tables = $db->query("SHOW TABLE STATUS")->fetchAll();

foreach ($tables as $table) {
    $tableName = $table['Name'];
    $collation = $table['Collation'];

    if (strpos($collation, 'utf8mb4') === false) {
        echo "⚠️  WARNING: Table $tableName has collation: $collation\n";
    } else {
        echo "✅ Table $tableName: $collation\n";
    }
}

// Sample data validation
echo "\nValidating data integrity...\n";

$sampleTables = ['cdb_members', 'cdb_threads', 'cdb_posts'];

foreach ($sampleTables as $table) {
    $count = $db->query("SELECT COUNT(*) FROM $table")->fetchColumn();
    $sample = $db->query("SELECT * FROM $table LIMIT 1")->fetch();

    echo "$table: $count rows\n";

    // Check for valid UTF-8
    foreach ($sample as $key => $value) {
        if (!is_string($value)) continue;

        if (!mb_check_encoding($value, 'UTF-8')) {
            echo "  ⚠️  Invalid UTF-8 in column: $key\n";
        }
    }
}

echo "\n✅ Validation complete!\n";
```

**Common Issues:**

**Issue 1: Double-encoded UTF-8**
```
Symptoms: Text looks like "å®¶ç"å®¶"
Fix: REVERSE conversion first
UPDATE table SET column = CONVERT(CAST(CONVERT(column USING utf8) AS BINARY) USING latin1);
```

**Issue 2: Truncated data**
```
Symptoms: Text cut off mid-character
Fix: Increase column size (VARCHAR needs 3x space for UTF-8)
ALTER TABLE table MODIFY column TEXT;
```

**Issue 3: Emoji not supported**
```
Symptoms: Emoji display as ?
Fix: Use utf8mb4 not utf8
ALTER DATABASE database CHARACTER SET utf8mb4;
```

---

## Phase 2: PHP File Conversion (Days 6-10)

### Step 1: Batch Convert PHP Files

**Tool:** `iconv` command-line

**Script:** `scripts/convert-php-files.sh`

```bash
#!/bin/bash

# Convert PHP files from GBK to UTF-8

SOURCE_DIR="/path/to/poketb.com/bbs"
TARGET_DIR="/path/to/modern-php/app"

# Create target directory
mkdir -p "$TARGET_DIR"

# Find all PHP files
find "$SOURCE_DIR" -name "*.php" -type f | while read -r file; do
    # Get relative path
    rel_path="${file#$SOURCE_DIR/}"
    target_file="$TARGET_DIR/$rel_path"

    # Create target directory
    mkdir -p "$(dirname "$target_file")"

    # Convert file
    iconv -f GBK -t UTF-8 "$file" > "$target_file"

    # Check for conversion errors
    if [ $? -ne 0 ]; then
        echo "⚠️  Conversion failed: $rel_path"
    else
        echo "✅ Converted: $rel_path"
    fi
done

echo "\n✅ PHP file conversion complete!"
```

**Execute:**
```bash
chmod +x scripts/convert-php-files.sh
./scripts/convert-php-files.sh 2>&1 | tee php_conversion_log.txt
```

---

### Step 2: Fix Character Encoding Issues

**Common Issues in PHP Files:**

**Issue 1: Hardcoded GBK strings**
```php
// Before (GBK encoded)
$message = '欢迎使用Discuz!';

// After (UTF-8 encoded, explicit)
$message = '欢迎使用Discuz!'; // File saved as UTF-8
```

**Issue 2: HTML meta tags**
```php
// Before
<meta http-equiv="Content-Type" content="text/html; charset=gbk">

// After
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
```

**Issue 3: Database connection charset**
```php
// Before
mysql_query("SET character_set_connection=gbk, character_set_results=gbk, character_set_client=binary");

// After
$pdo->exec("SET NAMES utf8mb4");
```

---

### Step 3: Update PHP String Functions

**Legacy Code (GBK-safe):**
```php
// String length in GBK
$length = strlen($string); // WRONG for GBK

// Substring in GBK
$substring = substr($string, 0, 10); // WRONG - may cut multi-byte character
```

**Modern Code (UTF-8):**
```php
// String length in UTF-8
$length = mb_strlen($string, 'UTF-8');

// Substring in UTF-8
$substring = mb_substr($string, 0, 10, 'UTF-8');

// String position in UTF-8
$position = mb_strpos($string, 'search', 0, 'UTF-8');
```

**Automated Fix Script:** `scripts/fix-string-functions.php`

```php
<?php
declare(strict_types=1);

/**
 * Fix string functions for UTF-8
 */

function fixPhpFile(string $filePath): void
{
    $content = file_get_contents($filePath);

    // Replace strlen with mb_strlen
    $content = preg_replace('/\bstrlen\s*\(/i', 'mb_strlen($1, \'UTF-8\')', $content);

    // Replace substr with mb_substr
    $content = preg_replace('/\bsubstr\s*\(/i', 'mb_substr($1, \'UTF-8\')', $content);

    // Replace strpos with mb_strpos
    $content = preg_replace('/\bstrpos\s*\(/i', 'mb_strpos($1, \'UTF-8\')', $content);

    // Add mbstring declaration if needed
    if (strpos($content, 'mb_') !== false && strpos($content, 'declare(strict_types=1)') !== false) {
        // File already has strict types, good
    }

    file_put_contents($filePath, $content);
}

// Process all PHP files
$directory = new RecursiveDirectoryIterator('/path/to/modern-php/app');
foreach (new RecursiveIteratorIterator($directory) as $file) {
    if ($file->getExtension() === 'php') {
        fixPhpFile($file->getPathname());
        echo "Fixed: {$file->getPathname()}\n";
    }
}
```

---

## Phase 3: Template Conversion (Days 11-15)

### Step 1: Convert HTML Templates

**Script:** `scripts/convert-templates.sh`

```bash
#!/bin/bash

TEMPLATE_DIR="/path/to/poketb.com/bbs/templates"
TARGET_DIR="/path/to/modern-php/resources/views"

find "$TEMPLATE_DIR" -name "*.htm" -type f | while read -r file; do
    rel_path="${file#$TEMPLATE_DIR/}"
    target_file="$TARGET_DIR/${rel_path%.htm}.blade.php"

    mkdir -p "$(dirname "$target_file")"

    # Convert from GBK to UTF-8
    iconv -f GBK -t UTF-8 "$file" > "$target_file"

    # Update charset meta tag
    sed -i 's/charset=gbk/charset=utf-8/gi' "$target_file"

    echo "Converted: $rel_path"
done
```

---

### Step 2: Update Template Syntax

**Legacy Discuz Template Syntax:**
```html
<!--{if $condition}-->
  <p>Content</p>
<!--{/if}-->

<!--{loop $array $value}-->
  <p>$value</p>
<!--{/loop}-->

<!--{template header}-->
```

**Modern Blade Syntax:**
```blade
@if($condition)
  <p>Content</p>
@endif

@foreach($array as $value)
  <p>{{ $value }}</p>
@endforeach

@include('header')
```

**Automated Migration:** `scripts/convert-templates.php`

```php
<?php
declare(strict_types=1);

function convertTemplateSyntax(string $content): string
{
    // Convert if statements
    $content = preg_replace('/<!--{if\s+(.+?)}-->/', '@if($1)', $content);
    $content = preg_replace('/<!--{else}-->/', '@else', $content);
    $content = preg_replace('/<!--{elseif\s+(.+?)}-->/', '@elseif($1)', $content);
    $content = preg_replace('/<!--{\/if}-->/', '@endif', $content);

    // Convert loops
    $content = preg_replace('/<!--{loop\s+(\S+)\s+(\S+)}-->/', '@foreach($1 as $2)', $content);
    $content = preg_replace('/<!--{loop\s+(\S+)\s+(\S+)\s+(\S+)}-->/', '@foreach($1 as $2 => $3)', $content);
    $content = preg_replace('/<!--{\/loop}-->/', '@endforeach', $content);

    // Convert includes
    $content = preg_replace('/<!--{template\s+(\S+)}-->/', '@include(\'$1\')', $content);

    // Convert variables
    $content = preg_replace('/\$\w+/', '{{$0}}', $content);

    return $content;
}
```

---

## Phase 4: Validation & Testing (Days 16-20)

### Step 1: Automated Tests

**Test Script:** `tests/utf8-validation.php`

```php
<?php
declare(strict_types=1);

use PHPUnit\Framework\TestCase;

class UTF8ValidationTest extends TestCase
{
    public function testDatabaseIsUtf8()
    {
        $result = DB::select("SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
                              FROM information_schema.SCHEMATA
                              WHERE SCHEMA_NAME = 'discuz'");

        $this->assertEquals('utf8mb4', $result[0]['DEFAULT_CHARACTER_SET_NAME']);
        $this->assertStringContainsString('utf8mb4', $result[0]['DEFAULT_COLLATION_NAME']);
    }

    public function testPhpFilesAreUtf8()
    {
        $files = glob('/path/to/app/**/*.php');

        foreach ($files as $file) {
            $content = file_get_contents($file);
            $this->assertTrue(mb_check_encoding($content, 'UTF-8'), "File $file is not valid UTF-8");
        }
    }

    public function testChineseCharactersDisplay()
    {
        $testString = '测试中文显示';
        $this->assertEquals('测试中文显示', $testString);
        $this->assertEquals(5, mb_strlen($testString, 'UTF-8'));
    }

    public function testEmojiSupport()
    {
        $emoji = '😀🎉';
        $this->assertTrue(mb_check_encoding($emoji, 'UTF-8'));
        $this->assertEquals(2, mb_strlen($emoji, 'UTF-8'));
    }
}
```

---

### Step 2: Manual Testing Checklist

**Display Testing:**
- [ ] Homepage displays Chinese characters correctly
- [ ] Forum names show correctly
- [ ] Thread titles are readable
- [ ] User profiles display properly
- [ ] Post content shows correctly

**Input Testing:**
- [ ] Can post in Chinese
- [ ] Can post with emoji
- [ ] Search works with Chinese
- [ ] Username registration in Chinese works

**Data Integrity:**
- [ ] No data loss after conversion
- [ ] No mojibake (garbled text)
- [ ] No truncated text
- [ ] Special characters work

---

## Common Problems & Solutions

### Problem 1: Mojibake (乱码)

**Symptom:** "å®¶ç"å®¶" instead of "家园"

**Cause:** Wrong charset declaration

**Solution:**
```php
// Ensure PHP header
header('Content-Type: text/html; charset=utf-8');

// Ensure HTML meta
<meta charset="utf-8">

// Ensure database connection
$pdo = new PDO('mysql:host=localhost;dbname=discuz;charset=utf8mb4');
```

---

### Problem 2: Mailformed UTF-8

**Symptom:** Warning: "Malformed UTF-8 characters"

**Cause:** Data not properly converted

**Solution:**
```php
// Clean malformed UTF-8
function cleanUtf8(string $string): string
{
    return mb_convert_encoding($string, 'UTF-8', 'UTF-8');
}
```

---

### Problem 3: File Upload Issues

**Symptom:** Uploaded filenames with Chinese characters become garbled

**Solution:**
```php
// Before
$filename = $_FILES['file']['name'];

// After - sanitize filename
$extension = pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION);
$filename = uniqid() . '.' . $extension;
```

---

## Rollback Plan

**If conversion fails:**

```bash
# Stop all services
systemctl stop nginx php-fpm

# Restore database backup
zcat backup_before_utf8_YYYYMMDD.sql.gz | mysql -u root -p discuz

# Restore files from backup
rm -rf /path/to/modern-php/app
cp -r /path/to/backup/app /path/to/modern-php/

# Restart services
systemctl start nginx php-fpm

echo "Rollback complete!"
```

---

## Success Criteria

✅ All tables use utf8mb4
✅ All PHP files are UTF-8 encoded
✅ All templates are UTF-8 encoded
✅ Chinese characters display correctly
✅ Emoji work
✅ No data loss
✅ No mojibake
✅ All tests pass

---

**Document Status:** ✅ Complete
**Next Document:** 09-testing-strategy.md
