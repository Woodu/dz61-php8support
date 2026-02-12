# P1 Core Features - Detailed Migration Guide

**Purpose:** Essential forum functionality needed for basic operations
**Priority:** CORE - Complete after P0, before P2
**Timeline:** Weeks 4-8 (25 working days)

---

## Overview

P1 features are the core functionality that makes a forum operational. Users can view forums, read threads, post replies, and manage their accounts. Without P1, the system works but isn't useful.

**P1 Features:**
1. Forum Navigation & Display (5 features)
2. Thread Management (5 features)
3. Posting System (6 features)
4. User Management (6 features)

---

## 1. Forum Navigation & Display

### 1.1 Forum Index

**Current File:** `/root/poketb-renew/poketb.com/bbs/index.php`

**Lines:** 1-300

**Current Code (PHP 4/5 style):**
```php
<?php
/*
    [Discuz!] (C)2001-2007 Comsenz Inc.
    This is NOT a freeware, use is subject to license terms

    $Id: index.php 13764 2008-05-19 03:14:01Z heyond $
*/

define('CURSCRIPT', 'index');

require_once './include/common.inc.php';
require_once DISCUZ_ROOT.'./include/forum.func.php';

$discuz_action = 1;

if($cacheindexlife && !$discuz_uid && $showoldetails != 'yes' && (!$_DCACHE['settings']['frameon'] || $_DCACHE['settings']['frameon'] && $_GET['frameon'] != 'yes') && empty($gid)) {

    $indexcache = getcacheinfo(0);

    if($timestamp - $indexcache['filemtime'] > $cacheindexlife) {
        @unlink($indexcache['filename']);
        define('CACHE_FILE', $indexcache['filename']);
        $styleid = $_DCACHE['settings']['styleid'];
    } elseif($indexcache['filename']) {
        @readfile($indexcache['filename']);
        $debug && debuginfo();
        $debug ? die('<script type="text/javascript">document.getElementById("debuginfo").innerHTML = " '.($debug ? 'Updated at '.gmdate("H:i:s", $indexcache['filemtime'] + 3600 * 8).', Processed in '.$debuginfo['time'].' second(s), '.$debuginfo['queries'].' Queries'.($gzipcompress ? ', Gzip enabled' : '') : '').'";</script>') : die();
    }
}

$validdays = $discuz_uid && !empty($groupexpiry) && $groupexpiry >= $timestamp ? ceil(($groupexpiry - $timestamp) / 86400) : 0;
if(isset($showoldetails)) {
    switch($showoldetails) {
        case 'no': dsetcookie('onlineindex', 0, 86400 * 365); break;
        case 'yes': dsetcookie('onlineindex', 1, 86400 * 365); break;
    }
} else {
    $showoldetails = false;
}

$currenttime = gmdate($timeformat, $timestamp + $timeoffset * 3600);
$lastvisittime = gmdate("$dateformat $timeformat", $lastvisit + $timeoffset * 3600);

$memberenc = rawurlencode($lastmember);
$newthreads = round(($timestamp - $lastvisit + 600) / 1000) * 1000;
$rsshead = $rssstatus ? ('<link rel="alternate" type="application/rss+xml" title="'.$bbname.'" href="'.$boardurl.'rss.php?auth='.$rssauth."\" />\n") : '';
$customtopics = '';
$hottagstatus = $tagstatus && $hottags && $_DCACHE['tags'];

$catlist = $forumlist = $sublist = $forumname = $collapseimg = $collapse = array();
$threads = $posts = $todayposts = $fids = $announcepm = 0;
$postdata = $historyposts ? explode("\t", $historyposts) : array();

foreach(array('forumlinks', 'birthdays') as $key) {
    if(!isset($_COOKIE['discuz_collapse']) || strpos($_COOKIE['discuz_collapse'], $key) === FALSE) {
        $collapseimg[$key] = 'collapsed_no.gif';
        $collapse[$key] = '';
    } else {
        $collapseimg[$key] = 'collapsed_yes.gif';
        $collapse[$key] = 'display: none';
    }
}

$gid = !empty($gid) ? intval($gid) : 0;
if(!$gid) {
    $announcements = '';
    if($_DCACHE['announcements']) {
        $readapmids = !empty($_DCOOKIE['readapmid']) ? explode('D', $_DCOOKIE['readapmid']) : array();
        foreach($_DCACHE['announcements'] as $announcement) {
            if(empty($announcement['groups']) || in_array($groupid, $announcement['groups'])) {
                if(empty($announcement['type'])) {
                    $announcements .= '<li><a href="announcement.php?id='.$announcement['id'].'">'.$announcement['subject'].
                        '<em>('.gmdate($dateformat, $announcement['starttime'] + $timeoffset * 3600).')</em></a></li>';
                } elseif($announcement['type'] == 1) {
                    $announcements .= '<li><a href="'.$announcement['message'].'" target="_blank">'.$announcement['subject'].
                        '<em>('.gmdate($dateformat, $announcement['starttime'] + $timeoffset * 3600).')</em></a></li>';
                }
            }
        }
    }
    unset($_DCACHE['announcements']);

    $sql = !empty($accessmasks) ?
                "SELECT f.fid, f.fup, f.type, f.name, f.threads, f.posts, f.todayposts, f.lastpost, f.inheritedmod, f.forumcolumns, f.simple, ff.description, ff.moderators, ff.icon, ff.viewperm, ff.redirect, a.allowview FROM {$tablepre}forums f
                    LEFT JOIN {$tablepre}forumfields ff ON ff.fid=f.fid
                    LEFT JOIN {$tablepre}access a ON a.uid='$discuz_uid' AND a.fid=f.fid
                    WHERE f.status>0 ORDER BY f.type, f.displayorder"
                : "SELECT f.fid, f.fup, f.type, f.name, f.threads, f.posts, f.todayposts, f.lastpost, f.inheritedmod, f.forumcolumns, f.simple, ff.description, ff.moderators, ff.icon, ff.viewperm, ff.redirect FROM {$tablepre}forums f
                    LEFT JOIN {$tablepre}forumfields ff USING(fid)
                    WHERE f.status>0 ORDER BY f.type, f.displayorder";

    $query = $db->query($sql);

    while($forum = $db->fetch_array($query)) {
        $forumname[$forum['fid']] = strip_tags($forum['name']);
        if($forum['type'] != 'group') {
            $threads += $forum['threads'];
            $posts += $forum['posts'];
            $todayposts += $forum['todayposts'];
        }
        $forum['redirect'] = dhtmlspecialchars($forum['redirect']);
        $forum['description'] = nl2br(dhtmlspecialchars($forum['description']));

        if($forum['type'] == 'group') {
            $forum['fid'] = $forum['fup'];
            $catlist[$forum['fid']] = $forum;
        } elseif($forum['type'] == 'forum') {
            $forumlist[$forum['fup']][$forum['fid']] = $forum;
        } elseif($forum['type'] == 'sub') {
            $forumlist[$forum['fup']][$forum['fid']] = $forum;
        }
    }

    // ... more processing ...
}

include template('index');
```

