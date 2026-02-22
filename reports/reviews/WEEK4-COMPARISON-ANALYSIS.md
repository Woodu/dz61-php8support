# Discuz! 6.1F vs Modern PHP 8.3 Migration - Week 4 Comparison

## 对比日期：2026-02-15

### 概述

本文档对比 Discuz! 6.1F 原始实现（poketb.com/bbs）与 Week 4 迁移实现（modern-php-migration-code）在以下方面的：
- 权限控制模型
- 论坛首页展示
- 版块显示与帖子列表
- 帖子查看与分页
- 发帖与回复功能
- 编辑与删除功能

---

## 一、权限控制模型对比

### 1.1 原始实现（Discuz! 6.1F）

#### 三层权限体系

**第一层：用户组全局权限（cdb_usergroups）**

数据库字段：
```sql
allowview      - 允许访问论坛
allowpost      - 允许发帖
allowreply     - 允许回复
allowpostpoll  - 允许发起投票
allowvote      - 允许投票
allowgetattach  - 允许下载附件
allowpostattach - 允许上传附件
```

代码检查顺序（newthread.inc.php: 54-66）：
```php
// 1. 首先检查用户组全局权限
if (!$discuz_uid &&
    ((!$forum['postperm'] && $allowpost) ||
     ($forum['postperm'] && forumperm($forum['postperm'])))) {
    showmessage('group_nopermission', NULL, 'NOPERM');
}
```

**第二层：板块级权限（cdb_forumfields）**

数据库字段：
```sql
viewperm       - 允许查看的用户组（逗号分隔）
postperm       - 允许发帖的用户组（逗号分隔）
replyperm      - 允许回复的用户组（逗号分隔）
getattachperm   - 允许下载附件的用户组（逗号分隔）
postattachperm - 允许上传附件的用户组（逗号分隔）
moderators     - 版主列表（逗号分隔的用户名）
password       - 版块密码（MD5）
edittime      - 编辑时间限制（秒数）
```

权限字段解析逻辑（forum.func.php: 370-405）：
```php
function forumperm($permstr) {
    global $groupid, $extgroupids;

    // 将当前用户的组ID放入数组
    $groupidarray = array($groupid);
    foreach(explode("\t", $extgroupids) as $extgroupid) {
        if(intval(trim($extgroupid))) {
            $groupidarray[] = $extgroupid;
        }
    }

    // 正则检查：组ID是否在权限字符串中
    // 格式：(^|\t)(组1|组2|组3)(\t|$)
    // $permstr 为空 → 允许所有组
    // $permstr 非空 → 检查当前组ID是否在其中
    return preg_match("/(^|\t)(".implode('|', $groupidarray).")(\t|$)/", $permstr);
}
```

板块权限检查（forum.func.php: 30-36）：
```php
if (!$forum['viewperm'] || ($forum['viewperm'] && forumperm($forum['viewperm'])) ||
    !empty($forum['allowview']) || (isset($forum['users']) && strstr($forum['users'], "\t$discuz_uid\t"))) {
    $forum['permission'] = 2;  // 需要密码
} elseif (!$forum['viewperm'] || ($forum['viewperm'] && forumperm($forum['viewperm']))) {
    $forum['permission'] = 1;  // 禁止访问
} else {
    $forum['permission'] = 1;  // 允许访问
}
```

**第三层：用户特定权限（cdb_access）**

数据库字段：
```sql
uid           - 用户ID
fid           - 板块ID
allowview     - 允许查看（0/1）
allowpost     - 允许发帖（0/1）
allowreply    - 允许回复（0/1）
allowgetattach - 允许下载附件（0/1）
allowpostattach - 允许上传附件（0/1）
adminuser     - 管理员标记
dateline      - 设置时间
```

权限检查逻辑（common.inc.php 中的SQL查询）：
```php
// 用户特定权限优先级最高
SELECT allowview, allowpost, allowreply
FROM cdb_access a
WHERE a.fid = $fid AND a.uid = $uid
```

**特权用户豁免**：
- 管理员（adminid=1,2,3）：绕过大部分权限检查
- 版主（moderators 字段）：板块级管理权限

#### 关键差异点

| 特性 | Legacy实现 | Modern实现 | 差异 |
|-----|----------|-----------|------|
| 全局组权限 | ✅ 检查 cdb_usergroups.allowpost | ❌ **未检查** | 缺失全局权限检查 |
| 板块权限解析 | ✅ Tab分隔的 extgroupids | ❌ **逗号分隔** | 解析格式错误 |
| 板块密码 | ✅ password + password 密码验证 | ❌ **未实现** | 缺失板块密码功能 |
| 版主判断 | ✅ moderators 字段（逗号分隔用户名） | ❌ **未检查** | 缺失版主权限 |
| 编辑时间限制 | ✅ edittime 字段（秒数） | ❌ **硬编码** | 使用固定时间限制 |
| 发帖积分消耗 | ✅ postcredits 字段 | ❌ **未检查** | 未验证积分消耗 |
| 扩展用户组 | ✅ extgroupids + forumperm() | ❌ **未实现** | 缺失扩展组检查 |

### 1.2 现代实现（ForumPermissionService）

**仅实现了两层权限**：

