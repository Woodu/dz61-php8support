# Week 14-16 详细行动计划
## Discuz! 6.1F 到 PHP 8.3 迁移项目

**制定日期**: 2026-02-19
**计划周期**: Week 14-16（3周）
**当前项目完成度**: 72%
**目标完成度**: 95%+（达到可投产状态）

---

## 📋 执行概览

| 周 | 主要目标 | 预期完成度 | 关键交付物 |
|----|----------|------------|-----------|
| **Week 14** | 质量保证与验证 | 72% → 80% | 测试修复、性能基准、文档更新 |
| **Week 15** | 交互表单实现 | 80% → 90% | 发帖UI、回复UI、BBCode编辑器 |
| **Week 16** | 管理员后台 | 90% → 95% | AdminCP、系统监控、部署准备 |

---

## 🎯 Week 14: 质量保证与验证（2026-02-20 ~ 2026-02-26）

### 目标
修复所有阻塞性问题，建立性能基准，更新文档以反映真实进度。

### 完成标准
- ✅ 所有测试套件通过率 ≥ 95%
- ✅ 性能基准报告生成
- ✅ 文档准确率 100%
- ✅ 零改表原则验证通过

---

### Day 1 (2026-02-20): 测试套件修复

#### 任务 14.1: 修复集成测试依赖注入 (P0)
**负责人**: 后端开发团队
**预计时间**: 2 小时
**优先级**: 🔴 P0 - 阻塞

**问题详情**:
```
文件: tests/Integration/Thread/ContentValidatorIntegrationTest.php:42
错误: TypeError in ForumPermissionService constructor
原因: 注入了错误的依赖类型（Cache 而非 FormulaPermissionService）
```

**修复步骤**:
1. 检查 `ContentValidatorIntegrationTest.php` 测试设置
2. 修正依赖注入：
   ```php
   // 错误写法：
   $permission = new ForumPermissionService($cache, $cache);

   // 正确写法：
   $formulaService = new FormulaPermissionService($cache);
   $permission = new ForumPermissionService($cache, $formulaService);
   ```
3. 验证所有集成测试的依赖注入
4. 运行集成测试套件

**验收标准**:
- [ ] 所有集成测试通过（0/7 → 7/7）
- [ ] 无依赖注入错误
- [ ] 测试执行时间 < 30 秒

**相关文件**:
- `tests/Integration/Thread/ContentValidatorIntegrationTest.php`
- `app/Security/Services/ForumPermissionService.php`
- `app/Security/Services/FormulaPermissionService.php`

---

#### 任务 14.2: 修复 E2E 测试投票表引用 (P0)
**负责人**: 测试团队
**预计时间**: 1 小时
**优先级**: 🔴 P0 - 阻塞

**问题详情**:
```
文件: tests/E2E/Scenarios/PollFlowTest.php:433
错误: Table 'discuz_utf8.cdb_pollvoters' doesn't exist
原因: Legacy 使用 cdb_polloptions.votes 字段追踪投票，无独立投票人表
```

**修复步骤**:
1. 检查 Legacy 投票系统结构：
   ```sql
   -- Legacy 结构
   cdb_polls: tid, multiple, visible, maxchoices, expiration
   cdb_polloptions: polloptionid, tid, displayorder, votes, option
   -- 注意: votes 字段直接存储该选项的票数
   ```
2. 更新测试以使用正确的表结构
3. 修改断言以匹配 Legacy 行为
4. 运行 E2E 测试套件

**验收标准**:
- [ ] E2E 测试中无表不存在的错误
- [ ] 投票测试通过（0/2 → 2/2）
- [ ] 断言与 Legacy 行为一致

**相关文件**:
- `tests/E2E/Scenarios/PollFlowTest.php`
- `app/Thread/Repositories/PollRepository.php`
- `poketb.com/bbs/viewthread.php` (Legacy 参考)

---

#### 任务 14.3: 创建 E2ETestCase 基类 (P0)
**负责人**: 测试团队
**预计时间**: 1 小时
**优先级**: 🔴 P0 - 阻塞

**问题详情**:
```
错误: Class "Discuz\Tests\Feature\E2ETestCase" not found
影响: 18 个特性测试无法运行
```

**实现步骤**:
1. 创建基类文件：
   ```php
   // 文件: tests/Feature/E2ETestCase.php
   <?php
   declare(strict_types=1);

   namespace Discuz\Tests\Feature;

   use PHPUnit\Framework\TestCase;
   use Discuz\Database\Database;

   abstract class E2ETestCase extends TestCase
   {
       protected function setUp(): void
       {
           parent::setUp();
           Database::setTestMode(true);
           // 设置测试数据库
           // 设置测试会话
       }

       protected function tearDown(): void
       {
           Database::setTestMode(false);
           parent::tearDown();
       }

       protected function createTestClient(): array
       {
           return [
               'base_uri' => 'http://localhost',
               'headers' => [
                   'Content-Type' => 'application/json'
               ]
           ];
       }
   }
   ```

2. 更新所有特性测试以继承此基类
3. 添加测试辅助方法（创建测试用户、登录等）

**验收标准**:
- [ ] E2ETestCase.php 创建完成
- [ ] 所有特性测试可以运行
- [ ] 测试数据库隔离正常工作

**相关文件**:
- `tests/Feature/E2ETestCase.php` (新建)
- `tests/Feature/AuthenticationFeatureTest.php`
- `tests/Feature/ForumFeatureTest.php`

---

### Day 2 (2026-02-21): 测试执行与覆盖率报告

#### 任务 14.4: 执行完整测试套件 (P0)
**负责人**: 测试团队
**预计时间**: 3 小时
**优先级**: 🔴 P0 - 关键

**执行步骤**:
```bash
# 1. 单元测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/unit/ \
  --testdox \
  > test-results-unit-2026-02-21.txt

# 2. 集成测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/Integration/ \
  --testdox \
  > test-results-integration-2026-02-21.txt

# 3. 特性测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/Feature/ \
  --testdox \
  > test-results-feature-2026-02-21.txt

# 4. E2E 测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/E2E/ \
  --testdox \
  > test-results-e2e-2026-02-21.txt

# 5. 生成覆盖率报告
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  --coverage-html storage/coverage-report/2026-02-21/ \
  --coverage-text \
  > test-coverage-2026-02-21.txt
```

**验收标准**:
- [ ] 所有测试套件执行完成
- [ ] 单元测试通过率 ≥ 99%
- [ ] 集成测试通过率 ≥ 95%
- [ ] E2E 测试通过率 ≥ 80%（UI 未完成的除外）
- [ ] 覆盖率报告生成成功
- [ ] 整体代码覆盖率 ≥ 85%

**交付物**:
- `storage/logs/test-results-unit-2026-02-21.txt`
- `storage/logs/test-results-integration-2026-02-21.txt`
- `storage/logs/test-results-feature-2026-02-21.txt`
- `storage/logs/test-results-e2e-2026-02-21.txt`
- `storage/coverage-report/2026-02-21/index.html`

---

#### 任务 14.5: 分析测试失败并修复 (P1)
**负责人**: 后端开发团队
**预计时间**: 4 小时
**优先级**: 🟡 P1 - 重要

**执行步骤**:
1. 分析测试失败原因：
   - 断言错误 vs 实现错误
   - 测试数据问题 vs 代码逻辑问题
   - 遗留兼容性问题

2. 分类修复：
   - **快速修复** (< 30分钟): 断言修正、测试数据调整
   - **中速修复** (1-2小时): 逻辑bug、兼容性问题
   - **慢速修复** (> 2小时): 架构问题（记录，延后处理）

3. 修复并重新运行测试

**验收标准**:
- [ ] 所有 P0 测试失败修复完成
- [ ] 测试通过率达到目标（≥ 95%）
- [ ] 失败测试有清晰的 JIRA/问题跟踪
- [ ] 修复文档化（代码注释、commit message）

---

### Day 3 (2026-02-22): 性能测试与基准建立

#### 任务 14.6: 执行性能测试 (P0)
**负责人**: 性能测试团队
**预计时间**: 4 小时
**优先级**: 🔴 P0 - 关键

**测试脚本清单**:
```bash
# 1. 论坛首页性能测试
docker exec -i discuz_modern_php php \
  tests/Performance/ForumHomepagePerformanceTest.php

# 2. 主题列表性能测试
docker exec -i discuz_modern_php php \
  tests/Performance/ThreadListPerformanceTest.php

# 3. 主题详情性能测试
docker exec -i discuz_modern_php php \
  tests/Performance/ThreadDetailPerformanceTest.php

# 4. BBCode 解析性能测试
docker exec -i discuz_modern_php php \
  tests/Performance/BBCodeParserPerformanceTest.php

# 5. 搜索性能测试（如果已实现）
docker exec -i discuz_modern_php php \
  tests/Performance/SearchPerformanceTest.php
```

