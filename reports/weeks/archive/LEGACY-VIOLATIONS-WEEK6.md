# Week 6 补全 - Legacy违规功能审查报告

**日期**: 2026-02-19
**审查范围**: Week 6 Frontend Implementation 补全工作
**审查原则**: 零表创建原则、Legacy功能对等、架构一致性

---

## 执行摘要

在Week 6补全的路由集成工作中发现了多处违反Legacy迁移原则的实现：
- ❌ 3个违规创建的表（需删除）
- ✅ 2个已批准的例外表
- ⚠️  多处方法名不匹配
- ⚠️  架构偏离（Database::query() workaround）

---

## 1. 数据库表违规

### 1.1 违规创建的表（需删除）

| 违规表 | Legacy对应表 | 字段差异 | 处理方案 |
|--------|-------------|---------|----------|
| `cdb_friends` | `cdb_buddys` | 新表使用了`user_id`/`friend_id`，Legacy使用`uid`/`buddyid` | **删除**，使用`cdb_buddys` |
| `cdb_user_profiles` | `cdb_memberfields` | 新表拆分了profile，Legacy统一在`cdb_memberfields` | **删除**，使用`cdb_memberfields` |
| `cdb_moderation_logs` | 无 | 全新功能，Legacy无对应 | **删除**（暂不需要） |

#### 1.1.1 cdb_friends vs cdb_buddys

**Legacy结构 (cdb_buddys)**:
```sql
uid mediumint(8) unsigned
buddyid mediumint(8) unsigned
grade tinyint(3) unsigned
dateline int(10) unsigned
description char(255)
```

**违规创建 (cdb_friends)**:
```sql
user_id mediumint unsigned
friend_id mediumint unsigned
status enum('pending','accepted','blocked')
created_at timestamp
updated_at timestamp
```

**问题**:
1. 表名不同（friends vs buddys）
2. 字段名不同（user_id vs uid, friend_id vs buddyid）
3. 新增了status字段（Legacy没有pending状态）

**处理方案**:
```sql
DROP TABLE cdb_friends;
-- 所有friendship相关代码必须使用cdb_buddys表结构
```

#### 1.1.2 cdb_user_profiles vs cdb_memberfields

**Legacy结构 (cdb_memberfields)** - 已包含所有用户资料：
```sql
uid mediumint(8) unsigned
nickname varchar(30)
site varchar(75)        -- 对应新的website
qq varchar(12)
location varchar(30)
bio text                -- 对应新的bio字段
...更多字段
```

**违规创建 (cdb_user_profiles)**:
```sql
user_id mediumint unsigned
gender int
birthday binary(0)
bio text
signature text
location varchar(30)
website varchar(75)
qq varchar(12)
wechat binary(0)        -- Legacy没有
real_name binary(0)     -- Legacy没有
id_number binary(0)     -- Legacy没有
```

**问题**:
1. 重复创建了用户资料存储
2. 添加了Legacy不存在的字段（wechat, real_name, id_number）
3. 拆分了应该统一在cdb_memberfields的数据

**处理方案**:
```sql
DROP TABLE cdb_user_profiles;
-- 所有user profile代码必须使用cdb_memberfields表结构
```

### 1.2 已批准的例外表（保留）

| 表名 | 批准理由 | 批准日期 |
|------|---------|---------|
| `cdb_credits` | Legacy cdb_creditslog结构不兼容，需要统一积分事务日志 | 2026-02-15 |
| `cdb_registration_log` | 新安全功能，Legacy不存在，用于注册审计 | 2026-02-15 |

---

## 2. 方法名不匹配问题

### 2.1 Repository方法名

| 控制器调用 | 实际Repository方法 | 状态 |
|-----------|------------------|------|
| `ThreadRepository::findById()` | `ThreadRepository::getThreadById()` | ❌ 不匹配 |
| `UserRepository::getById()` | `UserRepository::findById()` | ❌ 可能不匹配 |

**修复方案**: 统一方法命名规范
- Repository层使用 `get{Entity}ById()` 格式
- 或者统一为 `findById()` 格式

### 2.2 控制器依赖注入顺序

**问题**: `config/controllers.php` 中的依赖顺序与实际构造函数不匹配

**PaymentController构造函数**:
```php
public function __construct(
    PaymentService $paymentService,
    UserSessionService $userSession,
    CsrfTokenService $csrf,
    CreditTransactionService $creditService,
    ThreadRepository $threadRepository,
    UserRepository $userRepository,
    PermissionService $permission
)
```

