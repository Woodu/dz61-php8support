# CSS强制重新编译机制（开发/测试模式）

**生成时间**: 2026-02-20  
**状态**: ✅ 已完成  
**测试状态**: ✅ 所有测试通过

---

## 一、功能概述

在开发/测试环境下，每次请求都自动重新编译CSS模板，避免缓存问题导致的误差。类似于Legacy系统的 `$tplrefresh = 1` 机制。

---

## 二、实现机制

### 2.1 自动检测环境

`StyleCompilerService` 自动检测运行环境：

- **开发/测试环境** (`development`, `local`, `testing`, `dev`, `test`):
  - ✅ **总是重新编译** CSS（`forceRecompile = true`）
  - 确保CSS模板修改后立即生效
  - 避免缓存导致的样式不一致

- **生产环境** (`production`, `staging`):
  - ✅ **缓存启用**（`forceRecompile = false`）
  - 只在模板文件更新时重新编译
  - 提高性能

### 2.2 手动覆盖

可以通过环境变量手动控制：

```bash
# .env 文件
CSS_FORCE_RECOMPILE=true   # 强制总是重新编译
CSS_FORCE_RECOMPILE=false  # 禁用强制重新编译（使用自动检测）
```

### 2.3 代码控制

也可以通过代码动态控制：

```php
$compiler = new StyleCompilerService(...);

// 启用强制重新编译
$compiler->setForceRecompile(true);

// 禁用强制重新编译
$compiler->setForceRecompile(false);

// 检查当前设置
$isForced = $compiler->isForceRecompile();
```

---

## 三、配置方式

### 3.1 环境变量（推荐）

在 `.env` 文件中设置：

```bash
# 开发环境 - 总是重新编译
APP_ENV=development
# CSS_FORCE_RECOMPILE 会自动设置为 true

# 生产环境 - 使用缓存
APP_ENV=production
# CSS_FORCE_RECOMPILE 会自动设置为 false

# 手动覆盖
CSS_FORCE_RECOMPILE=true  # 强制总是重新编译（忽略APP_ENV）
```

### 3.2 配置文件

在 `config/style.php` 中配置：

```php
return [
    'force_recompile' => env('CSS_FORCE_RECOMPILE', null), // null = auto-detect
    // ... other settings
];
```

### 3.3 代码设置

在 `ViewRenderer` 中：

```php
$forceRecompile = null; // null = auto-detect from APP_ENV
$styleCompiler = new StyleCompilerService(
    $legacyTemplateDir,
    $legacyCacheDir,
    $outputDir,
    1,
    $forceRecompile // auto-detect if null
);
```

---

## 四、工作原理

### 4.1 自动检测流程

```
StyleCompilerService 构造
    ↓
检查 $forceRecompile 参数
    ↓
如果为 null → 调用 shouldForceRecompile()
    ↓
检查 CSS_FORCE_RECOMPILE 环境变量
    ↓
如果未设置 → 检查 APP_ENV
    ↓
development/local/testing → true
production/staging → false
```

### 4.2 ensureCompiled() 行为

```php
public function ensureCompiled($styleId, $type, $force = false)
{
    // 如果强制重新编译模式启用，总是重新编译
    if ($this->forceRecompile) {
        return $this->compileStyle($styleId);
    }
    
    // 否则检查是否需要更新
    if ($force || !$this->compiledCssExists(...) || $this->isCssTemplateUpdated(...)) {
        return $this->compileStyle($styleId);
    }
    
    return true;
}
```

---

## 五、使用场景

### 5.1 开发环境

**场景**: 开发时频繁修改CSS模板文件

**配置**:
```bash
APP_ENV=development
```

**行为**:
- ✅ 每次访问页面都重新编译CSS
- ✅ CSS模板修改后立即生效
- ✅ 无需手动删除缓存文件

### 5.2 测试环境

**场景**: 运行测试时确保CSS是最新的

**配置**:
```bash
APP_ENV=testing
```

**行为**:
- ✅ 每次测试都重新编译CSS
- ✅ 确保测试结果的一致性
- ✅ 避免缓存导致的测试失败

### 5.3 生产环境

**场景**: 生产环境需要性能优化

**配置**:
```bash
APP_ENV=production
```

**行为**:
- ✅ CSS文件缓存启用
- ✅ 只在模板更新时重新编译
- ✅ 提高页面加载性能

---

## 六、性能影响

### 6.1 开发/测试环境

