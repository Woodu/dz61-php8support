# PM System Service Layer Implementation

## Overview

Implemented the **Private Message Service Layer** following TDD methodology and PSR-12 standards for the Discuz! 6.1F migration project.

**Status**: ✅ **COMPLETE** - All 115 tests passing

---

## Implementation Summary

### 1. Exception Classes (4 new files)

Created domain-specific exceptions in `/app/PM/Exceptions/`:

| Exception | Purpose | Thrown When |
|-----------|---------|--------------|
| `PMLimitExceededException` | Daily PM limit exceeded | User exceeds 100 PMs/day |
| `PMBlockedException` | User blocked/blacklisted | Recipient blocked sender or blacklisted |
| `PMNotFoundException` | PM not found/unauthorized | PM doesn't exist or user lacks access |
| `PMValidationException` | Validation failure | Invalid input data (folder, usernames, etc.) |

**Location**: `/root/poketb-renew/modern-php-migration-code/app/PM/Exceptions/`

---

### 2. Service Class: PrivateMessageService

**File**: `/root/poketb-renew/modern-php-migration-code/app/PM/Services/PrivateMessageService.php`

**Dependencies**:
- `PrivateMessageRepositoryInterface` - Data access
- `UserRepository` - User validation
- `Connection` - Database transactions

#### Implemented Methods (12 core methods)

| Method | Purpose | Business Rules |
|--------|---------|----------------|
| `sendPM()` | Send private message | Validate recipient, check blocks, enforce daily limit |
| `getPMList()` | Get PM list with pagination | Validate folder, calculate offset, return DTOs |
| `getConversation()` | Get full conversation thread | Verify access, mark as read, return thread |
| `deletePMs()` | Batch delete PMs | Verify ownership, soft/hard delete |
| `markPMAsRead()` | Mark PM as read | Verify recipient, update status |
| `hasNewPMs()` | Check for new PMs | Query uc_newpm table |
| `getBlacklist()` | Get user's blacklist | Return blocked users list |
| `blacklistUser()` | Add user to blacklist | Validate: not self, user exists |
| `unblacklistUser()` | Remove from blacklist | Delete blacklist record |
| `getConversationPartners()` | Get recent conversations | Return partners with last message time |
| `sendBulkPMs()` | Batch send PMs | Send to multiple users, skip blocked |
| `countPMs()` | Count PMs in folder | Validate folder, return count |

---

### 3. Repository Extensions

Extended `PrivateMessageRepositoryInterface` with 6 new methods:

```php
public function create(int $fromUid, int $toUid, string $subject, string $message, ?int $replyPmid = null): int;
public function isBlocked(int $fromUid, int $toUid): bool;
public function isBlacklisted(int $uid, int $blacklistedUid): bool;
public function countSentByUser(int $uid, int $seconds): int;
public function getBlacklist(int $uid): array;
public function addToBlacklist(int $uid, int $blacklistedUid): bool;
public function removeFromBlacklist(int $uid, int $blacklistedUid): bool;
```

**Implemented in**: `/root/poketb-renew/modern-php-migration-code/app/PM/Repositories/PrivateMessageRepository.php`

---

### 4. DTO Enhancement

Added `fromEntity()` factory method to `PMListDTO`:

```php
public static function fromEntity(PrivateMessage $pm): PMListDTO
```

**Location**: `/root/poketb-renew/modern-php-migration-code/app/PM/DTOs/PMListDTO.php`

---

## Test Coverage

### Test File

**Location**: `/root/poketb-renew/modern-php-migration-code/tests/Unit/PM/Services/PrivateMessageServiceTest.php`

### Test Results

```
✅ All 30 Service tests passing
✅ All 115 PM tests passing (DTOs + Entities + Repositories + Services)
```

#### Test Categories