```php
// ForumPermissionService.php

public function canPostThread(int $forumId, int $userId, int $groupId): bool
{
    // ❌ 缺失：没有检查 cdb_usergroups.allowpost

    // 1. 检查用户特定权限（cdb_access）
    $userAccess = $this->getUserAccess($forumId, $userId);
    if ($userAccess !== null) {
        return (bool) $userAccess['allowpost'];
    }

    // 2. 检查板块权限（cdb_forumfields.postperm）
    return $this->checkForumPermission($forumId, $groupId, 'postperm');
}

private function checkForumPermission(int $forumId, int $groupId, string $permissionField): bool
{
    $sql = "SELECT {$permissionField} FROM cdb_forumfields WHERE fid = :fid";
    $row = $this->db->selectOne($sql, ['fid' => $forumId]);

    $permValue = $row[$permissionField] ?? '';

    // ❌ 错误：使用逗号分隔，而不是Tab分隔
    if ($permValue === '' || $permValue === null) {
        return true;
    }

    $allowedGroups = explode(',', $permValue);  // ❌ 应该是 explode("\t")
    return in_array((string)$groupId, $allowedGroups, true);
}
```

**缺失的关键功能**：

1. ❌ **未检查全局用户组权限**（cdb_usergroups）
2. ❌ **未检查扩展用户组**（extgroupids）
3. ❌ **未实现板块密码验证**（password）
4. ❌ **未检查版主身份**（moderators）
5. ❌ **未使用板块级编辑时间限制**（edittime）
6. ❌ **未使用发帖积分限制**（postcredits）
7. ❌ **权限字段分隔符错误**（逗号 vs Tab）

---

## 二、论坛首页（index.php）对比

### 2.1 原始实现（index.php: 1-380行）

**功能清单**：
1. ✅ 论坛缓存系统
2. ✅ 分类显示（group type）
3. ✅ 版块显示（forum type）
4. ✅ 子版块显示（sub type）
5. ✅ 板块折叠（collapsed cookie）
6. ✅ 版块图标（icon/flash）
7. ✅ 在线用户统计（onlinenum）
8. ✅ 公告显示（announcement）
9. ✅ 广告推荐（recommend）
10. ✅ 已访问论坛（visitedforums cookie）
11. ✅ RSS feed生成

**权限检查**（index.php: 82-106）：
```php
// SQL查询：LEFT JOIN cdb_forumfields ff ON ff.fid=f.fid
// LEFT JOIN cdb_access a ON a.fid=f.fid AND a.uid=$discuz_uid
// 返回字段：
//   f.viewperm, f.postperm, f.replyperm, ff.description, ff.moderators
//   a.allowview, a.allowpost, a.allowreply

if (!$forum['viewperm'] || ($forum['viewperm'] && forumperm($forum['viewperm'])) ||
    !empty($forum['allowview']) || (isset($forum['users']) && strstr($forum['users'], "\t$discuz_uid\t"))) {
    // 需要3级权限检查，但未完全实现
}
```

**板块折叠逻辑**（index.php: 124-130）：
```php
// 已访问论坛（visitedfid cookie）
if (isset($_DCOOKIE['visitedfid'])) {
    $visitedfids = explode('D', $_DCOOKIE['visitedfid']);
    foreach(explode('D', $_DCOOKIE['visitedfid']) as $fid) {
        if(isset($_DCOOKIE['fid'.$fid]) && $_DCOOKIE['fid'.$fid] > $lastvisit) {
            $forum['folder'] = ' class="new"';  // 新帖标记
        } else {
            $forum['folder'] = '';  // 无标记
        }
    }
}

// 子板块折叠状态（collapsed cookie）
if (substr($_DCOOKIE['collapsed'], 0, $prelength) == $cookiepre) {
    // 使用隐藏状态
} else {
    // 使用展开状态
}
```

### 2.2 现代实现（ForumController + ForumService）

**已实现功能**：
1. ✅ 论坛树形结构获取
2. ✅ 基本板块列表
3. ❌ 缺少分类折叠功能
4. ❌ 缺少板块折叠（collapsed cookie）
5. ❌ 缺少已访问论坛（visitedfid cookie）
6. ❌ 缺少公告显示
7. ❌ 缺少广告推荐
8. ❌ 缺少在线用户统计
9. ✅ 基本权限过滤

**缺失的关键功能**：

```php
// ❌ ForumController.php 缺少：
public function collapseAction(int $forumId): JsonResponse  // 折叠/展开板块
public function visitedAction(): JsonResponse          // 已访问论坛管理
public function markAllReadAction(): JsonResponse     // 全部标记已读
public function getAnnouncementsAction(): JsonResponse // 公告列表
public function getOnlineUsersAction(): JsonResponse  // 在线用户
public function getRecommendAction(): JsonResponse     // 推荐内容
```

---

## 三、版块显示与帖子列表（forumdisplay.php）对比

### 3.1 原始实现（forumdisplay.php: 1-390行）

**功能清单**：
1. ✅ 置顶/精华/普通帖分离
2. ✅ 帖子类型过滤（poll/reward/debate/trade/activity）
3. ✅ 排序方式（lastpost/dateline/replies/views）
4. ✅ 排序方向（asc/desc）
5. ✅ 分类板块子板块显示
6. ✅ 版块密码验证
7. ✅ 版块规则查看（rules）
8. ✅ 版块推荐显示（modrecommend）
9. ✅ 版块跳转菜单（forumjump）
10. ✅ 在线用户列表（whosonline）
11. ✅ 子板块折叠标记

