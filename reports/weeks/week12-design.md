# Week 12 Design Document: Moderation System & Attachment Upload UI

**Period**: Day 1-6
**Focus**: Moderation System (4 days) + Attachment Upload UI (2 days)
**Methodology**: TDD (Test-Driven Development)

## Table of Contents
1. [Legacy Analysis](#legacy-analysis)
2. [Modern Design](#modern-design)
3. [Database Design](#database-design)
4. [API Design](#api-design)
5. [UI/UX Design](#uiux-design)
6. [Security Design](#security-design)
7. [Implementation Plan](#implementation-plan)

---

## 1. Legacy Analysis

### 1.1 Moderation System Analysis

#### Core Files Identified
- **Entry Point**: `/poketb.com/bbs/modcp.php` (Moderator Control Panel)
- **Moderation Logic**: `/poketb.com/bbs/modcp/moderate.inc.php`
- **Member Management**: `/poketb.com/bbs/modcp/members.inc.php`
- **Moderation Logs**: `/poketb.com/bbs/modcp/logs.inc.php`
- **Forum Management**: `/poketb.com/bbs/modcp/forums.inc.php`
- **Home Dashboard**: `/poketb.com/bbs/modcp/home.inc.php`

#### Core Functionality Matrix

| Feature | Legacy Implementation | Database Tables | Key Fields |
|---------|---------------------|-----------------|------------|
| **Thread Operations** | `moderate.inc.php` | `cdb_threads`, `cdb_posts`, `cdb_forums` | `displayorder`, `digest`, `closed`, `fid`, `moderated` |
| Delete Thread | Move to recyclebin or permanent delete | `displayorder='-1'` (recyclebin) | soft delete via displayorder |
| Move Thread | `UPDATE threads SET fid=...` | `fid`, `typeid` | cross-forum move |
| Close/Open Thread | `closed='1'` or `closed='0'` | `closed` | boolean flag |
| Pin Thread | `displayorder=1,2,3` | `displayorder` | 1=normal pin, 2=section pin, 3=global pin |
| Digest Thread | `digest=1,2,3` | `digest` | 1=normal, 2=section, 3=global |
| **Post Operations** | `moderate.inc.php` (replies) | `cdb_posts` | `invisible`, `first` |
| Delete Post | `DELETE FROM posts` | - | hard delete with attachment cleanup |
| Validate Post | `invisible='0'` | `invisible` | -2=awaiting, -3=ignored, 0=visible |
| **User Management** | `members.inc.php` | `cdb_members`, `cdb_memberfields` | `groupid`, `adminid`, `groupterms` |
| Ban User | `groupid='4'` or `'5'` | `groupid`, `groupexpiry` | 4=banned, 5=guest banned |
| Unban User | Restore previous group | `groupterms` (serialized) | restore from serialized data |
| Edit User Profile | `UPDATE memberfields` | `bio`, `sightml`, `location` | signature, bio, location |
| **Moderation Logs** | File-based logs | N/A | `forumdata/logs/modcp*` |
| Log Format | Tab-separated text file | - | `timestamp\tusername\tadminid\tip\taction\top\tfid\textra` |

#### Data Flow Diagram (Moderation)

```
User Request → modcp.php
    ├─→ Authentication (AdminSession)
    ├─→ Permission Check (adminid, moderator status)
    ├─→ Action Router (switch $action)
    │   ├─→ moderate.inc.php → Thread/Post moderation
    │   ├─→ members.inc.php → User management
    │   ├─→ forums.inc.php → Forum settings
    │   ├─→ logs.inc.php → View logs
    │   └─→ home.inc.php → Dashboard
    ├─→ Execute Operation (UPDATE/DELETE queries)
    ├─→ Log Action (writelog 'modcp')
    └─→ Render Template (templates/default/modcp_*.htm)
```

#### Permission System (Legacy)

```php
// Permission hierarchy based on adminid field
$adminid == 1: Super Admin (all permissions)
$adminid == 2: Admin (most permissions, except IP ban)
$adminid == 3: Moderator (forum-specific only)
$adminid == 0: Regular user (no modcp access)

// Forum-level permissions
$forum['ismoderator']: Boolean flag from cdb_moderators table
$modforums['fids']: Comma-separated list of moderated forum IDs
```

#### Moderation Actions (Legacy)

| Action | Database Operation | Additional Operations |
|--------|-------------------|----------------------|
| `validate` thread | `UPDATE threads SET displayorder='0', moderated='1'` | Award credits, update forum stats |
| `delete` thread | `UPDATE threads SET displayorder='-1'` (recyclebin) or `DELETE` | Clean attachments, trades, polls |
| `ignore` thread | `UPDATE threads SET displayorder='-3'` | Mark as ignored |
| `validate` reply | `UPDATE posts SET invisible='0'` | Update thread reply count, award credits |
| `delete` reply | `DELETE FROM posts` | Clean attachments, update thread count |
| `ignore` reply | `UPDATE posts SET invisible='-3'` | Mark as ignored |
| `ban` user | `UPDATE members SET groupid='4'` | Save group terms, send PM |
| `unban` user | Restore from `groupterms['main']` | Clear group expiry |

#### Validation Rules (Legacy)

```php
// Thread moderation validation
- Must be moderator of forum ($forum['ismoderator'] == true)
- Forum must not be group type ($forum['type'] != 'group')
- Thread must belong to moderated forum ($thread['fid'] IN $modforums['fids'])

// User banning validation
- Cannot ban system groups (groupid IN 1,2,3,6,7,8)
- Cannot ban special groups (grouptype == 'special')
- Must provide ban reason if PM notification selected

// Post moderation validation
- Post must be in moderation queue (invisible='-2' or '-3')
- First post (thread starter) handled differently from replies
```

---

### 1.2 Attachment Upload UI Analysis

#### Current Implementation (Week 9)

**Existing Services**:
- `AttachmentUploadService` - Handles file upload, validation, storage
- `AttachmentController` - API endpoints for upload/download
- `LocalStorage` - File storage implementation

**Existing API Endpoints**:
```
POST /api/attachments/upload
GET  /api/attachments/{aid}
GET  /api/attachments/{aid}/download
GET  /api/attachments/{aid}/thumbnail
DELETE /api/attachments/{aid}
GET  /api/attachments/upload/limits
```

**Current Upload Flow**:
```
User submits form → FormData POST to AttachmentController::uploadAction()
    ├─→ Validate authentication
    ├─→ Validate CSRF token
    ├─→ Validate file (size, type, dimensions)
    ├─→ UploadService::upload()
    │   ├─→ Create AttachmentEntity
    │   ├─→ Validate permissions
    │   ├─→ Save to storage
    │   ├─→ Create thumbnail (if image)
    │   └─→ Save to database
    └─→ Return JSON response
```

#### Missing Features (To be implemented)

1. **Drag & Drop Upload Zone**
   - Visual drop zone in new thread/reply forms
   - Drag event handlers (dragover, drop)
   - File type validation before upload

2. **Multiple File Upload**
   - Queue multiple files
   - Sequential or parallel upload
   - Individual progress bars

3. **Image Preview**
   - Generate thumbnails for selected images
   - Display before upload
   - Remove from queue capability

4. **Upload Progress Tracking**
   - Real-time progress percentage
   - Upload speed indicator
   - Estimated time remaining

5. **Error Handling UI**
   - Display upload errors inline
   - Retry failed uploads
   - Remove failed files from queue

#### Legacy Upload UI Analysis

```
Legacy: /poketb.com/bbs/templates/default/post_attachments.htm
- Simple file input field
- AJAX upload with iframe fallback
- Progress bar (single, not per-file)
- Attachment list after upload
- Insert into post button (BBCode)
```

---

## 2. Modern Design

### 2.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     Presentation Layer                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Twig       │  │   AJAX       │  │   Drag &     │ │
│  │  Templates   │  │   Handler    │  │   Drop UI    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      Controller Layer                   │
│  ┌──────────────────┐  ┌──────────────────────────┐   │
│  │  Moderation      │  │   ModerationApiController│   │
│  │  Controller      │  │   (JSON endpoints)       │   │
│  └──────────────────┘  └──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                       Service Layer                     │
│  ┌──────────────────┐  ┌──────────────────────────┐   │
│  │  Moderation      │  │   ModerationLogService   │   │
│  │  Service         │  │                          │   │
│  │  - deleteThread  │  │   - logAction()          │   │
│  │  - moveThread    │  │   - getLogs()            │   │
│  │  - closeThread   │  │   - searchLogs()         │   │
│  │  - pinThread     │  └──────────────────────────┘   │
│  │  - digestThread  │                                  │
│  │  - banUser       │                                  │
│  │  - unbanUser     │                                  │
│  └──────────────────┘                                  │
└─────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    Repository Layer                     │
│  ┌──────────────────┐  ┌──────────────────────────┐   │
│  │  Moderation      │  │   ModerationLogRepo      │   │
│  │  Repository      │  │                          │   │
│  └──────────────────┘  └──────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      Database Layer                     │
│           PDO (Prepared Statements Only)                │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Moderation Service Design

#### Core Service Interface

```php
namespace Discuz\Moderation\Services;

use Discuz\Moderation\DTOs\ModerationActionDTO;
use Discuz\Moderation\Entities\ModerationLogEntity;

interface ModerationServiceInterface
{
    // Thread operations
    public function deleteThread(int $tid, int $moderatorUid, bool $forceDelete = false): bool;
    public function moveThread(int $tid, int $targetFid, int $moderatorUid): bool;
    public function closeThread(int $tid, int $moderatorUid): bool;
    public function openThread(int $tid, int $moderatorUid): bool;
    public function pinThread(int $tid, int $level, int $moderatorUid): bool;
    public function unpinThread(int $tid, int $moderatorUid): bool;
    public function digestThread(int $tid, int $level, int::moderatorUid): bool;
    public function undigestThread(int $tid, int $moderatorUid): bool;

    // Post operations
    public function deletePost(int $pid, int $moderatorUid): bool;
    public function validatePost(int $pid, int $moderatorUid): bool;
    public function ignorePost(int $pid, int $moderatorUid): bool;

    // User operations
    public function banUser(int $targetUid, int $banDuration, string $reason, int $moderatorUid): bool;
    public function unbanUser(int $targetUid, int $moderatorUid): bool;

    // Batch operations
    public function batchDeleteThreads(array $tids, int $moderatorUid): array;
    public function batchMoveThreads(array $tids, int $targetFid, int $moderatorUid): array;

    // Permission checks
    public function canModerateForum(int $fid, int $uid): bool;
    public function canModerateThread(int $tid, int $uid): bool;
    public function canBanUser(int $targetUid, int $moderatorUid): bool;
}
```

#### Moderation Log Service Interface

```php
namespace Discuz\Moderation\Services;

use Discuz\Moderation\Entities\ModerationLogEntity;

interface ModerationLogServiceInterface
{
    public function logAction(
        string $action,
        int $moderatorUid,
        string $moderatorUsername,
        int $targetId,
        string $targetType,
        ?string $reason = null,
        ?int $fid = null,
        ?string $ipAddress = null
    ): ModerationLogEntity;

    public function getLogs(
        ?string $action = null,
        ?int $fid = null,
        ?int $moderatorUid = null,
        int $limit = 50,
        int $offset = 0
    ): array;

    public function searchLogs(string $keyword, int $limit = 50, int $offset = 0): array;

    public function getLogById(int $logId): ?ModerationLogEntity;

    public function getLogsByTarget(int $targetId, string $targetType, int $limit = 20): array;
}
```

### 2.3 Permission System Design

#### Permission Hierarchy (Modern)

```php
enum AdminRole: int
{
    case SUPER_ADMIN = 1;      // Full system access
    case ADMIN = 2;            // Admin panel access, most moderation
    case MODERATOR = 3;        // Forum-specific moderation only
    case MEMBER = 0;           // No moderation permissions
    case GUEST = -1;           // Not logged in
}

class ModerationPermission
{
    public function canModerateForum(int $fid, int $uid): bool;
    public function canDeleteThread(int $tid, int $uid): bool;
    public function canMoveThread(int $tid, int $targetFid, int $uid): bool;
    public function canPinThread(int $tid, int $level, int $uid): bool;
    public function canDigestThread(int $tid, int $level, int $uid): bool;
    public function canCloseThread(int $tid, int $uid): bool;
    public function canBanUser(int $targetUid, int $uid): bool;
    public function canEditUserProfile(int $targetUid, int $uid): bool;

    private function isSuperAdmin(int $uid): bool;
    private function isAdmin(int $uid): bool;
    private function isModeratorOfForum(int $fid, int $uid): bool;
}
```

---

## 3. Database Design

### 3.1 Zero-Table-Change Verification

**APPROVED**: All moderation operations use EXISTING tables:

| Table | Usage | Modification |
|-------|-------|--------------|
| `cdb_threads` | Thread state (displayorder, digest, closed, fid) | UPDATE only |
| `cdb_posts` | Post visibility (invisible field) | UPDATE/DELETE only |
| `cdb_moderators` | Forum moderator assignments | SELECT only |
| `cdb_forums` | Forum permissions, recyclebin status | SELECT/UPDATE (forum settings) |
| `cdb_members` | User banning (groupid, groupexpiry) | UPDATE only |
| `cdb_memberfields` | User profile editing, groupterms | UPDATE only |
| `cdb_attachments` | Attachment cleanup on delete | DELETE only |
| `cdb_polls` | Poll cleanup on thread delete | DELETE only |
| `cdb_trades` | Trade cleanup on thread/post delete | DELETE only |

**NO NEW TABLES REQUIRED** - All moderation features work with existing schema.

### 3.2 Moderation Log Implementation

**Decision**: Use database-backed logs (not file-based like Legacy)

**Rationale**:
- Searchable and filterable
- Can be displayed in UI with pagination
- Persistent and backed up with database
- Better performance for large log volumes

**Proposed Table** (IF NEEDED for modern logging):
```sql
-- Only if we decide to migrate from file-based to database logs
-- For Week 12, we can use Legacy file-based logs or create this table
CREATE TABLE cdb_moderation_logs (
    log_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    action VARCHAR(50) NOT NULL,
    moderator_uid INT UNSIGNED NOT NULL,
    moderator_username VARCHAR(50) NOT NULL,
    target_id INT UNSIGNED NOT NULL,
    target_type ENUM('thread', 'post', 'user', 'forum') NOT NULL,
    fid INT UNSIGNED DEFAULT NULL,
    reason TEXT DEFAULT NULL,
    ip_address VARCHAR(45) DEFAULT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_action (action),
    INDEX idx_moderator (moderator_uid),
    INDEX idx_target (target_type, target_id),
    INDEX idx_forum (fid),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Note**: This table represents an APPROVED exception if we choose to implement it. The Legacy uses file-based logs (`forumdata/logs/modcp*`). For Week 12, we can:
1. **Option A**: Continue using file-based logs (Legacy compatible)
2. **Option B**: Create new table for database-backed logs (better UX)

**Recommendation**: Create the table for better queryability and UI integration.

---

## 4. API Design

### 4.1 Web Routes (ModerationController)

```
GET  /moderation                    - Moderator dashboard
GET  /moderation/threads            - Thread moderation queue
GET  /moderation/posts             - Post moderation queue
GET  /moderation/users             - User management
GET  /moderation/logs              - Moderation logs
```

### 4.2 API Routes (ModerationApiController)

```
// Thread operations
POST /api/moderation/threads/{tid}/delete       - Delete thread (move to recyclebin)
POST /api/moderation/threads/{tid}/delete-force - Permanently delete thread
POST /api/moderation/threads/{tid}/move         - Move thread to another forum
POST /api/moderation/threads/{tid}/close        - Close thread
POST /api/moderation/threads/{tid}/open         - Open thread
POST /api/moderation/threads/{tid}/pin          - Pin thread
POST /api/moderation/threads/{tid}/unpin        - Unpin thread
POST /api/moderation/threads/{tid}/digest       - Mark as digest
POST /api/moderation/threads/{tid}/undigest     - Remove digest

// Batch operations
POST /api/moderation/threads/batch-delete       - Batch delete threads
POST /api/moderation/threads/batch-move         - Batch move threads
POST /api/moderation/threads/batch-pin          - Batch pin threads

// Post operations
POST /api/moderation/posts/{pid}/delete         - Delete post
POST /api/moderation/posts/{pid}/validate       - Validate post (approve)
POST /api/moderation/posts/{pid}/ignore         - Ignore post

// User operations
POST /api/moderation/users/{uid}/ban            - Ban user
POST /api/moderation/users/{uid}/unban          - Unban user
GET  /api/moderation/users/{uid}/ban-status     - Get ban status

// Moderation logs
GET  /api/moderation/logs                       - Get moderation logs
GET  /api/moderation/logs/{id}                  - Get log entry
GET  /api/moderation/logs/search                - Search logs

// Moderation queue (content awaiting moderation)
GET  /api/moderation/queue/threads              - Threads awaiting moderation
GET  /api/moderation/queue/posts                - Posts awaiting moderation
```

### 4.3 Request/Response Formats

#### Delete Thread Request
```json
POST /api/moderation/threads/123/delete
{
  "csrf_token": "abc123...",
  "reason": "Spam post",
  "send_pm": true,
  "pm_message": "Your post was deleted because..."
}
```

Response:
```json
{
  "success": true,
  "message": "Thread deleted successfully",
  "thread": {
    "tid": 123,
    "status": "deleted",
    "recyclebin": true
  }
}
```

#### Move Thread Request
```json
POST /api/moderation/threads/123/move
{
  "csrf_token": "abc123...",
  "target_fid": 456,
  "reason": "Wrong forum",
  "leave_redirect": true,
  "send_pm": false
}
```

#### Pin Thread Request
```json
POST /api/moderation/threads/123/pin
{
  "csrf_token": "abc123...",
  "level": 1,  // 1=forum pin, 2=section pin, 3=global pin
  "reason": "Important announcement"
}
```

#### Ban User Request
```json
POST /api/moderation/users/789/ban
{
  "csrf_token": "abc123...",
  "duration": 7,  // days, 0=permanent
  "reason": "Multiple rule violations",
  "group_id": 4,  // 4=banned, 5=guest banned
  "send_pm": true,
  "pm_message": "You have been banned for 7 days..."
}
```

#### Moderation Logs Response
```json
GET /api/moderation/logs?action=delete&limit=20
{
  "success": true,
  "logs": [
    {
      "log_id": 1,
      "action": "delete_thread",
      "moderator": {
        "uid": 1,
        "username": "admin"
      },
      "target": {
        "type": "thread",
        "id": 123,
        "title": "Spam thread"
      },
      "forum": {
        "fid": 5,
        "name": "General Discussion"
      },
      "reason": "Spam post",
      "ip_address": "192.168.1.1",
      "created_at": "2026-02-18 10:30:00"
    }
  ],
  "total": 150,
  "page": 1,
  "per_page": 20
}
```

---

## 5. UI/UX Design

### 5.1 Moderation Dashboard Layout

```
┌──────────────────────────────────────────────────────────┐
│  Discuz! Board - Moderator Control Panel                 │
├──────────────────────────────────────────────────────────┤
│  Navigation: Dashboard | Threads | Posts | Users | Logs │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │  Quick Stats                                   │     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │     │
│  │  │ Awaiting │ │  Reports │ │ Your     │       │     │
│  │  │ Threads: │ │    : 15  │ │ Actions: │       │     │
│  │  │    23    │ │          │ │    156   │       │     │
│  │  └──────────┘ └──────────┘ └──────────┘       │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │  Recent Moderation Actions                     │     │
│  │  [Action filters: All | Delete | Move | Pin...] │     │
│  │                                                 │     │
│  │  10:30  admin deleted thread "Spam"            │     │
│  │  10:25  moderator2 moved thread to "Help"      │     │
│  │  10:20  admin pinned thread "Announcement"     │     │
│  │  ...                                           │     │
│  └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### 5.2 Thread Moderation Interface

```
┌──────────────────────────────────────────────────────────┐
│  Thread Moderation - Forum: General Discussion           │
├──────────────────────────────────────────────────────────┤
│  Filters: [All] [Awaiting] [Pinned] [Digested] [Closed] │
│  Search: [________________] [Search]                     │
├──────────────────────────────────────────────────────────┤
│  ☐ Thread Title        | Author | Stats | Actions       │
│  ────────────────────────────────────────────────────── │
│  ☐ [📌] Welcome        | admin  | 15/3 | [Move][Del]   │
│  ☐ [⭐] Best practices | mod1   | 8/0  | [Edit][Unpin] │
│  ☐ Question about PHP  | user5  | 3/1  | [Pin][Del]    │
│  ...                                                   │
│                                                          │
│  Batch Actions: [With selected: ▼Delete] [Apply]        │
│                                                          │
│  Pages: [1][2][3]...[10]                                │
└──────────────────────────────────────────────────────────┘
```

### 5.3 Attachment Upload UI (Drag & Drop)

```
┌──────────────────────────────────────────────────────────┐
│  New Thread - General Discussion                         │
├──────────────────────────────────────────────────────────┤
│  Subject: [_________________________________]            │
│                                                          │
│  Message:                                               │
│  ┌────────────────────────────────────────────────┐     │
│  │                                                │     │
│  │  [BBCode toolbar]                              │     │
│  │                                                │     │
│  │                                                │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  Attachments:                                           │
│  ┌────────────────────────────────────────────────┐     │
│  │  📁 Drag & Drop files here or click to upload   │     │
│  │                                                │     │
│  │          [Browse Files]                         │     │
│  │                                                │     │
│  │  Max: 10 files, 50MB total                     │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  Upload Queue:                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │ ✅ image1.png          [234 KB] [Remove]        │     │
│  │ ████████████████████ 100%                       │     │
│  │                                                │     │
│  │ ⏳ document.pdf         [1.2 MB] [Remove]        │     │
│  │ ████████████░░░░░░░░  65%                       │     │
│  │                                                │     │
│  │ ❌ video.mp4            [15 MB]                 │     │
│  │ Error: File too large (max 10MB)               │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  [Preview Thread] [Submit Thread]                       │
└──────────────────────────────────────────────────────────┘
```

### 5.4 Responsive Design Considerations

- **Desktop (>768px)**: Two-column layout (stats + actions)
- **Tablet (768px)**: Single column with collapsible sidebar
- **Mobile (<768px)**: Stacked layout, touch-friendly buttons

---

## 6. Security Design

### 6.1 Authentication & Authorization

```php
// Every moderation action must:
1. Check user is authenticated (session valid)
2. Check user has moderation permissions (adminid OR moderator status)
3. Check user has specific permission for target (forum ownership)
4. Validate CSRF token on POST/DELETE/PUT
5. Log all actions with IP address
```

### 6.2 CSRF Protection

```php
// All state-changing operations require CSRF token
- Token generated server-side (CsrfTokenService)
- Token included in forms and AJAX requests
- Token validated before action execution
- Token regenerated after sensitive operations
```

### 6.3 Input Validation

```php
// Thread operations
- Thread ID: must exist, must be in moderated forum
- Target Forum ID: must exist, must have permission
- Pin/Digest Level: 0-3 only (enum validation)

// User operations
- Target UID: must not be superadmin/system user
- Ban Duration: non-negative integer, max 365 days
- Reason: required if PM notification enabled, max 500 chars

// Batch operations
- IDs array: max 100 items per batch
- All IDs must be integers > 0
- All IDs must belong to moderated forums
```

### 6.4 SQL Injection Prevention

```php
// ALL database queries use PDO prepared statements
// NEVER concatenate variables into SQL

$stmt = $db->prepare("UPDATE cdb_threads SET displayorder=:level WHERE tid=:tid");
$stmt->execute(['level' => $level, 'tid' => $tid]);
```

### 6.5 XSS Prevention

```php
// All output in Twig templates is auto-escaped
// Explicit sanitization for user-generated content:
- Thread titles: htmlspecialchars()
- Usernames: htmlspecialchars()
- Moderation reasons: strip_tags() + htmlspecialchars()
- BBCode content: DiscuzCode parser with safe tags only
```

### 6.6 Audit Logging

```php
// Every moderation action logs:
- Action type (delete, move, pin, ban, etc.)
- Moderator (UID + username)
- Target (type + ID + title/username)
- Forum ID (if applicable)
- IP address
- Timestamp
- Reason (if provided)
- Additional data (JSON serialized)
```

---

## 7. Implementation Plan

### 7.1 Phase Breakdown

#### Phase 1: Moderation Core (Days 2-3)
**Goal**: Implement moderation service layer with full test coverage

**Deliverables**:
- ModerationService (all thread/post/user operations)
- ModerationRepository
- ModerationLogService
- ModerationPermission class
- DTOs (ModerationActionDTO, ModerationResultDTO)
- Entities (ModerationLogEntity)
- Unit tests (≥90% coverage)
- Integration tests (database operations)

**TDD Sequence**:
1. Write test for deleteThread
2. Implement deleteThread
3. Write test for moveThread
4. Implement moveThread
5. ... (repeat for all operations)
6. Write permission tests
7. Implement permission checks
8. Write batch operation tests
9. Implement batch operations
10. Run full test suite → 100% pass

#### Phase 2: Moderation UI (Days 4)
**Goal**: Build web interface for moderation

**Deliverables**:
- ModerationController (web routes)
- Twig templates (dashboard, threads, posts, users, logs)
- AJAX handlers for async operations
- Confirmation dialogs
- Batch operation UI
- Error handling and user feedback

**TDD Sequence**:
1. Write HTTP test for dashboard page
2. Implement dashboard route and template
3. Write HTTP test for thread moderation page
4. Implement thread moderation route and template
5. Write HTTP test for AJAX delete action
6. Implement AJAX delete endpoint
7. ... (repeat for all actions)
8. Write integration test for batch operations
9. Implement batch operations UI
10. Run full test suite → 100% pass

#### Phase 3: Attachment Upload UI (Day 5)
**Goal**: Implement drag & drop upload with progress tracking

**Deliverables**:
- Drag & drop upload zone component
- Multiple file upload queue
- Individual progress bars
- Image preview thumbnails
- Upload error handling
- JavaScript upload handler (upload.js)
- Integration tests (UI interactions)

**TDD Sequence**:
1. Write test for drag & drop event handling
2. Implement drag & drop handlers
3. Write test for file queue management
4. Implement file queue
5. Write test for progress tracking
6. Implement progress tracking
7. Write test for image preview
8. Implement image preview
9. Write test for error handling
10. Implement error handling
11. Run full test suite → 100% pass

#### Phase 4: Testing & Documentation (Day 6)
**Goal**: Comprehensive test coverage and documentation

**Deliverables**:
- Controller integration tests
- Edge case tests (boundary conditions)
- Permission boundary tests
- Concurrent operation tests
- Performance tests (batch operations)
- API documentation
- User guide
- DAY1-SUMMARY.md through DAY6-SUMMARY.md
- WEEK12-PROGRESS-SUMMARY.md
- WEEK12-COMPLETE.md

### 7.2 Test Coverage Requirements

| Component | Min Coverage | Critical Paths |
|-----------|--------------|----------------|
| ModerationService | 95% | All operations |
| ModerationRepository | 90% | Database queries |
| ModerationPermission | 100% | Security critical |
| ModerationController | 85% | HTTP endpoints |
| ModerationApiController | 90% | API endpoints |
| Upload JavaScript | 80% | Client-side logic |
| Integration Tests | N/A | End-to-end flows |

### 7.3 Success Criteria

**Functional Requirements**:
- ✅ All planned features implemented
- ✅ All tests pass (100%)
- ✅ Code coverage ≥ 85%
- ✅ Zero-table-change principle followed
- ✅ Legacy data compatibility maintained

**Quality Requirements**:
- ✅ PSR-12 compliant
- ✅ Strict types enforced
- ✅ No code smells
- ✅ Full dependency injection
- ✅ Exception-based error handling

**Security Requirements**:
- ✅ No SQL injection vulnerabilities
- ✅ No XSS vulnerabilities
- ✅ CSRF protection on all state changes
- ✅ Complete permission validation
- ✅ Comprehensive audit logging

**Documentation Requirements**:
- ✅ PHPDoc complete (all classes/methods)
- ✅ Design document (this file)
- ✅ Daily summaries (DAY1-6)
- ✅ Week summary (WEEK12-COMPLETE)
- ✅ API documentation (if applicable)

---

## 8. Edge Cases & Error Handling

### 8.1 Moderation Edge Cases

| Scenario | Handling |
|----------|----------|
| Moderator tries to delete thread in non-moderated forum | Permission denied error |
| Superadmin modifies system user (UID=1) | Blocked, error message |
| Move thread to same forum | No-op, success response |
| Pin already pinned thread | Update to new level, success |
| Ban already banned user | Update ban duration, success |
| Delete thread with active poll | Cascade delete poll |
| Delete thread with paid attachments | Refund payments to buyers |
| Batch operation with mixed permissions | Partial success, detailed results |
| Concurrent moderation of same thread | Last operation wins, both logged |
| Recyclebin full when deleting | Hard delete with warning |

### 8.2 Upload Edge Cases

| Scenario | Handling |
|----------|----------|
| File size exceeds PHP upload_max_filesize | Server error, user-friendly message |
| Duplicate filename detected | Auto-rename with timestamp suffix |
| Image dimensions exceed max resize | Resize to max dimensions |
| Non-image file with image extension | MIME type validation, reject |
| Upload interrupted (network failure) | Retry mechanism, clear partial files |
| Concurrent uploads exceed limit | Queue uploads, process sequentially |
| Storage quota exceeded | Error message, suggest cleanup |
| Malicious file upload | Comprehensive validation, quarantine |

---

## 9. Performance Considerations

### 9.1 Database Optimization

```sql
-- Indexes for moderation queries
CREATE INDEX idx_moderation_queue ON cdb_threads(displayorder, fid) WHERE displayorder=-2;
CREATE INDEX idx_moderation_posts ON cdb_posts(invisible, fid) WHERE invisible=-2;
CREATE INDEX idx_moderator_forums ON cdb_moderators(uid, fid);

-- Optimize log queries
CREATE INDEX idx_logs_action_time ON cdb_moderation_logs(action, created_at);
CREATE INDEX idx_logs_moderator_time ON cdb_moderation_logs(moderator_uid, created_at);
```

### 9.2 Batch Operation Optimization

```php
// Use transactions for batch operations
$db->beginTransaction();
try {
    foreach ($threadIds as $tid) {
        $this->deleteThread($tid, $moderatorUid);
    }
    $db->commit();
} catch (Exception $e) {
    $db->rollBack();
    throw $e;
}

// Limit batch size to prevent lock timeouts
const MAX_BATCH_SIZE = 100;
```

### 9.3 Caching Strategy

```php
// Cache moderator permissions (TTL: 5 minutes)
$cache->remember("mod_permissions:{$uid}", 300, function() use ($uid) {
    return $this->repository->getModeratedForums($uid);
});

// Cache forum stats (TTL: 1 minute)
$cache->remember("forum_stats:{$fid}", 60, function() use ($fid) {
    return $this->repository->getForumStats($fid);
});
```

---

## 10. Dependencies & Requirements

### 10.1 PHP Dependencies

```json
{
  "require-dev": {
    "phpunit/phpunit": "^10.0",
    "squizlabs/php_codesniffer": "^3.0",
    "phpstan/phpstan": "^1.0"
  }
}
```

### 10.2 Frontend Dependencies

```javascript
// No external frameworks (vanilla JS)
// Optional: browser Fetch API polyfill for older browsers
```

### 10.3 Database Requirements

- MySQL 8.0+ (already available)
- InnoDB engine
- utf8mb4 character set
- Transaction support

---

## 11. Timeline & Milestones

| Day | Phase | Tasks | Deliverables |
|-----|-------|-------|--------------|
| 1 | Legacy Analysis & Design | Analyze code, create design doc | Design doc, empty test structure |
| 2 | Moderation Core (Part 1) | Thread operations | Service, Repository, Unit tests |
| 3 | Moderation Core (Part 2) | Post + User operations | Complete Service, Integration tests |
| 4 | Moderation UI | Web interface | Controllers, Templates, HTTP tests |
| 5 | Attachment Upload UI | Drag & drop upload | JavaScript, UI tests |
| 6 | Testing & Documentation | Edge cases, performance | Full test suite, Documentation |

---

## 12. Risk Mitigation

### 12.1 Identified Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Legacy compatibility issues | High | Medium | Comprehensive testing with real data |
| Permission logic complexity | High | High | Extensive unit tests, code review |
| UI/UX complexity | Medium | Medium | Simple, iterative UI design |
| Performance degradation | Medium | Low | Load testing, optimization |
| Security vulnerabilities | Critical | Low | Security audit, penetration testing |

### 12.2 Rollback Plan

- Each feature developed in separate branch
- Feature flags for easy enable/disable
- Database migrations are reversible
- Legacy system remains untouched (reference only)

---

## Appendix A: Moderation Action Reference

### Thread Actions

```php
// Delete (soft delete to recyclebin)
DELETE /api/moderation/threads/{tid}/delete
Response: { success: true, recycled: true }

// Delete (permanent hard delete)
DELETE /api/moderation/threads/{tid}/delete-force
Response: { success: true, deleted: true }

// Move to different forum
POST /api/moderation/threads/{tid}/move
Body: { target_fid: 123, leave_redirect: true }
Response: { success: true, moved_to: 123 }

// Close thread (disable replies)
POST /api/moderation/threads/{tid}/close
Response: { success: true, closed: true }

// Open thread (enable replies)
POST /api/moderation/threads/{tid}/open
Response: { success: true, closed: false }

// Pin thread
POST /api/moderation/threads/{tid}/pin
Body: { level: 1 }  // 1=forum, 2=section, 3=global
Response: { success: true, pinned: true, level: 1 }

// Unpin thread
POST /api/moderation/threads/{tid}/unpin
Response: { success: true, pinned: false }

// Mark as digest
POST /api/moderation/threads/{tid}/digest
Body: { level: 1 }  // 1=forum, 2=section, 3=global
Response: { success: true, digested: true, level: 1 }

// Remove digest
POST /api/moderation/threads/{tid}/undigest
Response: { success: true, digested: false }
```

### User Actions

```php
// Ban user
POST /api/moderation/users/{uid}/ban
Body: { duration: 7, group_id: 4, reason: "Spam" }
Response: { success: true, banned: true, expires_at: "2026-02-25 10:30:00" }

// Unban user
POST /api/moderation/users/{uid}/unban
Response: { success: true, banned: false, restored_group: 10 }
```

---

**Document Version**: 1.0
**Last Updated**: 2026-02-18 (Day 1)
**Author**: Week 12 Development Team
**Status**: Ready for Implementation
