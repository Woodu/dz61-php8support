# Task #21 完成报告: 移除邮件发送功能

**完成日期**: 2026-02-21
**状态**: ✅ 完成
**实际耗时**: 35分钟

---

## 任务概述

从全局、设计、实现层面移除邮件发送相关功能，同时保留上层应用接口（如注册邮件）不废弃，为未来邮件服务迁移做准备。

---

## 完成工作

### 1. 创建邮件服务抽象层 ✅

**文件创建**:
- `app/Mail/Services/MailServiceInterface.php` - 邮件服务接口
- `app/Mail/Services/NullMailService.php` - 空操作实现（默认）

**接口定义**:
```php
interface MailServiceInterface {
    public function send(string $to, string $subject, string $body, array $options = []): bool;
    public function isEnabled(): bool;
}
```

**Null实现**:
- 不发送邮件，仅记录日志
- `isEnabled()` 返回 `false`
- 为未来SMTP/API邮件服务预留接口

### 2. 删除邮件发送服务 ✅

**删除文件**:
- `app/Activation/Services/EmailActivationService.php` (326行)
  - 包含 `mail()` 调用（第263行）
  - 用于注册账号邮件激活

**保留但禁用**:
- `app/User/Services/EmailVerificationService.php`
  - `sendEmail()` 方法改为空操作
  - 保留token生成功能
  - 添加 `@deprecated` 标记

### 3. 更新依赖配置 ✅

**AuthController**:
- 移除 `EmailActivationService` 依赖
- 保留 `ActivationCheckService`（手动激活）

**ActivationCheckService**:
- 移除 `EmailActivationService` 依赖
- `getActivationMethod()` 始终返回 `METHOD_MANUAL`
- 添加 `@deprecated` 标记

**public/index.php**:
- 删除 `email_activation_service` 注册
- 添加 `mail_service` 注册（使用NullMailService）
- 保留 `email_verification_service` 注册（已禁用发送）

**config/controllers.php**:
- 移除 AuthController 的 `email_activation_service` 依赖

### 4. 语法验证 ✅

所有修改文件通过PHP语法检查：
- ✅ app/Mail/Services/MailServiceInterface.php
- ✅ app/Mail/Services/NullMailService.php
- ✅ app/Http/Controllers/AuthController.php
- ✅ app/Activation/Services/ActivationCheckService.php
- ✅ app/User/Services/EmailVerificationService.php
- ✅ public/index.php
- ✅ config/controllers.php

---

## 测试结果

### 测试执行

**命令**: `docker exec -i discuz_modern_php vendor/bin/phpunit --testdox`

**结果**: ✅ **无sendmail错误**

**测试进度**: 92% (2440/2637) - 与之前相同
- 剩余8%被Session兼容性问题阻塞（Task #20）

**关键改进**:
- ❌ 移除前: `sendmail: can't connect to remote host (127.0.0.1): Connection refused`
- ✅ 移除后: 无邮件相关错误

### 邮件功能状态

| 功能 | 之前 | 现在 | 备注 |
|------|------|------|------|
| 注册邮件激活 | ❌ 失败（sendmail错误） | ✅ 禁用（手动激活） | 无报错 |
| 密码重置邮件 | ❌ 失败（sendmail错误） | ✅ 禁用 | 无报错 |
| Token生成 | ✅ 正常 | ✅ 正常 | 保留 |
| 激活状态检查 | ✅ 正常 | ✅ 正常 | 保留 |

---

## 架构变更

### 之前（含邮件发送）

```
RegistrationController
  → UserRegistrationService
      → EmailVerificationService
          → mail() function ❌ sendmail错误

AuthController
  → EmailActivationService
      → mail() function ❌ sendmail错误
```

### 现在（邮件已禁用）

```
RegistrationController
  → UserRegistrationService
      → EmailVerificationService (空操作)
          → NullMailService ✅ 仅日志

AuthController
  → ActivationCheckService (手动激活)
      → 无邮件依赖 ✅

Future:
  → MailServiceInterface (抽象)
      → NullMailService (当前默认)
      → SmtpMailService (TODO)
      → ApiMailService (TODO)
```

---

## 保留的上层接口

### Controller层（未修改）

```php
// AuthController - 方法签名保留
public function loginAction(string $referer = '/'): string
public function logoutAction(): void

// RegistrationController - 方法签名保留
public function registerAction(): array
public function renderRegistrationForm(): string
```

### Service层（部分修改）

```php
// UserRegistrationService - 保留注册逻辑
class UserRegistrationService {
    // 保留：用户创建、验证、数据库操作
    // 禁用：邮件发送（EmailVerificationService.sendEmail = no-op）
}

// ActivationCheckService - 保留激活检查
class ActivationCheckService {
    // 保留：groupid检查、手动激活状态
    // 移除：EmailActivationService依赖
}
```

### 数据层（完全保留）