- **影响**: 每次请求都编译CSS（约50-200ms）
- **可接受性**: ✅ 可接受（开发时性能不是主要考虑）

### 6.2 生产环境

- **影响**: 只在模板更新时编译（几乎无影响）
- **可接受性**: ✅ 最优性能

---

## 七、验证测试

### 7.1 测试开发模式

```bash
# 设置开发环境
export APP_ENV=development

# 访问页面（应该每次都重新编译）
curl http://localhost:8000/

# 检查编译时间戳
ls -la public/css/compiled/style_1_common.css

# 再次访问（应该再次重新编译）
sleep 1
curl http://localhost:8000/

# 检查时间戳是否更新
ls -la public/css/compiled/style_1_common.css
```

### 7.2 测试生产模式

```bash
# 设置生产环境
export APP_ENV=production

# 访问页面（应该使用缓存）
curl http://localhost:8000/

# 检查编译时间戳
ls -la public/css/compiled/style_1_common.css

# 再次访问（应该不重新编译）
sleep 1
curl http://localhost:8000/

# 检查时间戳是否相同
ls -la public/css/compiled/style_1_common.css
```

### 7.3 测试手动覆盖

```bash
# 在生产环境下强制重新编译
export APP_ENV=production
export CSS_FORCE_RECOMPILE=true

# 访问页面（应该每次都重新编译）
curl http://localhost:8000/
```

---

## 八、实现文件

### 8.1 核心服务

- ✅ `app/Style/Services/StyleCompilerService.php` - 已增强
  - `shouldForceRecompile()` - 自动检测方法
  - `setForceRecompile()` - 手动设置方法
  - `isForceRecompile()` - 检查当前设置

### 8.2 配置文件

- ✅ `config/style.php` - 新建配置文件
  - `force_recompile` - 强制重新编译设置

### 8.3 集成

- ✅ `app/View/ViewRenderer.php` - 已更新
  - 从配置文件读取设置
  - 自动传递给 `StyleCompilerService`

---

## 九、与Legacy系统对比

| 功能 | Legacy系统 | Modern系统 | 状态 |
|------|-----------|-----------|------|
| **开发模式强制重新编译** | `$tplrefresh = 1` | `APP_ENV=development` | ✅ |
| **自动检测环境** | 手动设置 | 自动检测 `APP_ENV` | ✅ |
| **手动覆盖** | 修改 `config.inc.php` | 设置 `CSS_FORCE_RECOMPILE` | ✅ |
| **运行时控制** | 不支持 | `setForceRecompile()` | ✅ |

---

## 十、总结

Modern系统已实现开发/测试环境下的自动强制重新编译机制：

1. ✅ **自动检测环境**: 根据 `APP_ENV` 自动启用/禁用
2. ✅ **手动覆盖**: 通过 `CSS_FORCE_RECOMPILE` 环境变量手动控制
3. ✅ **代码控制**: 通过 `setForceRecompile()` 方法动态控制
4. ✅ **性能优化**: 生产环境自动禁用，开发环境自动启用

**优势**:
- 开发时无需手动清除缓存
- 测试时确保CSS一致性
- 生产环境自动优化性能
- 与Legacy系统行为一致

---

## 十一、测试验证

### 11.1 测试脚本

已创建测试脚本 `scripts/test-force-recompile.php`，验证以下场景：

1. ✅ **开发环境自动启用** (`APP_ENV=development` → `forceRecompile=true`)
2. ✅ **生产环境自动禁用** (`APP_ENV=production` → `forceRecompile=false`)
3. ✅ **手动覆盖生效** (`CSS_FORCE_RECOMPILE=true` → `forceRecompile=true`)
4. ✅ **测试环境自动启用** (`APP_ENV=testing` → `forceRecompile=true`)
5. ✅ **运行时控制** (`setForceRecompile()` 方法正常工作)

### 11.2 测试结果

```
=== CSS Force Recompile Test ===

Test 1: Development environment (APP_ENV=development)
  ✅ PASS

Test 2: Production environment (APP_ENV=production)
  ✅ PASS

Test 3: Manual override (CSS_FORCE_RECOMPILE=true in production)
  ✅ PASS

Test 4: Testing environment (APP_ENV=testing)
  ✅ PASS

Test 5: Runtime control (setForceRecompile)
  ✅ PASS

=== All Tests Complete ===
```

---

**文档状态**: ✅ 完成  
**实现状态**: ✅ 已完成  
**测试状态**: ✅ 所有测试通过