**性能指标**:
| 指标 | 目标值 (P95) | 测量方法 |
|------|-------------|---------|
| 论坛首页加载时间 | < 300ms | Symfony StopWatch |
| 主题列表加载 | < 400ms | 数据库查询时间 |
| 主题详情加载 | < 500ms | 完整请求周期 |
| BBCode 解析 | < 50ms | 1000次迭代平均 |
| 并发用户支持 | 100+ | Apache Bench |

**验收标准**:
- [ ] 所有性能脚本执行完成
- [ ] 性能基准数据记录
- [ ] 性能报告生成
- [ ] 性能瓶颈识别（如有）

**交付物**:
- `storage/logs/performance-baseline-2026-02-22.md`
- `storage/logs/performance-raw-data-2026-02-22.json`

**性能报告模板**:
```markdown
# 性能基准报告
**日期**: 2026-02-22
**环境**: Docker PHP 8.3 + MySQL 8.0
**测试工具**: Symfony StopWatch + Apache Bench

## 测试结果

### 1. 论坛首页性能
- P50: ___ ms
- P95: ___ ms
- P99: ___ ms
- 数据库查询: ___ 次
- 内存使用: ___ MB

### 2. 主题列表性能
- P50: ___ ms
- P95: ___ ms
- 数据库查询: ___ 次

### 3. 主题详情性能
- P50: ___ ms
- P95: ___ ms
- BBCode 解析: ___ ms
- 数据库查询: ___ 次

### 4. 并发测试
- 10 并发: ___ req/s
- 50 并发: ___ req/s
- 100 并发: ___ req/s

## 性能瓶颈
[列出发现的问题]

## 优化建议
[列出优化方向]
```

---

### Day 4 (2026-02-23): 文档更新

#### 任务 14.7: 更新项目进度文档 (P0)
**负责人**: 技术文档团队
**预计时间**: 3 小时
**优先级**: 🔴 P0 - 关键

**需要更新的文件**:
1. `modern-php-migration-code/PROGRESS-REPORT.md`
2. `modern-php-migration-code/TASK-TRACKER.md`
3. `modern-php-migration-code/EXECUTION-PLAN-COMPLETE.md`
4. `modern-php-execution-plan/reports/weeks/WEEK13-COMPLETE.md`

**更新内容**:

**PROGRESS-REPORT.md**:
```markdown
## 项目进度（2026-02-23 更新）

### 当前状态
- **总体完成度**: 72%（修正，此前文档声称 100%）
- **P0 关键路径**: 100% ✅
- **P1 核心功能**: 80%
- **P2 增强功能**: 20%

### 测试状态（2026-02-21 验证）
- 单元测试: ___ / ___ (___% 通过)
- 集成测试: ___ / ___ (___% 通过)
- E2E 测试: ___ / ___ (___% 通过)
- 代码覆盖率: ___%

### 性能基准（2026-02-22 建立）
- 论坛首页: ___ ms (P95)
- 主题列表: ___ ms (P95)
- 主题详情: ___ ms (P95)
```

**TASK-TRACKER.md**:
```markdown
## Week 13 实际状态
- 计划任务: 6 项
- 实际完成: 3 项 (50%)
- 未完成: E2E 测试修复、性能测试、文档指南

## Week 14 实际状态
- 计划任务: 7 项
- 实际完成: ___ / 7
```

**EXECUTION-PLAN-COMPLETE.md**:
```markdown
## 进度修正声明（2026-02-23）

**此前文档声称的 100% 完成度不准确**

经过全面审查和测试验证，实际完成度为 72%。
主要差距：
- 交互表单前端 (0%)
- E2E 测试覆盖 (19%)
- 性能验证 (未执行)

**本文档已更新以反映真实状态**
```

**验收标准**:
- [ ] 所有进度文档更新为真实数据
- [ ] 添加"验证日期"戳
- [ ] 移除所有虚高声明
- [ ] 文档间数据一致

---

### Day 5 (2026-02-24): Week 6 补全与 Week 9 前端修复

#### 任务 14.8: 完成未完成的控制器 (P1)
**负责人**: 后端开发团队
**预计时间**: 6 小时
**优先级**: 🟡 P1 - 重要

**需要完成的控制器**:

**1. PaymentController 补全**
```php
// 文件: app/Http/Controllers/PaymentController.php
// 状态: 85% 完成
// 缺少:
- [ ] 支付历史查询接口
- [ ] 支付退款处理
- [ ] 支付回调验证增强
```

**2. PollController 补全**
```php
// 文件: app/Http/Controllers/PollController.php
// 状态: 80% 完成
// 缺少:
- [ ] 投票结果导出
- [ ] 投票截止提醒
- [ ] 多选投票验证增强
```

**3. PostController 补全**
```php
// 文件: app/Http/Controllers/PostController.php
// 状态: 90% 完成（后端）
// 缺少:
- [ ] 草稿保存功能
- [ ] 自动保存功能
- [ ] 编辑历史记录
```

**验收标准**:
- [ ] 所有控制器方法实现完成
- [ ] 单元测试覆盖率 ≥ 90%
- [ ] 集成测试通过
- [ ] API 文档更新

**相关文件**:
- `app/Http/Controllers/PaymentController.php`
- `app/Http/Controllers/PollController.php`
- `app/Http/Controllers/PostController.php`
- `tests/Unit/Http/Controllers/*Test.php`

---

#### 任务 14.9: 修复前端模板问题 (P1)
**负责人**: 前端开发团队
**预计时间**: 4 小时
**优先级**: 🟡 P1 - 重要

**需要修复的模板问题**:

**1. BBCode 渲染问题**
```twig
{# 模板: templates/thread/view.html.twig #}
{# 问题: BBCode 渲染不完整 #}
{# 修复: 确保所有 BBCode 标签正确渲染 #}
{{ post.message|raw }}  {# 确保自动转义已正确处理 #}
```

**2. 附件显示问题**
```twig
{# 模板: templates/thread/view.html.twig #}
{# 问题: 附件缩略图不显示 #}
{# 修复: 检查附件 URL 路径 #}
{% for attachment in post.attachments %}
  <img src="{{ attachment.url|e }}" alt="{{ attachment.filename|e }}">
{% endfor %}
```

**3. 分页导航问题**
```twig
{# 模板: templates/forum/threadlist.html.twig #}
{# 问题: 分页链接不工作 #}
{# 修复: 正确生成查询参数 #}
{% for page in pagination.pages %}
  <a href="?page={{ page }}">{{ page }}</a>
{% endfor %}
```

**验收标准**:
- [ ] 所有模板渲染正常
- [ ] 无 XSS 漏洞（转义正确）
- [ ] 响应式设计正常工作
- [ ] 浏览器兼容性测试通过（Chrome, Firefox, Safari）

---

### Day 6-7 (2026-02-25 ~ 2026-02-26): Week 14 收尾与总结

#### 任务 14.10: Week 14 验收与总结 (P0)
**负责人**: 项目经理
**预计时间**: 2 小时
**优先级**: 🔴 P0 - 关键

**验收清单**:
```markdown
## Week 14 完成清单

### 测试修复 ✅
- [ ] 集成测试通过率 ≥ 95%
- [ ] E2E 测试通过率 ≥ 80%
- [ ] 覆盖率报告生成
- [ ] 所有 P0 测试失败修复

### 性能测试 ✅
- [ ] 5 个性能脚本执行完成
- [ ] 性能基准报告生成
- [ ] 性能瓶颈识别
- [ ] 优化建议文档化

### 文档更新 ✅
- [ ] PROGRESS-REPORT.md 更新
- [ ] TASK-TRACKER.md 更新
- [ ] EXECUTION-PLAN-COMPLETE.md 更新
- [ ] WEEK13-COMPLETE.md 修正

### 控制器补全 ✅
- [ ] PaymentController 完成
- [ ] PollController 完成
- [ ] PostController 完成

### 前端修复 ✅
- [ ] BBCode 渲染正常
- [ ] 附件显示正常
- [ ] 分页导航正常

### 零改表验证 ✅
- [ ] 数据库表清单审查
- [ ] 无违规新表
- [ ] 视图合法性验证
```

**Week 14 总结报告**:
```markdown
# Week 14 总结报告
**日期**: 2026-02-26
**负责人**: [项目经理]

## 完成情况
- 计划任务: 10 项
- 实际完成: ___ / 10
- 完成率: ___%

## 主要成果
1. 测试套件修复：___ 测试通过
2. 性能基准建立：首页加载 ___ ms
3. 文档准确性：100%

## 遗留问题
1. [列出未解决的问题]

## Week 15 准备
- 交互表单实现准备就绪
- 前端开发环境配置完成
- 设计规范文档齐全

## 风险与建议
[列出风险和改进建议]
```

**交付物**:
- `modern-php-execution-plan/reports/weeks/WEEK14-COMPLETE.md`
- Week 14 总结报告

---

## 🎨 Week 15: 交互表单实现（2026-02-27 ~ 2026-03-05）