| Category | Tests | Coverage |
|----------|--------|----------|
| **sendPM** | 5 | Recipient validation, block checking, daily limit |
| **getPMList** | 4 | Folder validation, pagination, inbox/outbox |
| **getConversation** | 3 | Authorization, mark as read, thread retrieval |
| **deletePMs** | 2 | Ownership verification, batch deletion |
| **markPMAsRead** | 3 | Authorization, recipient verification |
| **hasNewPMs** | 2 | True/false cases |
| **getBlacklist** | 1 | Retrieval |
| **blacklistUser** | 3 | Self-check, validation, success |
| **unblacklistUser** | 1 | Success |
| **getConversationPartners** | 1 | Retrieval |
| **sendBulkPMs** | 3 | Empty list, blocked users, success |
| **countPMs** | 2 | Folder validation, counting |

---

## Business Rules Implemented

### Send PM Validation Flow

1. **Parse usernames** - Split comma-separated recipients
2. **Validate recipient** - User must exist in cdb_members
3. **Check blocks** - Sender not blocked by recipient (uc_friends delstatus=2)
4. **Check blacklist** - Recipient not blacklisted sender
5. **Daily limit** - Max 100 PMs/day (configurable)
6. **Create records** - Inbox (recipient) + Outbox (sender)
7. **Transaction safety** - Rollback on error

### Blacklist Management

- **Storage**: `uc_friends` table with `delstatus = 2`
- **Cannot blacklist self** - Validation enforces this
- **Remove existing friendship** - If friend relationship exists
- **Query blacklist** - Join with cdb_members for user details

### Conversation Thread

- **Authorization** - User must be sender or recipient
- **Mark as read** - Auto-mark when recipient views
- **Thread retrieval** - Get all PMs between two users
- **Delstatus handling** - Exclude deleted (delstatus=3) messages

---

## Database Schema Used

### Primary Tables

**uc_pms** - Private messages
- `pmid` (PK), `msgfrom`, `msgfromid`, `msgtoid`
- `folder` ('inbox'/'outbox'), `new` (0/1)
- `subject`, `message`, `dateline`
- `delstatus` (0=none, 1=sender, 2=recipient, 3=both)
- `related`, `fromappid`

**uc_newpm** - New PM notifications
- `uid` (PK)

**uc_friends** - Friendships + Blacklist
- `uid`, `friendid`, `delstatus` (0=friend, 2=blocked)

**cdb_members** - User data
- `uid`, `username`, etc.

---

## Key Design Decisions

### 1. Repository Pattern
- **Why**: Separate data access from business logic
- **Benefit**: Testable, swappable implementations

### 2. DTO Pattern
- **Why**: Type-safe data transfer
- **Benefit**: Validation in constructor, immutable with `readonly`

### 3. Exception Hierarchy
- **Why**: Domain-specific error handling
- **Benefit**: Caller can catch specific exceptions

### 4. Transaction Safety
- **Why**: Atomic operations for complex actions
- **Benefit**: Data consistency (e.g., sendPM creates 2 records)

### 5. Blacklist in uc_friends
- **Why**: Legacy compatibility
- **Benefit**: Single table for friends + blacklist (delstatus flag)

---

## Performance Considerations

### Database Queries

1. **Prepared statements** - All queries use PDO prepared statements
2. **Index usage** - Queries leverage existing indexes (uid, friendid, delstatus)
3. **Pagination** - Limit/offset for large lists
4. **Single-query blacklist check** - COUNT(*) instead of SELECT *

### Caching Opportunities

- **User info** - Cache username → UID lookups
- **Blacklist** - Cache blacklist status per user
- **New PM count** - uc_newpm table acts as cache

---

## Security Measures

### SQL Injection Prevention

✅ **All queries use prepared statements**
- PDO parameter binding
- No string concatenation in SQL

### Authorization

✅ **Ownership verification**
- PM delete: User must be sender or recipient
- PM view: User must be in conversation
- Mark as read: Only recipient can mark

### Validation

✅ **Input sanitization**
- DTO constructors validate data
- Service layer enforces business rules
- Type hints prevent invalid types

---

## Code Quality Metrics

### PSR-12 Compliance

✅ **All files follow PSR-12**
- 4 spaces indentation
- Strict types enabled
- Proper naming conventions

### Type Safety

✅ **100% typed code**
```php
declare(strict_types=1);

// All parameters and return types typed
public function sendPM(int $fromUid, SendPMDTO $dto): int
```

### Documentation

✅ **Complete PHPDoc**
- Class-level documentation
- Method-level documentation
- Parameter and return type docs

