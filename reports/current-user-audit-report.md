# 全面审查报告 - current_user 传递问题

**日期**: 2026-02-20
**问题**: /thread/16130 提示要登录，但登录状态存在

---

## 审查结果

### 使用Twig模板的Controller

| Controller | 传递current_user | 包含username | 状态 |
|-----------|----------------|------------|------|
| ForumController | ✅ | ✅ | ✅ 完整 |
| AuthController | ❌ → ✅ | ❌ → ✅ | ✅ 已修复 |
| ProfileController | ❌ → ✅ | ❌ → ✅ | ✅ 已修复 |
| ThreadViewController | ❌ → ✅ | ❌ → ✅ | ✅ 已修复 |

### 问题详情

#### 1. ThreadViewController ❌ → ✅

**影响**: `/thread/{tid}` 页面无法显示用户名

**修复位置**: `app/Thread/Controllers/ThreadViewController.php:112-144`

**修复前**:
```php
return view('forum/thread.html.twig', [
    'thread' => [...],
    'posts' => $postsData,
    // ❌ 缺少current_user
]);
```

**修复后**:
```php
return view('forum/thread.html.twig', [
    'thread' => [...],
    'posts' => $postsData,
    'current_user' => [
        'user_id' => $userId,
        'username' => $this->sessionService->getUsername(),
        'group_id' => $this->sessionService->getGroupId(),
        'admin_id' => $this->sessionService->getAdminId(),
    ],
]);
```

#### 2. AuthController ❌ → ✅

**影响**: `/auth/login` 页面无法正确显示登录状态

**修复位置**: `app/Http/Controllers/AuthController.php:128-136`

**修复**: 在 `$data` 数组中添加 `current_user` 字段

#### 3. ProfileController ❌ → ✅

**影响**: `/user/{id}` 个人资料页无法正确显示登录状态

**修复位置**: `app/Http/Controllers/ProfileController.php:109-117`

**修复**: 在 `view()` 参数中添加 `current_user` 字段

#### 4. ForumController ✅

**状态**: 已经正确传递 `current_user` 和所有必要字段

**位置**: `app/Forum/Controllers/ForumController.php:388-393` (首页)
**位置**: `app/Forum/Controllers/ForumController.php:488-492` (板块页)

---

## 标准 current_user 结构

所有Controller都应该使用以下标准结构：

```php
'current_user' => [
    'user_id' => $sessionService->getUserId(),
    'username' => $sessionService->getUsername(),
    'group_id' => $sessionService->getGroupId(),
    'admin_id' => $sessionService->getAdminId(),
],
```

### 字段说明

| 字段 | 类型 | 说明 | 必需 |
|------|------|------|------|
| `user_id` | int | 用户ID (0=访客) | ✅ 是 |
| `username` | string | 用户名 | ✅ 是 |
| `group_id` | int | 用户组ID | ✅ 是 |
| `admin_id` | int | 管理员ID (0=非管理员) | ✅ 是 |

---

## 模板使用方式

```twig
{# templates/components/menu.html.twig #}
<span class="avataonline">
    {% if current_user is defined and current_user.user_id > 0 %}
        <cite>
            <a href="{{ url('user/' ~ current_user.user_id) }}">
                {{ current_user.username }}
            </a>
        </cite>
        <a href="{{ url('auth/logout') }}">{{ 'logout'|trans }}</a>
    {% else %}
        <a href="{{ url('auth/register') }}">{{ 'register'|trans }}</a>
        <a href="{{ url('auth/login') }}">{{ 'login'|trans }}</a>
    {% endif %}
</span>
```

**关键检查点**:
1. `current_user.user_id > 0` - 判断是否登录
2. `current_user.username` - 显示用户名
3. 两个字段都必须存在，否则无法正确显示

---

## 验证清单

- [x] ForumController - 首页
- [x] ForumController - 板块页
- [x] AuthController - 登录页
- [x] ProfileController - 个人资料页
- [x] ThreadViewController - 帖子详情页

---

## 修复的文件

1. `app/Forum/Controllers/ForumController.php`
   - 第488-492行：添加 `username` 和 `admin_id`

2. `app/Thread/Controllers/ThreadViewController.php`
   - 第112-144行：添加 `current_user` 完整结构

3. `app/Http/Controllers/AuthController.php`
   - 第128-136行：添加 `current_user` 到 `$data`

4. `app/Http/Controllers/ProfileController.php`
   - 第109-117行：添加 `current_user` 到 `view()`

---

## 测试方法

### 浏览器测试

1. 访问首页确认登录状态
2. 访问板块页 `/forum/5` 确认用户名显示
3. 访问帖子页 `/thread/16130` 确认用户名显示
4. 访问个人资料页确认用户名显示

### 代码审查

```bash
# 运行审查脚本
./scripts/audit-current-user.sh

# 检查特定Controller
grep -A 5 "'current_user'" app/Thread/Controllers/ThreadViewController.php
```

---

## 后续建议

### 代码规范

**在所有Controller中使用辅助方法**:

```php
// 在基类Controller中添加辅助方法
abstract class BaseController
{
    protected function getCurrentUserArray(): array
    {
        return [
            'user_id' => $this->sessionService->getUserId(),
            'username' => $this->sessionService->getUsername(),
            'group_id' => $this->sessionService->getGroupId(),
            'admin_id' => $this->sessionService->getAdminId(),
        ];
    }
}

// 在子类中使用
return view('template.html.twig', [
    'current_user' => $this->getCurrentUserArray(),
    // ...其他数据
]);
```

### CI/CD 检查

添加CI检查确保所有使用Twig模板的Controller都传递了完整的 `current_user` 数据。

---

**修复完成时间**: 2026-02-20 19:00 UTC
**修复文件数**: 4个
**修复Controller数**: 3个 (ForumController补全，其他3个新增)
**测试状态**: ✅ 代码已修复，等待浏览器验证