### 目标
实现所有交互式表单的前端 UI，让用户可以通过 Web 界面创建内容。

### 完成标准
- ✅ 新主题表单功能完整
- ✅ 回复表单功能完整
- ✅ BBCode 编辑器功能完整
- ✅ 附件上传 UI 功能完整
- ✅ 投票创建 UI 功能完整
- ✅ 支付 UI 功能完整

---

### Day 1-2 (2026-02-27 ~ 2026-02-28): 新主题表单

#### 任务 15.1: 新主题表单 UI (P0)
**负责人**: 前端开发团队
**预计时间**: 12 小时（2 天）
**优先级**: 🔴 P0 - 关键

**Legacy 参考**:
```
poketb.com/bbs/post.php
poketb.com/bbs/templates/default/post_newthread.htm
```

**实现步骤**:

**1. 创建模板文件**
```twig
{# 文件: app/View/templates/thread/new.html.twig #}
{% extends 'base/layout.html.twig' %}

{% block title %}{{ forum.name }} - 发表新主题{% endblock %}

{% block content %}
<div class="new-thread-form">
  <h1>在 {{ forum.name }} 发表新主题</h1>

  <form method="post" action="{{ path('thread.create') }}" enctype="multipart/form-data">
    {{ csrf_field() }}

    <!-- 主题标题 -->
    <div class="form-group">
      <label for="subject">标题</label>
      <input type="text" id="subject" name="subject" maxlength="80" required>
      <small class="text-muted">最多 80 个字符</small>
    </div>

    <!-- 主题图标 -->
    <div class="form-group">
      <label>主题图标</label>
      <div class="thread-icons">
        {% for icon in icons %}
        <label class="icon-option">
          <input type="radio" name="iconid" value="{{ icon.id }}">
          <img src="{{ icon.url }}" alt="{{ icon.name }}">
        </label>
        {% endfor %}
      </div>
    </div>

    <!-- BBCode 编辑器 -->
    <div class="form-group">
      <label for="message">内容</label>
      <textarea id="message" name="message" rows="15" required></textarea>

      <!-- BBCode 工具栏 -->
      <div class="bbcode-toolbar">
        <button type="button" data-tag="b">粗体</button>
        <button type="button" data-tag="i">斜体</button>
        <button type="button" data-tag="u">下划线</button>
        <button type="button" data-tag="url">链接</button>
        <button type="button" data-tag="img">图片</button>
        <button type="button" data-tag="quote">引用</button>
        <button type="button" data-tag="code">代码</button>
      </div>
    </div>

    <!-- 附件上传 -->
    <div class="form-group">
      <label for="attachments">附件</label>
      <input type="file" id="attachments" name="attachments[]" multiple>
      <div class="upload-progress" style="display: none;">
        <progress value="0" max="100"></progress>
        <span class="progress-text">0%</span>
      </div>
    </div>

    <!-- 投票选项（可选） -->
    <div class="form-group">
      <label>
        <input type="checkbox" name="add_poll" value="1">
        添加投票
      </label>

      <div id="poll-options" style="display: none;">
        <div class="poll-settings">
          <label>
            <input type="checkbox" name="poll_multiple" value="1">
            允许多选
          </label>
          <label>
            <input type="checkbox" name="poll_visible" value="1" checked>
            投票前可见
          </label>
        </div>

        <div id="poll-choices">
          <input type="text" name="poll_options[]" placeholder="选项 1" required>
          <input type="text" name="poll_options[]" placeholder="选项 2" required>
          <input type="text" name="poll_options[]" placeholder="选项 3">
        </div>

        <button type="button" id="add-poll-option">添加选项</button>
      </div>
    </div>

    <!-- 支付设置（可选） -->
    <div class="form-group">
      <label>
        <input type="checkbox" name="add_payment" value="1">
        设置为付费主题
      </label>

      <div id="payment-settings" style="display: none;">
        <label>价格（积分）</label>
        <input type="number" name="price" min="0" value="0">
      </div>
    </div>

    <!-- 提交按钮 -->
    <div class="form-actions">
      <button type="submit" class="btn btn-primary">发表主题</button>
      <button type="button" class="btn btn-secondary" id="preview-btn">预览</button>
      <a href="{{ path('forum.view', {fid: forum.fid}) }}" class="btn btn-cancel">取消</a>
    </div>
  </form>
</div>
{% endblock %}
```

**2. 创建 JavaScript 功能**
```javascript
// 文件: public/js/new-thread.js
document.addEventListener('DOMContentLoaded', function() {
  // BBCode 工具栏
  const toolbar = document.querySelector('.bbcode-toolbar');
  const message = document.getElementById('message');

  toolbar.addEventListener('click', function(e) {
    if (e.target.type === 'button') {
      const tag = e.target.dataset.tag;
      const selection = message.value.substring(message.selectionStart, message.selectionEnd);

      if (selection) {
        insertBBCode(tag, selection);
      } else {
        const placeholder = prompt(`请输入 ${tag} 内容:`);
        if (placeholder) {
          insertBBCode(tag, placeholder);
        }
      }
    }
  });

  function insertBBCode(tag, content) {
    const bbcode = `[${tag}]${content}[/${tag}]`;
    message.setRangeText(bbcode, message.selectionStart, message.selectionEnd, 'select');
    message.focus();
  }

  // 附件上传进度
  const fileInput = document.getElementById('attachments');
  const progress = document.querySelector('.upload-progress');

  fileInput.addEventListener('change', function() {
    progress.style.display = 'block';

    const formData = new FormData();
    for (const file of fileInput.files) {
      formData.append('attachments[]', file);
    }

    fetch('/api/upload', {
      method: 'POST',
      body: formData,
      headers: {
        'X-Requested-With': 'XMLHttpRequest'
      }
    })
    .then(response => response.json())
    .then(data => {
      if (data.success) {
        progress.querySelector('progress').value = 100;
        progress.querySelector('.progress-text').textContent = '100%';
      }
    });
  });

  // 投票选项
  const addPollCheckbox = document.querySelector('input[name="add_poll"]');
  const pollOptions = document.getElementById('poll-options');
  const addOptionBtn = document.getElementById('add-poll-option');
  const pollChoices = document.getElementById('poll-choices');

  addPollCheckbox.addEventListener('change', function() {
    pollOptions.style.display = this.checked ? 'block' : 'none';
  });

  addOptionBtn.addEventListener('click', function() {
    const optionCount = pollChoices.querySelectorAll('input').length;
    const input = document.createElement('input');
    input.type = 'text';
    input.name = 'poll_options[]';
    input.placeholder = `选项 ${optionCount + 1}`;
    pollChoices.appendChild(input);
  });

  // 预览功能
  document.getElementById('preview-btn').addEventListener('click', function() {
    const formData = new FormData(document.querySelector('form'));

    fetch('/api/preview', {
      method: 'POST',
      body: formData
    })
    .then(response => response.json())
    .then(data => {
      // 显示预览窗口
      const previewWindow = window.open('', 'preview');
      previewWindow.document.write(data.html);
    });
  });
});
```

**3. 更新控制器**
```php
// 文件: app/Http/Controllers/ThreadController.php
// 方法: newThreadFormAction()

public function newThreadFormAction(Request $request, int $fid): Response
{
    // 验证用户权限
    $user = $this->authService->getCurrentUser();
    $forum = $this->forumRepository->findById($fid);

    if (!$this->permissionService->canPostThread($user, $forum)) {
        throw new ForbiddenException('您没有权限在此版块发表主题');
    }

    // 获取可用的主题图标
    $icons = $this->getThreadIcons();

    // 渲染表单
    return $this->viewRenderer->render('thread/new.html.twig', [
        'forum' => $forum,
        'icons' => $icons,
        'user' => $user
    ]);
}

private function getThreadIcons(): array
{
    // Legacy 使用文件系统存储图标
    $icons = [];
    for ($i = 1; $i <= 9; $i++) {
        $icons[] = [
            'id' => $i,
            'name' => "图标 {$i}",
            'url' => "/images/icons/icon{$i}.gif"
        ];
    }
    return $icons;
}
```

**验收标准**:
- [ ] 表单 UI 完整实现
- [ ] BBCode 工具栏功能正常
- [ ] 附件上传功能正常（带进度显示）
- [ ] 投票选项动态添加正常
- [ ] 支付设置显示正常
- [ ] 预览功能正常
- [ ] 移动端响应式设计正常
- [ ] 浏览器兼容性测试通过

**相关文件**:
- `app/View/templates/thread/new.html.twig` (新建)
- `public/js/new-thread.js` (新建)
- `app/Http/Controllers/ThreadController.php`
- `public/css/new-thread.css` (新建)

---

### Day 3-4 (2026-03-01 ~ 2026-03-02): 回复表单

#### 任务 15.2: 回复表单 UI (P0)
**负责人**: 前端开发团队
**预计时间**: 12 小时（2 天）
**优先级**: 🔴 P0 - 关键