**置顶帖分离逻辑**（forumdisplay.php: 195-250）：
```php
// 全局置顶（displayorder > 0）
$stickytids = $_DCACHE['globalstick']['global']['tids'] ? unserialize($_DCACHE['globalstick']['global']['tids']) : '';
$stickytids = array($stickytids);

if ($stickytids) {
    // 使用 index: displayorder DESC, displayorder > 0
    $sql = "WHERE displayorder IN (".implode(',', array_fill(0, count($stickytids) - 1, '?')).")";
    $query = $db->query("SELECT * FROM {$tablepre}threads t WHERE fid='$fid' AND $sql ORDER BY displayorder DESC, lastpost DESC LIMIT $start_limit, $ppp");
} else {
    // 使用 index: (fid, displayorder, lastpost)
    $sql = "WHERE fid='$fid'";
}

// 子板块置顶（displayorder > 0 且子板块）
if ($subexists && $forum['threadtypes']) {
    // 需要处理子板块的置顶帖
}

// 精华帖（digest > 0）
if ($filter == 'digest') {
    $sql = " AND digest>0";
}
```

**版块密码验证**（forumdisplay.php: 51-58）：
```php
if ($forum['password']) {
    if ($action == 'pwverify') {
        // 验证密码（MD5）
        if ($pw != $forum['password']) {
            showmessage('forum_passwd_correct', "forumdisplay.php?fid=$fid");
        }
        dsetcookie('fidpw'.$fid, $pw, 31536000);
    } elseif (!isset($_DCOOKIE['fidpw'.$fid]) || $_DCOOKIE['fidpw'.$fid] != $forum['password']) {
        // 显示密码输入表单
        include template('forumdisplay_passwd');
        exit();
    }
    }
}
```

### 3.2 现代实现（ThreadRepository + ThreadController）

**已实现功能**：
1. ✅ 基本帖子列表查询
2. ✅ 简单过滤（sticky/digest）
3. ✅ 排序（lastpost/dateline/replies/views）
4. ❌ 缺少全局置顶（globalstick）
5. ❌ 缺少置顶帖分离查询
6. ❌ 缺少版块密码验证
7. ❌ 缺少版块规则显示
8. ❌ 缺少子板块处理
9. ❌ 缺少论坛跳转菜单
10. ❌ 缺少在线用户列表

**ThreadRepository 查询逻辑差异**：

```php
// Modern 实现（ThreadRepository.php: 22-73）
public function findByForum(
    int $forumId,
    int $page = 1,
    int $perPage = 20,
    string $sort = 'lastpost',
    string $order = 'desc'
): array {
    // ❌ 问题：单次查询，未分离置顶帖
    $sql = "SELECT * FROM {$this->threadsTable}
            WHERE fid = :fid
            ORDER BY {$orderClause}
            LIMIT :offset, :limit";
    // ❌ 缺少：全局置顶处理
    // ❌ 缺少：displayorder 优先级
}

// Legacy 实现（forumdisplay.php: 195-250）
// 1. 全局置顶帖（使用全局缓存）
// 2. 普通置顶帖（displayorder > 0）
// 3. 普通帖子
// 分别查询，最后合并
```

**缺失的置顶帖处理**：
```php
// ❌ Modern ThreadRepository 缺少：
public function findGlobalSticky(array $stickyThreadIds): array;
public function findStickyByForumWithSubs(int $forumId): array;
public function findDigestByForumOrdered(int $forumId, int $limit): array;
```

---

## 四、帖子查看与分页（viewthread.php）对比

### 4.1 原始实现（viewthread.php: 1-250行）

**功能清单**：
1. ✅ 帖子信息获取（cdb_threads）
2. ✅ 帖子阅读权限检查（readperm）
3. ✅ 帖子查看计数（views++）
4. ✅ 帖子回复列表（分页显示）
5. ✅ 帖子类型特殊处理（poll/reward/debate/trade/activity）
6. ✅ 帖子订阅状态（subscribed）
7. ✅ 帖子关闭状态（closed）
8. ✅ 帖子最后发帖人（lastposter）
9. ✅ 帖子最后回复时间（lastpost）
10. ✅ 精华帖判断（digest > 0）
11. ✅ 投票帖内容（polldata）
12. ✅ 悬赏帖内容（rewarddata）
13. ✅ 辩论帖内容（debatedata）
14. ✅ 交易帖内容（tradedata）
15. ✅ 活动帖内容（activitydata）
16. ✅ BBCode解析（discuzcode.func.php）

**阅读权限检查**（viewthread.php + forum.func.php）：
```php
// viewthread.php 直接使用 forum() 函数获取论坛信息
$thread = $db->fetch_first("SELECT * FROM {$tablepre}threads t WHERE tid='$tid'");

// forum.func.php: 30-36 权限检查
if (!$forum['viewperm'] || $forum['viewperm'] && forumperm($forum['viewperm'])) ||
    !empty($forum['allowview']) || (isset($forum['users']) && strstr($forum['users'], "\t$discuz_uid\t"))) {
    $forum['permission'] = 2;  // 需要密码
}

// cdb_threads.readperm 字段（阅读权限）
// 0: 全部可读
// 1: 需要特定积分等级
// 2: 需要特定积分
// 3: 需要特定积分
// 空值: 不限制（但需要其他权限通过）
```