**Migrated Code (PHP 8.3):**
```php
<?php
declare(strict_types=1);

namespace Discuz\Controllers;

use Discuz\Http\Request;
use Discuz\Http\Response;
use Discuz\Auth\AuthService;
use Discuz\Cache\CacheRepository;
use Discuz\Database\Database;
use Discuz\View\ViewFactory;

/**
 * Forum Index Controller
 * Displays the main forum index page
 */
class ForumIndexController
{
    protected AuthService $auth;
    protected CacheRepository $cache;
    protected Database $db;
    protected ViewFactory $view;

    public function __construct(
        AuthService $auth,
        CacheRepository $cache,
        Database $db,
        ViewFactory $view
    ) {
        $this->auth = $auth;
        $this->cache = $cache;
        $this->db = $db;
        $this->view = $view;
    }

    /**
     * Display forum index
     */
    public function index(Request $request): Response
    {
        // Get current user
        $user = $this->auth->user();

        // Try to get from cache (for guests only)
        $cacheKey = 'forum_index_' . ($user ? 'user_' . $user->id : 'guest');
        $useCache = !$user && $this->cache->has('cacheindexlife');

        if ($useCache) {
            return $this->getCachedIndex($cacheKey);
        }

        // Load data
        $data = $this->getIndexData($request, $user);

        return $this->view->render('forum.index', $data);
    }

    /**
     * Get cached index page
     */
    protected function getCachedIndex(string $cacheKey): Response
    {
        $html = $this->cache->get($cacheKey);

        if (!$html) {
            // Regenerate cache
            $html = $this->regenerateIndexCache();
            $this->cache->put($cacheKey, $html, $this->cache->get('cacheindexlife') ?: 3600);
        }

        return response($html);
    }

    /**
     * Get index page data
     */
    protected function getIndexData(Request $request, ?User $user): array
    {
        // Get announcements
        $announcements = $this->getAnnouncements($user);

        // Get forums structure
        $forums = $this->getForumsStructure($user);

        // Get statistics
        $stats = $this->getForumStats();

        // Get birthdays
        $birthdays = $this->getBirthdays();

        // Get online users
        $online = $this->getOnlineUsers();

        return [
            'announcements' => $announcements,
            'categories' => $forums['categories'],
            'forums' => $forums['forums'],
            'stats' => $stats,
            'birthdays' => $birthdays,
            'online' => $online,
            'user' => $user,
            'show_online_details' => $request->cookie('onlineindex') === '1',
            'current_time' => now(),
        ];
    }

    /**
     * Get announcements for user
     */
    protected function getAnnouncements(?User $user): array
    {
        return $this->cache->remember('announcements', 3600, function() use ($user) {
            $query = $this->db->table('announcements')
                ->where('starttime', '<=', time())
                ->where(function($query) {
                    $query->where('endtime', '>=', time())
                        ->orWhereNull('endtime');
                })
                ->orderBy('displayorder', 'desc')
                ->orderBy('starttime', 'desc');

            // Filter by user group if logged in
            if ($user) {
                $query->where(function($query) use ($user) {
                    $query->whereNull('groups')
                        ->orWhereRaw("FIND_IN_SET(?, groups)", [$user->groupId]);
                });
            } else {
                $query->whereNull('groups');
            }

            return $query->fetchAll();
        });
    }

    /**
     * Get forums structure
     */
    protected function getForumsStructure(?User $user): array
    {
        return $this->cache->remember('forums_structure_' . ($user ? $user->id : 'guest'), 300, function() use ($user) {
            // Get all forums
            $query = $this->db->query("
                SELECT
                    f.fid, f.fup, f.type, f.name, f.status,
                    f.threads, f.posts, f.todayposts, f.lastpost,
                    f.inheritedmod, f.forumcolumns, f.simple,
                    f.displayorder,
                    ff.description, ff.moderators, ff.icon,
                    ff.viewperm, ff.redirect, ff.password
                FROM cdb_forums f
                LEFT JOIN cdb_forumfields ff USING(fid)
                WHERE f.status > 0
                ORDER BY f.type, f.displayorder
            ");

            $forums = [];
            $categories = [];

            while ($forum = $query->fetch()) {
                // Check permissions
                if (!$this->canViewForum($forum, $user)) {
                    continue;
                }

                $forum['description'] = nl2br(htmlspecialchars($forum['description']));
                $forum['has_password'] = !empty($forum['password']);

                if ($forum['type'] === 'group') {
                    $categories[$forum['fid']] = $forum;
                } elseif ($forum['type'] === 'forum') {
                    $forums['forums'][$forum['fup']][$forum['fid']] = $forum;
                } elseif ($forum['type'] === 'sub') {
                    $forums['subforums'][$forum['fup']][$forum['fid']] = $forum;
                }
            }

            return [
                'categories' => $categories,
                'forums' => $forums,
            ];
        });
    }

    /**
     * Check if user can view forum
     */
    protected function canViewForum(array $forum, ?User $user): bool
    {
        // Check basic status
        if ($forum['status'] <= 0) {
            return false;
        }

        // Check redirect
        if ($forum['redirect']) {
            return true; // Redirects are always viewable
        }

        // Check view permission
        if ($forum['viewperm']) {
            if (!$user) {
                return false;
            }

            $allowedGroups = explode("\t", $forum['viewperm']);
            if (!in_array($user->groupId, $allowedGroups)) {
                return false;
            }
        }

        return true;
    }

    /**
     * Get forum statistics
     */
    protected function getForumStats(): array
    {
        return $this->cache->remember('forum_stats', 3600, function() {
            return [
                'threads' => $this->db->table('threads')->sum('threads'),
                'posts' => $this->db->table('posts')->sum('posts'),
                'members' => $this->db->table('members')->count(),
                'last_member' => $this->db->table('members')
                    ->orderBy('uid', 'desc')
                    ->selectOne(),
            ];
        });
    }

    /**
     * Get today's birthdays
     */
    protected function getBirthdays(): array
    {
        return $this->cache->remember('birthdays_today', 3600, function() {
            $todayMonthDay = date('nd');

            return $this->db->table('memberfields')
                ->whereRaw("SUBSTRING(bday, 6, 5) = ?", [$todayMonthDay])
                ->leftJoin('members', 'memberfields.uid', '=', 'members.uid')
                ->select('members.username', 'members.uid')
                ->get();
        });
    }

    /**
     * Get online users
     */
    protected function getOnlineUsers(): array
    {
        return [
            'total' => $this->cache->remember('online_count', 60, function() {
                return $this->db->table('sessions')
                    ->where('lastactivity', '>', time() - 900)
                    ->count();
            }),
            'members' => $this->cache->remember('online_members', 60, function() {
                return $this->db->table('sessions')
                    ->where('lastactivity', '>', time() - 900)
                    ->where('uid', '>', 0)
                    ->leftJoin('members', 'sessions.uid', '=', 'members.uid')
                    ->select('members.uid', 'members.username', 'members.adminid')
                    ->orderBy('members.lastactivity', 'desc')
                    ->limit(50)
                    ->get();
            }),
        ];
    }

    /**
     * Regenerate index cache
     */
    protected function regenerateIndexCache(): string
    {
        ob_start();
        // Render view without user (guest)
        $data = $this->getIndexData(request(), null);
        echo $this->view->render('forum.index', $data)->getContent();
        return ob_get_clean();
    }
}
```

