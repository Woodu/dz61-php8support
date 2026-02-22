# Week 11 发帖系统设计文档 (Posting System Design Document)

**文档版本**: 1.0
**创建日期**: 2026-02-18
**负责人**: Week 11 Development Lead
**状态**: Phase 1 - Legacy Analysis Complete

---

## 目录 (Table of Contents)

1. [项目概述](#项目概述)
2. [Legacy 系统分析](#legacy-系统分析)
3. [现代架构设计](#现代架构设计)
4. [数据库表使用](#数据库表使用)
5. [路由定义](#路由定义)
6. [控制器设计](#控制器设计)
7. [服务层设计](#服务层设计)
8. [模板结构](#模板结构)
9. [BBCode 集成](#bbcode-集成)
10. [附件上传集成](#附件上传集成)
11. [测试计划](#测试计划)
12. [实现优先级](#实现优先级)

---

## 项目概述

### Week 11 目标

实现完整的 Discuz! 发帖功能,包括:

1. ✅ **新主题表单** (New Thread Form) - 用户可以创建新主题
2. ✅ **回复表单** (Reply Form) - 用户可以回复主题,支持引用
3. ✅ **BBCode 集成** (BBCode Integration) - BBCode 解析器集成到模板
4. ✅ **附件上传 UI** (Attachment Upload UI) - 文件上传,支持拖拽
5. ✅ **账户激活集成** (Account Activation) - 激活流程集成到登录

### 关键约束

- ✅ **零表更改原则**: 仅使用现有表 (cdb_threads, cdb_posts, cdb_attachments, cdb_forums, cdb_usergroups)
- ✅ **Legacy 兼容性**: 现有 293,477 个帖子必须正确显示
- ✅ **安全要求**: CSRF 保护、XSS 防护、SQL 注入防护
- ✅ **性能要求**: >85% 测试覆盖率,所有测试通过

---

## Legacy 系统分析

### 1. 发帖流程分析 (Posting Flow Analysis)

#### 1.1 新主题流程 (New Thread Flow)

**Legacy 文件**: `/root/poketb-renew/poketb.com/bbs/post.php?action=newthread`

**流程步骤**:

```php
// 1. 权限检查 (Lines 298-74)
if(!$discuz_uid && !((!$forum['postperm'] && $allowpost) || ($forum['postperm'] && forumperm($forum['postperm'])))) {
    showmessage('group_nopermission', NULL, 'NOPERM');
}

// 2. 表单提交检查 (Line 86)
if(!submitcheck('topicsubmit', 0, $seccodecheck, $secqaacheck)) {
    // 显示表单
    include template('post_newthread');
}

// 3. 数据验证 (Lines 120-141)
if($subject == '' || $message == '') {
    showmessage('post_sm_isnull');
}

if($post_invalid = checkpost()) {
    showmessage($post_invalid);
}

if(checkflood()) {
    showmessage('post_flood_ctrl');
}

// 4. 主题数据插入 (Lines 484-488)
$db->query("INSERT INTO {$tablepre}threads (fid, readperm, price, iconid, typeid, author, authorid, subject, dateline, lastpost, lastposter, displayorder, digest, special, attachment, subscribed, moderated, msgnotify)
    VALUES ('$fid', '$readperm', '$price', '$iconid', '$typeid', '$author', '$discuz_uid', '$subject', '$timestamp', '$timestamp', '$author', '$displayorder', '$digest', '$special', '$attachment', '$subscribed', '$moderated', '$msgnotify')");

$tid = $db->insert_id();

// 5. 帖子数据插入 (Lines 570-574)
$db->query("INSERT INTO {$tablepre}posts (fid, tid, first, author, authorid, subject, dateline, message, useip, invisible, anonymous, usesig, htmlon, bbcodeoff, smileyoff, parseurloff, attachment)
    VALUES ('$fid', '$tid', '1', '$discuz_user', '$discuz_uid', '$subject', '$timestamp', '$message', '$onlineip', '$pinvisible', '$isanonymous', '$usesig', '$htmlon', '$bbcodeoff', '$smileyoff', '$parseurloff', '$attachment')");

$pid = $db->insert_id();

// 6. 附件处理 (Lines 584-608)
if($attachment) {
    foreach($attachments as $key => $attach) {
        $db->query("INSERT INTO {$tablepre}attachments (tid, pid, dateline, readperm, price, filename, description, filetype, filesize, attachment, downloads, isimage, uid, thumb, remote)
            VALUES ('$tid', '$pid', '$timestamp', '$attach[perm]', '$attach[price]', '$attach[name]', '$attach[description]', '$attach[type]', '$attach[size]', '$attach[attachment]', '0', '$attach[isimage]', '$attach[uid]', '$attach[thumb]', '$attach[remote]')");

        $searcharray[] = '[local]'.$localid[$key].'[/local]';
        $replacearray[] = '[attach]'.$db->insert_id().'[/attach]';
    }

    $message = str_replace($searcharray, $replacearray, preg_replace($pregarray, $replacearray, $message));
    $db->query("UPDATE {$tablepre}posts SET message='$message' WHERE pid='$pid'");
}

// 7. 版块统计更新 (Lines 812-818)
$db->query("UPDATE {$tablepre}forums SET lastpost='$lastpost', threads=threads+1, posts=posts+1, todayposts=todayposts+1 WHERE fid='$fid'", 'UNBUFFERED');
```

**关键发现**:

1. **双表插入**: 先插入 `cdb_threads` 获取 `tid`,再插入 `cdb_posts` 获取 `pid`
2. **附件占位符**: `[local]xxx[/local]` → `[attach]aid[/attach]` 替换机制
3. **BBCode 存储**: 原始 BBCode 存储在数据库,显示时解析
4. **积分更新**: 用户发帖获得积分,版块统计更新
5. **审核机制**: `modnewthreads` 控制是否需要审核 (displayorder=-2)

#### 1.2 回复流程 (Reply Flow)

**Legacy 文件**: `/root/poketb-renew/poketb.com/bbs/include/newreply.inc.php`

**流程步骤**:

```php
// 1. 引用处理 (Lines 105-203)
if(isset($repquote)) {
    $thaquote = $db->fetch_first("SELECT tid, fid, author, authorid, first, message, useip, dateline, anonymous, status FROM {$tablepre}posts WHERE pid='$repquote' AND invisible='0'");

    // 清理 BBCode (Lines 149-173)
    $message = preg_replace(array(
        "/\[hide=?\d*\](.+?)\[\/hide\]/is",
        "/\[quote](.*)\[\/quote]/siU",
        $language['post_edit_regexp'],
        "/\[($bbcodes)=?.*\]/iU",
        "/\[\/($bbcodes)\]/i",
    ), array(
        "[b]$language[post_hidden][/b]",
        '',
        '',
        '',
        ''
    ), $message);

    $message = cutstr(strip_tags($message), 200);

    // 生成引用 BBCode (Line 197)
    $message = "[quote]$language[post_reply_quote] [url={$boardurl}redirect.php?goto=findpost&pid=$repquote&ptid=$tid][img]{$boardurl}images/common/back.gif[/img][/url]\n$message [/quote]\n";
}

// 2. 回复插入 (Lines 491-497)
$db->query("INSERT INTO {$tablepre}posts (fid, tid, first, author, authorid, subject, dateline, message, useip, invisible, anonymous, usesig, htmlon, bbcodeoff, smileyoff, parseurloff, attachment)
    VALUES ('$fid', '$tid', '0', '$discuz_user', '$discuz_uid', '$subject', '$timestamp', '$message', '$onlineip', '$pinvisible', '$isanonymous', '$usesig', '$htmlon', '$bbcodeoff', '$smileyoff', '$parseurloff', '$attachment')");

$pid = $db->insert_id();

// 3. 主题更新 (Lines 719)
$db->query("UPDATE {$tablepre}threads SET lastposter='$author', lastpost='$timestamp', replies=replies+1 ".($attachment ? ', attachment=\'1\'' : '').", subscribed='".($subscribed || $newsubscribed ? 1 : 0)."' WHERE tid='$tid'", 'UNBUFFERED');
```

**关键发现**:

1. **引用格式**: `[quote]原帖内容[/quote]` BBCode 格式
2. **BBCode 清理**: 引用时移除隐藏内容、编辑标记等
3. **first 标志**: `first=1` 表示主题帖,`first=0` 表示回复
4. **主题更新**: 更新 `lastposter`, `lastpost`, `replies`

### 2. 表单字段分析 (Form Fields Analysis)

#### 2.1 新主题表单字段 (New Thread Form Fields)

| 字段 | 类型 | 必填 | 说明 | Legacy 代码位置 |
|------|------|------|------|----------------|
| `fid` | int | ✅ | 版块 ID | Line 298 |
| `subject` | string | ✅ | 标题 (1-80 字符) | Line 120 |
| `message` | string | ✅ | 内容 (BBCode) | Line 120 |
| `typeid` | int | ❌ | 分类 ID (如果版块要求) | Line 162 |
| `iconid` | int | ❌ | 图标 ID | Line 164 |
| `readperm` | int | ❌ | 阅读权限 | Line 170 |
| `price` | int | ❌ | 售价 (付费主题) | Line 174 |
| `tags` | string | ❌ | 标签 | Line 930 |
| `anonymous` | bool | ❌ | 匿名发布 | Line 172 |
| `htmlon` | bool | ❌ | 启用 HTML | Line 564 |
| `bbcodeoff` | bool | ❌ | 禁用 BBCode | Line 558 |
| `smileyoff` | bool | ❌ | 禁用表情 | Line 560 |
| `parseurloff` | bool | ❌ | 禁用 URL 解析 | Line 562 |
| `usesig` | bool | ❌ | 使用签名 | Line 479 |
| `emailnotify` | bool | ❌ | 邮件通知 | Line 480 |
| `attachments` | array | ❌ | 附件列表 | Line 144 |

#### 2.2 回复表单字段 (Reply Form Fields)

| 字段 | 类型 | 必填 | 说明 | Legacy 代码位置 |
|------|------|------|------|----------------|
| `tid` | int | ✅ | 主题 ID | Line 69 |
| `message` | string | ✅ | 回复内容 (BBCode) | Line 311 |
| `subject` | string | ❌ | 回复标题 (可选) | Line 311 |
| `reply_to` | int | ❌ | 回复的帖子 ID (引用) | Line 105 |
| `quote` | bool | ❌ | 是否引用原帖 | Line 105 |
| `anonymous` | bool | ❌ | 匿名回复 | Line 483 |
| `attachments` | array | ❌ | 附件列表 | Line 419 |

### 3. 验证规则分析 (Validation Rules)

#### 3.1 标题验证 (Subject Validation)

```php
// Legacy: include/common.inc.php
$subject = isset($subject) ? dhtmlspecialchars(censor(trim($subject))) : '';
$subject = !empty($subject) ? str_replace("\t", ' ', $subject) : $subject;

// 验证规则:
if($subject == '') {
    showmessage('post_sm_isnull'); // Line 120 newthread.inc.php
}

// 数据库字段: char(80)
// 最大长度: 80 字符
// 禁止字符: Tab 字符 (替换为空格)
```

#### 3.2 内容验证 (Message Validation)

```php
// Legacy: include/post.func.php
function checkpost() {
    // 1. 检查长度
    if(strlen($message) < 1) {
        return 'post_sm_isnull';
    }

    // 2. 检查重复内容
    // 3. 检查禁止词汇
    // 4. 检查 HTML 标签 (如果禁用 HTML)
    // 5. 检查 BBCode 语法
}

// 新回复特殊验证 (Lines 343-363 newreply.inc.php)
if(is_array($smilies)) {
    $temp = $message;
    foreach($smilies as $sm){
        $temp = str_replace($sm['code'], '', $temp);
    }
    $temp = trim($temp);
    if(strlen($temp) < 4) {
        showmessage('回复内容不能全是表情符号，请说点有意义的。谢谢理解，你不说啥也没事。但请返回');
    }
}
```

#### 3.3 附件验证 (Attachment Validation)

```php
// Legacy: include/post.func.php → attach_upload()
// 验证项:
1. 文件大小 ($maxattachsize)
2. 文件扩展名 ($attachextensions)
3. MIME 类型
4. 病毒扫描 (如果启用)
5. 附件权限 ($allowpostattach)
6. 每日上传配额
```

### 4. BBCode 处理分析 (BBCode Processing)

#### 4.1 存储 vs 显示 (Storage vs Display)

**关键发现**: BBCode **原始存储**在数据库,显示时解析

```php
// 存储 (数据库中的 message 字段):
[b]粗体文字[/b]
[url=http://example.com]链接[/url]
[img]attachment.jpg[/img]
[quote]引用内容[/quote]

// 显示解析 (include/discuzcode.func.php):
function discuzcode($message, $smileyoff, $bbcodeoff, $htmlon, $allowsmilies, $allowbbcode, $allowimgcode, $allowhtml, $jammer = 0, $ubb = 0) {
    // 1. HTML 转义 (防止 XSS)
    if(!$htmlon) {
        $message = dhtmlspecialchars($message);
    }

    // 2. 解析 BBCode 标签
    if(!$bbcodeoff) {
        $message = preg_replace("/\[(b|i|u)\](.+?)\[\/\\1\]/is", "<\\1>\\2</\\1>", $message);
        $message = preg_replace("/\[url=(https?.+?)\](.+?)\[\/url\]/i", "<a href=\"\\1\" target=\"_blank\">\\2</a>", $message);
        // ... 更多 BBCode 解析
    }

    return $message;
}
```

#### 4.2 附件 BBCode (Attachment BBCode)

```php
// 上传时占位符 (Lines 594-600 newthread.inc.php)
$searcharray[] = '[local]'.$localid[$key].'[/local]';
$replacearray[] = '[attach]'.$db->insert_id().'[/attach]';

// 存储在数据库的格式:
[attach]12345[/attach]

// 显示时解析 (include/attachment.func.php):
// 1. 查询附件表获取文件信息
// 2. 生成 HTML:
//    - 图片: <img src="...">
//    - 其他: <a href="download.php?aid=12345">filename.ext</a>
```

### 5. 权限检查分析 (Permission Checks)

#### 5.1 发帖权限矩阵 (Post Permission Matrix)

```php
// Legacy: cdb_forums 表字段
$forum['allowpost']       // -1=禁止, 0=继承, 1=允许
$forum['postperm']        // 允许的用户组列表 (格式: "\t1\t3\t5\t")
$forum['allowreply']      // -1=禁止, 0=继承, 1=允许
$forum['replyperm']       // 允许的用户组列表

// Legacy: cdb_usergroups 表字段
$usergroup['allowpost']   // 是否允许发帖
$usergroup['allowreply']  // 是否允许回复
$usergroup['allowpostattach'] // 是否允许上传附件

// 权限检查函数 (include/misc.func.php)
function forumperm($permstr) {
    // 检查当前用户组是否在权限字符串中
    // 格式: "\t1\t3\t5\t"
}
```

#### 5.2 审核机制 (Moderation Queue)

```php
// 审核条件:
1. $forum['modnewposts'] == 1  // 版块设置需要审核
2. $modnewthreads == 1        // 全局设置需要审核
3. $censormod == 1            // 包含敏感词

// 审核帖子标记:
$threads->displayorder = -2   // 待审核主题
$posts->invisible = -2        // 待审核回复

// 审核后更新:
$threads->displayorder = 0    // 正常显示
$posts->invisible = 0         // 正常显示
```

### 6. 安全机制分析 (Security Mechanisms)

#### 6.1 CSRF 保护 (CSRF Protection)

```php
// Legacy: post.php
// 生成 token (隐式)
<form method="post" action="">
    <input type="hidden" name="formhash" value="<?php echo formhash(); ?>" />
    <!-- 表单字段 -->
</form>

// 验证 token (include/common.inc.php)
function submitcheck($var, $allowget = 0, $seccodecheck = 0, $secqaacheck = 0) {
    if($allowget || ($_SERVER['REQUEST_METHOD'] == 'POST' && !empty($_POST[$var]) && $_POST['formhash'] == formhash())) {
        return true;
    }
    return false;
}
```

#### 6.2 XSS 防护 (XSS Protection)

```php
// Legacy: include/common.inc.php
function dhtmlspecialchars($string) {
    if(is_array($string)) {
        foreach($string as $key => $val) {
            $string[$key] = dhtmlspecialchars($val);
        }
    } else {
        $string = str_replace('&', '&amp;', $string);
        $string = str_replace('"', '&quot;', $string);
        $string = str_replace('<', '&lt;', $string);
        $string = str_replace('>', '&gt;', $string);
        $string = preg_replace('/&amp;(#\d{4,5};)/', '&\\1', $string);
    }
    return $string;
}

// 应用场景:
1. 标题: $subject = dhtmlspecialchars(censor(trim($subject)));
2. BBCode 禁用时: $message = dhtmlspecialchars($message);
3. 用户名: $author = dhtmlspecialchars($username);
```

#### 6.3 SQL 注入防护 (SQL Injection Protection)

```php
// Legacy: 使用变量插值 (不安全!)
$db->query("INSERT INTO {$tablepre}threads (subject) VALUES ('$subject')");

// 现代化改进: 使用 PDO prepared statements
$stmt = $pdo->prepare("INSERT INTO cdb_threads (subject) VALUES (:subject)");
$stmt->execute(['subject' => $subject]);
```

---

## 现代架构设计

### 1. 已实现组件 (Existing Components)

#### 1.1 PostController (已实现)

**文件**: `/root/poketb-renew/modern-php-migration-code/app/Http/Controllers/PostController.php`

**状态**: ✅ 已实现,但缺少以下功能:

- [ ] BBCode 实时预览
- [ ] 附件上传 UI 集成
- [ ] 引用功能完整实现
- [ ] 表单验证增强

**现有方法**:

```php
// 已实现的方法:
PostController::newThreadFormAction()     // 显示新主题表单
PostController::newThreadAction()          // 提交新主题
PostController::replyFormAction()          // 显示回复表单
PostController::replyAction()              // 提交回复
```

#### 1.2 BBCode Parser (已实现)

**文件**: `/root/poketb-renew/modern-php-migration-code/app/View/BBCodeParser.php`

**状态**: ✅ 已实现,但是 STUB (仅转义,未解析)

```php
// 当前实现 (临时方案)
public static function parse(?string $bbcode): string
{
    // TEMPORARY: Just escape the content
    return htmlspecialchars($bbcode, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}
```

**需要扩展**: 完整的 BBCode 解析器实现

#### 1.3 Attachment Service (已实现)

**文件**: `/root/poketb-renew/modern-php-migration-code/app/Attachment/Services/AttachmentUploadService.php`

**状态**: ✅ 已实现,功能完整

```php
// 已实现的功能:
- 文件验证 (大小、类型、MIME)
- 权限检查
- 文件存储
- 缩略图生成
- 数据库记录创建
```

#### 1.4 Permission Service (已实现)

**文件**: `/root/poketb-renew/modern-php-migration-code/app/Security/Services/PermissionService.php`

**状态**: ✅ 已实现,功能完整

```php
// 权限检查方法:
PermissionService::canPostInForum()           // 检查版块发帖权限
PermissionService::canReplyInForum()           // 检查版块回复权限
PermissionService::canPostAttachmentInForum()  // 检查版块上传附件权限
```

### 2. 需要实现的组件 (Components to Implement)

#### 2.1 BBCode Twig 扩展 (BBCode Twig Extension)

**目标**: 在 Twig 模板中使用 `{{ post.message|bbcode }}` 过滤器

**实现**:

```php
// app/View/Twig/BbcodeExtension.php
<?php
declare(strict_types=1);

namespace Discuz\View\Twig;

use Twig\Extension\AbstractExtension;
use Twig\TwigFilter;
use Discuz\View\BBCodeParser;

class BbcodeExtension extends AbstractExtension
{
    public function getFilters(): array
    {
        return [
            new TwigFilter('bbcode', [BBCodeParser::class, 'parse'], ['is_safe' => ['html']]),
        ];
    }
}

// 注册到 Twig (app/View/ViewRenderer.php)
$twig->addExtension(new BbcodeExtension());
```

#### 2.2 Post Repository (帖子仓库)

**文件**: `app/Post/Repositories/PostRepository.php`

**职责**:

```php
// 查询帖子
public function findById(int $pid): ?PostEntity
public function findByThreadId(int $tid, int $limit = 20, int $offset = 0): array
public function countByThreadId(int $tid): int

// 创建帖子
public function create(PostEntity $post): int

// 更新帖子
public function updateMessage(int $pid, string $message): bool
public function updateAttachmentFlag(int $pid, bool $hasAttachment): bool

// 删除帖子
public function delete(int $pid): bool
```

#### 2.3 Thread Repository (主题仓库) - 扩展

**文件**: `app/Thread/Repositories/ThreadRepository.php` (已存在,需扩展)

**需要添加的方法**:

```php
// 更新主题统计
public function incrementReplies(int $tid): bool
public function updateLastPost(int $tid, string $lastPoster, int $dateline): bool
public function updateAttachmentFlag(int $tid, bool $hasAttachment): bool

// 主题锁定/解锁
public function close(int $tid, bool $closed = true): bool
public function isClosed(int $tid): bool
```

#### 2.4 Forum Repository (版块仓库) - 扩展

**文件**: `app/Forum/Repositories/ForumRepository.php` (已存在,需扩展)

**需要添加的方法**:

```php
// 更新版块统计
public function incrementThreads(int $fid): bool
public function incrementPosts(int $fid): bool
public function updateLastPost(int $fid, string $lastpost): bool
```

---

## 数据库表使用

### 1. cdb_threads (主题表)

**现有数据**: 46,984 条记录

**字段映射**:

| Legacy 字段 | 类型 | 说明 | 现代 PHP 8.3 类型 |
|-------------|------|------|-------------------|
| `tid` | mediumint unsigned | 主题 ID (主键) | `int` |
| `fid` | smallint unsigned | 版块 ID | `int` |
| `iconid` | smallint unsigned | 图标 ID | `int` |
| `typeid` | smallint unsigned | 分类 ID | `?int` (nullable) |
| `readperm` | tinyint unsigned | 阅读权限 | `int` |
| `price` | smallint | 售价 | `int` |
| `author` | char(15) | 作者用户名 | `string` |
| `authorid` | mediumint unsigned | 作者 UID | `int` |
| `subject` | char(80) | 标题 | `string` |
| `dateline` | int unsigned | 发布时间 | `int` (timestamp) |
| `lastpost` | int unsigned | 最后回复时间 | `int` (timestamp) |
| `lastposter` | char(15) | 最后回复用户名 | `string` |
| `views` | int unsigned | 查看次数 | `int` |
| `replies` | mediumint unsigned | 回复数量 | `int` |
| `displayorder` | tinyint(1) | 显示顺序 | `int` |
| `highlight` | tinyint(1) | 高亮 | `int` |
| `digest` | tinyint(1) | 精华 | `int` |
| `special` | tinyint(1) | 特殊类型 | `int` |
| `attachment` | tinyint(1) | 包含附件 | `bool` |
| `subscribed` | tinyint(1) | 已订阅 | `bool` |
| `moderated` | tinyint(1) | 已审核 | `bool` |
| `closed` | mediumint unsigned | 关闭状态 | `int` |
| `msgnotify` | int | 消息通知 | `int` |

**使用场景**:

```php
// 插入新主题
INSERT INTO cdb_threads (fid, author, authorid, subject, dateline, lastpost, lastposter, displayorder, digest, special, attachment, subscribed, moderated, msgnotify)
VALUES (:fid, :author, :authorId, :subject, :dateline, :dateline, :author, 0, 0, :special, :hasAttachment, :subscribed, 0, :msgNotify)

// 更新统计
UPDATE cdb_threads SET replies = replies + 1, lastpost = :dateline, lastposter = :author, attachment = :hasAttachment WHERE tid = :tid
```

### 2. cdb_posts (帖子表)

**现有数据**: 293,477 条记录

**字段映射**:

| Legacy 字段 | 类型 | 说明 | 现代 PHP 8.3 类型 |
|-------------|------|------|-------------------|
| `pid` | int unsigned | 帖子 ID (主键) | `int` |
| `fid` | smallint unsigned | 版块 ID | `int` |
| `tid` | mediumint unsigned | 主题 ID | `int` |
| `first` | tinyint(1) | 是否主题帖 (1=是, 0=否) | `bool` |
| `author` | varchar(15) | 作者用户名 | `string` |
| `authorid` | mediumint unsigned | 作者 UID | `int` |
| `subject` | varchar(80) | 标题 (可选,回复时) | `string` |
| `dateline` | int unsigned | 发布时间 | `int` (timestamp) |
| `message` | mediumtext | 内容 (BBCode) | `string` |
| `useip` | varchar(15) | 发布 IP | `string` |
| `invisible` | tinyint(1) | 可见性 (-2=待审核, 0=正常) | `int` |
| `anonymous` | tinyint(1) | 是否匿名 | `bool` |
| `usesig` | tinyint(1) | 使用签名 | `bool` |
| `htmlon` | tinyint(1) | 启用 HTML | `bool` |
| `bbcodeoff` | tinyint(1) | 禁用 BBCode | `bool` |
| `smileyoff` | tinyint(1) | 禁用表情 | `bool` |
| `parseurloff` | tinyint(1) | 禁用 URL 解析 | `bool` |
| `attachment` | tinyint(1) | 包含附件 | `bool` |
| `rate` | smallint | 评分 | `int` |
| `ratetimes` | tinyint unsigned | 评分次数 | `int` |
| `status` | tinyint(1) | 状态 | `int` |

**使用场景**:

```php
// 插入主题帖 (first=1)
INSERT INTO cdb_posts (fid, tid, first, author, authorid, subject, dateline, message, useip, invisible, anonymous, usesig, htmlon, bbcodeoff, smileyoff, parseurloff, attachment)
VALUES (:fid, :tid, 1, :author, :authorId, :subject, :dateline, :message, :useip, :invisible, :anonymous, :usesig, :htmlon, :bbcodeoff, :smileyoff, :parseurloff, :attachment)

// 插入回复 (first=0)
INSERT INTO cdb_posts (fid, tid, first, author, authorid, dateline, message, useip, invisible, anonymous, usesig, htmlon, bbcodeoff, smileyoff, parseurloff, attachment)
VALUES (:fid, :tid, 0, :author, :authorId, :dateline, :message, :useip, :invisible, :anonymous, :usesig, :htmlon, :bbcodeoff, :smileyoff, :parseurloff, :attachment)

// 更新附件 BBCode
UPDATE cdb_posts SET message = :message WHERE pid = :pid
```

### 3. cdb_attachments (附件表)

**现有数据**: 14,749 条记录

**字段映射**:

| Legacy 字段 | 类型 | 说明 | 现代 PHP 8.3 类型 |
|-------------|------|------|-------------------|
| `aid` | mediumint unsigned | 附件 ID (主键) | `int` |
| `tid` | mediumint unsigned | 主题 ID | `int` |
| `pid` | int unsigned | 帖子 ID | `int` |
| `dateline` | int unsigned | 上传时间 | `int` (timestamp) |
| `readperm` | tinyint unsigned | 阅读权限 | `int` |
| `price` | smallint unsigned | 下载价格 | `int` |
| `filename` | char(100) | 原始文件名 | `string` |
| `description` | char(100) | 描述 | `string` |
| `filetype` | char(50) | MIME 类型 | `string` |
| `filesize` | int unsigned | 文件大小 (字节) | `int` |
| `attachment` | char(100) | 存储路径 | `string` |
| `downloads` | mediumint | 下载次数 | `int` |
| `isimage` | tinyint unsigned | 是否图片 | `bool` |
| `uid` | mediumint unsigned | 上传者 UID | `int` |
| `thumb` | tinyint unsigned | 有缩略图 | `bool` |
| `remote` | tinyint unsigned | 是否远程存储 | `bool` |

**使用场景**:

```php
// 插入附件记录
INSERT INTO cdb_attachments (tid, pid, dateline, readperm, price, filename, description, filetype, filesize, attachment, downloads, isimage, uid, thumb, remote)
VALUES (:tid, :pid, :dateline, :readperm, :price, :filename, :description, :filetype, :filesize, :attachment, 0, :isimage, :uid, :thumb, 0)

// 查询帖子附件
SELECT * FROM cdb_attachments WHERE pid = :pid
```

### 4. cdb_forums (版块表)

**现有数据**: 96 条记录

**用于权限检查的字段**:

| Legacy 字段 | 类型 | 说明 | 现代 PHP 8.3 类型 |
|-------------|------|------|-------------------|
| `fid` | smallint unsigned | 版块 ID (主键) | `int` |
| `allowpost` | tinyint(1) | 允许发帖 | `bool` |
| `allowreply` | tinyint(1) | 允许回复 | `bool` |
| `postperm` | mediumtext | 发帖权限字符串 | `?string` |
| `replyperm` | mediumtext | 回复权限字符串 | `?string` |
| `allowpostattach` | tinyint(1) | 允许上传附件 | `bool` |
| `postattachperm` | mediumtext | 上传附件权限 | `?string` |
| `threads` | mediumint unsigned | 主题数量 | `int` |
| `posts` | mediumint unsigned | 帖子数量 | `int` |
| `todayposts` | mediumint unsigned | 今日发帖数 | `int` |
| `lastpost` | char(110) | 最新帖子 | `string` |

**使用场景**:

```php
// 权限检查
SELECT allowpost, postperm, allowreply, replyperm, allowpostattach, postattachperm
FROM cdb_forums WHERE fid = :fid

// 更新统计
UPDATE cdb_forums SET threads = threads + 1, posts = posts + 1, todayposts = todayposts + 1, lastpost = :lastpost WHERE fid = :fid
```

### 5. cdb_usergroups (用户组表)

**用于权限检查的字段**:

| Legacy 字段 | 类型 | 说明 | 现代 PHP 8.3 类型 |
|-------------|------|------|-------------------|
| `groupid` | smallint unsigned | 用户组 ID (主键) | `int` |
| `allowpost` | tinyint(1) | 允许发帖 | `bool` |
| `allowreply` | tinyint(1) | 允许回复 | `bool` |
| `allowpostattach` | tinyint(1) | 允许上传附件 | `bool` |
| `allowpostpoll` | tinyint(1) | 允许发布投票 | `bool` |

---

## 路由定义

### 1. 现有路由 (Existing Routes)

**文件**: `/root/poketb-renew/modern-php-migration-code/config/routes.php`

**已定义的路由**:

```php
// 新主题表单 (Line 583)
GET /api/v1/post/newthread?forum_id={fid}
→ PostController::newThreadFormAction

// 提交新主题 (Line 615)
POST /api/v1/post/newthread
→ PostController::newThreadAction

// 回复表单 (Line 638)
GET /api/v1/thread/{id}/reply?reply_to={pid}
→ PostController::replyFormAction

// 提交回复 (Line 666)
POST /api/v1/thread/{id}/reply
→ PostController::replyAction
```

### 2. 需要添加的路由 (Routes to Add)

```php
// ============================================================
// Post Management Routes (New)
// ============================================================

/**
 * 编辑帖子表单
 *
 * GET /api/v1/post/{id}/edit
 *
 * Response 200:
 * - success: bool - True
 * - post: object - Post data (id, subject, message, etc.)
 * - csrf_token: string - CSRF token
 *
 * Response 401: Unauthorized
 * Response 403: Permission denied
 * Response 404: Post not found
 */
$router->get('/post/:id/edit', [PostController::class, 'editFormAction']);

/**
 * 更新帖子
 *
 * POST /api/v1/post/{id}
 *
 * Request body:
 * - subject: string (required for first post) - Updated subject
 * - message: string (required) - Updated message (BBCode)
 * - reason: string (optional) - Edit reason
 * - csrf_token: string (required) - CSRF token
 *
 * Response 200:
 * - success: bool - True
 * - message: string - "Post updated successfully"
 * - post: object - Updated post data
 *
 * Response 400: Invalid input
 * Response 401: Unauthorized
 * Response 403: Invalid CSRF token or permission denied
 * Response 404: Post not found
 */
$router->post('/post/:id', [PostController::class, 'updateAction']);

/**
 * 删除帖子
 *
 * DELETE /api/v1/post/{id}
 *
 * Request body:
 * - csrf_token: string (required) - CSRF token
 * - reason: string (optional) - Deletion reason
 *
 * Response 200:
 * - success: bool - True
 * - message: string - "Post deleted successfully"
 *
 * Response 401: Unauthorized
 * Response 403: Invalid CSRF token or permission denied
 * Response 404: Post not found
 */
$router->delete('/post/:id', [PostController::class, 'deleteAction']);

/**
 * BBCode 实时预览
 *
 * POST /api/v1/post/preview
 *
 * Request body:
 * - message: string (required) - BBCode content to preview
 *
 * Response 200:
 * - success: bool - True
 * - html: string - Rendered HTML
 *
 * Response 400: Invalid input
 */
$router->post('/post/preview', [PostController::class, 'previewAction']);

/**
 * 上传附件 (临时,关联到帖子)
 *
 * POST /api/v1/post/{id}/attachments/upload
 *
 * Request body (multipart/form-data):
 * - file: file (required) - Attachment file
 * - description: string (optional) - Attachment description
 * - csrf_token: string (required) - CSRF token
 *
 * Response 200:
 * - success: bool - True
 * - attachment: object - Attachment data (id, filename, url, thumbnail_url)
 *
 * Response 400: Invalid file
 * Response 401: Unauthorized
 * Response 403: Invalid CSRF token or permission denied
 */
$router->post('/post/:id/attachments/upload', [AttachmentController::class, 'uploadForPostAction']);

/**
 * 获取帖子附件列表
 *
 * GET /api/v1/post/{id}/attachments
 *
 * Response 200:
 * - success: bool - True
 * - attachments: array - List of attachments
 *
 * Response 404: Post not found
 */
$router->get('/post/:id/attachments', [AttachmentController::class, 'listByPostAction']);

/**
 * 删除附件
 *
 * DELETE /api/v1/post/{id}/attachments/{aid}
 *
 * Request body:
 * - csrf_token: string (required) - CSRF token
 *
 * Response 200:
 * - success: bool - True
 * - message: string - "Attachment deleted successfully"
 *
 * Response 401: Unauthorized
 * Response 403: Invalid CSRF token or permission denied
 * Response 404: Attachment not found
 */
$router->delete('/post/:id/attachments/:aid', [AttachmentController::class, 'deleteFromPostAction']);

/**
 * 引用帖子 (获取引用 BBCode)
 *
 * GET /api/v1/post/{id}/quote
 *
 * Response 200:
 * - success: bool - True
 * - bbcode: string - Quote BBCode block
 * - post: object - Original post data (id, author, dateline)
 *
 * Response 404: Post not found
 */
$router->get('/post/:id/quote', [PostController::class, 'quoteAction']);
```

### 3. 页面路由 (Page Routes - HTML Templates)

```php
// ============================================================
// Page Routes (HTML, not API)
// ============================================================

/**
 * 新主题页面
 *
 * GET /post/newthread?forum_id={fid}
 *
 * Renders: templates/post/new_thread.html.twig
 *
 * Query parameters:
 * - forum_id: int (required) - Forum ID
 *
 * Context data:
 * - forum: object - Forum information
 * - user: object - Current user
 * - csrf_token: string - CSRF token
 * - attachments: array - Existing attachments (if any)
 */
$router->get('/post/newthread', function ($request, $response) {
    // Render template
    return $this->view->render($response, 'post/new_thread.html.twig', [
        'forum' => $forumData,
        'user' => $userData,
        'csrf_token' => $csrfToken,
    ]);
});

/**
 * 回复页面
 *
 * GET /thread/{id}/reply
 *
 * Renders: templates/post/reply.html.twig
 *
 * Path parameters:
 * - id: int (required) - Thread ID
 *
 * Query parameters:
 * - reply_to: int (optional) - Post ID to quote
 *
 * Context data:
 * - thread: object - Thread information
 * - user: object - Current user
 * - csrf_token: string - CSRF token
 * - quote: object (optional) - Quoted post data
 */
$router->get('/thread/:id/reply', function ($request, $response, $args) {
    // Render template
    return $this->view->render($response, 'post/reply.html.twig', [
        'thread' => $threadData,
        'user' => $userData,
        'csrf_token' => $csrfToken,
        'quote' => $quoteData,
    ]);
});

/**
 * 编辑帖子页面
 *
 * GET /post/{id}/edit
 *
 * Renders: templates/post/edit.html.twig
 *
 * Path parameters:
 * - id: int (required) - Post ID
 *
 * Context data:
 * - post: object - Post data
 * - thread: object - Thread information
 * - user: object - Current user
 * - csrf_token: string - CSRF token
 * - attachments: array - Post attachments
 */
$router->get('/post/:id/edit', function ($request, $response, $args) {
    // Render template
    return $this->view->render($response, 'post/edit.html.twig', [
        'post' => $postData,
        'thread' => $threadData,
        'user' => $userData,
        'csrf_token' => $csrfToken,
        'attachments' => $attachments,
    ]);
});
```

---

## 控制器设计

### 1. PostController 扩展 (Extended PostController)

**文件**: `app/Http/Controllers/PostController.php`

**需要添加的方法**:

#### 1.1 编辑表单

```php
/**
 * Display edit form
 *
 * Route: GET /api/v1/post/{id}/edit
 *
 * @param int $postId Post ID
 * @return array Form data
 */
public function editFormAction(int $postId): array
{
    // 1. 检查登录
    $userId = $this->userSession->getUserId();
    if ($userId === 0) {
        return ['success' => false, 'error' => 'Authentication required'];
    }

    // 2. 查询帖子
    $post = $this->postRepository->findById($postId);
    if ($post === null) {
        return ['success' => false, 'error' => 'Post not found'];
    }

    // 3. 权限检查
    //    - 是作者 OR 是版主
    //    - 未超时 (edittimelimit)
    //    - 主题帖只能在无回复时编辑
    if (!$this->canEditPost($post, $userId)) {
        return ['success' => false, 'error' => 'Permission denied'];
    }

    // 4. 查询附件
    $attachments = $this->attachmentRepository->findByPid($postId);

    // 5. 生成 CSRF token
    $csrfToken = $this->csrf->generate();

    return [
        'success' => true,
        'post' => [
            'id' => $post->pid,
            'threadId' => $post->tid,
            'subject' => $post->subject,
            'message' => $post->message,
            'isFirst' => $post->first === 1,
        ],
        'attachments' => array_map(fn($a) => $a->toArray(), $attachments),
        'csrf_token' => $csrfToken,
    ];
}
```

#### 1.2 更新帖子

```php
/**
 * Update post
 *
 * Route: POST /api/v1/post/{id}
 *
 * @param int $postId Post ID
 * @param array $data Updated data
 * @return array Update result
 */
public function updateAction(int $postId, array $data): array
{
    // 1. 检查登录
    $userId = $this->userSession->getUserId();
    if ($userId === 0) {
        return ['success' => false, 'error' => 'Authentication required'];
    }

    // 2. 验证 CSRF
    if (!$this->csrf->validate($data['csrf_token'] ?? null)) {
        return ['success' => false, 'error' => 'Invalid security token'];
    }

    // 3. 查询帖子
    $post = $this->postRepository->findById($postId);
    if ($post === null) {
        return ['success' => false, 'error' => 'Post not found'];
    }

    // 4. 权限检查
    if (!$this->canEditPost($post, $userId)) {
        return ['success' => false, 'error' => 'Permission denied'];
    }

    // 5. 验证数据
    $validator = new PostValidator();
    $errors = $validator->validate($data);
    if (!empty($errors)) {
        return ['success' => false, 'errors' => $errors];
    }

    // 6. 更新数据
    try {
        $this->postRepository->updateMessage($postId, $data['message']);

        // 如果是主题帖,同时更新主题标题
        if ($post->first === 1 && isset($data['subject'])) {
            $this->threadRepository->updateSubject($post->tid, $data['subject']);
        }

        return [
            'success' => true,
            'message' => 'Post updated successfully',
        ];

    } catch (\Exception $e) {
        error_log('PostController::updateAction - Error: ' . $e->getMessage());
        return ['success' => false, 'error' => 'Update failed'];
    }
}
```

#### 1.3 删除帖子

```php
/**
 * Delete post
 *
 * Route: DELETE /api/v1/post/{id}
 *
 * @param int $postId Post ID
 * @param array $data Request data (csrf_token, reason)
 * @return array Delete result
 */
public function deleteAction(int $postId, array $data): array
{
    // 1. 检查登录
    $userId = $this->userSession->getUserId();
    if ($userId === 0) {
        return ['success' => false, 'error' => 'Authentication required'];
    }

    // 2. 验证 CSRF
    if (!$this->csrf->validate($data['csrf_token'] ?? null)) {
        return ['success' => false, 'error' => 'Invalid security token'];
    }

    // 3. 查询帖子
    $post = $this->postRepository->findById($postId);
    if ($post === null) {
        return ['success' => false, 'error' => 'Post not found'];
    }

    // 4. 权限检查
    //    - 是作者 OR 是版主
    //    - 如果是主题帖,需要有回复时不能删除
    if (!$this->canDeletePost($post, $userId)) {
        return ['success' => false, 'error' => 'Permission denied'];
    }

    // 5. 删除帖子
    try {
        $this->postRepository->delete($postId);

        // 如果是主题帖,删除整个主题
        if ($post->first === 1) {
            $this->threadRepository->delete($post->tid);
        }

        return [
            'success' => true,
            'message' => 'Post deleted successfully',
        ];

    } catch (\Exception $e) {
        error_log('PostController::deleteAction - Error: ' . $e->getMessage());
        return ['success' => false, 'error' => 'Delete failed'];
    }
}
```

#### 1.4 BBCode 预览

```php
/**
 * Preview BBCode rendering
 *
 * Route: POST /api/v1/post/preview
 *
 * @param array $data Request data (message)
 * @return array Preview result
 */
public function previewAction(array $data): array
{
    // 1. 验证数据
    if (empty($data['message'])) {
        return ['success' => false, 'error' => 'Message is required'];
    }

    // 2. 解析 BBCode
    $html = BBCodeParser::parse($data['message']);

    return [
        'success' => true,
        'html' => $html,
    ];
}
```

#### 1.5 引用帖子

```php
/**
 * Get quote BBCode for a post
 *
 * Route: GET /api/v1/post/{id}/quote
 *
 * @param int $postId Post ID to quote
 * @return array Quote data
 */
public function quoteAction(int $postId): array
{
    // 1. 检查登录
    $userId = $this->userSession->getUserId();
    if ($userId === 0) {
        return ['success' => false, 'error' => 'Authentication required'];
    }

    // 2. 查询帖子
    $post = $this->postRepository->findById($postId);
    if ($post === null) {
        return ['success' => false, 'error' => 'Post not found'];
    }

    // 3. 检查主题是否可回复
    $thread = $this->threadRepository->findById($post->tid);
    if ($thread === null || $thread->closed) {
        return ['success' => false, 'error' => 'Thread is closed'];
    }

    // 4. 清理 BBCode (移除隐藏内容、编辑标记等)
    $cleanMessage = $this->cleanMessageForQuote($post->message);

    // 5. 生成引用 BBCode
    $quoteBBCode = sprintf(
        "[quote][i]%s 原帖由 %s 发布于 %s[/i]\n%s[/quote]\n",
        $post->author,
        date('Y-m-d H:i', $post->dateline),
        $cleanMessage
    );

    return [
        'success' => true,
        'bbcode' => $quoteBBCode,
        'post' => [
            'id' => $post->pid,
            'author' => $post->author,
            'dateline' => $post->dateline,
        ],
    ];
}

/**
 * Clean message for quoting
 *
 * Removes:
 * - [hide] tags (hidden content)
 * - [quote] tags (nested quotes)
 * - Edited marks
 * - Attach tags (keep [attach]id[/attach])
 */
private function cleanMessageForQuote(string $message): string
{
    // 移除隐藏内容
    $message = preg_replace('/\[hide=?\d*\].*?\[\/hide\]/is', '[b][隐藏内容][/b]', $message);

    // 移除嵌套引用
    $message = preg_replace('/\[quote\].*?\[\/quote\]/is', '', $message);

    // 移除编辑标记
    $message = preg_replace('/\n\n\[i\]\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.+?\[\/i\]\n$/s', '', $message);

    // 截取前 500 字符
    $message = mb_substr($message, 0, 500, 'UTF-8');

    return trim($message);
}
```

### 2. 辅助方法 (Helper Methods)

```php
/**
 * Check if user can edit post
 */
private function canEditPost(PostEntity $post, int $userId): bool
{
    // 1. 版主可以编辑任何帖子
    if ($this->permission->isModerator($post->fid)) {
        return true;
    }

    // 2. 作者可以编辑自己的帖子 (在时限内)
    if ($post->authorid === $userId) {
        // 检查编辑时限 (默认 10 分钟)
        $editTimeLimit = 600; // 从配置读取
        if (time() - $post->dateline <= $editTimeLimit) {
            return true;
        }
    }

    return false;
}

/**
 * Check if user can delete post
 */
private function canDeletePost(PostEntity $post, int $userId): bool
{
    // 1. 版主可以删除任何帖子
    if ($this->permission->isModerator($post->fid)) {
        return true;
    }

    // 2. 主题帖只能在无回复时删除
    if ($post->first === 1) {
        $replyCount = $this->postRepository->countByThreadId($post->tid) - 1;
        if ($replyCount > 0) {
            return false;
        }
    }

    // 3. 作者可以删除自己的帖子
    if ($post->authorid === $userId) {
        return true;
    }

    return false;
}
```

---

## 服务层设计

### 1. PostService (帖子服务)

**文件**: `app/Post/Services/PostService.php`

**职责**:

```php
<?php
declare(strict_types=1);

namespace Discuz\Post\Services;

use Discuz\Post\Entities\PostEntity;
use Discuz\Post\Repositories\PostRepository;
use Discuz\Thread\Repositories\ThreadRepository;
use Discuz\Forum\Repositories\ForumRepository;
use Discuz\User\Repositories\UserRepository;
use Discuz\Security\Services\PermissionService;
use Discuz\Post\DTOs\PostCreationDTO;
use RuntimeException;

/**
 * Post Service
 *
 * Handles post creation, update, deletion business logic
 *
 * @package Discuz\Post\Services
 */
class PostService
{
    private PostRepository $postRepository;
    private ThreadRepository $threadRepository;
    private ForumRepository $forumRepository;
    private UserRepository $userRepository;
    private PermissionService $permission;

    public function __construct(
        PostRepository $postRepository,
        ThreadRepository $threadRepository,
        ForumRepository $forumRepository,
        UserRepository $userRepository,
        PermissionService $permission
    ) {
        $this->postRepository = $postRepository;
        $this->threadRepository = $threadRepository;
        $this->forumRepository = $forumRepository;
        $this->userRepository = $userRepository;
        $this->permission = $permission;
    }

    /**
     * Create new thread with first post
     *
     * @param PostCreationDTO $dto Post creation data
     * @param int $userId User ID
     * @param int $groupId User group ID
     * @param string $ipAddress User IP
     * @return array{tid: int, pid: int} Created thread and post IDs
     * @throws RuntimeException On validation or permission error
     */
    public function createThread(PostCreationDTO $dto, int $userId, int $groupId, string $ipAddress): array
    {
        // 1. Validate data
        $this->validateThreadCreation($dto, $userId, $groupId);

        // 2. Check forum permissions
        if (!$this->permission->canPostInForum($dto->forumId, $userId)) {
            throw new RuntimeException('You do not have permission to post in this forum');
        }

        // 3. Check flood control
        // TODO: Implement flood control check

        // 4. Create thread record
        $thread = new ThreadEntity(
            fid: $dto->forumId,
            iconid: $dto->iconId ?? 0,
            typeid: $dto->typeid ?? 0,
            readperm: $dto->readperm ?? 0,
            price: $dto->price ?? 0,
            author: $dto->author,
            authorid: $userId,
            subject: $dto->subject,
            dateline: time(),
            lastpost: time(),
            lastposter: $dto->author,
            views: 0,
            replies: 0,
            displayorder: 0,
            digest: 0,
            special: $dto->special ?? 0,
            attachment: $dto->hasAttachments ? 1 : 0,
            subscribed: $dto->subscribe ? 1 : 0,
            moderated: 0,
            closed: 0,
            msgnotify: $dto->emailNotify ? 1 : 0
        );

        $tid = $this->threadRepository->create($thread);

        // 5. Create first post
        $post = new PostEntity(
            fid: $dto->forumId,
            tid: $tid,
            first: 1,
            author: $dto->author,
            authorid: $userId,
            subject: $dto->subject,
            dateline: time(),
            message: $dto->message,
            useip: $ipAddress,
            invisible: 0,
            anonymous: $dto->anonymous ? 1 : 0,
            usesig: $dto->useSig ? 1 : 0,
            htmlon: $dto->htmlOn ? 1 : 0,
            bbcodeoff: $dto->bbcodeOff ? 1 : 0,
            smileyoff: $dto->smileyOff ? 1 : 0,
            parseurloff: $dto->parseUrlOff ? 1 : 0,
            attachment: $dto->hasAttachments ? 1 : 0
        );

        $pid = $this->postRepository->create($post);

        // 6. Update forum stats
        $lastpost = sprintf('%s\t%s\t%d\t%s', $tid, $dto->subject, time(), $dto->author);
        $this->forumRepository->incrementThreads($dto->forumId);
        $this->forumRepository->updateLastPost($dto->forumId, $lastpost);

        // 7. Award credits to user
        // TODO: Implement credit system integration

        return ['tid' => $tid, 'pid' => $pid];
    }

    /**
     * Create reply to thread
     *
     * @param PostCreationDTO $dto Post creation data
     * @param int $userId User ID
     * @param int $groupId User group ID
     * @param string $ipAddress User IP
     * @return int Created post ID
     * @throws RuntimeException On validation or permission error
     */
    public function createReply(PostCreationDTO $dto, int $userId, int $groupId, string $ipAddress): int
    {
        // 1. Validate data
        $this->validateReplyCreation($dto, $userId, $groupId);

        // 2. Get thread
        $thread = $this->threadRepository->findById($dto->threadId);
        if ($thread === null) {
            throw new RuntimeException('Thread not found');
        }

        // 3. Check if thread is closed
        if ($thread->closed) {
            throw new RuntimeException('Thread is closed');
        }

        // 4. Check forum permissions
        if (!$this->permission->canReplyInForum($thread->fid, $userId)) {
            throw new RuntimeException('You do not have permission to reply in this forum');
        }

        // 5. Check flood control
        // TODO: Implement flood control check

        // 6. Create reply post
        $post = new PostEntity(
            fid: $thread->fid,
            tid: $dto->threadId,
            first: 0,
            author: $dto->author,
            authorid: $userId,
            subject: '',
            dateline: time(),
            message: $dto->message,
            useip: $ipAddress,
            invisible: 0,
            anonymous: $dto->anonymous ? 1 : 0,
            usesig: $dto->useSig ? 1 : 0,
            htmlon: $dto->htmlOn ? 1 : 0,
            bbcodeoff: $dto->bbcodeOff ? 1 : 0,
            smileyoff: $dto->smileyOff ? 1 : 0,
            parseurloff: $dto->parseUrlOff ? 1 : 0,
            attachment: $dto->hasAttachments ? 1 : 0
        );

        $pid = $this->postRepository->create($post);

        // 7. Update thread stats
        $this->threadRepository->incrementReplies($dto->threadId);
        $this->threadRepository->updateLastPost($dto->threadId, $dto->author, time());

        // 8. Update forum stats
        $lastpost = sprintf('%s\t%s\t%d\t%s', $dto->threadId, $thread->subject, time(), $dto->author);
        $this->forumRepository->incrementPosts($thread->fid);
        $this->forumRepository->updateLastPost($thread->fid, $lastpost);

        // 9. Award credits to user
        // TODO: Implement credit system integration

        return $pid;
    }

    /**
     * Validate thread creation data
     */
    private function validateThreadCreation(PostCreationDTO $dto, int $userId, int $groupId): void
    {
        // 1. Subject validation
        if (empty($dto->subject)) {
            throw new RuntimeException('Subject is required');
        }

        if (mb_strlen($dto->subject, 'UTF-8') > 80) {
            throw new RuntimeException('Subject is too long (max 80 characters)');
        }

        // 2. Message validation
        if (empty($dto->message)) {
            throw new RuntimeException('Message is required');
        }

        // Check for all-smilies message
        $messageWithoutSmilies = preg_replace('/\[[^\]]+\]/', '', $dto->message);
        if (mb_strlen(trim($messageWithoutSmilies), 'UTF-8') < 4) {
            throw new RuntimeException('Message cannot contain only smilies');
        }

        // 3. Forum validation
        $forum = $this->forumRepository->findById($dto->forumId);
        if ($forum === null) {
            throw new RuntimeException('Forum not found');
        }

        if ($forum->type === 'group') {
            throw new RuntimeException('Cannot post in forum group');
        }

        // 4. User validation
        $user = $this->userRepository->findById($userId);
        if ($user === null) {
            throw new RuntimeException('User not found');
        }

        // 5. Special thread validation
        if ($dto->special > 0) {
            $this->validateSpecialThread($dto, $groupId);
        }
    }

    /**
     * Validate reply creation data
     */
    private function validateReplyCreation(PostCreationDTO $dto, int $userId, int $groupId): void
    {
        // 1. Message validation
        if (empty($dto->message)) {
            throw new RuntimeException('Message is required');
        }

        // Check for all-smilies message
        $messageWithoutSmilies = preg_replace('/\[[^\]]+\]/', '', $dto->message);
        if (mb_strlen(trim($messageWithoutSmilies), 'UTF-8') < 4) {
            throw new RuntimeException('Message cannot contain only smilies');
        }

        // 2. Thread validation
        $thread = $this->threadRepository->findById($dto->threadId);
        if ($thread === null) {
            throw new RuntimeException('Thread not found');
        }
    }

    /**
     * Validate special thread types (poll, reward, etc.)
     */
    private function validateSpecialThread(PostCreationDTO $dto, int $groupId): void
    {
        switch ($dto->special) {
            case 1: // Poll
                if (!$this->permission->canPostPoll()) {
                    throw new RuntimeException('You do not have permission to post polls');
                }
                // Validate poll options
                if (empty($dto->specialData['poll_options']) || count($dto->specialData['poll_options']) < 2) {
                    throw new RuntimeException('Poll must have at least 2 options');
                }
                break;

            case 2: // Payment thread
                if (!$this->permission->canPostReward()) {
                    throw new RuntimeException('You do not have permission to post reward threads');
                }
                // Validate price
                if (!isset($dto->price) || $dto->price <= 0) {
                    throw new RuntimeException('Price is required for reward threads');
                }
                break;

            // TODO: Add more special thread types
        }
    }
}
```

---

## 模板结构

### 1. 新主题表单模板

**文件**: `templates/post/new_thread.html.twig`

```twig
{# extends 'layout/base.html.twig' #}

{% block title %}发新主题 - {{ forum.name }}{% endblock %}

{% block content %}
<div class="post-new-thread">
    <h1>在 {{ forum.name }} 发新主题</h1>

    <form id="newThreadForm" method="post" action="/api/v1/post/newthread" enctype="multipart/form-data">
        {# CSRF Token #}
        <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
        <input type="hidden" name="forum_id" value="{{ forum.id }}">

        {# Subject Field #}
        <div class="form-group">
            <label for="subject">标题 <span class="required">*</span></label>
            <input type="text" id="subject" name="subject" maxlength="80" required
                   class="form-control" placeholder="请输入标题 (1-80 字符)">
            <small class="form-text">最多 80 个字符</small>
        </div>

        {# Message Field #}
        <div class="form-group">
            <label for="message">内容 <span class="required">*</span></label>

            {# BBCode Toolbar #}
            <div class="bbcode-toolbar">
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="b" title="粗体">
                    <b>B</b>
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="i" title="斜体">
                    <i>I</i>
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="u" title="下划线">
                    <u>U</u>
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="url" title="链接">
                    🔗
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="img" title="图片">
                    🖼️
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="quote" title="引用">
                    ❝
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="code" title="代码">
                    &lt;/&gt;
                </button>
            </div>

            {# Message Textarea #}
            <textarea id="message" name="message" rows="15" required
                      class="form-control bbcode-editor"
                      placeholder="支持 BBCode 格式，例如: [b]粗体[/b], [url=http://example.com]链接[/url]"></textarea>

            {# Live Preview #}
            <div class="bbcode-preview mt-2">
                <h6>实时预览</h6>
                <div id="bbcodePreview" class="border p-2" style="min-height: 100px;">
                    <em class="text-muted">预览将在此显示...</em>
                </div>
            </div>
        </div>

        {# Attachment Upload #}
        <div class="form-group">
            <label>附件上传</label>

            {# Drag & Drop Zone #}
            <div id="attachmentDropZone" class="attachment-drop-zone">
                <p>拖拽文件到这里，或点击选择文件</p>
                <input type="file" id="attachmentInput" name="attachments[]" multiple
                       class="form-control-file" style="display: none;">
                <button type="button" class="btn btn-outline-secondary" id="selectAttachmentBtn">
                    选择文件
                </button>
            </div>

            {# Attachment List #}
            <div id="attachmentList" class="attachment-list mt-2"></div>
            <small class="form-text">
                允许的文件类型: jpg, jpeg, gif, png, pdf, zip, rar 等<br>
                最大文件大小: 10 MB
            </small>
        </div>

        {# Options #}
        <div class="form-group">
            <div class="form-check">
                <input type="checkbox" id="anonymous" name="anonymous" class="form-check-input">
                <label for="anonymous" class="form-check-label">匿名发布</label>
            </div>
            <div class="form-check">
                <input type="checkbox" id="subscribe" name="subscribe" class="form-check-input" checked>
                <label for="subscribe" class="form-check-label">订阅此主题</label>
            </div>
        </div>

        {# Submit Buttons #}
        <div class="form-group">
            <button type="submit" class="btn btn-primary" id="submitBtn">发布主题</button>
            <button type="button" class="btn btn-secondary" id="previewBtn">预览</button>
            <a href="/forum/{{ forum.id }}" class="btn btn-outline-secondary">取消</a>
        </div>
    </form>
</div>

{# JavaScript #}
{% block scripts %}
<script src="/js/bbcode-editor.js"></script>
<script src="/js/attachment-upload.js"></script>
<script>
$(document).ready(function() {
    // Initialize BBCode Editor
    const bbcodeEditor = new BBCodeEditor('#message', {
        preview: '#bbcodePreview',
        toolbar: '.bbcode-toolbar'
    });

    // Initialize Attachment Upload
    const attachmentUpload = new AttachmentUpload('#attachmentDropZone', {
        url: '/api/v1/post/attachments/upload',
        csrfToken: '{{ csrf_token }}',
        maxFiles: 5,
        maxSize: 10 * 1024 * 1024, // 10 MB
        onUploadComplete: function(file, response) {
            if (response.success) {
                addAttachmentToList(response.attachment);
            }
        }
    });

    // Form submission
    $('#newThreadForm').on('submit', function(e) {
        e.preventDefault();

        const formData = new FormData(this);
        formData.append('attachments', JSON.stringify(attachmentUpload.getUploadedFiles()));

        $.ajax({
            url: $(this).attr('action'),
            method: 'POST',
            data: formData,
            processData: false,
            contentType: false,
            success: function(response) {
                if (response.success) {
                    window.location.href = '/thread/' + response.thread.id;
                } else {
                    showError(response.error || '发布失败');
                }
            },
            error: function() {
                showError('网络错误，请重试');
            }
        });
    });

    // Preview button
    $('#previewBtn').on('click', function() {
        const message = $('#message').val();
        $.post('/api/v1/post/preview', { message: message }, function(response) {
            if (response.success) {
                $('#bbcodePreview').html(response.html);
            }
        });
    });

    function addAttachmentToList(attachment) {
        const html = `
            <div class="attachment-item" data-aid="${attachment.id}">
                <span>${attachment.filename}</span>
                <button type="button" class="btn btn-sm btn-danger remove-attachment">删除</button>
            </div>
        `;
        $('#attachmentList').append(html);
    }

    function showError(message) {
        alert(message); // TODO: Replace with proper error display
    }
});
</script>
{% endblock %}
{% endblock %}
```

### 2. 回复表单模板

**文件**: `templates/post/reply.html.twig`

```twig
{# extends 'layout/base.html.twig' #}

{% block title %}回复主题 - {{ thread.subject }}{% endblock %}

{% block content %}
<div class="post-reply">
    <h1>回复主题: {{ thread.subject }}</h1>

    {# Original Thread Preview #}
    <div class="thread-preview mb-3">
        <h3>主题内容</h3>
        <div class="thread-content">
            {{ thread.message|bbcode|raw }}
        </div>
    </div>

    {# Quote Display (if replying to a post) #}
    {% if quote %}
    <div class="quote-preview mb-3">
        <h3>引用 {{ quote.author }} 的帖子</h3>
        <div class="quote-content">
            {{ quote.message|bbcode|raw }}
        </div>
        <input type="hidden" name="reply_to" value="{{ quote.id }}">
    </div>
    {% endif %}

    <form id="replyForm" method="post" action="/api/v1/thread/{{ thread.id }}/reply" enctype="multipart/form-data">
        {# CSRF Token #}
        <input type="hidden" name="csrf_token" value="{{ csrf_token }}">

        {# Message Field #}
        <div class="form-group">
            <label for="message">回复内容 <span class="required">*</span></label>

            {# BBCode Toolbar #}
            <div class="bbcode-toolbar">
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="b" title="粗体">
                    <b>B</b>
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="i" title="斜体">
                    <i>I</i>
                </button>
                <button type="button" class="btn btn-sm btn-secondary" data-bbcode="quote" title="引用">
                    ❝
                </button>
            </div>

            {# Message Textarea #}
            <textarea id="message" name="message" rows="10" required
                      class="form-control bbcode-editor"
                      placeholder="输入回复内容...">{% if quote %}[quote]{{ quote.message }}[/quote]{% endif %}</textarea>

            {# Live Preview #}
            <div class="bbcode-preview mt-2">
                <h6>实时预览</h6>
                <div id="bbcodePreview" class="border p-2" style="min-height: 100px;">
                    <em class="text-muted">预览将在此显示...</em>
                </div>
            </div>
        </div>

        {# Attachment Upload #}
        <div class="form-group">
            <label>附件上传</label>
            <div id="attachmentDropZone" class="attachment-drop-zone">
                <p>拖拽文件到这里</p>
                <input type="file" id="attachmentInput" name="attachments[]" multiple>
            </div>
            <div id="attachmentList" class="attachment-list mt-2"></div>
        </div>

        {# Options #}
        <div class="form-group">
            <div class="form-check">
                <input type="checkbox" id="anonymous" name="anonymous" class="form-check-input">
                <label for="anonymous" class="form-check-label">匿名回复</label>
            </div>
        </div>

        {# Submit Buttons #}
        <div class="form-group">
            <button type="submit" class="btn btn-primary">发表回复</button>
            <button type="button" class="btn btn-secondary" id="previewBtn">预览</button>
            <a href="/thread/{{ thread.id }}" class="btn btn-outline-secondary">取消</a>
        </div>
    </form>
</div>
{% endblock %}
```

### 3. BBCode 编辑器 JavaScript

**文件**: `public/js/bbcode-editor.js`

```javascript
/**
 * BBCode Editor
 *
 * Provides toolbar buttons and live preview for BBCode editing
 */
class BBCodeEditor {
    constructor(textarea, options = {}) {
        this.textarea = $(textarea);
        this.preview = $(options.preview || '#bbcodePreview');
        this.toolbar = $(options.toolbar || '.bbcode-toolbar');
        this.debounceTimer = null;

        this.init();
    }

    init() {
        // Initialize toolbar buttons
        this.toolbar.find('[data-bbcode]').on('click', (e) => {
            e.preventDefault();
            const bbcode = $(e.currentTarget).data('bbcode');
            this.insertBBCode(bbcode);
        });

        // Live preview (debounced)
        this.textarea.on('input', () => {
            clearTimeout(this.debounceTimer);
            this.debounceTimer = setTimeout(() => {
                this.updatePreview();
            }, 500);
        });

        // Initial preview
        this.updatePreview();
    }

    insertBBCode(tag) {
        const textarea = this.textarea[0];
        const start = textarea.selectionStart;
        const end = textarea.selectionEnd;
        const text = textarea.value;
        const selectedText = text.substring(start, end);

        let insertion = '';
        let cursorOffset = 0;

        switch (tag) {
            case 'url':
                const url = prompt('请输入链接地址:', 'http://');
                if (url) {
                    insertion = `[url=${url}]${selectedText || '链接文字'}[/url]`;
                    cursorOffset = insertion.length;
                }
                break;

            case 'img':
                const imgUrl = prompt('请输入图片地址:', 'http://');
                if (imgUrl) {
                    insertion = `[img]${imgUrl}[/img]`;
                    cursorOffset = insertion.length;
                }
                break;

            default:
                insertion = `[${tag}]${selectedText}[/${tag}]`;
                cursorOffset = tag.length + 2;
        }

        if (insertion) {
            textarea.value = text.substring(0, start) + insertion + text.substring(end);
            textarea.selectionStart = start + cursorOffset;
            textarea.selectionEnd = start + cursorOffset;
            textarea.focus();

            this.updatePreview();
        }
    }

    updatePreview() {
        const message = this.textarea.val();

        if (!message.trim()) {
            this.preview.html('<em class="text-muted">预览将在此显示...</em>');
            return;
        }

        // Call preview API
        $.post('/api/v1/post/preview', { message: message }, (response) => {
            if (response.success) {
                this.preview.html(response.html);
            }
        }).fail(() => {
            this.preview.html('<em class="text-danger">预览加载失败</em>');
        });
    }
}

// Export for use in templates
window.BBCodeEditor = BBCodeEditor;
```

### 4. 附件上传 JavaScript

**文件**: `public/js/attachment-upload.js`

```javascript
/**
 * Attachment Upload Handler
 *
 * Handles drag-and-drop file upload with progress tracking
 */
class AttachmentUpload {
    constructor(dropZone, options = {}) {
        this.dropZone = $(dropZone);
        this.url = options.url || '/api/v1/attachments/upload';
        this.csrfToken = options.csrfToken || '';
        this.maxFiles = options.maxFiles || 5;
        this.maxSize = options.maxSize || 10 * 1024 * 1024; // 10 MB
        this.uploadedFiles = [];
        this.onUploadComplete = options.onUploadComplete || function() {};

        this.init();
    }

    init() {
        // Prevent default drag behaviors
        ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
            this.dropZone[0].addEventListener(eventName, this.preventDefaults, false);
        });

        // Highlight drop zone
        ['dragenter', 'dragover'].forEach(eventName => {
            this.dropZone[0].addEventListener(eventName, () => {
                this.dropZone.addClass('highlight');
            }, false);
        });

        ['dragleave', 'drop'].forEach(eventName => {
            this.dropZone[0].addEventListener(eventName, () => {
                this.dropZone.removeClass('highlight');
            }, false);
        });

        // Handle dropped files
        this.dropZone[0].addEventListener('drop', (e) => {
            const files = e.dataTransfer.files;
            this.handleFiles(files);
        }, false);

        // Handle file input
        const fileInput = this.dropZone.find('input[type="file"]');
        fileInput.on('change', (e) => {
            this.handleFiles(e.target.files);
        });
    }

    preventDefaults(e) {
        e.preventDefault();
        e.stopPropagation();
    }

    handleFiles(files) {
        if (this.uploadedFiles.length + files.length > this.maxFiles) {
            alert(`最多只能上传 ${this.maxFiles} 个文件`);
            return;
        }

        ([...files]).forEach(file => this.uploadFile(file));
    }

    uploadFile(file) {
        // Validate file size
        if (file.size > this.maxSize) {
            alert(`文件 "${file.name}" 太大 (最大 ${this.maxSize / 1024 / 1024} MB)`);
            return;
        }

        // Validate file type
        const allowedTypes = ['image/jpeg', 'image/gif', 'image/png', 'image/webp',
                              'application/pdf', 'application/zip',
                              'application/x-zip-compressed'];
        if (!allowedTypes.includes(file.type)) {
            alert(`文件类型 "${file.type}" 不允许上传`);
            return;
        }

        // Prepare form data
        const formData = new FormData();
        formData.append('file', file);
        formData.append('csrf_token', this.csrfToken);

        // Upload with progress
        $.ajax({
            url: this.url,
            type: 'POST',
            data: formData,
            processData: false,
            contentType: false,
            xhr: () => {
                const xhr = new window.XMLHttpRequest();
                xhr.upload.addEventListener('progress', (e) => {
                    if (e.lengthComputable) {
                        const percentComplete = (e.loaded / e.total) * 100;
                        this.updateProgress(file.name, percentComplete);
                    }
                }, false);
                return xhr;
            },
            success: (response) => {
                if (response.success) {
                    this.uploadedFiles.push(response.attachment);
                    this.onUploadComplete(file, response);
                } else {
                    alert(`上传失败: ${response.error || '未知错误'}`);
                }
            },
            error: () => {
                alert(`上传 "${file.name}" 失败`);
            }
        });
    }

    updateProgress(filename, percent) {
        // Update progress bar (if exists)
        const progressBar = $(`[data-file="${filename}"] .progress-bar`);
        if (progressBar.length) {
            progressBar.css('width', percent + '%');
        }
    }

    getUploadedFiles() {
        return this.uploadedFiles;
    }

    removeFile(aid) {
        this.uploadedFiles = this.uploadedFiles.filter(f => f.aid !== aid);
    }
}

// Export for use in templates
window.AttachmentUpload = AttachmentUpload;
```

---

## BBCode 集成

### 1. Twig 过滤器扩展 (Twig Filter Extension)

**文件**: `app/View/Twig/BbcodeExtension.php`

```php
<?php
declare(strict_types=1);

namespace Discuz\View\Twig;

use Twig\Extension\AbstractExtension;
use Twig\TwigFilter;
use Discuz\View\BBCodeParser;

/**
 * BBCode Twig Extension
 *
 * Provides {{ content|bbcode }} filter for templates
 *
 * @package Discuz\View\Twig
 */
class BbcodeExtension extends AbstractExtension
{
    /**
     * Get Twig filters
     *
     * @return array TwigFilter[]
     */
    public function getFilters(): array
    {
        return [
            new TwigFilter('bbcode', [$this, 'parseBBCode'], ['is_safe' => ['html']]),
        ];
    }

    /**
     * Parse BBCode to HTML
     *
     * @param string|null $bbcode BBCode content
     * @return string HTML output
     */
    public function parseBBCode(?string $bbcode): string
    {
        if ($bbcode === null || $bbcode === '') {
            return '';
        }

        return BBCodeParser::parse($bbcode);
    }
}
```

### 2. 注册扩展 (Register Extension)

**文件**: `app/View/ViewRenderer.php`

**需要添加的代码**:

```php
use Discuz\View\Twig\BbcodeExtension;

public function getTwig(): \Twig\Environment
{
    if ($this->twig !== null) {
        return $this->twig;
    }

    // ... existing code ...

    // Add BBCode extension
    $this->twig->addExtension(new BbcodeExtension());

    return $this->twig;
}
```

### 3. 模板使用 (Template Usage)

```twig
{# Example: Display post message with BBCode rendered #}
<div class="post-message">
    {{ post.message|bbcode|raw }}
</div>

{# Example: Display signature with BBCode rendered #}
<div class="user-signature">
    {{ user.signature|bbcode|raw }}
</div>
```

---

## 附件上传集成

### 1. 附件上传流程 (Attachment Upload Flow)

```
用户选择文件
    ↓
JavaScript 验证 (大小、类型)
    ↓
FormData 包装
    ↓
AJAX 上传到 /api/v1/attachments/upload
    ↓
AttachmentUploadService::upload()
    ├─ 文件验证
    ├─ 权限检查
    ├─ 生成文件名
    ├─ 保存到存储
    ├─ 生成缩略图 (如果是图片)
    └─ 插入数据库
    ↓
返回 attachment JSON
    ↓
JavaScript 添加到附件列表
    ↓
显示预览 (缩略图或图标)
```

### 2. 附件与帖子关联 (Attachment-Post Association)

**策略**: 临时上传,提交时关联

```javascript
// 1. 用户上传附件 (临时)
attachmentUpload.uploadFile(file);
// → 返回 { aid: 123, filename: "image.jpg", url: "/uploads/..." }

// 2. 用户提交表单
$.ajax({
    url: '/api/v1/post/newthread',
    data: {
        // ... 帖子字段 ...
        attachments: JSON.stringify([
            { aid: 123, description: "图片描述" },
            { aid: 124, description: "文档" }
        ])
    }
});

// 3. 后端处理
// a. 创建帖子 (返回 pid)
// b. 更新附件记录: UPDATE cdb_attachments SET pid = :pid WHERE aid IN (123, 124)
// c. 生成 BBCode: [attach]123[/attach]
// d. 更新帖子内容: UPDATE cdb_posts SET message = :message WHERE pid = :pid
```

### 3. 附件显示 (Attachment Display)

**在帖子内容中插入 BBCode 占位符**:

```javascript
// 上传完成后,在编辑器中插入 [attach]aid[/attach]
function insertAttachmentBBCode(aid) {
    const textarea = document.getElementById('message');
    const bbcode = `[attach]${aid}[/attach]`;
    insertAtCursor(textarea, bbcode);
}
```

**显示时解析 BBCode**:

```php
// BBCodeParser::parse() 中处理 [attach] 标签
$message = preg_replace_callback(
    '/\[attach\](\d+)\[\/attach\]/i',
    function ($matches) {
        $aid = (int) $matches[1];
        $attachment = $attachmentRepository->findById($aid);

        if ($attachment === null) {
            return '[附件不存在]';
        }

        if ($attachment->isImage()) {
            return sprintf('<img src="/uploads/%s" alt="%s" class="post-image">',
                $attachment->attachment,
                htmlspecialchars($attachment->filename)
            );
        } else {
            return sprintf('<a href="/download/%d" class="post-attachment">%s</a> (%s)',
                $aid,
                htmlspecialchars($attachment->filename),
                formatFileSize($attachment->filesize)
            );
        }
    },
    $message
);
```

---

## 测试计划

### 1. 单元测试 (Unit Tests)

#### 1.1 PostServiceTest

**文件**: `tests/unit/Post/PostServiceTest.php`

```php
<?php
declare(strict_types=1);

namespace Discuz\Tests\Unit\Post;

use PHPUnit\Framework\TestCase;
use Discuz\Post\Services\PostService;
use Discuz\Post\DTOs\PostCreationDTO;

class PostServiceTest extends TestCase
{
    private PostService $postService;

    protected function setUp(): void
    {
        // Initialize test dependencies
        $this->postService = new PostService(
            // Mock repositories
        );
    }

    public function testCreateThreadWithValidData()
    {
        // Arrange
        $dto = PostCreationDTO::fromArray([
            'forumId' => 1,
            'author' => 'testuser',
            'authorId' => 1,
            'subject' => 'Test Thread',
            'message' => '[b]Test content[/b]',
        ]);

        // Act
        $result = $this->postService->createThread($dto, 1, 10, '127.0.0.1');

        // Assert
        $this->assertArrayHasKey('tid', $result);
        $this->assertArrayHasKey('pid', $result);
        $this->assertGreaterThan(0, $result['tid']);
        $this->assertGreaterThan(0, $result['pid']);
    }

    public function testCreateThreadWithEmptySubjectThrowsException()
    {
        // Arrange
        $dto = PostCreationDTO::fromArray([
            'forumId' => 1,
            'author' => 'testuser',
            'authorId' => 1,
            'subject' => '',
            'message' => 'Test content',
        ]);

        // Expect exception
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Subject is required');

        // Act
        $this->postService->createThread($dto, 1, 10, '127.0.0.1');
    }

    public function testCreateThreadWithSubjectTooLongThrowsException()
    {
        // Arrange
        $dto = PostCreationDTO::fromArray([
            'forumId' => 1,
            'author' => 'testuser',
            'authorId' => 1,
            'subject' => str_repeat('A', 81), // 81 characters
            'message' => 'Test content',
        ]);

        // Expect exception
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Subject is too long');

        // Act
        $this->postService->createThread($dto, 1, 10, '127.0.0.1');
    }

    public function testCreateThreadWithAllSmiliesThrowsException()
    {
        // Arrange
        $dto = PostCreationDTO::fromArray([
            'forumId' => 1,
            'author' => 'testuser',
            'authorId' => 1,
            'subject' => 'Test Thread',
            'message' => '[img]/smilies/1.gif[/img] [img]/smilies/2.gif[/img]',
        ]);

        // Expect exception
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Message cannot contain only smilies');

        // Act
        $this->postService->createThread($dto, 1, 10, '127.0.0.1');
    }

    // TODO: Add more tests...
    // - testCreateThreadWithoutPermissionThrowsException
    // - testCreateReplyWithValidData
    // - testCreateReplyToClosedThreadThrowsException
    // - testQuoteGeneration
    // - testAttachmentProcessing
}
```

#### 1.2 BBCodeParserTest (扩展)

**文件**: `tests/unit/View/BBCodeParserTest.php` (已存在,需扩展)

```php
// 新增测试用例
public function testParseAttachTag()
{
    // Mock attachment repository
    // Test [attach]123[/attach] parsing
}

public function testParseQuoteTag()
{
    $bbcode = '[quote][i]原帖由 testuser 发布于 2026-02-18[/i]\nOriginal message[/quote]';
    $html = BBCodeParser::parse($bbcode);

    $this->assertStringContainsString('<blockquote>', $html);
    $this->assertStringContainsString('Original message', $html);
}

public function testParseHideTagRemovesContent()
{
    $bbcode = 'Visible content [hide=10]Hidden content[/hide] more visible';
    $html = BBCodeParser::parse($bbcode);

    // Hidden content should be replaced with placeholder
    $this->assertStringContainsString('[隐藏内容]', $html);
    $this->assertStringNotContainsString('Hidden content', $html);
}
```

### 2. 集成测试 (Integration Tests)

#### 2.1 CreateThreadFlowTest

**文件**: `tests/integration/Post/CreateThreadFlowTest.php`

```php
<?php
declare(strict_types=1);

namespace Discuz\Tests\Integration\Post;

use Discuz\Tests\Integration\TestCase;
use Discuz\Database\Database;

class CreateThreadFlowTest extends TestCase
{
    public function testCompleteCreateThreadFlow()
    {
        // Arrange: Create test user and forum
        $userId = $this->createTestUser();
        $forumId = $this->createTestForum();

        // Act: Submit thread creation request
        $response = $this->post('/api/v1/post/newthread', [
            'forum_id' => $forumId,
            'subject' => 'Integration Test Thread',
            'message' => '[b]Bold content[/b] with [url=http://example.com]link[/url]',
            'csrf_token' => $this->getCsrfToken(),
        ]);

        // Assert: Check response
        $this->assertEquals(200, $response['success']);
        $this->assertArrayHasKey('thread', $response);
        $this->assertArrayHasKey('id', $response['thread']);

        // Assert: Check database
        $db = Database::getInstance();
        $thread = $db->fetchOne("SELECT * FROM cdb_threads WHERE tid = ?", [$response['thread']['id']]);
        $this->assertNotNull($thread);
        $this->assertEquals('Integration Test Thread', $thread['subject']);

        $post = $db->fetchOne("SELECT * FROM cdb_posts WHERE tid = ? AND first = 1", [$response['thread']['id']]);
        $this->assertNotNull($post);
        $this->assertStringContainsString('[b]Bold content[/b]', $post['message']);

        // Cleanup
        $this->deleteTestThread($response['thread']['id']);
        $this->deleteTestUser($userId);
        $this->deleteTestForum($forumId);
    }

    public function testUnauthenticatedUserCannotCreateThread()
    {
        // Act: Try to create thread without authentication
        $response = $this->post('/api/v1/post/newthread', [
            'forum_id' => 1,
            'subject' => 'Test',
            'message' => 'Test content',
        ]);

        // Assert
        $this->assertEquals(false, $response['success']);
        $this->assertEquals('auth_required', $response['error_code']);
    }

    // TODO: Add more integration tests...
    // - testCreateReplyFlow
    // - testQuoteFlow
    // - testAttachmentUploadFlow
    // - testEditPostFlow
    // - testDeletePostFlow
    // - testPermissionChecks
}
```

### 3. 功能测试清单 (Functional Test Checklist)

#### 3.1 新主题功能测试

- [ ] **TC1**: 已登录用户可以发布新主题
  - 前置条件: 用户已登录,有发帖权限
  - 操作: 填写标题和内容,点击发布
  - 预期: 主题创建成功,跳转到主题页面

- [ ] **TC2**: 未登录用户无法发布新主题
  - 前置条件: 用户未登录
  - 操作: 访问 `/post/newthread?forum_id=1`
  - 预期: 重定向到登录页面

- [ ] **TC3**: 标题为必填项
  - 操作: 留空标题,填写内容,点击发布
  - 预期: 显示"标题不能为空"错误

- [ ] **TC4**: 标题长度限制
  - 操作: 输入 81 个字符的标题
  - 预期: 显示"标题过长"错误

- [ ] **TC5**: 内容为必填项
  - 操作: 填写标题,留空内容,点击发布
  - 预期: 显示"内容不能为空"错误

- [ ] **TC6**: 内容不能全是表情符号
  - 操作: 内容只包含 `[img]smilie.gif[/img]`
  - 预期: 显示"内容不能全是表情符号"错误

- [ ] **TC7**: BBCode 正确存储和显示
  - 操作: 输入 `[b]粗体[/b]` 和 `[url=http://example.com]链接[/url]`
  - 预期: 数据库存储原始 BBCode,页面显示解析后的 HTML

- [ ] **TC8**: 附件上传成功
  - 操作: 上传图片文件
  - 预期: 显示上传进度,完成后显示缩略图

- [ ] **TC9**: 附件类型验证
  - 操作: 尝试上传 `.exe` 文件
  - 预期: 显示"不允许的文件类型"错误

- [ ] **TC10**: 附件大小限制
  - 操作: 上传超过 10 MB 的文件
  - 预期: 显示"文件太大"错误

- [ ] **TC11**: CSRF 保护
  - 操作: 提交表单时修改 CSRF token
  - 预期: 显示"无效的安全令牌"错误

- [ ] **TC12**: 版块权限检查
  - 操作: 在无权限的版块尝试发帖
  - 预期: 显示"您没有权限在此版块发帖"

#### 3.2 回复功能测试

- [ ] **TC13**: 已登录用户可以回复主题
  - 前置条件: 用户已登录,有回复权限
  - 操作: 填写回复内容,点击发表
  - 预期: 回复成功,跳转到主题页面

- [ ] **TC14**: 引用功能正常
  - 操作: 点击"引用"按钮
  - 预期: 编辑器显示 `[quote]...[/quote]` 格式

- [ ] **TC15**: 引用内容清理
  - 操作: 引用包含 `[hide]` 标签的帖子
  - 预期: 隐藏内容被替换为 `[隐藏内容]`

- [ ] **TC16**: 关闭的主题无法回复
  - 前置条件: 主题已关闭
  - 操作: 尝试回复
  - 预期: 显示"主题已关闭"错误

- [ ] **TC17**: 回复统计更新正确
  - 操作: 回复主题
  - 预期: 主题的 `replies` 字段 +1

#### 3.3 编辑功能测试

- [ ] **TC18**: 作者可以编辑自己的帖子
  - 前置条件: 用户是帖子作者
  - 操作: 访问 `/post/{id}/edit`,修改内容
  - 预期: 编辑成功

- [ ] **TC19**: 版主可以编辑任何帖子
  - 前置条件: 用户是版主
  - 操作: 编辑其他用户的帖子
  - 预期: 编辑成功

- [ ] **TC20**: 编辑时限检查
  - 前置条件: 帖子发布超过 10 分钟
  - 操作: 尝试编辑
  - 预期: 显示"编辑时限已过"错误 (非版主)

- [ ] **TC21**: 主题帖编辑更新标题
  - 操作: 编辑主题帖,修改标题
  - 预期: 主题表和帖子表的标题都更新

#### 3.4 删除功能测试

- [ ] **TC22**: 作者可以删除自己的帖子
  - 前置条件: 用户是帖子作者
  - 操作: 点击删除
  - 预期: 删除成功

- [ ] **TC23**: 版主可以删除任何帖子
  - 前置条件: 用户是版主
  - 操作: 删除其他用户的帖子
  - 预期: 删除成功

- [ ] **TC24**: 有回复的主题不能删除
  - 前置条件: 主题有回复
  - 操作: 尝试删除主题帖
  - 预期: 显示"主题有回复,无法删除"错误

#### 3.5 BBCode 功能测试

- [ ] **TC25**: 粗体标签解析
  - 输入: `[b]粗体文字[/b]`
  - 预期: `<b>粗体文字</b>`

- [ ] **TC26**: 斜体标签解析
  - 输入: `[i]斜体文字[/i]`
  - 预期: `<i>斜体文字</i>`

- [ ] **TC27**: 链接标签解析
  - 输入: `[url=http://example.com]链接文字[/url]`
  - 预期: `<a href="http://example.com" target="_blank">链接文字</a>`

- [ ] **TC28**: 图片标签解析
  - 输入: `[img]/uploads/image.jpg[/img]`
  - 预期: `<img src="/uploads/image.jpg" alt="" class="post-image">`

- [ ] **TC29**: 引用标签解析
  - 输入: `[quote]引用内容[/quote]`
  - 预期: `<blockquote>引用内容</blockquote>`

- [ ] **TC30**: 代码标签解析
  - 输入: `[code]console.log("Hello");[/code]`
  - 预期: `<pre><code>console.log("Hello");</code></pre>`

- [ ] **TC31**: 隐藏标签解析
  - 输入: `[hide=10]隐藏内容[/hide]`
  - 预期: `[隐藏内容,回复后可见]` (对未付费用户)

- [ ] **TC32**: 附件 BBCode 解析
  - 输入: `[attach]12345[/attach]`
  - 预期: 根据附件类型显示图片或下载链接

- [ ] **TC33**: 嵌套标签解析
  - 输入: `[b][i]粗斜体[/i][/b]`
  - 预期: `<b><i>粗斜体</i></b>`

#### 3.6 安全测试

- [ ] **TC34**: XSS 防护
  - 输入: `<script>alert('XSS')</script>`
  - 预期: 脚本被转义,不执行

- [ ] **TC35**: SQL 注入防护
  - 输入: `'; DROP TABLE cdb_threads; --`
  - 预期: 内容被安全存储,不影响数据库

- [ ] **TC36**: CSRF 攻击防护
  - 操作: 提交伪造的跨站请求
  - 预期: CSRF token 验证失败,请求被拒绝

- [ ] **TC37**: 路径遍历防护
  - 操作: 附件文件名包含 `../../`
  - 预期: 文件名被清理,路径遍历失败

- [ ] **TC38**: 文件上传安全
  - 操作: 上传包含 PHP 代码的图片
  - 预期: 文件类型验证失败,上传被拒绝

---

## 实现优先级

### Phase 1: BBCode 集成 (最高优先级) ⏱️ 1 天

**原因**: 现有 293,477 个帖子需要显示,BBCode 解析是关键

**任务**:
1. ✅ 扩展 `BBCodeParser` 完整实现
2. ✅ 创建 `BbcodeExtension` Twig 扩展
3. ✅ 在 `ViewRenderer` 中注册扩展
4. ✅ 添加单元测试

**测试**:
- [ ] 所有 Legacy 帖子正确显示
- [ ] BBCode 标签正确解析
- [ ] 附件 BBCode 正确显示
- [ ] 测试覆盖率 >90%

### Phase 2: 新主题表单 (次高优先级) ⏱️ 2 天

**任务**:
1. ✅ 创建 `PostService`
2. ✅ 扩展 `PostController` (如果需要)
3. ✅ 创建 `templates/post/new_thread.html.twig`
4. ✅ 创建 JavaScript BBCode 编辑器
5. ✅ 添加单元测试和集成测试

**测试**:
- [ ] 表单显示正常
- [ ] 数据验证正确
- [ ] 权限检查正确
- [ ] 数据库插入成功
- [ ] 测试覆盖率 >85%

### Phase 3: 回复表单 (第三优先级) ⏱️ 2 天

**任务**:
1. ✅ 扩展 `PostController` 添加引用功能
2. ✅ 创建 `templates/post/reply.html.twig`
3. ✅ 实现引用 BBCode 生成
4. ✅ 添加单元测试和集成测试

**测试**:
- [ ] 回复表单显示正常
- [ ] 引用功能正确
- [ ] 主题更新正确
- [ ] 版块统计更新正确
- [ ] 测试覆盖率 >85%

### Phase 4: 附件上传 UI (第四优先级) ⏱️ 2 天

**任务**:
1. ✅ 扩展 `AttachmentController` 添加临时上传
2. ✅ 创建 JavaScript 拖拽上传组件
3. ✅ 实现附件预览和删除
4. ✅ 集成到发帖表单
5. ✅ 添加集成测试

**测试**:
- [ ] 拖拽上传工作正常
- [ ] 进度显示正确
- [ ] 文件验证正确
- [ ] 附件预览显示
- [ ] 测试覆盖率 >85%

### Phase 5: 账户激活集成 (第五优先级) ⏱️ 1 天

**任务**:
1. ✅ 修改 `AuthController@login` 检查激活状态
2. ✅ 创建激活提示模板
3. ✅ 集成 `ActivationService`
4. ✅ 添加流程测试

**测试**:
- [ ] 未激活用户登录时显示提示
- [ ] 激活链接发送成功
- [ ] 激活后可以正常发帖
- [ ] 测试覆盖率 >90%

---

## 附录

### A. Legacy 函数映射 (Legacy Function Mapping)

| Legacy 函数 | 现代 PHP 8.3 类 | 文件 |
|-------------|-----------------|------|
| `dhtmlspecialchars()` | `htmlspecialchars()` | Built-in |
| `censor()` | `ContentValidator::censor()` | TODO |
| `checkpost()` | `PostValidator::validate()` | TODO |
| `checkflood()` | `FloodControlService::check()` | TODO |
| `forumperm()` | `PermissionService::checkForumPermission()` | ✅ 已实现 |
| `submitcheck()` | `CsrfTokenService::validate()` | ✅ 已实现 |
| `discuzcode()` | `BBCodeParser::parse()` | ⚠️ Stub (需完善) |
| `attach_upload()` | `AttachmentUploadService::upload()` | ✅ 已实现 |
| `updatepostcredits()` | `CreditService::awardForPost()` | TODO |

### B. 错误代码映射 (Error Code Mapping)

| Legacy 错误消息 | 现代 Error Code | HTTP 状态码 |
|----------------|-----------------|-------------|
| `post_sm_isnull` | `missing_fields` | 400 |
| `post_flood_ctrl` | `flood_control` | 429 |
| `group_nopermission` | `permission_denied` | 403 |
| `post_newthread_mod_succeed` | (成功) | 200 |
| `post_reply_succeed` | (成功) | 200 |
| `thread_nonexistence` | `thread_not_found` | 404 |
| `forum_nonexistence` | `forum_not_found` | 404 |

### C. 数据库查询优化建议

```php
// ❌ BAD: N+1 query problem
foreach ($attachments as $attachment) {
    $author = $db->fetch_first("SELECT username FROM cdb_members WHERE uid = {$attachment['uid']}");
}

// ✅ GOOD: Single query with JOIN
$attachments = $db->query("
    SELECT a.*, m.username
    FROM cdb_attachments a
    LEFT JOIN cdb_members m ON a.uid = m.uid
    WHERE a.pid = {$pid}
");

// ✅ GOOD: Batch loading with IN
$uids = array_unique(array_column($attachments, 'uid'));
$users = $db->query("SELECT uid, username FROM cdb_members WHERE uid IN (" . implode(',', $uids) . ")");
```

### D. 性能基准 (Performance Benchmarks)

| 操作 | 目标响应时间 | 实际响应时间 | 状态 |
|------|-------------|-------------|------|
| 新主题表单加载 | < 100ms | TBD | ⏳ |
| 提交新主题 | < 500ms | TBD | ⏳ |
| 回复表单加载 | < 100ms | TBD | ⏳ |
| 提交回复 | < 500ms | TBD | ⏳ |
| BBCode 解析 (单帖) | < 50ms | TBD | ⏳ |
| 附件上传 (10MB) | < 30s | TBD | ⏳ |

### E. 依赖项清单 (Dependency Checklist)

**必须实现的依赖**:

- [ ] `PostService` - 帖子业务逻辑
- [ ] `PostRepository` - 帖子数据访问 (需要扩展)
- [ ] `PostValidator` - 帖子数据验证
- [ ] `FloodControlService` - 发帖洪水控制 (可能需要实现)
- [ ] `CreditService` - 积分系统 (部分功能需要)
- [ ] `BBCodeParser` - 完整 BBCode 解析 (当前是 stub)
- [ ] `BBCodeEditor` JavaScript - 前端编辑器
- [ ] `AttachmentUpload` JavaScript - 前端上传组件

---

## 总结

本文档详细分析了 Legacy Discuz! 6.1F 的发帖系统,并设计了现代 PHP 8.3 实现。关键要点:

1. **零表更改**: 所有功能使用现有表,无需新建
2. **Legacy 兼容**: 现有 293,477 个帖子必须正确显示
3. **安全优先**: CSRF、XSS、SQL 注入防护全覆盖
4. **TDD 方法**: 测试驱动开发,>85% 覆盖率
5. **分阶段实施**: 5 个阶段,共 8 天开发时间

**下一步行动**:

1. 获得设计文档批准
2. 开始 Phase 2 (测试计划编写)
3. 实现 Phase 1 (BBCode 集成) - 最高优先级

---

**文档结束**

**创建日期**: 2026-02-18
**最后更新**: 2026-02-18
**版本**: 1.0
