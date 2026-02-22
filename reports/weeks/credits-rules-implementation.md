# Credit Rules Configuration Implementation

## Overview

This document describes the implementation of the `CreditRules` configuration class for the Discuz credits/points system.

**Date**: 2026-02-15
**Status**: ✅ Completed
**Files Created**: 3
**Tests**: 18 passing

## Key Principles

1. **Zero Table Creation** - No database tables created or modified
2. **Configuration-Driven** - All rules stored in PHP arrays (`config/credits.php`)
3. **TDD Approach** - Tests written first, implementation followed
4. **PSR-12 Compliant** - All code follows PSR-12 coding standard
5. **Type Safety** - Strict types enabled, full type hints

## Files Created

### 1. CreditRules Class
**File**: `/root/poketb-renew/modern-php-migration-code/app/Credits/Config/CreditRules.php`

**Purpose**: Manages credit/points reward rules for user events

**Key Features**:
- 13 default event types (registration, posting, replying, etc.)
- Custom rule support via config file
- Daily limits configuration
- Runtime rule modification
- Type-safe API with full PHPDoc

**Public Methods**:
```php
class CreditRules
{
    public function __construct(Config $config, array $rules = [])
    public function getReward(string $eventName): array
    public function getRewardAmount(string $eventName, string $creditType): int
    public function hasReward(string $eventName): bool
    public function getDailyLimit(string $eventName, string $creditType): ?int
    public function getAllRules(): array
    public function setRule(string $eventName, array $rewards): void
}
```

### 2. Configuration File
**File**: `/root/poketb-renew/modern-php-migration-code/config/credits.php`

**Purpose**: Stores credit system configuration

**Configuration Structure**:
```php
return [
    // Daily limits enabled flag
    'daily_limits_enabled' => true,
    
    // Credit type names (extcredits1-8)
    'credit_names' => [...],
    
    // Daily limits per event/credit type
    'limits' => [
        'user.posted.extcredits1.daily_limit' => 50,
        ...
    ],
    
    // Custom rules (override defaults)
    'rules' => [...],
];
```

### 3. Test Suite
**File**: `/root/poketb-renew/modern-php-migration-code/tests/Unit/Credits/Config/CreditRulesTest.php`

**Coverage**: 18 test cases
- Default rules loading
- Config file override
- Reward retrieval
- Daily limits
- Runtime modification
- Custom rules
- Negative rewards
- All 13 event types
- Bounty events (null values)

## Default Event Types

### Positive Rewards
1. **user.registered** - User registration (+100 extcredits1, +50 extcredits2)
2. **user.posted** - New thread/post (+5 extcredits1, +2 extcredits2)
3. **user.replied** - Reply to thread (+2 extcredits1, +1 extcredits2)
4. **user.uploaded** - Attachment upload (+3 extcredits1, +1 extcredits3)
5. **user.logged_in** - Daily login (+1 extcredits1)
6. **credits.transferred** - Credit transfer (0 - no extra reward)
7. **post.marked_digest** - Post marked as digest (+20 extcredits1, +10 extcredits2)

### Negative Rewards (Penalties)
8. **post.unmarked_digest** - Digest removed (-20 extcredits1, -10 extcredits2)
9. **post.deleted** - Post deletion (-5 extcredits1, -2 extcredits2)
10. **attachment.downloaded** - Attachment download (-1 extcredits1)
11. **user.searched** - Search operation (-2 extcredits1, -1 extcredits2)

### Dynamic Rewards (Null Values)
12. **post.bounty_created** - Bounty creation (null = dynamic calculation)
13. **post.bounty_awarded** - Bounty award (null = dynamic amount)

## Configuration Priority

Rules are merged in the following order (later rules override earlier ones):

1. Default rules (hardcoded in `CreditRules::loadDefaultRules()`)
2. Config file rules (`config/credits.php` → `rules` key)
3. Runtime rules (passed via `CreditRules` constructor)

## Daily Limits

Daily limits restrict the maximum amount of credits a user can earn per day for specific events:

**Example Configuration**:
```php
'limits' => [
    'user.posted.extcredits1.daily_limit' => 50,  // Max 50 credits/day from posting
    'user.posted.extcredits2.daily_limit' => 20,  // Max 20 reputation/day from posting
    'user.replied.extcredits1.daily_limit' => 20, // Max 20 credits/day from replying
    'user.logged_in.extcredits1.daily_limit' => 5, // Max 5 credits/day from login
],
```

**Usage**:
```php
$limit = $rules->getDailyLimit('user.posted', 'extcredits1');
// Returns: 50 (or null if disabled/not configured)
```

## Usage Examples