---

## Legacy Analysis Integration

### Legacy Code Studied

**UC Client PM Controller**: `poketb.com/bbs/uc_client/control/pm.php`
**UC Client PM Model**: `poketb.com/bbs/uc_client/model/pm.php`

### Business Rules Preserved

| Legacy Feature | Modern Implementation |
|----------------|----------------------|
| Daily PM limit | `countSentByUser()` + `$dailyPMLimit` |
| Block checking | `isBlocked()` + `isBlacklisted()` |
| Blacklist (delstatus=2) | `addToBlacklist()` / `removeFromBlacklist()` |
| Threaded conversations | `getThread()` with `related` PMs |
| New PM notifications | `uc_newpm` table management |
| Soft delete | `delstatus` bit flags (1=sender, 2=recipient) |

---

## Integration Points

### Dependencies

```php
use Discuz\PM\Repositories\PrivateMessageRepositoryInterface;
use Discuz\PM\DTOs\SendPMDTO;
use Discuz\PM\DTOs\DeletePMDTO;
use Discuz\PM\DTOs\PMListDTO;
use Discuz\PM\Entities\PrivateMessage;
use Discuz\PM\Exceptions\PMLimitExceededException;
use Discuz\PM\Exceptions\PMBlockedException;
use Discuz\PM\Exceptions\PMNotFoundException;
use Discuz\PM\Exceptions\PMValidationException;
use Discuz\User\Repository\UserRepository;
use Discuz\Database\Connection;
```

### Usage Example

```php
// Send PM
$service = new PrivateMessageService($pmRepo, $userRepo, $db);
$dto = new SendPMDTO('recipient', 'Hello', 'Message text');
$pmid = $service->sendPM($senderUid, $dto);

// Get PM List
$pmList = $service->getPMList($uid, 'inbox', 1, 20);

// Get Conversation
$thread = $service->getConversation($pmid, $uid);

// Mark as Read
$service->markPMAsRead($pmid, $uid);
```

---

## Future Enhancements

### Potential Improvements

1. **Caching layer** - Redis cache for user info, blacklist
2. **Rate limiting** - More granular PM limits (per recipient)
3. **Attachments** - File attachments in PMs
4. **Encryption** - End-to-end encryption for sensitive PMs
5. **Search** - Full-text search across PMs
6. **Threading** - UI improvements for conversation view
7. **Push notifications** - Real-time PM notifications

---

## Testing Commands

### Run PM Service Tests

```bash
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/Unit/PM/Services/PrivateMessageServiceTest.php --testdox
```

### Run All PM Tests

```bash
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/Unit/PM/ --testdox
```

### With Coverage

```bash
docker exec -i discuz_modern_php php vendor/bin/phpunit --coverage-html coverage tests/Unit/PM/
```

---

## File Manifest

### Created Files (8)

```
app/PM/Exceptions/PMLimitExceededException.php
app/PM/Exceptions/PMBlockedException.php
app/PM/Exceptions/PMNotFoundException.php
app/PM/Exceptions/PMValidationException.php
app/PM/Services/PrivateMessageService.php
tests/Unit/PM/Services/PrivateMessageServiceTest.php
```

### Modified Files (3)

```
app/PM/Repositories/PrivateMessageRepositoryInterface.php (added 7 methods)
app/PM/Repositories/PrivateMessageRepository.php (implemented 7 methods)
app/PM/DTOs/PMListDTO.php (added fromEntity method)
```

---

## Completion Checklist

- ✅ Created 4 exception classes
- ✅ Implemented PrivateMessageService with 12 methods
- ✅ Extended repository with 7 new methods
- ✅ Added DTO factory method
- ✅ Wrote 30 unit tests (all passing)
- ✅ All 115 PM tests passing
- ✅ PSR-12 compliance
- ✅ Strict types enabled
- ✅ Complete PHPDoc
- ✅ Transaction safety
- ✅ SQL injection prevention
- ✅ Authorization checks
- ✅ Business rule enforcement

---

**Implementation Date**: 2026-02-14
**PHP Version**: 8.3
**Test Framework**: PHPUnit 10.5
**Code Style**: PSR-12