**帖子查看计数**（viewthread.php: 41-44）：
```php
// 立即更新（不使用缓存）
$db->query("UPDATE {$tablepre}threads SET views=views+1 WHERE tid='$tid'");
```

**帖子阅读标记**（viewthread.php: 33-40）：
```php
$oldtopics = isset($_DCOOKIE['oldtopics']) ? $_DCOOKIE['oldtopics'] : 'D';
if(strpos($oldtopics, 'D'.$tid.'D') === FALSE) {
    $oldtopics = 'D'.$tid.$oldtopics;
    if(strlen($oldtopics) > 3072) {
        $oldtopics = preg_replace("((D\d+)+D).*$", "\\1", substr($oldtopics, 0, 3072));
    }
    dsetcookie('oldtopics', $oldtopics, 3600);
}

// 标记论坛已读（forumn 范畴）
if ($lastvisit < $thread['lastpost'] && (!isset($_DCOOKIE['fid'.$fid]) || $thread['lastpost'] > $_DCOOKIE['fid'.$fid])) {
    dsetcookie('fid'.$fid, $thread['lastpost'], 3600);
}
```

### 4.2 现代实现（ThreadViewService + ThreadViewController）

**已实现功能**：
1. ✅ 帖子信息获取
2. ❌ 缺少阅读权限检查（readperm）
3. ✅ 帖子查看计数（incrementViews）
4. ✅ 帖子回复列表（分页）
5. ⚠️ 帖子类型处理不完整
6. ❌ 缺少帖子阅读标记（oldtopics cookie）
7. ❌ 缺少论坛已读标记（fid cookie）
8. ❌ 缺少 BBCode 解析（discuzcode.func.php）
9. ❌ 缺少投票帖数据
10. ❌ 缺少悬赏帖数据
11. ❌ 缺少辩论帖数据
12. ❌ 缺少交易帖数据
13. ❌ 缺少活动帖数据

**ThreadViewService 实现对比**：
```php
// Modern 实现（ThreadViewService.php）
public function getThreadWithPosts(
    int $threadId,
    int $userId,
    int $page = 1,
    int $perPage = 20
): ThreadView {
    // ❌ 缺少：readperm 权限检查
    // ❌ 缺少：special 类型数据获取不完整
    // ❌ 缺少：oldtopics 阅读标记
    // ❌ 缺少：fid 已读标记

    $thread = $this->threadRepository->findById($threadId);
    $posts = $this->postRepository->findByThread($threadId, $page, $perPage);

    return new ThreadView($thread, $posts, $page, $perPage, $total);
}
```

**缺失的关键方法**：
```php
// ❌ ThreadViewService.php 缺少：
public function checkReadPermission(int $threadId, int $userId, int $groupId): bool;
public function markThreadAsRead(int $threadId, int $userId): void;
public function markForumAsRead(int $forumId, int $userId): void;
public function isThreadRead(int $threadId, int $userId): bool;
public function getPollData(int $threadId): ?array;
public function getRewardData(int $threadId): ?array;
public function getDebateData(int $threadId): ?array;
public function getTradeData(int $threadId): ?array;
public function getActivityData(int $threadId): ?array;
```

---

## 五、发帖功能（newthread.inc.php）对比

### 5.1 原始实现（newthread.inc.php: 1-220行）

**功能清单**：
1. ✅ 特殊类型权限检查（allowpostpoll/allowpostreward）
2. ✅ 发帖积分限制检查（postcredits）
3. ✅ 主题长度验证（1-80字符）
4. ✅ 内容长度验证（1-50000字符）
5. ✅ 图片数量限制（20个）
6. ✅ 链接数量限制（50个）
7. ✅ 版块密码检查
8. ✅ 版块规则检查
9. ✅ 投票帖表单（poll）
10. ✅ 悬赏帖表单（reward）
11. ✅ 辩论帖表单（debate）
12. ✅ 交易帖表单（trade）
13. ✅ 活动帖表单（activity）
14. ✅ 类型图标选择（iconid）
15. ✅ 帖子分类选择（typeid）
16. ✅ 匿名发帖（anonymous）
17. ✅ 订阅回复通知（subscrib）

**权限检查顺序**（newthread.inc.php: 43-74）：
```php
// 1. 特殊类型权限
if (($special == 1 && !$allowpostpoll)) {
    showmessage('group_nopermission', NULL, 'NOPERM');
}

// 2. 发帖积分检查
checklowerlimit($postcredits);

// 3. 版块权限检查
if (!$discuz_uid &&
    ((!$forum['postperm'] && $allowpost) ||
     ($forum['postperm'] && forumperm($forum['postperm'])))) {
    showmessage('group_nopermission', NULL, 'NOPERM');
}

// 4. 版块密码检查
if ($forum['password']) {
    if (!isset($_DCOOKIE['fidpw'.$fid]) || $_DCOOKIE['fidpw'.$fid] != $forum['password']) {
        showmessage('forum_passwd', NULL, 'NOPERM');
    }
}
```

