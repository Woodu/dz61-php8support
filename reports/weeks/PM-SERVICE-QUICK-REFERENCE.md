# PrivateMessageService - Quick Reference Guide

## Constructor

```php
use Discuz\PM\Services\PrivateMessageService;
use Discuz\PM\Repositories\PrivateMessageRepository;
use Discuz\User\Repository\UserRepository;
use Discuz\Database\Connection;

$service = new PrivateMessageService(
    $pmRepository,      // PrivateMessageRepositoryInterface
    $userRepository,     // UserRepository
    $connection          // Connection
);
```

---

## Method Reference

### 1. sendPM(int $fromUid, SendPMDTO $dto): int

Send a private message to a recipient.

**Parameters**:
- `$fromUid` - Sender user ID
- `$dto` - SendPMDTO with toUsernames, subject, message, replyPmid

**Returns**: PM ID of sent message

**Throws**:
- `PMValidationException` - Recipient not found
- `PMBlockedException` - Sender blocked or blacklisted
- `PMLimitExceededException` - Daily limit exceeded

**Example**:
```php
$dto = new SendPMDTO('recipient_username', 'Subject', 'Message text');
$pmid = $service->sendPM($senderUid, $dto);
```

---

### 2. getPMList(int $uid, string $folder, int $page, int $perPage): array

Get paginated list of PMs for user.

**Parameters**:
- `$uid` - User ID
- `$folder` - 'inbox' or 'outbox'
- `$page` - Page number (1-based)
- `$perPage` - Items per page

**Returns**: Array of PMListDTO

**Throws**:
- `PMValidationException` - Invalid folder

**Example**:
```php
$inbox = $service->getPMList($uid, 'inbox', 1, 20);  // Page 1, 20 per page
$outbox = $service->getPMList($uid, 'outbox', 1, 20);
```

---

### 3. getConversation(int $pmid, int $uid): array

Get full conversation thread for a PM.

**Parameters**:
- `$pmid` - PM ID
- `$uid` - User ID (for authorization)

**Returns**: Array of PrivateMessage entities

**Throws**:
- `PMNotFoundException` - PM not found or unauthorized

**Side Effect**: Marks PM as read if user is recipient

**Example**:
```php
$thread = $service->getConversation($pmid, $uid);
foreach ($thread as $pm) {
    echo $pm->getMessage() . "\n";
}
```

---

### 4. deletePMs(int $uid, DeletePMDTO $dto): int

Delete multiple PMs.

**Parameters**:
- `$uid` - User ID
- `$dto` - DeletePMDTO with pmids array and hardDelete flag

**Returns**: Number of PMs deleted

**Throws**:
- `PMNotFoundException` - User not authorized

**Example**:
```php
$dto = new DeletePMDTO([1, 2, 3], false);  // soft delete
$deleted = $service->deletePMs($uid, $dto);

$hardDto = new DeletePMDTO([1, 2, 3], true);  // hard delete
$hardDeleted = $service->deletePMs($uid, $hardDto);
```

---

### 5. markPMAsRead(int $pmid, int $uid): bool

Mark a PM as read.

**Parameters**:
- `$pmid` - PM ID
- `$uid` - User ID

**Returns**: True on success

**Throws**:
- `PMNotFoundException` - PM not found or not recipient

**Example**:
```php
$service->markPMAsRead($pmid, $uid);
```

---

### 6. hasNewPMs(int $uid): bool

Check if user has new PMs.

**Parameters**:
- `$uid` - User ID

**Returns**: True if has new PMs

**Example**:
```php
if ($service->hasNewPMs($uid)) {
    echo "You have new messages!";
}
```

---

### 7. getBlacklist(int $uid): array

Get user's blacklist.

**Parameters**:
- `$uid` - User ID

**Returns**: Array of blocked users [uid, username]

**Example**:
```php
$blacklist = $service->getBlacklist($uid);
foreach ($blacklist as $user) {
    echo $user['username'] . "\n";
}
```

---

### 8. blacklistUser(int $uid, string $username): bool

Add a user to blacklist.

**Parameters**:
- `$uid` - User ID
- `$username` - Username to blacklist

**Returns**: True on success

**Throws**:
- `PMValidationException` - Cannot blacklist yourself or user not found

**Example**:
```php
$service->blacklistUser($uid, 'annoying_user');
```

---

### 9. unblacklistUser(int $uid, string $username): bool

Remove user from blacklist.

**Parameters**:
- `$uid` - User ID
- `$username` - Username to unblacklist

**Returns**: True on success

**Example**:
```php
$service->unblacklistUser($uid, 'forgiven_user');
```

---

### 10. getConversationPartners(int $uid, int $limit): array

Get list of users with recent conversations.

**Parameters**:
- `$uid` - User ID
- `$limit` - Maximum results (default 20)

**Returns**: Array of [uid, username, last_message_time]

**Example**:
```php
$partners = $service->getConversationPartners($uid, 20);
foreach ($partners as $partner) {
    echo "Chat with {$partner['username']}: {$partner['last_message_time']}\n";
}
```

---

### 11. sendBulkPMs(int $fromUid, array $usernames, string $subject, string $message): int

Send PMs to multiple users.

**Parameters**:
- `$fromUid` - Sender user ID
- `$usernames` - Array of usernames
- `$subject` - PM subject
- `$message` - PM message

**Returns**: Number of PMs successfully sent

**Throws**:
- `PMValidationException` - Empty recipient list

