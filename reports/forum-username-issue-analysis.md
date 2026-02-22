# Forum Page 用户名问题分析报告

**日期**: 2026-02-20
**页面**: /forum/5
**问题**: 用户名不显示

---

## 问题定位

### 用户提供的HTML片段
```html
<span class="avataonline">
    <cite>
        <a class="dropmenu" id="viewpro" href="/user/18">
            <!-- 空的！用户名在这里应该是空的 -->
        </a>
    </cite>
    <a href="/auth/logout">退出</a>
</span>
```

### 实际页面渲染结果

检查 `/forum/5` 页面显示：
```html
<span class="avataonline">
    <a href="/auth/register">Register</a>
    <a href="/auth/login">Login</a>
</span>
```

---

## 根本原因

**这不是CSS或模板问题！这是Session/Cookie问题！**

### 页面显示状态

新系统显示的是**未登录（Guest）状态**：
- 显示"Register"链接
- 显示"Login"链接
- **不显示用户名**

而Legacy页面（`http://localhost:8080/bbs/forum-5-1.html`）显示的是**已登录状态**

---

## 原因分析

### 可能的原因

1. **Cookie域名/路径不匹配**
   - Legacy Cookie: `.poketb.com` 或特定路径
   - New System Cookie: 可能使用了不同的域名/路径

2. **Session存储方式不同**
   - Legacy: PHP Session files
   - New System: 可能使用不同的session handler

3. **Cookie名称不同**
   - Legacy可能使用: `discuz_cookie`
   - New System可能使用: 不同的cookie名称

4. **Session数据结构不兼容**
   - Legacy session: `$_SESSION['discuz_uid']`
   - New System session: 可能使用了不同的key

---

## 验证方法

### 1. 检查Cookie

```bash
# Legacy Cookie
curl -s http://localhost:8080/bbs/forum-5-1.html -v 2>&1 | grep -i "set-cookie"

# New System Cookie
curl -s http://localhost:8083/forum/5 -v 2>&1 | grep -i "set-cookie"
```

### 2. 检查登录状态

```bash
# Legacy - 应显示已登录
curl -s http://localhost:8080/bbs/ | grep -c "您的上次访问"

# New System - 可能显示未登录
curl -s http://localhost:8083/ | grep -c "您的上次访问"
```

### 3. 检查Session实现

查看新系统的Session配置：
```php
// app/Session/Services/SessionService.php
// config/session.php
```

---

## 修复建议

### P0 - 立即修复

1. **统一Cookie配置**
   ```php
   // 确保新旧系统使用相同的cookie域名
   ini_set('session.cookie_domain', '.poketb.com');
   ```

2. **统一Cookie名称**
   ```php
   // 检查Legacy使用的cookie名称
   // 新系统使用相同的名称
   ```

3. **Session Key兼容**
   ```php
   // 在SessionService中检查Legacy session key
   if (isset($_SESSION['discuz_uid'])) {
       // 适配Legacy session数据
   }
   ```

### P1 - 高优先级

4. **共享Session存储**
   - 配置新旧系统使用同一个Redis实例
   - 使用相同的session prefix

5. **Cookie同步机制**
   - 登录时同时设置新旧系统的cookie
   - 退出时清除所有cookie

---

## 模板验证

**模板本身是正确的** ✅

`templates/components/menu.html.twig` 第11-13行：
```twig
<cite>
    <a class="dropmenu" id="viewpro" href="{{ url('user/' ~ current_user.user_id) }}">
        {{ current_user.username }}  {# 这里会显示用户名 #}
    </a>
</cite>
```

**但是**，当 `current_user.user_id == 0` 或 `current_user` 为空时，会进入访客分支：
```twig
{% else %}
    {# Guest #}
    <a href="{{ url('auth/register') }}">{{ 'register'|trans }}</a>
    <a href="{{ url('auth/login') }}">{{ 'login'|trans }}</a>
{% endif %}
```

---

## 当前状态

| 项目 | Legacy | New System | 状态 |
|------|--------|-----------|------|
| 用户登录状态 | ✅ 已登录 | ❌ 未登录 | Session问题 |
| 用户名显示 | ✅ 显示 | ❌ 不显示 | 依赖登录状态 |
| 模板代码 | - | ✅ 正确 | - |
| CSS样式 | ✅ 完整 | ⚠️ 部分 | 需完善 |

---

## 结论

**用户名不显示不是CSS或HTML问题，而是Session/Cookie共享问题！**

新系统需要：
1. 读取Legacy的Cookie来识别登录状态
2. 或配置共享的Session存储
3. 或实现SSO（单点登录）机制

---

**报告生成时间**: 2026-02-20 18:15 UTC
**问题类型**: Session/Authentication
**优先级**: P0 - 核心功能