**Legacy 参考**:
```
poketb.com/bbs/post.php?action=reply
poketb.com/bbs/templates/default/post_reply.htm
```

**实现步骤**:

**1. 创建模板文件**
```twig
{# 文件: app/View/templates/thread/reply.html.twig #}
{% extends 'base/layout.html.twig' %}

{% block title %}{{ thread.subject }} - 回复主题{% endblock %}

{% block content %}
<div class="reply-form">
  <h1>回复: {{ thread.subject }}</h1>

  <!-- 显示主题内容（参考） -->
  <div class="thread-preview">
    <h3>主题内容</h3>
    <div class="post-content">
      {{ thread.firstPost.message|raw }}
    </div>
  </div>

  <form method="post" action="{{ path('thread.reply', {tid: thread.tid}) }}" enctype="multipart/form-data">
    {{ csrf_field() }}

    <!-- BBCode 编辑器 -->
    <div class="form-group">
      <label for="message">回复内容</label>
      <textarea id="message" name="message" rows="15" required></textarea>

      <!-- BBCode 工具栏（同新主题表单） -->
      <div class="bbcode-toolbar">
        <button type="button" data-tag="b">粗体</button>
        <button type="button" data-tag="i">斜体</button>
        <button type="button" data-tag="u">下划线</button>
        <button type="button" data-tag="url">链接</button>
        <button type="button" data-tag="img">图片</button>
        <button type="button" data-tag="quote">引用</button>
        <button type="button" data-tag="code">代码</button>
      </div>
    </div>

    <!-- 附件上传 -->
    <div class="form-group">
      <label for="attachments">附件</label>
      <input type="file" id="attachments" name="attachments[]" multiple>
      <div class="upload-info">
        <small>允许的文件类型: jpg, png, gif, pdf, doc, docx, zip</small>
        <small>最大文件大小: {{ upload_max_filesize() }}</small>
      </div>
    </div>

    <!-- 回复选项 -->
    <div class="form-group">
      <label>
        <input type="checkbox" name="email_notify" value="1">
        有回复时邮件通知我
      </label>
    </div>

    <!-- 提交按钮 -->
    <div class="form-actions">
      <button type="submit" class="btn btn-primary">发表回复</button>
      <button type="button" class="btn btn-secondary" id="preview-btn">预览</button>
      <a href="{{ path('thread.view', {tid: thread.tid}) }}" class="btn btn-cancel">取消</a>
    </div>
  </form>
</div>
{% endblock %}
```

**2. 引用回复功能**
```javascript
// 文件: public/js/reply-thread.js

// 点击"引用"按钮时触发
function quotePost(postId) {
  // 获取被引用的帖子内容
  fetch(`/api/post/${postId}`)
    .then(response => response.json())
    .then(data => {
      const author = data.author;
      const message = data.message;
      const timestamp = data.dateline;

      // 生成 BBCode 引用格式
      const quote = `[quote][b]${author}[/b] 发表于 ${timestamp}\n${message}[/quote]\n`;

      // 插入到编辑器
      const messageBox = document.getElementById('message');
      messageBox.value = quote + messageBox.value;
      messageBox.focus();
    });
}

// 在主题视图中添加"引用"按钮
document.querySelectorAll('.btn-quote').forEach(button => {
  button.addEventListener('click', function() {
    const postId = this.dataset.postId;
    quotePost(postId);
  });
});
```

**验收标准**:
- [ ] 回复表单 UI 完整实现
- [ ] 引用功能正常
- [ ] BBCode 工具栏功能正常
- [ ] 附件上传功能正常
- [ ] 邮件通知选项正常
- [ ] 预览功能正常
- [ ] 移动端响应式设计正常

**相关文件**:
- `app/View/templates/thread/reply.html.twig` (新建)
- `public/js/reply-thread.js` (新建)
- `app/Http/Controllers/ThreadController.php`

---

### Day 5-6 (2026-03-03 ~ 2026-03-04): BBCode 编辑器增强

#### 任务 15.3: BBCode 编辑器增强 (P1)
**负责人**: 前端开发团队
**预计时间**: 12 小时（2 天）
**优先级**: 🟡 P1 - 重要

**功能增强**:

**1. 实时预览**
```javascript
// 文件: public/js/bbcode-editor.js

class BBCodeEditor {
  constructor(textarea, preview) {
    this.textarea = textarea;
    this.preview = preview;

    // 自动保存
    this.initAutoSave();

    // 实时预览
    this.initLivePreview();

    // 快捷键支持
    this.initShortcuts();
  }

  initAutoSave() {
    let timeout;
    this.textarea.addEventListener('input', () => {
      clearTimeout(timeout);
      timeout = setTimeout(() => {
        localStorage.setItem('draft-' + this.getThreadId(), this.textarea.value);
      }, 1000);
    });

    // 恢复草稿
    const draft = localStorage.getItem('draft-' + this.getThreadId());
    if (draft && confirm('发现未保存的草稿，是否恢复？')) {
      this.textarea.value = draft;
    }
  }

  initLivePreview() {
    this.textarea.addEventListener('input', () => {
      this.updatePreview();
    });
  }

  async updatePreview() {
    const bbcode = this.textarea.value;

    const response = await fetch('/api/bbcode/preview', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ bbcode })
    });

    const data = await response.json();
    this.preview.innerHTML = data.html;
  }

  initShortcuts() {
    this.textarea.addEventListener('keydown', (e) => {
      // Ctrl+B: 粗体
      if (e.ctrlKey && e.key === 'b') {
        e.preventDefault();
        this.wrapSelection('b');
      }

      // Ctrl+I: 斜体
      if (e.ctrlKey && e.key === 'i') {
        e.preventDefault();
        this.wrapSelection('i');
      }

      // Ctrl+Shift+S: 删除线
      if (e.ctrlKey && e.shiftKey && e.key === 'S') {
        e.preventDefault();
        this.wrapSelection('s');
      }

      // Tab: 缩进
      if (e.key === 'Tab') {
        e.preventDefault();
        this.insertText('    ');
      }
    });
  }

  wrapSelection(tag) {
    const start = this.textarea.selectionStart;
    const end = this.textarea.selectionEnd;
    const text = this.textarea.value;
    const selection = text.substring(start, end);

    this.textarea.value =
      text.substring(0, start) +
      `[${tag}]${selection}[/${tag}]` +
      text.substring(end);

    this.textarea.selectionStart = start + tag.length + 2;
    this.textarea.selectionEnd = end + tag.length + 2;
    this.textarea.focus();
  }

  insertText(text) {
    const start = this.textarea.selectionStart;
    const value = this.textarea.value;

    this.textarea.value = value.substring(0, start) + text + value.substring(start);
    this.textarea.selectionStart = this.textarea.selectionEnd = start + text.length;
    this.textarea.focus();
  }

  getThreadId() {
    // 从 URL 或 data 属性获取主题 ID
    return this.textarea.dataset.threadId || 'new';
  }
}

// 初始化编辑器
document.addEventListener('DOMContentLoaded', () => {
  const textarea = document.getElementById('message');
  const preview = document.getElementById('preview');

  if (textarea && preview) {
    new BBCodeEditor(textarea, preview);
  }
});
```

**2. 表情符号插入器**
```javascript
// 文件: public/js/smilies.js

class SmilieInserter {
  constructor(editor, toolbar) {
    this.editor = editor;
    this.toolbar = toolbar;
    this.init();
  }

  async init() {
    // 加载表情列表
    const response = await fetch('/api/smilies');
    this.smilies = await response.json();

    // 创建表情选择器
    this.createSmiliePicker();
  }

  createSmiliePicker() {
    const button = document.createElement('button');
    button.type = 'button';
    button.textContent = '表情';
    button.className = 'btn-smilies';

    const picker = document.createElement('div');
    picker.className = 'smilie-picker';
    picker.style.display = 'none';

    this.smilies.forEach(smilie => {
      const img = document.createElement('img');
      img.src = smilie.url;
      img.alt = smilie.code;
      img.title = smilie.description;

      img.addEventListener('click', () => {
        this.editor.insertText(smilie.code);
        picker.style.display = 'none';
      });

      picker.appendChild(img);
    });

    button.addEventListener('click', () => {
      picker.style.display = picker.style.display === 'none' ? 'block' : 'none';
    });

    this.toolbar.appendChild(button);
    this.toolbar.appendChild(picker);
  }
}
```

**3. 图片粘贴上传**
```javascript
// 文件: public/js/paste-upload.js

document.getElementById('message').addEventListener('paste', async (e) => {
  const items = e.clipboardData.items;

  for (const item of items) {
    if (item.type.indexOf('image') !== -1) {
      e.preventDefault();

      const file = item.getAsFile();
      const formData = new FormData();
      formData.append('image', file);

      try {
        const response = await fetch('/api/upload/image', {
          method: 'POST',
          body: formData
        });

        const data = await response.json();

        if (data.success) {
          // 插入图片 BBCode
          const textarea = e.target;
          const bbcode = `[img]${data.url}[/img]`;
          textarea.setRangeText(bbcode, textarea.selectionStart, textarea.selectionEnd, 'end');
        }
      } catch (error) {
        console.error('图片上传失败:', error);
        alert('图片上传失败，请重试');
      }
    }
  }
});
```