**Note**: Skips blocked users (doesn't throw exception)

**Example**:
```php
$sent = $service->sendBulkPMs(
    $adminUid,
    ['user1', 'user2', 'user3'],
    'Announcement',
    'System maintenance tonight'
);
echo "Sent to {$sent} users";
```

---

### 12. countPMs(int $uid, string $folder): int

Count PMs in a folder.

**Parameters**:
- `$uid` - User ID
- `$folder` - 'inbox' or 'outbox'

**Returns**: Number of PMs

**Throws**:
- `PMValidationException` - Invalid folder

**Example**:
```php
$inboxCount = $service->countPMs($uid, 'inbox');
$outboxCount = $service->countPMs($uid, 'outbox');
```

---

## Exception Handling

```php
use Discuz\PM\Exceptions\{
    PMLimitExceededException,
    PMBlockedException,
    PMNotFoundException,
    PMValidationException
};

try {
    $pmid = $service->sendPM($fromUid, $dto);
} catch (PMValidationException $e) {
    // Invalid input (user not found, etc.)
    error_log("Validation error: {$e->getMessage()}");
} catch (PMBlockedException $e) {
    // User blocked or blacklisted
    error_log("Blocked: {$e->getMessage()}");
} catch (PMLimitExceededException $e) {
    // Daily limit exceeded
    error_log("Limit exceeded: {$e->getMessage()}");
} catch (PMNotFoundException $e) {
    // PM not found or unauthorized
    error_log("Not found: {$e->getMessage()}");
}
```

---

## Configuration

### Daily PM Limit

Default: 100 PMs per day

To modify (if needed in future):

```php
// In PrivateMessageService class
private int $dailyPMLimit = 100;  // Adjust as needed
```

---

## Usage Patterns

### Send PM with Error Handling

```php
try {
    $dto = new SendPMDTO($recipient, $subject, $message);
    $pmid = $service->sendPM($senderUid, $dto);

    echo "PM sent successfully (ID: {$pmid})";
} catch (PMValidationException $e) {
    echo "Recipient not found";
} catch (PMBlockedException $e) {
    echo "Cannot send: blocked or blacklisted";
} catch (PMLimitExceededException $e) {
    echo "Daily limit reached";
}
```

### Display Inbox with Pagination

```php
$page = (int) ($_GET['page'] ?? 1);
$perPage = 20;

try {
    $pms = $service->getPMList($uid, 'inbox', $page, $perPage);

    foreach ($pms as $pmDto) {
        echo "From: {$pmDto->msgfrom}\n";
        echo "Subject: {$pmDto->subject}\n";
        echo "New: " . ($pmDto->isNew ? 'Yes' : 'No') . "\n";
        echo "---\n";
    }
} catch (PMValidationException $e) {
    echo "Invalid folder";
}
```

### View Conversation Thread

```php
try {
    $thread = $service->getConversation($pmid, $uid);

    echo "<div class='conversation'>";
    foreach ($thread as $pm) {
        $class = $pm->getMsgfromid() === $uid ? 'sent' : 'received';
        echo "<div class='message {$class}'>";
        echo "<strong>{$pm->getMsgfrom()}</strong>: ";
        echo htmlspecialchars($pm->getMessage());
        echo "</div>";
    }
    echo "</div>";
} catch (PMNotFoundException $e) {
    echo "Conversation not found";
}
```

### Manage Blacklist

```php
// Add to blacklist
if ($action === 'block') {
    try {
        $service->blacklistUser($uid, $username);
        echo "User blocked";
    } catch (PMValidationException $e) {
        echo $e->getMessage();  // "Cannot blacklist yourself"
    }
}

// Remove from blacklist
if ($action === 'unblock') {
    $service->unblacklistUser($uid, $username);
    echo "User unblocked";
}

// View blacklist
$blacklist = $service->getBlacklist($uid);
echo "<ul>";
foreach ($blacklist as $user) {
    echo "<li>{$user['username']}</li>";
}
echo "</ul>";
```

---

## Testing Examples

### Unit Test Example

```php
public function test_send_pm_succeeds(): void
{
    // Setup mocks
    $dto = new SendPMDTO('recipient', 'Subject', 'Message');

    $this->userRepository->expects($this->once())
        ->method('findByUsername')
        ->willReturn(['uid' => 2, 'username' => 'recipient']);

    $this->pmRepository->expects($this->once())
        ->method('isBlocked')
        ->willReturn(false);

    // Test
    $result = $this->service->sendPM(1, $dto);

    // Assert
    $this->assertEquals(123, $result);
}
```

---

## Database Queries Used

### Send PM

```sql
-- Get sender username
SELECT username FROM cdb_members WHERE uid = ?;

-- Create inbox PM
INSERT INTO uc_pms (msgfrom, msgfromid, msgtoid, folder, new, subject, dateline, message, delstatus, related, fromappid)
VALUES (?, ?, ?, 'inbox', 1, ?, ?, ?, 0, ?, 0);

-- Create outbox PM
INSERT INTO uc_pms (msgfrom, msgfromid, msgtoid, folder, new, subject, dateline, message, delstatus, related, fromappid)
VALUES (?, ?, ?, 'outbox', 0, ?, ?, ?, 0, ?, 0);

-- Add to new PM table
INSERT INTO uc_newpm (uid) VALUES (?);
```

### Check Blocked/Blacklist

```sql
SELECT COUNT(*) as count
FROM uc_friends
WHERE uid = ? AND friendid = ? AND delstatus = 2;
```

### Get Conversation Thread

```sql
SELECT pmid, msgfrom, msgfromid, msgtoid, folder, new, subject, dateline, message, delstatus, related, fromappid
FROM uc_pms
WHERE ((msgfromid = ? AND msgtoid = ?) OR (msgfromid = ? AND msgtoid = ?))
  AND delstatus IN (0, 1, 2)
ORDER BY dateline ASC;
```

---

**Last Updated**: 2026-02-14
**PHP Version**: 8.3
**Service Version**: 1.0