**内容验证**（newthread.inc.php: 87-93）：
```php
$subject = trim($subject);
$message = trim($message);

if ($subject == '' || $message == '') {
    showmessage('post_sm_isnull');
}

// 特殊类型数据验证
if ($special == 1) {
    // 投票票验证：至少2个选项
    if (count($polloptions['options']) < 2) {
        showmessage('post_poll_optionless', NULL, 'NOPERM');
    }
} elseif ($special == 2) {
    // 悬赏帖验证：价格>0，截止时间>当前时间
    if ($price > 0 && $timestamp > $deadline) {
        showmessage('reward_credits_invalid', NULL, 'NOPERM');
    }
} elseif ($special == 3) {
    // 辩论帖验证：正反方立场
}
```

### 5.2 现代实现（ThreadCreationService + ContentValidator）

**已实现功能**：
1. ⚠️ 权限检查不完整（缺少全局组权限）
2. ✅ 防洪水控制（FloodControlService）
3. ✅ 基本内容验证（ContentValidator）
4. ⚠️ 特殊类型验证不完整
5. ❌ 缺少发帖积分检查（postcredits）
6. ❌ 缺少版块密码验证
7. ❌ 缺少版块规则检查（rules）
8. ❌ 缺少类型图标选择（iconid）
9. ❌ 缺少帖子分类选择（typeid）
10. ❌ 缺少匿名发帖（anonymous）
11. ❌ 缺少订阅回复通知（subscrib）

**权限检查差异对比**：
```php
// Modern 实现（ThreadCreationService.php: 141-148）
public function validatePermissions(int $forumId, int $userId, int $groupId): void
{
    // ❌ 只检查了板块级权限
    $canPost = $this->permissionService->canPostThread($forumId, $userId, $groupId);

    if (!$canPost) {
        throw new \RuntimeException("You do not have permission to post threads in this forum");
    }

    // ❌ 缺失：未检查 cdb_usergroups.allowpost
    // ❌ 缺失：未检查 cdb_forumfields.postcredits
    // ❌ 缺失：未检查 special 类型权限（allowpostpoll/allowpostreward）
}
```

**ContentValidator 验证逻辑对比**：
```php
// Modern 实现（ContentValidator.php）
public function validateSpecialType(int $special, ?array $specialData): void
{
    return match ($special) {
        1 => $this->validatePollData($data),    // ✅ 投票帖
        2 => $this->validateRewardData($data),  // ✅ 悬赏帖
        3 => $this->validateDebateData($data),  // ✅ 辩论帖
        4 => $this->validateTradeData($data),   // ✅ 交易帖
        5 => $this->validateActivityData($data), // ✅ 活动帖
        default => throw ThreadException::invalidSpecialType("Invalid special type: {$special}"),
    };
}

// Legacy 实现包含更多验证：
// 1. 版块规则（rules）- ❌ 未实现
// 2. 类型图标权限- ❌ 未实现
// 3. 分类权限- ❌ 未实现
// 4. 积分消耗- ❌ 未实现
// 5. 匿名发帖- ⚠️ 部分实现
// 6. 订阅通知- ❌ 未实现
```

---

## 六、回复功能（newreply.inc.php）对比

### 6.1 原始实现（newreply.inc.php: 1-365行）

**功能清单**：
1. ✅ 回复权限检查（allowreply）
2. ✅ 回复积分限制检查（replycredits）
3. ✅ 主题/内容验证
4. ✅ 引用回复功能（quotepid）
5. ✅ 回复通知（replynotify）
6. ✅ 悬赏帖回复检查（不能回复自己发的悬赏帖）
7. ✅ 辩论帖回复检查（不能回复已结束的辩论）
8. ✅ 交易帖回复检查（不能回复已售罄的帖）
9. ✅ 活动帖回复检查（不能回复已截止的活动）
10. ✅ BBCode 引用格式化

**引用格式化**（newreply.inc.php: 103-149）：
```php
// 获取被引用的帖子
$thaquote = $db->fetch_first("SELECT pid, fid, author, authorid, first, message, useip, dateline, anonymous, status
    FROM {$tablepre}posts p
    WHERE pid='$repquote' AND invisible='0'");

// 检查权限
if (!$thaquote['status']) {
    if ($bannedmessages && $thaquote['authorid']) {
        $message = $language['post_banned'];
    } elseif ($thaquote['status'] & 1) {
        $message = $language['post_single_banned'];
    }
}

// 格式化引用内容
$time = gmdate("$dateformat $timeformat", $thaquote['dateline'] + $timeoffset * 3600);
$message = "[quote]{$thaquote['author']} ($time)\n";
$message .= preg_replace("/\n/", "\n> ", $thaquote['message']);
$message .= "\n[/quote]";
```

### 6.2 现代实现（PostReplyService + PostReplyController）

**已实现功能**：
1. ⚠️ 权限检查不完整（缺少全局组权限）
2. ❌ 缺少回复积分限制检查（replycredits）
3. ✅ 基本内容验证
4. ✅ 基本引用格式化（formatQuote）
5. ❌ 缺少回复通知（replynotify）
6. ❌ 缺少被禁用户检查（bannedmessages）
7. ❌ 缺少特殊类型回复检查

**PostReplyService 实现对比**：
```php
// Modern 实现（PostReplyService.php: 45-126）
public function createReply(PostReply $reply, int $userId, int $groupId, string $userIp): Post
{
    // ❌ 缺失：未检查 cdb_usergroups.allowreply
    // ❌ 缺失：未检查 cdb_forumfields.replycredits
    // ❌ 缺失：未检查 replynotify 设置

    $this->validatePermissions($forumId, $userId, $groupId);
    $this->validateContent($reply);
    $this->floodControl->checkReplyFlood($userId, $userIp);
    $this->floodControl->checkReplyHourlyLimit($userId, $userIp);

    $message = $reply->getMessage();
    if ($reply->shouldQuote() && $reply->getReplyTo() !== null) {
        $quotedPost = $this->postRepository->findById($reply->getReplyTo());
        if ($quotedPost !== null) {
            $message = $this->formatQuote($quotedPost) . "\n\n" . $message;
        }
    }

    // 创建回复...
}
```