**config/controllers.php** (已修复):
```php
\Discuz\Http\Controllers\PaymentController::class => [
    'payment_service',
    'user_session_service',
    'csrf_service',
    'credit_service',
    'thread_repository',
    'user_repository',
    'permission_service',
]
```

---

## 3. 架构违规

### 3.1 Database类添加query()方法

**问题**: `app/Database/Database.php` 添加了`query()`方法作为临时workaround

**违规原因**:
1. Database类应该是简单的PDO wrapper
2. 添加业务方法（query()）破坏了单一职责原则
3. 使SessionManager直接依赖Database而非Repository

**正确架构**:
```
Controller → Service → Repository → Database (PDO)
```

**当前错误架构**:
```
SessionManager → Database::query() (bypassing Repository)
```

**修复方案**:
```php
// 1. 删除Database::query()方法
// 2. 使用file session driver（已配置SESSION_DRIVER=file）
// 3. 如果需要database session，创建SessionRepository
```

### 3.2 Bootstrap服务创建

**问题**: Bootstrap直接new服务实例，而非使用Container

**当前代码**:
```php
self::$app['auth_service'] = new AuthService(
    self::$app['db'],
    self::$app['user_session_service'],
    new PasswordVerifier()
);
```

**正确做法**: 使用ServiceContainer统一管理

---

## 4. 功能缺失或不匹配

### 4.1 Session存储

**Legacy**: 使用自定义的`cdb_sessions`表存储在线用户信息
**Modern**: 尝试使用PHP标准session_handler

**问题**:
- Legacy的cdb_sessions不是PHP session，是在线用户列表
- Modern不应该复用cdb_sessions表存储PHP session

**正确方案**: 使用file session driver

### 4.2 Friendship功能

**Legacy**:
- 表: `cdb_buddys`
- 功能: 好友系统，无pending状态

**Modern** (错误实现):
- 表: `cdb_friends`
- 功能: 添加了pending/approval流程（Legacy没有）

**修复**: 回退到Legacy的简单好友系统

---

## 5. 修复优先级

### P0 - 立即修复（阻塞路由集成）

1. ✅ ~~删除Database::query()方法~~ (已用file session替代)
2. ⚠️ 修复Repository方法名不匹配
3. ⚠️ 删除违规表：`cdb_friends`, `cdb_user_profiles`, `cdb_moderation_logs`

### P1 - 短期修复（Week 13-14）

4. 统一FriendshipService使用`cdb_buddys`
5. 统一UserProfileService使用`cdb_memberfields`
6. 更新所有Repository方法调用

### P2 - 架构重构（后续）

7. 重构SessionManager使用Repository模式（如果需要database session）
8. Bootstrap使用ServiceContainer创建服务

---

## 6. 删除违规表的SQL

```sql
-- 删除违规创建的表
DROP TABLE IF EXISTS cdb_friends;
DROP TABLE IF EXISTS cdb_user_profiles;
DROP TABLE IF EXISTS cdb_moderation_logs;

-- 验证Legacy表存在
SELECT COUNT(*) FROM cdb_buddys;
SELECT COUNT(*) FROM cdb_memberfields;
```

---

## 7. 代码修复清单

### 7.1 FriendshipService
- [ ] 更新使用`cdb_buddys`表
- [ ] 移除pending/approval逻辑
- [ ] 使用Legacy字段名（uid, buddyid, grade, dateline）

### 7.2 UserProfileService
- [ ] 更新使用`cdb_memberfields`表
- [ ] 移除不存在的字段（wechat, real_name, id_number）
- [ ] 使用Legacy字段名（uid, nickname, site, qq, bio, location）

### 7.3 ThreadRepository
- [ ] 统一方法名：`findById()` 或 `getThreadById()`
- [ ] 更新PaymentController调用

### 7.4 Database类
- [ ] 移除`query()`方法（如果不再需要）

---

## 8. 总结

**关键原则**:
1. ✅ 不得创建Legacy已有的表
2. ✅ 不得添加Legacy没有的功能字段
3. ✅ 必须使用Legacy的字段名和表名
4. ✅ Repository层必须匹配Legacy数据库结构

**Week 6 补全偏离原则的根本原因**:
- 没有充分检查Legacy已有的表结构和功能
- 自行设计了"更好"的表结构（如friends带pending状态）
- 使用了现代化的字段名（user_id vs uid）

**立即行动**:
1. 删除3个违规表
2. 修复Repository方法调用
3. 更新FriendshipService使用cdb_buddys
4. 更新UserProfileService使用cdb_memberfields

---

**报告生成时间**: 2026-02-19
**审查人**: Claude Code Agent
**状态**: 待修复