**Key Changes:**
- ✅ MVC pattern with controllers
- ✅ Separate data fetching logic
- ✅ Permission-based filtering
- ✅ Redis/file cache support
- ✅ Type hints throughout
- ✅ Prepared statements
- ✅ Cleaner template rendering
- ✅ Better separation of concerns

**Database Tables Involved:**
- `cdb_forums` - Forum information
- `cdb_forumfields` - Extended forum data
- `cdb_announcements` - Announcements
- `cdb_threads` - Thread counts
- `cdb_posts` - Post counts
- `cdb_members` - Member stats
- `cdb_memberfields` - Birthdays
- `cdb_sessions` - Online users
- `cdb_access` - Permission overrides

**Testing Checklist:**
- [ ] Index page loads
- [ ] Forums display in correct order
- [ ] Sub-forums display under parent
- [ ] Forum stats are accurate
- [ ] Announcements display for correct user groups
- [ ] Permission check works
- [ ] Online users count is correct
- [ ] Birthdays display correctly
- [ ] Guest cache works
- [ ] User-specific data loads
- [ ] Categories display correctly
- [ ] Redirect forums show correctly
- [ ] Password-protected forums are marked

**Dependencies:**
- P0: Core Bootstrap
- P0: Database Layer
- P0: Caching System
- P0: Authentication