**验收标准**:
- [ ] 实时预览功能正常
- [ ] 自动保存草稿功能正常
- [ ] 快捷键支持正常（Ctrl+B, Ctrl+I 等）
- [ ] 表情符号插入功能正常
- [ ] 图片粘贴上传功能正常
- [ ] 移动端兼容性正常

**相关文件**:
- `public/js/bbcode-editor.js` (新建)
- `public/js/smilies.js` (新建)
- `public/js/paste-upload.js` (新建)
- `public/css/bbcode-editor.css` (新建)

---

### Day 7 (2026-03-05): 附件上传 UI

#### 任务 15.4: 附件上传 UI (P0)
**负责人**: 前端开发团队
**预计时间**: 8 小时
**优先级**: 🔴 P0 - 关键

**实现步骤**:

**1. 拖拽上传组件**
```html
<!-- 模板片段 -->
<div class="upload-zone" id="upload-zone">
  <div class="upload-icon">📁</div>
  <p>拖拽文件到此处上传</p>
  <p>或者</p>
  <button type="button" class="btn-select-files">选择文件</button>
  <input type="file" id="file-input" multiple style="display: none;">
</div>

<div class="upload-list" id="upload-list">
  <!-- 上传的文件列表 -->
</div>
```

```javascript
// 文件: public/js/upload-zone.js

class UploadZone {
  constructor(zone, fileInput, uploadList) {
    this.zone = zone;
    this.fileInput = fileInput;
    this.uploadList = uploadList;
    this.init();
  }

  init() {
    // 拖拽事件
    this.zone.addEventListener('dragover', (e) => {
      e.preventDefault();
      this.zone.classList.add('dragover');
    });

    this.zone.addEventListener('dragleave', () => {
      this.zone.classList.remove('dragover');
    });

    this.zone.addEventListener('drop', (e) => {
      e.preventDefault();
      this.zone.classList.remove('dragover');
      this.handleFiles(e.dataTransfer.files);
    });

    // 点击选择文件
    this.zone.querySelector('.btn-select-files').addEventListener('click', () => {
      this.fileInput.click();
    });

    this.fileInput.addEventListener('change', () => {
      this.handleFiles(this.fileInput.files);
    });
  }

  async handleFiles(files) {
    for (const file of files) {
      await this.uploadFile(file);
    }
  }

  async uploadFile(file) {
    // 创建上传项
    const item = this.createUploadItem(file);
    this.uploadList.appendChild(item);

    const formData = new FormData();
    formData.append('attachment', file);

    try {
      const response = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
        headers: {
          'X-Requested-With': 'XMLHttpRequest'
        }
      });

      const data = await response.json();

      if (data.success) {
        this.updateUploadItem(item, 'complete', data);
      } else {
        this.updateUploadItem(item, 'error', data.error);
      }
    } catch (error) {
      this.updateUploadItem(item, 'error', error.message);
    }
  }

  createUploadItem(file) {
    const item = document.createElement('div');
    item.className = 'upload-item uploading';
    item.innerHTML = `
      <div class="file-icon">📄</div>
      <div class="file-info">
        <div class="file-name">${file.name}</div>
        <div class="file-size">${this.formatSize(file.size)}</div>
        <progress value="0" max="100"></progress>
      </div>
      <button type="button" class="btn-remove">✕</button>
    `;

    item.querySelector('.btn-remove').addEventListener('click', () => {
      item.remove();
    });

    return item;
  }

  updateUploadItem(item, status, data) {
    item.className = `upload-item ${status}`;

    if (status === 'complete') {
      item.querySelector('progress').value = 100;
      item.dataset.aid = data.aid; // 附件 ID
    } else if (status === 'error') {
      item.querySelector('.file-name').textContent += ` (上传失败: ${data})`;
    }
  }

  formatSize(bytes) {
    if (bytes < 1024) return bytes + ' B';
    if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
    return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
  }
}

// 初始化上传区域
document.addEventListener('DOMContentLoaded', () => {
  const zone = document.getElementById('upload-zone');
  const fileInput = document.getElementById('file-input');
  const uploadList = document.getElementById('upload-list');

  if (zone && fileInput && uploadList) {
    new UploadZone(zone, fileInput, uploadList);
  }
});
```

**2. 上传进度跟踪**
```javascript
// 使用 XMLHttpRequest 跟踪上传进度
function uploadFileWithProgress(file, onProgress) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    const formData = new FormData();
    formData.append('attachment', file);

    xhr.upload.addEventListener('progress', (e) => {
      if (e.lengthComputable) {
        const percent = (e.loaded / e.total) * 100;
        onProgress(percent);
      }
    });

    xhr.addEventListener('load', () => {
      if (xhr.status === 200) {
        const data = JSON.parse(xhr.responseText);
        resolve(data);
      } else {
        reject(new Error('上传失败'));
      }
    });

    xhr.addEventListener('error', () => {
      reject(new Error('网络错误'));
    });

    xhr.open('POST', '/api/upload');
    xhr.send(formData);
  });
}
```

**验收标准**:
- [ ] 拖拽上传功能正常
- [ ] 点击选择文件功能正常
- [ ] 上传进度显示正常
- [ ] 上传完成后显示附件信息
- [ ] 可以删除已上传的附件
- [ ] 文件类型验证正常
- [ ] 文件大小限制正常
- [ ] 移动端兼容性正常

**相关文件**:
- `app/View/templates/partials/upload-zone.html.twig` (新建)
- `public/js/upload-zone.js` (新建)
- `public/css/upload-zone.css` (新建)
- `app/Http/Controllers/UploadController.php`

---

## 🔐 Week 16: 管理员后台实现（2026-03-06 ~ 2026-03-12）

### 目标
实现管理员后台（AdminCP），提供系统管理和监控功能。

### 完成标准
- ✅ 管理员认证系统完整
- ✅ 管理员仪表盘功能完整
- ✅ 用户管理功能完整
- ✅ 版块管理功能完整
- ✅ 系统监控功能完整

---

### Day 1-2 (2026-03-06 ~ 2026-03-07): 管理员认证系统

#### 任务 16.1: 管理员认证 (P0)
**负责人**: 后端开发团队
**预计时间**: 12 小时（2 天）
**优先级**: 🔴 P0 - 关键

**Legacy 参考**:
```
poketb.com/bbs/admincp.php
poketb.com/bbs/admincp.login.php
```

**实现步骤**:

**1. 创建管理员认证服务**
```php
<?php
declare(strict_types=1);

namespace Discuz\Admin\Services;

use Discuz\User\Repository\UserRepository;
use Discuz\Security\Services\Password\PasswordVerifier;
use Discuz\Session\Services\SessionService;

/**
 * 管理员认证服务
 */
class AdminAuthService
{
    private UserRepository $userRepository;
    private PasswordVerifier $passwordVerifier;
    private SessionService $sessionService;

    public function __construct(
        UserRepository $userRepository,
        PasswordVerifier $passwordVerifier,
        SessionService $sessionService
    ) {
        $this->userRepository = $userRepository;
        $this->passwordVerifier = $passwordVerifier;
        $this->sessionService = $sessionService;
    }

    /**
     * 管理员登录
     */
    public function login(string $username, string $password, bool $remember = false): array
    {
        // 查找用户
        $user = $this->userRepository->findByUsername($username);

        if (!$user) {
            throw new AdminAuthException('用户名或密码错误');
        }

        // 验证是否为管理员
        if (!$this->isAdmin($user)) {
            throw new AdminAuthException('您不是管理员');
        }

        // 验证密码
        if (!$this->passwordVerifier->verify($password, $user->password, $user->salt)) {
            throw new AdminAuthException('用户名或密码错误');
        }

        // 创建管理员会话
        $this->sessionService->set('admin_logged_in', true);
        $this->sessionService->set('admin_uid', $user->uid);
        $this->sessionService->set('admin_username', $user->username);

        // 记录登录日志
        $this->logAdminAction($user->uid, 'login', [
            'ip' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? ''
        ]);

        return [
            'uid' => $user->uid,
            'username' => $user->username,
            'adminid' => $user->adminid
        ];
    }

    /**
     * 管理员登出
     */
    public function logout(): void
    {
        $uid = $this->sessionService->get('admin_uid');

        // 记录登出日志
        if ($uid) {
            $this->logAdminAction($uid, 'logout', []);
        }

        // 清除管理员会话
        $this->sessionService->delete('admin_logged_in');
        $this->sessionService->delete('admin_uid');
        $this->sessionService->delete('admin_username');
    }

    /**
     * 检查是否已登录
     */
    public function isLoggedIn(): bool
    {
        return $this->sessionService->get('admin_logged_in', false) === true;
    }

    /**
     * 获取当前登录的管理员
     */
    public function getCurrentAdmin(): ?array
    {
        if (!$this->isLoggedIn()) {
            return null;
        }

        return [
            'uid' => $this->sessionService->get('admin_uid'),
            'username' => $this->sessionService->get('admin_username')
        ];
    }

    /**
     * 验证是否为管理员
     */
    private isAdmin(array $user): bool
    {
        // Legacy: adminid = 1 为超级管理员
        // adminid = 2, 3 为其他管理员
        return in_array($user['adminid'] ?? 0, [1, 2, 3], true);
    }

    /**
     * 记录管理员操作日志
     */
    private function logAdminAction(int $uid, string $action, array $data): void
    {
        // 记录到 cdb_admin_logs 表（如果存在）或 cdb_moderation_logs
    }
}

class AdminAuthException extends \Exception
{
}
```