---

## 七、编辑功能（editpost.inc.php）对比

### 7.1 原始实现（editpost.inc.php: 1-150行）

**功能清单**：
1. ✅ 编辑权限检查（alloweditpost）
2. ✅ 编辑时间限制检查（edittime + 版主豁免）
3. ✅ 首帖编辑处理（更新thread.subject）
4. ✅ 回复编辑处理（仅更新post.message）
5. ✅ 版主编辑所有帖子（ismoderator）
6. ✅ 管理员编辑所有帖子（adminid in 1,2,3）
7. ✅ 编辑原因记录（编辑间隔限制）
8. ✅ 删除权限检查（删除自己的帖或版主删除）
9. ✅ 软删除/硬删除选项
10. ✅ 交易帖数量修改（不能编辑已售罄）
11. ✅ 辩论帖立场修改（未结束的辩论）

**编辑时间限制**（editpost.inc.php: 83-87）：
```php
// 版块级编辑时间限制（edittime 字段）
$edittimelimit = $forum['edittime'] && $timestamp - $orig['dateline'] > $forum['edittime'];

// 权限检查顺序
if ($adminid != $orig['authorid'] && $adminid != $forum['moderators']) {
    // 非作者，非版主，非管理员
    if (!$alloweditpost || $edittimelimit) {
        showmessage('post_edit_nopermission', NULL, 'HALTED');
    }
}

// 版主豁免
if (in_array($adminid, explode(',', $forum['moderators']))) {
    $alloweditpost = 1;  // 版主可以编辑所有帖子
}
```

### 7.2 现代实现（PostEditService + PostEditController）

**已实现功能**：
1. ⚠️ 权限检查不完整（缺少版主检查）
2. ⚠️ 时间限制硬编码（未使用edittime字段）
3. ✅ 首帖编辑（更新thread.subject）
4. ✅ 回复编辑（仅更新post.message）
5. ❌ 缺少软删除/硬删除选项
6. ❌ 缺少版主编辑权限（moderators）
7. ❌ 缺少管理员编辑权限（adminid in 1,2,3）

**PostEditService 时间限制**（PostEditService.php: 39-50）：
```php
// ❌ 硬编码时间限制（按用户组）
private array $editTimeLimits = [
    1 => 600,   // 10分钟
    2 => 600,   // 10分钟
    3 => 3600,  // 1小时
    4 => 7200,  // 2小时
    5 => 0,     // 无限制
    6 => 0,
    7 => 0,
    8 => 0,
];

// Legacy 实现（editpost.inc.php）：
// ✅ 使用 cdb_forumfields.edittime 字段（可按板块配置）
$edittimelimit = $forum['edittime'] && $timestamp - $orig['dateline'] > $forum['edittime'];
```

**PostEditService 权限检查**（PostEditService.php: 208-213）：
```php
// ❌ 缺失：未检查版主身份
private function validateEditPermission(Post $post, int $userId, int $groupId): void
{
    if (!$this->canEditPost($post, $userId, $groupId)) {
        $remaining = $this->getRemainingEditTime($post, $groupId);
        if ($remaining === 0) {
            throw new \RuntimeException("Edit time has expired for this post");
        }
        throw new \RuntimeException("You do not have permission to edit this post");
    }
}

// Legacy 实现包含：
// 1. ismoderator 检查（moderators 字段，逗号分隔用户名）
// 2. adminid 检查（1,2,3 为管理员）
// 3. alloweditpost 检查（cdb_usergroups）
// 4. edittime 检查（cdb_forumfields.edittime）
```

---

## 八、核心缺失功能总结

### 8.1 权限控制缺失

| 功能 | Legacy | Modern | 优先级 |
|-----|--------|--------|--------|
| 全局组权限检查 | ✅ | ❌ | P0 |
| 扩展用户组支持 | ✅ | ❌ | P0 |
| 版主权限检查 | ✅ | ❌ | P0 |
| 管理员权限检查 | ✅ | ❌ | P0 |
| 板块密码验证 | ✅ | ❌ | P1 |
| 板块规则显示 | ✅ | ❌ | P1 |
| 编辑时间限制（数据库） | ✅ | ❌ | P0（硬编码） |
| 发帖/回复积分限制 | ✅ | ❌ | P1 |
| 特殊类型权限 | ✅ | ❌ | P0 |
| 阅读权限（readperm） | ✅ | ❌ | P1 |

### 8.2 论坛功能缺失