---

### 1.2 Forum Display

**Current File:** `/root/poketb-renew/poketb.com/bbs/forumdisplay.php`

**Lines:** 1-500

**Key Logic to Migrate:**
1. Forum permissions checking
2. Thread listing with pagination
3. Sticky/normal thread separation
4. Thread filtering (by type, digest, etc.)
5. Forum password verification
6. Moderator detection
7. Sub-forum display
8. Thread icons and tags

**Migrated Controller Structure:**
```php
<?php
declare(strict_types=1);

namespace Discuz\Controllers;

use Discuz\Http\Request;
use Discuz\Http\Response;
use Discuz\Auth\AuthService;
use Discuz\Database\Database;

/**
 * Forum Display Controller
 */
class ForumDisplayController
{
    public function show(Request $request, int $forumId): Response
    {
        $forum = $this->loadForum($forumId);
        $this->checkPermission($forum);

        $threads = $this->getThreads(
            $forumId,
            $request->query('page', 1),
            $request->query('filter', ''),
            $request->query('sort', 'lastpost')
        );

        $subforums = $this->getSubforums($forumId);
        $moderators = $this->getModerators($forumId);

        return view('forum.display', [
            'forum' => $forum,
            'threads' => $threads,
            'subforums' => $subforums,
            'moderators' => $moderators,
        ]);
    }
}
```

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Forum loads with correct threads
- [ ] Pagination works
- [ ] Sticky threads appear first
- [ ] Permission check works
- [ ] Password-protected forums prompt
- [ ] Sub-forums display
- [ ] Moderators list shows
- [ ] Thread filters work
- [ ] Sorting works
- [ ] Thread icons display