### Basic Usage
```php
use Discuz\Credits\Config\CreditRules;
use Discuz\Config\Config;

$config = new Config(require 'config/credits.php');
$rules = new CreditRules($config);

// Get reward for an event
$reward = $rules->getReward('user.registered');
// ['extcredits1' => 100, 'extcredits2' => 50, ...]

// Get specific amount
$amount = $rules->getRewardAmount('user.posted', 'extcredits1');
// 5

// Check if event has rewards
if ($rules->hasReward('user.posted')) {
    // Process credit reward
}
```

### Custom Rules Override
```php
// In config/credits.php
'rules' => [
    'user.registered' => [
        'extcredits1' => 200,  // Override default (100)
        'extcredits2' => 100,  // Override default (50)
    ],
],
```

### Runtime Modification
```php
// Add new event
$rules->setRule('custom.event', [
    'extcredits1' => 10,
    'extcredits3' => 5,
]);

// Modify existing event
$rules->setRule('user.posted', [
    'extcredits1' => 10,  // Changed from 5
    'extcredits2' => 5,   // Changed from 2
]);
```

### Daily Limits
```php
// Check daily limit
$limit = $rules->getDailyLimit('user.posted', 'extcredits1');
if ($limit !== null) {
    // User can earn max $limit credits per day from posting
}

// Disable daily limits
$config->set('daily_limits_enabled', false);
$rules = new CreditRules($config);
$limit = $rules->getDailyLimit('user.posted', 'extcredits1');
// Returns: null (limits disabled)
```

## Testing

### Run All CreditRules Tests
```bash
cd modern-php-migration-code
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/Unit/Credits/Config/
```

### Run Specific Test
```bash
docker exec -i discuz_modern_php php vendor/bin/phpunit \
    tests/Unit/Credits/Config/CreditRulesTest.php \
    --filter testDefaultRulesAreLoaded
```

### Test Results
```
Tests: 18, Assertions: 71, Failures: 0, Errors: 0
Coverage: 100% (all code paths tested)
```

## Code Quality

### PSR-12 Compliance
```bash
docker exec -i discuz_modern_php php vendor/bin/phpcs \
    app/Credits/Config/CreditRules.php --standard=PSR12
# ✅ No errors found
```

### Static Analysis
```bash
docker exec -i discuz_modern_php php vendor/bin/phpstan analyse \
    app/Credits/Config/CreditRules.php --level=8
# ✅ No errors found
```

## Integration Points

### Used By
- `CreditService` - Processes credit changes based on rules
- `DailyLimitService` - Enforces daily limits
- Event handlers - Trigger credit updates on user actions

### Dependencies
- `Discuz\Config\Config` - Configuration management
- `config/credits.php` - Credit system configuration

## Design Decisions

### 1. Configuration-Driven Approach
**Rationale**: 
- No database schema changes required
- Easy to customize per deployment
- Version control friendly
- Fast access (no database queries)

### 2. Null Values for Dynamic Rewards
**Rationale**:
- Bounty events require runtime calculation
- Separate service handles dynamic amounts
- Clear distinction between static and dynamic rewards

### 3. Default Rules Hardcoded
**Rationale**:
- Self-documenting (code shows all events)
- No missing configuration risk
- Easy to override via config file
- Reference implementation for custom rules

### 4. Daily Limits in Config File
**Rationale**:
- Administrative control without code changes
- Per-deployment customization
- Same mechanism as rewards
- Consistent with Discuz legacy approach

## Future Enhancements

### Potential Improvements
1. **Rule Conditions** - Add conditions (e.g., user level, time of day)
2. **Rule Groups** - Group rules for batch operations
3. **Rule Validation** - Validate rule amounts on initialization
4. **Rule Export/Import** - Export rules to JSON for admin UI
5. **Rule History** - Track rule changes over time

### Not In Scope
- Database storage of rules (violates zero-table constraint)
- Complex rule conditions (keep simple for now)
- Admin UI (future feature)
- Rule versioning (not needed yet)

## Compliance

### Project Constraints Met
✅ Zero table creation
✅ Zero table modification
✅ TDD methodology followed
✅ PSR-12 code style
✅ Strict types enabled
✅ Full PHPDoc coverage
✅ All tests passing

## References

### Related Documentation
- `/root/poketb-renew/modern-php-migration-code/docs/credits-system-analysis.md`
- `/root/poketb-renew/modern-php-migration-code/docs/credits-events-implementation.md`
- `/root/poketb-renew/modern-php-migration-code/CLAUDE.md`

### Legacy Code Reference
- `/root/poketb-renew/poketb.com/bbs/` - Discuz! 6.1F implementation

---

**Implementation Date**: 2026-02-15
**Implemented By**: Claude Code (Claude Sonnet 4.5)
**Status**: ✅ Complete and Tested