**2. 创建管理员权限服务**
```php
<?php
declare(strict_types=1);

namespace Discuz\Admin\Services;

/**
 * 管理员权限服务
 */
class AdminPermissionService
{
    /**
     * 检查管理员权限
     */
    public function can(int $adminId, string $permission): bool
    {
        // 超级管理员（adminid = 1）拥有所有权限
        if ($adminId === 1) {
            return true;
        }

        // 从数据库加载管理员权限
        $permissions = $this->loadAdminPermissions($adminId);

        return in_array($permission, $permissions, true);
    }

    /**
     * 从数据库加载管理员权限
     */
    private function loadAdminPermissions(int $adminId): array
    {
        // Legacy: cdb_adminpermissions 表
        // SELECT permissions FROM cdb_adminpermissions WHERE uid = ?
        // 返回权限数组
        return [];
    }

    /**
     * 常用权限常量
     */
    public const VIEW_DASHBOARD = 'admin.view_dashboard';
    public const MANAGE_USERS = 'admin.manage_users';
    public const MANAGE_FORUMS = 'admin.manage_forums';
    public const MANAGE_THREADS = 'admin.manage_threads';
    public const MANAGE_SETTINGS = 'admin.manage_settings';
    public const VIEW_LOGS = 'admin.view_logs';
}
```

**3. 创建管理员登录控制器**
```php
<?php
declare(strict_types=1);

namespace Discuz\Http\Controllers\Admin;

use Discuz\Admin\Services\AdminAuthService;
use Discuz\Http\Request;
use Discuz\Http\Response;
use Discuz\View\ViewRenderer;

/**
 * 管理员认证控制器
 */
class AdminAuthController
{
    private AdminAuthService $adminAuthService;
    private ViewRenderer $viewRenderer;

    public function __construct(
        AdminAuthService $adminAuthService,
        ViewRenderer $viewRenderer
    ) {
        $this->adminAuthService = $adminAuthService;
        $this->viewRenderer = $viewRenderer;
    }

    /**
     * 显示登录页面
     */
    public function loginFormAction(): Response
    {
        // 如果已登录，重定向到仪表盘
        if ($this->adminAuthService->isLoggedIn()) {
            return Response::redirect('/admin/dashboard');
        }

        return $this->viewRenderer->render('admin/login.html.twig');
    }

    /**
     * 处理登录请求
     */
    public function loginAction(Request $request): Response
    {
        $username = $request->post('username', '');
        $password = $request->post('password', '');
        $remember = $request->post('remember') === '1';

        try {
            $admin = $this->adminAuthService->login($username, $password, $remember);

            return Response::redirect('/admin/dashboard');
        } catch (AdminAuthException $e) {
            return $this->viewRenderer->render('admin/login.html.twig', [
                'error' => $e->getMessage()
            ]);
        }
    }

    /**
     * 登出
     */
    public function logoutAction(): Response
    {
        $this->adminAuthService->logout();

        return Response::redirect('/admin/login');
    }
}
```

**验收标准**:
- [ ] 管理员登录功能正常
- [ ] 管理员权限验证正常
- [ ] 会话管理正常
- [ ] 登录日志记录正常
- [ ] 登出功能正常

**相关文件**:
- `app/Admin/Services/AdminAuthService.php` (新建)
- `app/Admin/Services/AdminPermissionService.php` (新建)
- `app/Http/Controllers/Admin/AdminAuthController.php` (新建)
- `app/View/templates/admin/login.html.twig` (新建)

---

### Day 3-4 (2026-03-08 ~ 2026-03-09): 管理员仪表盘

#### 任务 16.2: 管理员仪表盘 (P0)
**负责人**: 前端开发团队
**预计时间**: 12 小时（2 天）
**优先级**: 🔴 P0 - 关键

**实现步骤**:

**1. 仪表盘模板**
```twig
{# 文件: app/View/templates/admin/dashboard.html.twig #}
{% extends 'admin/layout.html.twig' %}

{% block title %}管理员后台 - 控制面板{% endblock %}

{% block content %}
<div class="dashboard">
  <h1>控制面板</h1>

  <!-- 统计卡片 -->
  <div class="stats-cards">
    <div class="stat-card">
      <div class="stat-icon">👥</div>
      <div class="stat-info">
        <div class="stat-value">{{ stats.user_count|number_format }}</div>
        <div class="stat-label">注册用户</div>
      </div>
    </div>

    <div class="stat-card">
      <div class="stat-icon">💬</div>
      <div class="stat-info">
        <div class="stat-value">{{ stats.thread_count|number_format }}</div>
        <div class="stat-label">主题总数</div>
      </div>
    </div>

    <div class="stat-card">
      <div class="stat-icon">📝</div>
      <div class="stat-info">
        <div class="stat-value">{{ stats.post_count|number_format }}</div>
        <div class="stat-label">帖子总数</div>
      </div>
    </div>

    <div class="stat-card">
      <div class="stat-icon">📎</div>
      <div class="stat-info">
        <div class="stat-value">{{ stats.attachment_count|number_format }}</div>
        <div class="stat-label">附件总数</div>
      </div>
    </div>
  </div>

  <!-- 快速操作 -->
  <div class="quick-actions">
    <h2>快速操作</h2>
    <div class="action-buttons">
      <a href="{{ path('admin.users') }}" class="btn-action">用户管理</a>
      <a href="{{ path('admin.forums') }}" class="btn-action">版块管理</a>
      <a href="{{ path('admin.threads') }}" class="btn-action">主题管理</a>
      <a href="{{ path('admin.settings') }}" class="btn-action">系统设置</a>
      <a href="{{ path('admin.logs') }}" class="btn-action">日志查看</a>
    </div>
  </div>

  <!-- 系统信息 -->
  <div class="system-info">
    <h2>系统信息</h2>
    <table class="info-table">
      <tr>
        <td>Discuz! 版本</td>
        <td>8.3.0 (Modern PHP)</td>
      </tr>
      <tr>
        <td>PHP 版本</td>
        <td>{{ system.php_version }}</td>
      </tr>
      <tr>
        <td>MySQL 版本</td>
        <td>{{ system.mysql_version }}</td>
      </tr>
      <tr>
        <td>服务器时间</td>
        <td>{{ system.server_time|date('Y-m-d H:i:s') }}</td>
      </tr>
      <tr>
        <td>内存使用</td>
        <td>{{ system.memory_usage }}</td>
      </tr>
    </table>
  </div>

  <!-- 最近活动 -->
  <div class="recent-activity">
    <h2>最近活动</h2>
    <ul class="activity-list">
      {% for activity in recent_activities %}
      <li>
        <span class="activity-time">{{ activity.created_at|date('H:i') }}</span>
        <span class="activity-text">{{ activity.message }}</span>
      </li>
      {% endfor %}
    </ul>
  </div>

  <!-- 图表 -->
  <div class="charts">
    <h2>统计图表</h2>
    <div class="chart-container">
      <canvas id="registrations-chart"></canvas>
    </div>
    <div class="chart-container">
      <canvas id="posts-chart"></canvas>
    </div>
  </div>
</div>
{% endblock %}
```