| 功能 | Legacy | Modern | 优先级 |
|-----|--------|--------|--------|
| 论坛折叠状态 | ✅ | ❌ | P1 |
| 已访问论坛管理 | ✅ | ❌ | P1 |
| 公告系统 | ✅ | ❌ | P1 |
| 广告推荐 | ✅ | ❌ | P2 |
| 在线用户列表 | ✅ | ❌ | P1 |
| 论坛跳转菜单 | ✅ | ❌ | P1 |
| 子板块显示 | ✅ | ❌ | P1 |
| 全局置顶帖 | ✅ | ❌ | P0 |
| 版块图标选择 | ✅ | ❌ | P1 |
| 帖子分类选择 | ✅ | ❌ | P1 |
| 阅读标记（oldtopics） | ✅ | ❌ | P0 |
| 论坛已读标记（fid cookie）| ✅ | ❌ | P0 |
| BBCode完整解析 | ✅ | ❌ | P0 |
| 回复通知 | ✅ | ❌ | P1 |
| 禁用用户检查 | ✅ | ❌ | P0 |
| 软删除选项 | ✅ | ❌ | P2 |

### 8.3 性能优化缺失

| 优化 | Legacy | Modern | 影响 |
|-----|--------|--------|------|
| 置顶帖分离查询 | ✅ | ❌ | **严重** |
| 全局置顶缓存 | ✅ | ❌ | **严重** |
| 帖子缓存 | ✅ | ❌ | **严重** |
| 在线用户缓存 | ✅ | ❌ | 中等 |
| 版块树缓存 | ✅ | ❌ | 中等 |

---

## 九、修复优先级建议

### P0 - 关键缺失（必须修复）

1. **补充全局用户组权限检查**（cdb_usergroups）
   - ForumPermissionService 必须检查 allowpost/allowreply
   - Priority: CRITICAL

2. **修正权限字段分隔符解析**（extgroupids）
   - ForumPermissionService 必须使用 Tab 分隔而不是逗号
   - Priority: CRITICAL

3. **添加版主权限检查**（moderators）
   - ForumPermissionService 需要检查 moderat
   - PostEditService 需要检查 moderat
   - Priority: CRITICAL

4. **置顶帖分离查询**
   - ThreadRepository 需要分离置顶帖和普通帖
   - Priority: CRITICAL

5. **添加阅读权限检查**（readperm）
   - ThreadViewService 需要检查 cdb_threads.readperm
   - Priority: HIGH

6. **实现缓存系统**
   - 论坛树、帖子列表、置顶帖缓存
   - Priority: HIGH

### P1 - 重要缺失（影响体验）

1. **板块密码验证**（password）
2. **板块规则显示**（rules）
3. **论坛折叠状态**（collapsed）
4. **已访问论坛管理**（visitedfid）
5. **帖子阅读标记**（oldtopics + fid cookie）
6. **在线用户列表**（whosonline）
7. **论坛跳转菜单**（forumjump）
8. **版块图标选择**（iconid）
9. **帖子分类选择**（typeid）

### P2 - 增强功能（可选）

1. **软删除/硬删除选项**
2. **回复通知**（replynotify）
3. **匿名发帖**（anonymous）
4. **订阅回复通知**（subscrib）
5. **广告推荐系统**（modrecommend）

---

## 十、数据库字段使用对比

### 10.1 Legacy 使用但 Modern 未使用的字段

**cdb_forums 表**：
```sql
type          - ✅ Modern 使用
- status        - ❌ Modern 完全忽略
- displayorder  - ⚠️ Modern 仅用于排序
- threads       - ❌ Modern 完全忽略
- posts         - ❌ Modern 完全忽略
- todayposts    - ❌ Modern 完全忽略
- lastpost      - ⚠️ Modern 未有效使用（仅存储）
```

**cdb_forumfields 表**：
```sql
moderators     - ❌ Modern 完全未使用
- edittime       - ❌ Modern 使用硬编码时间限制
- postcredits    - ❌ Modern 完全未检查
- replycredits   - ❌ Modern 完全未检查
- rules          - ❌ Modern 完全未实现
- password       - ❌ Modern 完全未实现
- viewperm       - ✅ Modern 使用但解析错误
- postperm       - ✅ Modern 使用但解析错误
- replyperm      - ✅ Modern 使用但解析错误
```

**cdb_threads 表**：
```sql
readperm      - ❌ Modern 完全未检查
- special       - ⚠️ Modern 部分实现不完整
- price         - ⚠️ Modern 未验证
- typeid        - ⚠️ Modern 未验证权限
- iconid        - ❌ Modern 完全未验证
```

**cdb_usergroups 表**：
```sql
allowview      - ❌ Modern 完全未检查
- allowpost      - ❌ Modern 完全未检查
- allowreply     - ❌ Modern 完全未检查
- allowpostpoll  - ❌ Modern 完全未检查
- alloweditpost  - ❌ Modern 完全未检查
```

**cdb_access 表**：
```sql
allowview     - ✅ Modern 部分使用
- allowpost     - ✅ Modern 部分使用
- allowreply    - ✅ Modern 部分使用
- allowgetattach - ❌ Modern 完全未使用
- allowpostattach - ❌ Modern 完全未使用
```

---

## 十一、修复方案建议

### 11.1 ForumPermissionService 修复