**Dependencies:**
- Forum Index (1.1)
- Permissions System

---

### 1.3 Thread Viewing

**Current File:** `/root/poketb-renew/poketb.com/bbs/viewthread.php`

**Lines:** 1-600

**Key Features to Migrate:**
1. Thread + posts loading
2. Post pagination
3. Permission checks
4. Attachment display
5. User info for each post
6. Poll display
7. Thread payment (paid threads)
8. Signature display
9. Quick reply
10. New reply form

**Migrated Controller:**
```php
<?php
declare(strict_types=1);

namespace Discuz\Controllers;

class ViewThreadController
{
    public function show(Request $request, int $threadId): Response
    {
        $thread = $this->loadThread($threadId);
        $posts = $this->getPosts($threadId, $page);

        // Increment view count
        $this->incrementViews($threadId);

        return view('thread.view', [
            'thread' => $thread,
            'posts' => $posts,
            'pagination' => $this->buildPagination(...),
            'quick_reply' => $this->canReply($thread),
        ]);
    }
}
```

**Estimated Migration Effort:** 3 days

**Testing Checklist:**
- [ ] Thread loads correctly
- [ ] Posts display in order
- [ ] Pagination works
- [ ] View count increments
- [ ] Attachments display
- [ ] User signatures show
- [ ] Poll displays and works
- [ ] Quick reply submits
- [ ] Permission check works
- [ ] Paid threads gate correctly

**Dependencies:**
- Forum Display (1.2)
- Post Functions (2.3)

---

## 2. Thread Management

### 2.1 New Thread

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/newthread.inc.php`

**Lines:** 1-400

**Migrated Service:**
```php
<?php
declare(strict_types=1);

namespace Discuz\Services;

class ThreadService
{
    public function createThread(CreateThreadRequest $request): Thread
    {
        // Validate
        $this->validateForum($request->forumId);
        $this->validatePermissions($request->userId, $request->forumId);
        $this->validateFloodControl($request->userId);

        // Create thread
        $threadId = $this->db->table('threads')->insertGetId([
            'fid' => $request->forumId,
            'subject' => $request->subject,
            'authorid' => $request->userId,
            'author' => $request->username,
            'dateline' => time(),
            'lastpost' => time(),
            'lastposter' => $request->username,
            'displayorder' => 0,
            'digest' => 0,
            // ... more fields
        ]);

        // Create first post
        $this->createPost($threadId, $request);

        // Update forum stats
        $this->updateForumStats($request->forumId);

        // Update user stats
        $this->updateUserStats($request->userId);

        return $this->getThread($threadId);
    }
}
```

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Thread creates successfully
- [ ] First post is created
- [ ] Forum stats update
- [ ] User stats update
- [ ] Flood control works
- [ ] Permission check works
- [ ] Attachments upload
- [ ] Polls create correctly
- [ ] Drafts save
- [ ] Preview works

**Dependencies:**
- Thread Viewing (1.3)
- Post Functions (2.3)
- Authentication (P0)

---

### 2.2 Thread Polls

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/viewthread_poll.inc.php`

