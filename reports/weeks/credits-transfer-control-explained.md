# 积分转账控制系统完整说明

**日期**: 2026-02-14
**目的**: 解释旧系统的银行插件和积分转账控制逻辑

---

## 🎯 核心发现

### 旧系统的"银行插件"（Bank System）

**关键发现**：Discuz! 6.1F 有一个**独立的银行插件系统**，用于管理特定积分类型（如"金币"）的转账、存储、利息等功能。

#### 银行系统特点

1. **独立于普通积分系统**
   - 使用专门的 `moneycredits` 字段
   - 有独立的银行表（`cdb_banklist`, `cdb_bankoperation`, `cdb_banklog`）
   - 只有特定权限用户可以操作
   - 支持存款、取款、转账、利息等银行功能

2. **与 extcredits1-8 的关系**
   - extcredits1-8 是**普通积分系统**
   - 普通积分**不能**直接转账
   - 某些 extcredits 类型**可以**兑换**成银行积分（金币）
   - 兑换是单向的（extcredits → bank，不可逆）

3. **后台配置控制**
   - 每种 extcredits 类型都有独立的转账控制：
     - `allowexchangeout` - 是否允许转出
     - `allowexchangein` - 是否允许转入
   - 后台可以单独配置每种积分类型的规则

---

## 📋 extcredits1-8 配置结构详解

### 每种积分类型的完整配置

```php
$credits['extcredits1'] = [
    'title' => '金币',              // 积分名称
    'unit' => '枚',                // 单位
    'ratio' => 1,                  // 兑换汇率（1金币=1枚）
    'available' => true,           // 是否启用
    'showinthread' => true,       // 是否在帖子中显示
    'allowexchangeout' => true,    // ★ 是否允许转出★
    'allowexchangein' => true,     // ★ 是否允许转入★
    'lowerlimit' => 0,             // 最小余额限制（不能低于此值）
];

$credits['extcredits2'] = [
    'title' => '威望',
    'unit' => '点',
    'ratio' => 1,
    'available' => true,
    'showinthread' => true,
    'allowexchangeout' => false,   // �望不可转出
    'allowexchangein' => false,    // 威望不可转入
    'lowerlimit' => 0,
];

$credits['extcredits3'] = [
    'title' => '金钱',          // 银行积分
    'unit' => '枚',
    'ratio' => 0.1,             // 1金钱=10枚（汇率）
    'available' => true,
    'showinthread' => true,
    'allowexchangeout' => true,    // 金钱可以转出
    'allowexchangein' => true,     // 金钱可以转入
    'lowerlimit' => 0,
];

// ... extcredits4-8 类似
```

### 关键字段说明

#### allowexchangeout 和 allowexchangein

**作用**：控制该积分类型是否可以参与转账

**可能的值组合**：
1. `allowexchangeout=true, allowexchangein=true` - **可以转入和转出**（正常转账）
2. `allowexchangeout=false, allowexchangein=false` - **不能转入也不能转出**（系统积分，如经验值）
3. `allowexchangeout=false, allowexchangein=true` - **只能转入不能转出**（充值积分，如威望）
4. `allowexchangeout=true, allowexchangein=false` - **只能转出不能转入**（消耗积分，如道具）

**默认值**：
```php
'allowexchangeout' => null,  // 默认不允许转出
'allowexchangein' => null,   // 默认不允许转入
```

---

## 🔒 转账验证逻辑详解

### Legacy 转账功能（memcp.php）

**文件**: `include/memcp.php`

**验证流程**:
```php
// 1. 检查操作类型
if ($action == 'transfer' || $action == 'exchange') {

    // 2. 检查积分类型
    $creditstrans = $_GET['creditstrans'];

    // 3. 检查用户密码
    $password = $_GET['password'];

    // 4. 验证通过后检查转账规则
    if ($action == 'transfer') {
        // 检查该积分类型是否允许转账
        $allowexchangeout = $credits[$creditstrans]['allowexchangeout'];
        $allowexchangein = $credits[$creditstrans]['allowexchangein'];

        if (!$allowexchangeout || !$allowexchangein) {
            showmessage('credits_transfer_invalid');
        }
    }

    // 5. 检查余额是否足够
    $amount = $_GET['amount'];
    $lowerlimit = $credits[$creditstrans]['lowerlimit'];

    if ($user['extcredits'.$creditstrans] - $amount < $lowerlimit) {
        showmessage('credits_transfer_lowerlimit');
    }
}
```