```php
class ForumPermissionService
{
    private Connection $db;
    private array $cache = [];

    // ✅ 添加：扩展用户组支持
    public function getExtGroupIds(int $userId): string
    {
        $sql = "SELECT extgroupids FROM {$tablepre}memberfields
                 WHERE uid = :uid";
        $row = $this->db->selectOne($sql, ['uid' => $userId]);
        return $row['extgroupids'] ?? '';
    }

    // ✅ 修复：权限字段解析（Tab分隔）
    private function checkForumPermission(int $forumId, int $groupId, string $permissionField): bool
    {
        $sql = "SELECT {$permissionField} FROM {$tablepre}forumfields
                 WHERE fid = :fid";
        $row = $this->db->selectOne($sql, ['fid' => $forumId]);

        $permValue = $row[$permissionField] ?? '';

        // 空值 = 所有组允许
        if ($permValue === '' || $permValue === null) {
            return true;
        }

        // ⚠️ 修正：使用 Tab 分隔符（与 Legacy 一致）
        $allowedGroups = explode("\t", $permValue);

        // ✅ 添加：检查扩展用户组
        $extGroupIds = $this->getExtGroupIds($userId);
        $allowedGroups = array_merge($allowedGroups, explode("\t", $extGroupIds));

        return in_array((string)$groupId, $allowedGroups, true);
    }

    // ✅ 添加：版主检查
    public function isModerator(int $userId, string $moderators): bool
    {
        if (empty($moderators)) {
            return false;
        }

        $sql = "SELECT username FROM {$tablepre}members
                 WHERE uid = :uid";
        $row = $this->db->selectOne($sql, ['uid' => $userId]);

        $moderatorNames = explode(',', $moderators);
        return in_array($row['username'], $moderatorNames, true);
    }

    // ✅ 修改：canPostThread 方法
    public function canPostThread(int $forumId, int $userId, int $groupId): bool
    {
        // 1. 管理员直接通过
        if ($this->isAdmin($userId)) {
            return true;
        }

        // 2. 检查是否为版主
        $forum = $this->getForumFields($forumId);
        if ($this->isModerator($userId, $forum['moderators'])) {
            return true;
        }

        // 3. 检查全局组权限（cdb_usergroups）
        if (!$this->checkUserGroupPermission($groupId, 'allowpost')) {
            return false;
        }

        // 4. 检查用户特定权限（cdb_access）
        $userAccess = $this->getUserAccess($forumId, $userId);
        if ($userAccess !== null) {
            return (bool) $userAccess['allowpost'];
        }

        // 5. 检查板块级权限（cdb_forumfields.postperm）
        return $this->checkForumPermission($forumId, $groupId, 'postperm');
    }
}
```

### 11.2 ThreadRepository 修复

```php
class ThreadRepository
{
    // ✅ 添加：置顶帖分离查询
    public function findByForumSticky(
        int $forumId,
        int $page = 1,
        int $perPage = 20
    ): array {
        $offset = ($page - 1) * $perPage;

        // 1. 全局置顶（使用全局缓存）
        $globalSticky = $this->getGlobalStickyThreads();
        if (!empty($globalSticky)) {
            $stickyIds = array_keys($globalSticky);
            $placeholders = array_fill(0, count($stickyIds) - 1, '?');
            $sql = "SELECT * FROM {$this->threadsTable}
                     WHERE tid IN (".implode(',', $placeholders).")
                     ORDER BY displayorder DESC, lastpost DESC
                     LIMIT :offset, :limit";
            // ...
        }

        // 2. 板块置顶（displayorder > 0）
        $forumStickySql = "SELECT * FROM {$this->threadsTable}
                                WHERE fid = :fid AND displayorder > 0
                                ORDER BY displayorder DESC, lastpost DESC
                                LIMIT :offset, :limit";

        // 3. 普通帖子
        $normalSql = "SELECT * FROM {$this->threadsTable}
                           WHERE fid = :fid AND displayorder = 0
                           ORDER BY lastpost DESC
                           LIMIT :offset, :limit";

        // 合并结果
        return array_merge($globalSticky, $forumSticky, $normal);
    }

    // ✅ 添加：全局置顶缓存支持
    private function getGlobalStickyThreads(): array
    {
        $cacheKey = 'global:sticky:threads';

        if ($this->cache !== null) {
            $cached = $this->cache->get($cacheKey);
            if ($cached !== null) {
                $stickyIds = $cached ? unserialize($cached) : [];
            }
        }

        if (empty($stickyIds)) {
            return [];
        }

        // 查询全局置顶帖
        $sql = "SELECT tid FROM {$this->threadsTable}
                 WHERE displayorder > 0
                 ORDER BY displayorder DESC, lastpost DESC
                 LIMIT 20";

        $results = $this->db->select($sql, []);

        return $this->cache->set($cacheKey, serialize($results), 3600);
    }
}
```

---

## 十二、总结

### 核心问题

1. **权限控制体系不完整**：缺失全局组权限、扩展组、版主权限检查
2. **权限字段解析错误**：extgroupids 应该用 Tab 分隔而不是逗号
3. **数据库字段未充分利用**：大量字段完全未使用
4. **功能缺失严重**：折叠状态、已访问论坛、阅读标记、板块密码等
5. **性能优化缺失**：无缓存系统，置顶帖处理低效
6. **特殊类型支持不完整**：poll/reward/debate/trade/activity 数据验证不完整

### 修复策略

1. **立即修复**（P0）：权限控制核心问题
2. **高优先级**（P0-P1）：置顶帖分离、缓存系统
3. **中优先级**（P1）：论坛功能完善（折叠、密码、规则等）
4. **低优先级**（P2）：增强功能（软删除、通知等）

---

**生成时间**: 2026-02-15
**对比分析人**: Claude Sonnet 4.5
**文档版本**: 1.0