**2. 统计数据服务**
```php
<?php
declare(strict_types=1);

namespace Discuz\Admin\Services;

use Discuz\Database\Database;

/**
 * 管理员统计服务
 */
class AdminStatsService
{
    private Database $db;

    public function __construct(Database $db)
    {
        $this->db = $db;
    }

    /**
     * 获取仪表盘统计数据
     */
    public function getDashboardStats(): array
    {
        return [
            'user_count' => $this->getUserCount(),
            'thread_count' => $this->getThreadCount(),
            'post_count' => $this->getPostCount(),
            'attachment_count' => $this->getAttachmentCount(),
            'online_users' => $this->getOnlineUserCount(),
            'today_posts' => $this->getTodayPostCount()
        ];
    }

    private function getUserCount(): int
    {
        $stmt = $this->db->prepare('SELECT COUNT(*) FROM cdb_members');
        $stmt->execute();
        return (int)$stmt->fetchColumn();
    }

    private function getThreadCount(): int
    {
        $stmt = $this->db->prepare('SELECT COUNT(*) FROM cdb_threads');
        $stmt->execute();
        return (int)$stmt->fetchColumn();
    }

    private function getPostCount(): int
    {
        $stmt = $this->db->prepare('SELECT COUNT(*) FROM cdb_posts');
        $stmt->execute();
        return (int)$stmt->fetchColumn();
    }

    private function getAttachmentCount(): int
    {
        $stmt = $this->db->prepare('SELECT COUNT(*) FROM cdb_attachments');
        $stmt->execute();
        return (int)$stmt->fetchColumn();
    }

    private function getOnlineUserCount(): int
    {
        // Legacy: cdb_sessions 表或 cdb_onlinelist
        $stmt = $this->db->prepare('SELECT COUNT(*) FROM cdb_sessions WHERE uid > 0');
        $stmt->execute();
        return (int)$stmt->fetchColumn();
    }

    private function getTodayPostCount(): int
    {
        $stmt = $this->db->prepare('
            SELECT COUNT(*) FROM cdb_posts
            WHERE dateline >= ?
        ');
        $stmt->execute([strtotime('today')]);
        return (int)$stmt->fetchColumn();
    }

    /**
     * 获取最近活动
     */
    public function getRecentActivity(int $limit = 20): array
    {
        // 从 cdb_moderation_logs 或 cdb_admin_logs 获取
        $stmt = $this->db->prepare('
            SELECT * FROM cdb_moderation_logs
            ORDER BY created_at DESC
            LIMIT ?
        ');
        $stmt->execute([$limit]);
        return $stmt->fetchAll();
    }
}
```

**验收标准**:
- [ ] 仪表盘显示正常
- [ ] 统计数据准确
- [ ] 快速操作链接正常
- [ ] 系统信息显示正常
- [ ] 最近活动显示正常
- [ ] 图表渲染正常（使用 Chart.js）

**相关文件**:
- `app/View/templates/admin/dashboard.html.twig` (新建)
- `app/Admin/Services/AdminStatsService.php` (新建)
- `app/Http/Controllers/Admin/AdminDashboardController.php` (新建)

---

### Day 5-6 (2026-03-10 ~ 2026-03-11): 用户与版块管理

#### 任务 16.3: 用户管理 (P0)
**负责人**: 后端开发团队
**预计时间**: 12 小时（2 天）
**优先级**: 🔴 P0 - 关键

**实现步骤**:

**1. 用户列表模板**
```twig
{# 文件: app/View/templates/admin/users.html.twig #}
{% extends 'admin/layout.html.twig' %}

{% block title %}管理员后台 - 用户管理{% endblock %}

{% block content %}
<div class="users-management">
  <h1>用户管理</h1>

  <!-- 搜索和筛选 -->
  <div class="search-filters">
    <form method="get" class="search-form">
      <input type="text" name="username" placeholder="搜索用户名" value="{{ filters.username }}">
      <select name="groupid">
        <option value="">所有用户组</option>
        {% for group in usergroups %}
        <option value="{{ group.groupid }}" {% if filters.groupid == group.groupid %}selected{% endif %}>
          {{ group.grouptitle }}
        </option>
        {% endfor %}
      </select>
      <button type="submit" class="btn-search">搜索</button>
    </form>
  </div>

  <!-- 用户列表 -->
  <table class="data-table">
    <thead>
      <tr>
        <th><input type="checkbox" id="select-all"></th>
        <th>UID</th>
        <th>用户名</th>
        <th>用户组</th>
        <th>邮箱</th>
        <th>注册时间</th>
        <th>最后访问</th>
        <th>帖子数</th>
        <th>状态</th>
        <th>操作</th>
      </tr>
    </thead>
    <tbody>
      {% for user in users %}
      <tr>
        <td><input type="checkbox" name="uids[]" value="{{ user.uid }}"></td>
        <td>{{ user.uid }}</td>
        <td>{{ user.username }}</td>
        <td>{{ user.grouptitle }}</td>
        <td>{{ user.email }}</td>
        <td>{{ user.regdate|date('Y-m-d') }}</td>
        <td>{{ user.lastvisit|date('Y-m-d H:i') }}</td>
        <td>{{ user.posts }}</td>
        <td>
          {% if user.adminid > 0 %}
            <span class="badge-admin">管理员</span>
          {% endif %}
          {% if user.groupid == 4 %}
            <span class="badge-banned">已禁用</span>
          {% endif %}
        </td>
        <td>
          <a href="{{ path('admin.user.edit', {uid: user.uid}) }}" class="btn-edit">编辑</a>
          <a href="{{ path('admin.user.delete', {uid: user.uid}) }}" class="btn-delete" onclick="return confirm('确定删除？')">删除</a>
        </td>
      </tr>
      {% endfor %}
    </tbody>
  </table>

  <!-- 批量操作 -->
  <div class="bulk-actions">
    <select name="action">
      <option value="">批量操作...</option>
      <option value="ban">禁用用户</option>
      <option value="unban">解除禁用</option>
      <option value="delete">删除用户</option>
    </select>
    <button type="submit" class="btn-apply">应用</button>
  </div>

  <!-- 分页 -->
  <div class="pagination">
    {{ pagination|raw }}
  </div>
</div>
{% endblock %}
```

**2. 用户编辑模板**
```twig
{# 文件: app/View/templates/admin/user-edit.html.twig #}
{% extends 'admin/layout.html.twig' %}

{% block title %}管理员后台 - 编辑用户{% endblock %}

{% block content %}
<div class="user-edit">
  <h1>编辑用户: {{ user.username }}</h1>

  <form method="post" action="{{ path('admin.user.update', {uid: user.uid}) }}">
    {{ csrf_field() }}

    <!-- 基本信息 -->
    <fieldset>
      <legend>基本信息</legend>
      <div class="form-group">
        <label>用户名</label>
        <input type="text" name="username" value="{{ user.username }}" readonly>
      </div>
      <div class="form-group">
        <label>邮箱</label>
        <input type="email" name="email" value="{{ user.email }}">
      </div>
      <div class="form-group">
        <label>用户组</label>
        <select name="groupid">
          {% for group in usergroups %}
          <option value="{{ group.groupid }}" {% if user.groupid == group.groupid %}selected{% endif %}>
            {{ group.grouptitle }}
          </option>
          {% endfor %}
        </select>
      </div>
    </fieldset>

    <!-- 管理员设置 -->
    <fieldset>
      <legend>管理员设置</legend>
      <div class="form-group">
        <label>
          <input type="checkbox" name="is_admin" value="1" {% if user.adminid > 0 %}checked{% endif %}>
          设为管理员
        </label>
      </div>
      {% if user.adminid > 0 %}
      <div class="form-group">
        <label>管理员级别</label>
        <select name="adminid">
          <option value="1" {% if user.adminid == 1 %}selected{% endif %}>超级管理员</option>
          <option value="2" {% if user.adminid == 2 %}selected{% endif %}>管理员</option>
          <option value="3" {% if user.adminid == 3 %}selected{% endif %}>版主</option>
        </select>
      </div>
      {% endif %}
    </fieldset>

    <!-- 积分设置 -->
    <fieldset>
      <legend>积分设置</legend>
      <div class="form-group">
        <label>积分</label>
        <input type="number" name="credits" value="{{ user.credits }}">
      </div>
      <div class="form-group">
        <label> extcredits1</label>
        <input type="number" name="extcredits1" value="{{ user.extcredits1 }}">
      </div>
      <div class="form-group">
        <label> extcredits2</label>
        <input type="number" name="extcredits2" value="{{ user.extcredits2 }}">
      </div>
    </fieldset>

    <!-- 状态设置 -->
    <fieldset>
      <legend>状态设置</legend>
      <div class="form-group">
        <label>
          <input type="checkbox" name="status" value="1" {% if user.groupid != 4 %}checked{% endif %}>
          启用
        </label>
      </div>
    </fieldset>

    <div class="form-actions">
      <button type="submit" class="btn-primary">保存更改</button>
      <a href="{{ path('admin.users') }}" class="btn-cancel">取消</a>
    </div>
  </form>
</div>
{% endblock %}
```

**验收标准**:
- [ ] 用户列表显示正常
- [ ] 用户搜索功能正常
- [ ] 用户编辑功能正常
- [ ] 批量操作功能正常
- [ ] 用户状态修改正常

**相关文件**:
- `app/View/templates/admin/users.html.twig` (新建)
- `app/View/templates/admin/user-edit.html.twig` (新建)
- `app/Http/Controllers/Admin/AdminUserController.php` (新建)

---

#### 任务 16.4: 版块管理 (P0)
**负责人**: 后端开发团队
**预计时间**: 8 小时
**优先级**: 🔴 P0 - 关键

**实现步骤**:

**1. 版块列表模板**
```twig
{# 文件: app/View/templates/admin/forums.html.twig #}
{% extends 'admin/layout.html.twig' %}

{% block title %}管理员后台 - 版块管理{% endblock %}

{% block content %}
<div class="forums-management">
  <h1>版块管理</h1>

  <!-- 版块树形列表 -->
  <div class="forums-tree">
    {% for category in categories %}
    <div class="forum-category">
      <h3>{{ category.name }}</h3>

      <table class="data-table">
        <thead>
          <tr>
            <th>显示顺序</th>
            <th>版块名称</th>
            <th>主题数</th>
            <th>帖子数</th>
            <th>今日帖数</th>
            <th>状态</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          {% for forum in category.forums %}
          <tr>
            <td>{{ forum.displayorder }}</td>
            <td>{{ forum.name }}</td>
            <td>{{ forum.threads }}</td>
            <td>{{ forum.posts }}</td>
            <td>{{ forum.todayposts }}</td>
            <td>
              {% if forum.status == 1 %}
                <span class="badge-success">启用</span>
              {% else %}
                <span class="badge-disabled">禁用</span>
              {% endif %}
            </td>
            <td>
              <a href="{{ path('admin.forum.edit', {fid: forum.fid}) }}" class="btn-edit">编辑</a>
            </td>
          </tr>
          {% endfor %}
        </tbody>
      </table>
    </div>
    {% endfor %}
  </div>

  <!-- 添加版块按钮 -->
  <div class="actions">
    <a href="{{ path('admin.forum.create') }}" class="btn-primary">添加版块</a>
  </div>
</div>
{% endblock %}
```

**验收标准**:
- [ ] 版块列表显示正常
- [ ] 版块编辑功能正常
- [ ] 版块创建功能正常
- [ ] 版块排序功能正常

**相关文件**:
- `app/View/templates/admin/forums.html.twig` (新建)
- `app/View/templates/admin/forum-edit.html.twig` (新建)
- `app/Http/Controllers/Admin/AdminForumController.php` (新建)

---

### Day 7 (2026-03-12): Week 16 收尾与总结

#### 任务 16.5: Week 14-16 总体验收 (P0)
**负责人**: 项目经理
**预计时间**: 4 小时
**优先级**: 🔴 P0 - 关键

**验收清单**:
```markdown
## Week 14-16 完成清单

### Week 14: 质量保证与验证 ✅
- [ ] 测试套件修复完成（≥ 95% 通过）
- [ ] 性能基准建立
- [ ] 文档准确性验证（100%）
- [ ] 控制器补全完成

### Week 15: 交互表单实现 ✅
- [ ] 新主题表单功能完整
- [ ] 回复表单功能完整
- [ ] BBCode 编辑器功能完整
- [ ] 附件上传 UI 功能完整

### Week 16: 管理员后台 ✅
- [ ] 管理员认证系统完整
- [ ] 管理员仪表盘功能完整
- [ ] 用户管理功能完整
- [ ] 版块管理功能完整

### 零改表原则验证 ✅
- [ ] Week 14-16 无新增违规表
- [ ] 所有新表均已批准

### 投产准备度 ✅
- [ ] 整体完成度 ≥ 95%
- [ ] 测试覆盖率 ≥ 90%
- [ ] 性能基准达标
- [ ] 文档完整性 100%
```

**最终报告**:
```markdown
# Week 14-16 最终报告
**日期**: 2026-03-12
**项目经理**: [姓名]

## 项目状态
- **开始完成度**: 72%
- **最终完成度**: 95%
- **提升幅度**: +23%

## 主要成果

### Week 14: 质量保证与验证
- 测试套件通过率: ___%
- 性能基准: 首页加载 ___ ms
- 文档准确性: 100%

### Week 15: 交互表单实现
- 新主题表单: ✅ 100%
- 回复表单: ✅ 100%
- BBCode 编辑器: ✅ 100%
- 附件上传: ✅ 100%

### Week 16: 管理员后台
- 管理员认证: ✅ 100%
- 管理员仪表盘: ✅ 100%
- 用户管理: ✅ 100%
- 版块管理: ✅ 100%

## 投产就绪性评估

### 已就绪 ✅
- [x] 核心功能完整（认证、发帖、回复、浏览）
- [x] 管理后台完整（用户管理、版块管理）
- [x] 测试覆盖充分（≥ 90%）
- [x] 性能基准达标
- [x] 安全性验证通过
- [x] 文档完整准确

### 待优化 ⚠️
- [ ] 搜索功能（Week 8，可延后）
- [ ] 高级主题功能（可延后）

### 建议投产时间
**2026-03-15**（Week 16 完成后 3 天）

## 风险评估

### 低风险 ✅
- 数据迁移: 100% 完成，零数据丢失
- 安全漏洞: 0 个高危漏洞
- 零改表原则: 100% 合规

### 中风险 🟡
- 性能压测: 建议生产环境压力测试
- 用户体验: 建议小规模试用后全面上线

## 后续计划

### Week 17-18: 生产部署
1. 生产环境配置
2. 数据备份策略
3. 监控告警设置
4. 灰度发布计划

### Week 19+: 持续优化
1. 性能优化
2. 功能增强
3. 用户体验改进

## 团队感谢
感谢所有团队成员的努力付出！
```

---

## 📊 三周行动计划汇总

### 时间线
```
Week 14: 2026-02-20 ~ 2026-02-26 (质量保证与验证)
Week 15: 2026-02-27 ~ 2026-03-05 (交互表单实现)
Week 16: 2026-03-06 ~ 2026-03-12 (管理员后台实现)
```

### 关键里程碑
| 日期 | 里程碑 | 预期完成度 |
|------|--------|-----------|
| 2026-02-24 | Week 14 测试修复完成 | 75% |
| 2026-03-02 | Week 15 新主题表单完成 | 85% |
| 2026-03-09 | Week 16 管理员认证完成 | 92% |
| 2026-03-12 | Week 14-16 全部完成 | 95% |

### 交付物清单
**Week 14**:
- [ ] 测试结果报告
- [ ] 性能基准报告
- [ ] 更新的进度文档
- [ ] 控制器补全代码

**Week 15**:
- [ ] 新主题表单 UI
- [ ] 回复表单 UI
- [ ] BBCode 编辑器组件
- [ ] 附件上传组件

**Week 16**:
- [ ] 管理员认证系统
- [ ] 管理员仪表盘
- [ ] 用户管理界面
- [ ] 版块管理界面

---

## 🚨 风险与应对

### 高风险 🔴
1. **测试套件修复耗时超出预期**
   - 应对: 预留 2 天缓冲时间
   - 备选: 先修复 P0 测试，P1 测试可延后

2. **性能测试发现严重性能问题**
   - 应对: 准备优化方案（数据库索引、缓存）
   - 备选: 延后非关键功能

### 中风险 🟡
1. **前端 UI 实现复杂度超出预期**
   - 应对: 简化 UI 设计，分阶段实现
   - 备选: 使用现成 UI 组件库

2. **管理员后台功能需求变更**
   - 应对: 与产品方确认优先级
   - 备选: 基础功能优先，高级功能延后

### 低风险 🟢
1. **文档更新工作量大**
   - 应对: 使用模板批量生成
   - 备选: 先更新核心文档

---

## 📝 资源需求

### 人员配置
- 后端开发: 2 人 × 3 周 = 6 人周
- 前端开发: 1 人 × 2 周（Week 15-16） = 2 人周
- 测试工程师: 1 人 × 1 周（Week 14） = 1 人周
- 项目经理: 0.5 人 × 3 周 = 1.5 人周

**总计**: 10.5 人周

### 技术资源
- 开发环境: Docker PHP 8.3 + MySQL 8.0
- 测试环境: 独立测试数据库
- 前端工具: Twig 3.x, Chart.js, jQuery/Alpine.js
- 监控工具: Symfony StopWatch, Blackfire

---

## ✅ 成功标准

### Week 14 完成标准
- [ ] 所有 P0 测试通过（≥ 95%）
- [ ] 性能基准报告生成
- [ ] 文档准确率 100%

### Week 15 完成标准
- [ ] 用户可通过 UI 创建主题
- [ ] 用户可通过 UI 回复主题
- [ ] BBCode 编辑器功能完整
- [ ] 附件上传功能完整

### Week 16 完成标准
- [ ] 管理员可登录后台
- [ ] 管理员可管理用户
- [ ] 管理员可管理版块
- [ ] 系统监控功能正常

### 三周总体完成标准
- [ ] 项目完成度 ≥ 95%
- [ ] 测试覆盖率 ≥ 90%
- [ ] 零安全漏洞
- [ ] 零改表违规
- [ ] 生产就绪度 ≥ 95%

---

**文档版本**: 1.0
**最后更新**: 2026-02-19
**审核状态**: 待审核

---

## 📞 联系方式

如有问题或需要调整计划，请联系：
- 项目经理: [姓名/邮箱]
- 技术负责人: [姓名/邮箱]
- 前端负责人: [姓名/邮箱]
- 测试负责人: [姓名/邮箱]