---

## 🏗️ 现代化实现方案

### 1. CreditRules 配置增强

**文件**: `app/Credits/Config/CreditRules.php`

**增加配置项**：
```php
private array $transferRules = [
    'extcredits1' => [
        'allowTransferOut' => true,
        'allowTransferIn' => true,
        'minTransferAmount' => 10,     // 最小转账金额
        'maxTransferAmount' => 10000,  // 最大转账金额
    ],
    'extcredits2' => [
        'allowTransferOut' => false,  // 威望不可转出
        'allowTransferIn' => false,   // 威望不可转入
    ],
    'extcredits3' => [
        'allowTransferOut' => true,
        'allowTransferIn' => true,
        'minTransferAmount' => 1,      // 金钱最小转账
        'maxTransferAmount' => 5000, // 金钱最大转账
    ],
];

public function getTransferRules(string $creditType): array
{
    return $this->transferRules[$creditType] ?? [];
}

public function canTransferOut(string $creditType): bool
{
    $rules = $this->getTransferRules($creditType);
    return $rules['allowTransferOut'] ?? false;
}

public function canTransferIn(string $creditType): bool
{
    $rules = $this->getTransferRules($creditType);
    return $rules['allowTransferIn'] ?? false;
}

public function getMinTransferAmount(string $creditType): int
{
    $rules = $this->getTransferRules($creditType);
    return $rules['minTransferAmount'] ?? 0;
}

public function getMaxTransferAmount(string $creditType): int
{
    $rules = $this->getTransferRules($creditType);
    return $rules['maxTransferAmount'] ?? PHP_INT_MAX;
}
```

### 2. CreditService->transferCredits() 增强

**文件**: `app/Credits/Services/CreditService.php`

**增加验证逻辑**:
```php
public function transferCredits(int $fromUserId, int $toUserId, string $creditType, int $amount): bool
{
    // 1. 基础验证
    if ($fromUserId === $toUserId) {
        throw new \InvalidArgumentException('Cannot transfer to yourself');
    }

    // 2. 验证积分类型
    $this->validateCreditType($creditType);

    // 3. 验证金额
    $this->validateAmount($amount);

    // 4. ★ 检查转账规则★
    if (!$this->rules->canTransferOut($creditType)) {
        throw new \RuntimeException(
            "Credit type '{$creditType}' does not allow transfer out"
        );
    }

    if (!$this->rules->canTransferIn($creditType)) {
        throw new \RuntimeException(
            "Credit type '{$creditType}' does not allow transfer in"
        );
    }

    // 5. 检查最小/最大转账金额
    $minAmount = $this->rules->getMinTransferAmount($creditType);
    $maxAmount = $this->rules->getMaxTransferAmount($creditType);

    if ($amount < $minAmount) {
        throw new \InvalidArgumentException(
            "Minimum transfer amount is {$minAmount}"
        );
    }

    if ($amount > $maxAmount) {
        throw new \InvalidArgumentException(
            "Maximum transfer amount is {$maxAmount}"
        );
    }

    // 6. 调用 Repository 执行转账
    return $this->repository->transferCredits(
        $fromUserId,
        $toUserId,
        $creditType,
        $amount
    );
}
```

### 3. 配置文件更新

**文件**: `config/credits.php`

```php
return [
    // ... 其他配置 ...

    // 积分类型转账规则
    'transfer_rules' => [
        'extcredits1' => [
            'allow_transfer_out' => true,
            'allow_transfer_in' => true,
            'min_transfer_amount' => 10,
            'max_transfer_amount' => 10000,
        ],
        'extcredits2' => [
            'allow_transfer_out' => false,
            'allow_transfer_in' => false,
        ],
        'extcredits3' => [
            'allow_transfer_out' => true,
            'allow_transfer_in' => true,
            'min_transfer_amount' => 1,
            'max_transfer_amount' => 5000,
        ],
        // ... extcredits4-8
    ],
];
```

---