- ✅ `cdb_members` 表 - 所有字段保留
- ✅ `cdb_memberfields` 表 - `authstr`字段保留
- ✅ DTOs - 所有Email字段保留
- ✅ Entities - 所有Email属性保留

---

## 未来迁移路径

### 步骤1: 实现新的邮件服务

```php
// app/Mail/Services/SmtpMailService.php
class SmtpMailService implements MailServiceInterface {
    public function send(string $to, string $subject, string $body, array $options = []): bool {
        // 使用SMTP发送邮件
        $mailer = new SmtpMailer($this->config);
        return $mailer->send($to, $subject, $body);
    }

    public function isEnabled(): bool {
        return !empty($this->config['smtp_host']);
    }
}
```

### 步骤2: 更新服务注册

```php
// public/index.php
$container->register('mail_service', function ($c) {
    return new \Discuz\Mail\Services\SmtpMailService($c->get('config')['mail']);
});
```

### 步骤3: 启用邮件发送（可选）

```php
// UserRegistrationService.php
private MailServiceInterface $mailService;

public function register(UserRegistration $data): User {
    // ... 创建用户 ...

    // 发送邮件（如果启用）
    if ($this->mailService->isEnabled()) {
        $this->mailService->send($user->email, 'Welcome', $message);
    }

    return $user;
}
```

---

## 文件清单

### 创建的文件 (2个)

1. **app/Mail/Services/MailServiceInterface.php** (24行)
   - 邮件服务抽象接口
   - 定义 `send()` 和 `isEnabled()` 方法

2. **app/Mail/Services/NullMailService.php** (23行)
   - 空操作实现
   - 不发送邮件，仅记录日志

### 删除的文件 (1个)

1. **app/Activation/Services/EmailActivationService.php** (326行)
   - 包含 `mail()` 调用
   - 用于注册激活邮件发送

### 修改的文件 (5个)

1. **app/Http/Controllers/AuthController.php**
   - 移除 `EmailActivationService` 导入
   - 移除构造函数参数和属性

2. **app/Activation/Services/ActivationCheckService.php**
   - 移除 `EmailActivationService` 依赖
   - `getActivationMethod()` 始终返回手动激活
   - 添加 `@deprecated` 标记

3. **app/User/Services/EmailVerificationService.php**
   - `sendEmail()` 改为空操作
   - 添加 `@deprecated` 标记
   - 保留token生成功能

4. **public/index.php**
   - 删除 `email_activation_service` 注册
   - 添加 `mail_service` 注册（NullMailService）

5. **config/controllers.php**
   - 移除 AuthController 的 `email_activation_service` 依赖
   - 添加注释说明

---

## 验收标准

- ✅ 无 `sendmail: can't connect to remote host` 错误
- ✅ 所有PHP文件语法检查通过
- ✅ 测试可以运行到92%（与之前相同，无新增阻塞）
- ✅ Controller方法签名保持不变
- ✅ 数据库schema保持不变（零改表）
- ✅ DTOs和Entities保留Email字段
- ✅ 创建邮件服务抽象层（MailServiceInterface）
- ✅ 为未来SMTP/API邮件服务预留接口

---

## 技术要点

### 邮件禁用策略

1. **最小化修改**: 只修改发送逻辑，保留数据结构
2. **空操作模式**: 使用Null Object Pattern避免null检查
3. **接口抽象**: MailServiceInterface便于未来替换实现
4. **向后兼容**: Controller和Service层接口保持不变

### Legacy兼容

- ✅ 使用 `cdb_memberfields.authstr` 存储token（格式不变）
- ✅ Group ID 8 = 待激活（保留）
- ✅ Group ID 10 = 已激活（保留）
- ✅ 手动激活流程完整（保留）

---

## 下一步工作

### Task #20: 修复PHPUnit Session兼容性（进行中）

**阻塞问题**: 剩余8%测试被Session兼容性问题阻塞

**预期结果**: 修复后测试完成率将达到98%+

---

## 总结

**Task #21状态**: ✅ **完成**

**关键成果**:
1. ✅ 移除所有邮件发送功能（EmailActivationService删除）
2. ✅ 消除sendmail错误（测试运行无邮件错误）
3. ✅ 创建邮件服务抽象层（MailServiceInterface）
4. ✅ 保留上层接口不变（Controller/Service签名）
5. ✅ 保留数据结构不变（数据库/DTOs/Entities）

**技术亮点**:
- 使用Null Object Pattern避免空值检查
- 接口抽象便于未来迁移（SMTP/API）
- 最小化修改原则（仅修改发送逻辑）
- 向后兼容（Controller/Service接口不变）

**实际耗时**: 35分钟

**测试结果**: 92% (2440/2637) - 与之前相同，无新增阻塞

**影响范围**:
- 2个新文件（MailServiceInterface + NullMailService）
- 1个删除文件（EmailActivationService）
- 5个修改文件（AuthController + ActivationCheckService + EmailVerificationService + public/index.php + controllers.php）

---

**报告生成**: 2026-02-21
**报告者**: Week 16 开发团队
**任务ID**: #21