**Lines:** 1-200

**Estimated Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Poll displays in thread
- [ ] Voting works
- [ ] Multiple choice polls work
- [ ] Vote counts update
- [ ] Poll results show
- [ ] Permission check (can't vote twice)
- [ ] Poll expiration works

**Dependencies:**
- Thread Viewing (1.3)

---

### 2.3 Thread Payment

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/threadpay.inc.php`

**Lines:** 1-150

**Estimated Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Paid threads require payment
- [ ] Credits deduct correctly
- [ ] Payment record saves
- [ ] Authors earn credits
- [ ] Admin bypass works
- [ ] Refund works (if applicable)

**Dependencies:**
- Thread Viewing (1.3)
- Credit System

---

## 3. Posting System

### 3.1 Post Controller

**Current File:** `/root/poketb-renew/poketb.com/bbs/post.php`

**Lines:** 1-600

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] New thread works
- [ ] New reply works
- [ ] Edit post works
- [ ] Delete post works
- [ ] Attachments upload
- [ ] Preview works
- [ ] Draft saves
- [ ] Rate limiting works
- [ ] Permission checks work

**Dependencies:**
- New Thread (2.1)
- Thread Viewing (1.3)

---

### 3.2 New Reply

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/newreply.inc.php`

**Lines:** 1-350

**Estimated Migration Effort:** 1.5 days

**Testing Checklist:**
- [ ] Reply creates successfully
- [ ] Thread bump updates
- [ ] User post count increments
- [ ] Notifications send (if applicable)
- [ ] Quote reply works
- [ ] Attachments attach
- [ ] Flood control works

**Dependencies:**
- Post Controller (3.1)

---

### 3.3 Edit Post

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/editpost.inc.php`

**Lines:** 1-500

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] User can edit own post
- [ ] Edit time limit enforced
- [ ] Mod can edit any post
- [ ] Edit reason saves
- [ ] Edit history tracks
- [ ] Permission check works
- [ ] Attachments update
- [ ] First post edit updates thread subject

**Dependencies:**
- Post Controller (3.1)

---

### 3.4 Post Functions

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/post.func.php`

**Lines:** 1-1500

**Estimated Migration Effort:** 3 days

**Testing Checklist:**
- [ ] BBCode parsing works
- [ ] Smilies convert correctly
- [ ] HTML sanitization works
- [ ] Censoring works
- [ ] Keyword links work
- [ ] Word wrap works
- [ ] Draft functions work
- [ ] Attachment handling works

**Dependencies:**
- None (utility functions)

---

### 3.5 Attachment Upload

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/attachment.func.php`

**Lines:** 1-800

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Files upload successfully
- [ ] File size limits enforce
- [ ] File type checking works
- [ ] Thumbnails generate
- [ ] Image resizing works
- [ ] Storage paths work
- [ ] Permissions check
- [ ] Quota limits enforce
- [ ] Virus scanning (if enabled)

**Dependencies:**
- Post Functions (3.4)

---

### 3.6 DiscuzCode Parser

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/discuzcode.func.php`

**Lines:** 1-1200

**Estimated Migration Effort:** 3 days

**Testing Checklist:**
- [ ] Bold/italic/underline work
- [ ] Links auto-parse
- [ ] Images display
- [ ] Code blocks format
- [ ] Quote blocks format
- [ ] Lists format
- [ ] Tables format
- [ ] Nested tags work
- [ ] XSS protection works
- [ ] Sanitization is safe

**Dependencies:**
- None (utility functions)

---

## 4. User Management

### 4.1 Member Profile

**Current File:** `/root/poketb-renew/poketb.com/bbs/member.php`

**Lines:** 1-800

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Profile loads correctly
- [ ] User info displays
- [ ] Statistics show
- [ ] Recent posts list
- [ ] Contact info shows
- [ ] Privacy settings respected
- [ ] Admin sees all info
- [ ] Avatar displays

**Dependencies:**
- Authentication (P0)

---

### 4.2 Member List

**Current File:** `/root/poketb-renew/poketb.com/bbs/member.php:54-200`

**Estimated Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Member list loads
- [ ] Pagination works
- [ ] Sorting works
- [ ] Filtering works
- [ ] Search works
- [ ] Permission check works

**Dependencies:**
- Member Profile (4.1)

---

### 4.3 Online Users

**Current File:** `/root/poketb-renew/poketb.com/bbs/member.php:24-52`

**Estimated Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Online list loads
- [ ] Pagination works
- [ ] User actions show
- [ ] IP display works (for admins)
- [ ] Invisibility respected

**Dependencies:**
- Session Management (P0)

---

### 4.4 User CP

**Current File:** `/root/poketb-renew/poketb.com/bbs/memcp.php`

**Lines:** 1-400

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Profile updates save
- [ ] Avatar uploads
- [ ] Signature updates
- [ ] Password changes work
- [ ] Email changes verify
- [ ] Preferences save
- [ ] Privacy settings work

**Dependencies:**
- Member Profile (4.1)

---

### 4.5 User Registration

**Current File:** Various (in misc.php)

**Estimated Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Registration form loads
- [ ] Validation works
- [ ] Username uniqueness checks
- [ ] Email verification sends
- [ ] Password strength enforces
- [ ] CAPTCHA works
- [ ] Agreement acceptance works
- [ ] Welcome email sends
- [ ] User creates successfully
- [ ] Auto-login works

**Dependencies:**
- Authentication (P0)
- UCenter Integration (if used)

---

### 4.6 Avatar System

**Current File:** Various in uc_client/

**Estimated Migration Effort:** 1.5 days

**Testing Checklist:**
- [ ] Avatars upload
- [ ] Image resizing works
- [ ] Cropping works
- [ ] Avatar gallery works
- [ ] Gravatar support works
- [ ] Avatar permissions work

**Dependencies:**
- User CP (4.4)

---

## Summary: P1 Core Features

**Total Features:** 22 sub-features
**Estimated Time:** 4-5 weeks (25 working days)
**Dependencies:** P0 complete

### Week 4: Forum Navigation (Days 16-20)
- Day 16-17: Forum Index (1.1)
- Day 18-19: Forum Display (1.2)
- Day 20: Thread Viewing (1.3)

### Week 5: Thread Management (Days 21-25)
- Day 21-22: New Thread (2.1)
- Day 23: Thread Polls (2.2)
- Day 24: Thread Payment (2.3)
- Day 25: Testing & Bug Fixes

### Week 6: Posting System (Days 26-30)
- Day 26-27: Post Controller (3.1)
- Day 28: New Reply (3.2)
- Day 29-30: Edit Post (3.3)

### Week 7: Post Processing (Days 31-35)
- Day 31-33: Post Functions (3.4)
- Day 34-35: Attachment Upload (3.5)

### Week 8: BBCode & User Mgmt (Days 36-40)
- Day 36-38: DiscuzCode Parser (3.6)
- Day 39-40: Member Profile (4.1)

### Weeks 9-10: Remaining User Features (Days 41-50)
- Day 41-42: Member List (4.2)
- Day 43: Online Users (4.3)
- Day 44-45: User CP (4.4)
- Day 46-47: User Registration (4.5)
- Day 48-49: Avatar System (4.6)
- Day 50: P1 Testing & Bug Fixes

### Success Criteria
✅ Users can browse forums
✅ Users can read threads
✅ Users can post new threads
✅ Users can reply to threads
✅ Users can edit their posts
✅ Users can manage profiles
✅ Users can register accounts
✅ Attachments upload correctly
✅ BBCode renders properly
✅ All P1 tests pass

**Next Phase:** P2 Important Features (Document 03)