## 📊 不同积分类型的转账规则

### 类型 1: 金币（extcredits1）
**特点**: 全功能积分，可转账
- ✅ allowTransferOut: true - 可以转出
- ✅ allowTransferIn: true - 可以转入
- ✅ 用途：购买道具、转账给其他用户、兑换

### 类型 2: �望（extcredits2）
**特点**: 荣誉积分，不可转账
- ❌ allowTransferOut: false - 不可转出
- ❌ allowTransferIn: false - 不可转入
- ✅ 用途：发帖奖励、回复奖励、系统奖励

### 类型 3: 金钱（extcredits3）
**特点**: 银行积分，可转账
- ✅ allowTransferOut: true - 可以转出
- ✅ allowTransferIn: true - 可以转入
- ✅ 用途：银行系统、购买高级道具、大额转账
- ⚠️ 特殊规则：可能需要手续费、利息等

### 类型 4-8: 其他扩展积分
**特点**: 根据实际需求配置
- 大多数：allowTransferOut=true, allowTransferIn=true
- 某些可能：allowTransferOut=false（仅充值）
- 某些可能：allowTransferIn=false（仅消耗）

---

## 🎯 实施建议

### 优先级：P0（高优先级）

#### Phase 1: 基础转账控制（必须实现）
1. ✅ **CreditRules 增强**
   - 添加 `getTransferRules()` 方法
   - 添加 `canTransferOut()` 方法
   - 添加 `canTransferIn()` 方法
   - 添加 `getMinTransferAmount()` 方法
   - 添加 `getMaxTransferAmount()` 方法

2. ✅ **CreditService 增强**
   - 在 `transferCredits()` 中调用转账规则验证
   - 抛出不符合规则的操作（异常）
   - 验证最小/最大转账金额

3. ✅ **配置文件更新**
   - 添加 `transfer_rules` 配置项
   - 为每种 extcredits 类型配置规则

#### Phase 2: 银行系统（可选，后续实现）
- 独行表设计
- 银行业务逻辑
- 独行管理后台
- 金币存取款功能
- 利息计算

---

## ✅ 测试用例

### 转账控制测试

```php
// 测试 1: 正常转账（金币）
$dto = new TransferCreditsDto(
    fromUserId: 1,
    toUserId: 2,
    creditType: 'extcredits1',  // 金币 - 可转账
    amount: 100
);
$creditService->transferCredits($dto);
// ✅ 应该成功

// 测试 2: 不可转账类型（威望）
$dto = new TransferCreditsDto(
    fromUserId: 1,
    toUserId: 2,
    creditType: 'extcredits2',  // 威望 - 不可转账
    amount: 100
);
// ❌ 应该抛出异常
```

### 最小/最大金额测试

```php
// 测试 3: 小于最小金额
$dto = new TransferCreditsDto(
    fromUserId: 1,
    toUserId: 2,
    creditType: 'extcredits3',  // 金钱 - 最小1枚
    amount: 0
);
// ❌ 应该抛出异常：最小转账金额是 1

// 测试 4: 大于最大金额
$dto = new TransferCreditsDto(
    fromUserId: 1,
    toUserId: 2,
    creditType: 'extcredits3',  // 金钱 - 最大5000枚
    amount: 10000
);
// ❌ 应该抛出异常：最大转账金额是 5000
```

---

## 🎯 总结

### 关键要点

1. **不是所有积分都可以转账**
   - extcredits1-8 每种类型有独立的转账控制
   - 通过 `allowexchangeout` 和 `allowexchangein` 配置

2. **银行系统是独立的**
   - 银行系统有自己的表和逻辑
   - 银行积分（moneycredits）与 extcredits1-8 分离
   - 兑换是单向的（extcredits → moneycredits）

3. **后台配置灵活性**
   - 管理员可以为每种积分类型设置不同规则
   - 可以随时启用/禁用转账功能
   - 可以设置最小/最大转账金额

4. **现代化实现**
   - 通过 CreditRules 管理转账配置
   - 通过 CreditService 验证转账权限
   - 保持 Legacy 兼容性

---

**生成时间**: 2026-02-14
**探索者**: Explore Agent a0e3ae7
**文档版本**: 1.0
**状态**: ✅ 完成
