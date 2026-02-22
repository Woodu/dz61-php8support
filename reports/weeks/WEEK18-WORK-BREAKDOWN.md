# Week 18 工作拆解计划

**日期**: 2026-02-21
**状态**: ⏳ 待开始
**预计周期**: 3个工作日 (20小时)
**主要目标**: Reward Thread迁移 + 附件UI优化 + 性能优化

---

## 📊 Week 18 任务概览

### 任务优先级（按零改表原则和Legacy功能对比）

| 优先级 | 任务 | 预计时间 | Legacy功能 | 说明 |
|--------|------|----------|-----------|------|
| **P0** | Task #1: Reward Thread分析 | 2小时 | ✅ 有 | Legacy有完整悬赏功能，必须迁移 |
| **P0** | Task #2: RewardService实现 | 4小时 | ✅ 有 | 悬赏创建、最佳答案、积分发放 |
| **P0** | Task #3: RewardController集成 | 2小时 | ✅ 有 | 悬赏表单UI和交互 |
| **P1** | Task #4: 附件上传UI优化 | 8小时 | ⚠️ 部分 | Legacy有基础上传，增强UX |
| **P2** | Task #5: 性能优化 | 4小时 | ✅ 有 | 优化现有BBCode和文本处理 |

**总计**: 20小时 (3个工作日)

---

## 🔴 P0 优先级: Reward Thread迁移 (8小时)

### Task #1: Reward Thread分析 (2小时)

**目标**: 分析Legacy悬赏主题功能，验证数据库表结构

#### 子任务 #1.1: 验证Legacy数据库表 (30分钟)

**检查内容**:
- [ ] 验证 `cdb_reward` 表是否存在
- [ ] 分析表结构和字段
- [ ] 检查索引配置
- [ ] 统计现有悬赏主题数量

**SQL查询**:
```sql
-- 检查表是否存在
SHOW TABLES LIKE 'cdb_reward';

-- 查看表结构
DESCRIBE cdb_reward;

-- 统计悬赏主题数
SELECT COUNT(*) FROM cdb_reward;

-- 查看示例数据
SELECT * FROM cdb_reward LIMIT 5;
```

**如果表不存在**:
- 检查Legacy是否使用 `cdb_threads` 表存储悬赏信息
- 查找 `price`, `reward` 等字段
- 分析Legacy悬赏实现方式

#### 子任务 #1.2: 分析Legacy悬赏功能 (45分钟)

**检查文件**: `poketb.com/bbs/`

**需要分析的文件**:
- [ ] `newthread.inc.php` - 悬赏主题创建逻辑
- [ ] `viewthread.inc.php` - 悬赏主题显示逻辑
- [ ] `misc.inc.php` - 最佳答案选择逻辑

**分析内容**:
- 悬赏创建流程
- 悬赏状态管理
- 最佳答案选择机制
- 积分扣除和发放逻辑
- 悬赏过期处理

**输出**: `REWARD-THREAD-LEGACY-ANALYSIS.md`

#### 子任务 #1.3: 设计现代化方案 (45分钟)

**Service层设计**:
```
app/Reward/
├── Services/
│   ├── RewardService.php          # 悬赏核心服务
│   └── RewardExpirationService.php # 悬赏过期处理
├── Repositories/
│   └── RewardRepository.php        # 悬赏数据访问
├── DTOs/
│   ├── RewardCreation.php          # 悬赏创建DTO
│   └── RewardAnswer.php            # 答案DTO
└── Exceptions/
    └── RewardException.php         # 悬赏异常
```

**Controller层设计**:
```
app/Http/Controllers/
└── RewardController.php            # 悬赏控制器
```

**模板设计**:
```
templates/reward/
├── create.html.twig                # 悬赏创建表单
└── answer.html.twig                # 最佳答案选择
```

**输出**: `REWARD-THREAD-MODERN-DESIGN.md`

#### 子任务 #1.4: 创建任务测试计划 (20分钟)

**测试用例**:
- [ ] 创建悬赏主题测试
- [ ] 悬赏状态查询测试
- [ ] 最佳答案选择测试
- [ ] 积分扣除测试
- [ ] 积分发放测试
- [ ] 悬赏过期测试

**输出**: 测试用例清单

---

### Task #2: RewardService实现 (4小时)

#### 子任务 #2.1: 创建RewardRepository (1小时)

**文件**: `app/Reward/Repositories/RewardRepository.php`

**方法**:
```php
class RewardRepository
{
    public function findById(int $threadId): ?array
    public function create(array $data): int
    public function update(int $threadId, array $data): bool
    public function delete(int $threadId): bool
    public function getActiveRewards(int $limit = 10): array
    public function getExpiredRewards(): array
    public function markAsResolved(int $threadId, int $postId): bool
    public function markAsExpired(int $threadId): bool
}
```

**测试**: `tests/Unit/Reward/RewardRepositoryTest.php`

#### 子任务 #2.2: 创建RewardService (2小时)

**文件**: `app/Reward/Services/RewardService.php`

**核心方法**:
```php
class RewardService
{
    // 创建悬赏
    public function createReward(RewardCreation $dto): int

    // 查询悬赏状态
    public function getRewardStatus(int $threadId): ?RewardStatus

    // 选择最佳答案
    public function selectBestAnswer(int $threadId, int $postId, int $authorId): bool

    // 检查是否可以参与悬赏
    public function canParticipate(int $threadId, int $userId): bool

    // 检查悬赏是否过期
    public function isExpired(int $threadId): bool

    // 发放悬赏积分
    public function payoutReward(int $threadId, int $winnerId): bool

    // 退款（无人回答时）
    public function refundReward(int $threadId): bool
}
```

**DTOs**:
- `RewardCreation` - 悬赏创建参数
- `RewardStatus` - 悬赏状态对象

**测试**: `tests/Unit/Reward/RewardServiceTest.php` (30+测试用例)

#### 子任务 #2.3: 创建RewardExpirationService (1小时)

**文件**: `app/Reward/Services/RewardExpirationService.php`

**方法**:
```php
class RewardExpirationService
{
    // 检查并处理过期悬赏
    public function processExpiredRewards(): int

    // 自动退款
    public function autoRefund(int $threadId): bool

    // 标记为过期
    public function markAsExpired(int $threadId): bool
}
```

**测试**: `tests/Unit/Reward/RewardExpirationServiceTest.php`

---

### Task #3: RewardController集成 (2小时)

#### 子任务 #3.1: 创建RewardController (1小时)

**文件**: `app/Http/Controllers/RewardController.php`

**方法**:
```php
class RewardController
{
    // 显示悬赏创建表单
    public function showCreateFormAction(int $forumId): array

    // 处理悬赏创建
    public function createRewardAction(int $forumId, array $data): array

    // 显示答案选择界面
    public function showAnswersAction(int $threadId): array

    // 选择最佳答案
    public function selectBestAnswerAction(int $threadId, array $data): array

    // 查询悬赏状态API
    public function statusAction(int $threadId): array
}
```

**路由配置**:
```php
// Reward Routes
['GET', '/reward/create/:forumId', [RewardController::class, 'showCreateFormAction']],
['POST', '/reward/create/:forumId', [RewardController::class, 'createRewardAction']],
['GET', '/reward/:threadId/answers', [RewardController::class, 'showAnswersAction']],
['POST', '/reward/:threadId/select-answer', [RewardController::class, 'selectBestAnswerAction']],
['GET', '/api/v1/reward/:threadId/status', [RewardController::class, 'statusAction']],
```

#### 子任务 #3.2: 创建悬赏表单UI (1小时)

**文件**: `templates/reward/create.html.twig`

**功能**:
- [ ] 悬赏积分输入
- [ ] 悬赏期限设置
- [ ] 悬赏说明输入
- [ ] 积分余额显示
- [ ] CSRF token集成

**文件**: `templates/reward/answer.html.twig`

**功能**:
- [ ] 显示所有回复列表
- [ ] 选择最佳答案单选框
- [ ] 悬赏状态显示
- [ ] 提交按钮

---

## 🟡 P1 优先级: 附件上传UI优化 (8小时)

### Task #4: 附件上传UI优化 (8小时)

#### 子任务 #4.1: 实现拖拽上传 (3小时)

**文件**: `public/js/attachment-upload.js` (新建)

**功能**:
- [ ] 拖拽区域实现
- [ ] 拖拽事件监听 (dragover, drop, dragleave)
- [ ] 文件类型验证
- [ ] 文件大小验证
- [ ] 视觉反馈（拖拽高亮）

**HTML结构**:
```html
<div id="drop-zone" class="drop-zone">
    <div class="drop-message">
        <i class="fas fa-cloud-upload-alt"></i>
        <p>拖拽文件到此处上传</p>
        <p class="text-muted">或</p>
        <button class="btn btn-primary">选择文件</button>
    </div>
</div>
```

**CSS样式**:
```css
.drop-zone {
    border: 2px dashed #ccc;
    border-radius: 5px;
    padding: 40px;
    text-align: center;
    transition: all 0.3s;
}

.drop-zone.dragover {
    border-color: #007bff;
    background-color: #f0f8ff;
}
```

#### 子任务 #4.2: 实现批量上传 (3小时)

**功能**:
- [ ] 多文件选择 (`<input type="file" multiple>`)
- [ ] 文件队列管理
- [ ] 并发上传控制（最多3个同时）
- [ ] 文件列表显示
- [ ] 单个文件删除

**JavaScript逻辑**:
```javascript
class AttachmentUploader {
    constructor(maxFiles = 10, maxConcurrent = 3) {
        this.queue = [];
        this.uploading = 0;
        this.maxFiles = maxFiles;
        this.maxConcurrent = maxConcurrent;
    }

    addFiles(files) {
        // 添加到队列
        // 触发上传
    }

    uploadNext() {
        // 上传下一个文件
    }

    removeFile(fileId) {
        // 从队列移除
    }
}
```

#### 子任务 #4.3: 实现上传进度条 (2小时)

**功能**:
- [ ] 单个文件进度条
- [ ] 总体进度条
- [ ] 上传速度显示
- [ ] 剩余时间估算
- [ ] 上传完成提示

**HTML结构**:
```html
<div class="upload-progress">
    <div class="progress-item">
        <span class="filename">image.jpg</span>
        <div class="progress">
            <div class="progress-bar" style="width: 45%"></div>
        </div>
        <span class="progress-text">45% (120 KB/s)</span>
    </div>
</div>
```

**AJAX上传**:
```javascript
const formData = new FormData();
formData.append('file', file);
formData.append('_csrf_token', csrfToken);

const xhr = new XMLHttpRequest();
xhr.upload.addEventListener('progress', (e) => {
    const percent = (e.loaded / e.total) * 100;
    updateProgress(percent);
});
xhr.open('POST', '/api/v1/attachments/upload');
xhr.send(formData);
```

---

## 🟢 P2 优先级: 性能优化 (4小时)

### Task #5: 性能优化 (4小时)

#### 子任务 #5.1: BBCode预览缓存 (2小时)

**文件**: `app/BBCode/Services/BBCodeParser.php` (可能需要创建)

**优化策略**:
- [ ] 使用Redis缓存解析后的HTML
- [ ] 缓存键: `bbcode:{content_hash}`
- [ ] 缓存TTL: 1小时
- [ ] 缓存失效: 内容更新时清除

**实现**:
```php
class BBCodeParser
{
    public function parseWithCache(string $bbcode): string
    {
        $hash = md5($bbcode);
        $cacheKey = "bbcode:{$hash}";

        // 尝试从缓存读取
        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        // 解析BBCode
        $html = $this->parse($bbcode);

        // 存入缓存
        $this->cache->set($cacheKey, $html, 3600);

        return $html;
    }
}
```

**测试**: 验证缓存命中率达到80%+

#### 子任务 #5.2: 大文本分块处理 (2小时)

**文件**: `templates/post/new_thread.html.twig`

**优化策略**:
- [ ] 分块加载长文本（每2000行一个块）
- [ ] 虚拟滚动（只渲染可见区域）
- [ ] 延迟加载（滚动到时加载）
- [ ] 使用requestAnimationFrame优化渲染

**JavaScript实现**:
```javascript
class LargeTextHandler {
    constructor(textarea, chunkSize = 2000) {
        this.textarea = textarea;
        this.chunkSize = chunkSize;
    }

    // 分块处理
    processChunks(text) {
        const chunks = [];
        for (let i = 0; i < text.length; i += this.chunkSize) {
            chunks.push(text.substring(i, i + this.chunkSize));
        }
        return chunks;
    }

    // 虚拟滚动
    handleScroll() {
        const scrollTop = this.textarea.scrollTop;
        const visibleChunk = Math.floor(scrollTop / this.chunkSize);
        this.renderChunk(visibleChunk);
    }
}
```

---

## 📁 Week 18 交付文档

### 必须交付的文档 (5份)

1. **`REWARD-THREAD-LEGACY-ANALYSIS.md`** (≥2,000字)
   - Legacy悬赏功能分析
   - 数据库表结构
   - 业务逻辑分析

2. **`REWARD-THREAD-MODERN-DESIGN.md`** (≥1,500字)
   - 现代化设计方案
   - Service层设计
   - Controller层设计

3. **`TASK2-REWARD-SERVICE-COMPLETE.md`** (≥2,000字)
   - RewardService实现报告
   - 测试结果
   - 代码示例

4. **`TASK4-ATTACHMENT-UI-COMPLETE.md`** (≥1,500字)
   - 附件UI优化报告
   - 拖拽上传实现
   - 批量上传实现

5. **`WEEK18-FINAL-SUMMARY.md`** (≥2,000字)
   - Week 18完整总结
   - 任务完成情况
   - Week 19建议

### 可选文档

6. **`PERFORMANCE-OPTIMIZATION-REPORT.md`** (≥1,000字)
   - 性能优化详情
   - 优化前后对比

---

## 📊 Week 18 成功标准

### Reward Thread迁移
- [ ] Legacy悬赏功能100%迁移
- [ ] RewardService实现完整
- [ ] RewardController集成完整
- [ ] 悬赏表单UI完整
- [ ] 30+测试用例通过

### 附件上传UI优化
- [ ] 拖拽上传功能实现
- [ ] 批量上传功能实现（最多10个文件）
- [ ] 上传进度条显示
- [ ] 文件类型和大小验证

### 性能优化
- [ ] BBCode预览缓存实现
- [ ] 缓存命中率达到80%+
- [ ] 大文本分块处理实现

### 文档完整
- [ ] 5份必须文档全部完成
- [ ] TASK-TRACKER.md更新
- [ ] PROGRESS-REPORT.md更新

---

## 🎯 Week 18 后的状态预期

**项目进度**: 82% → 87% (+5%)
**P0 Critical Path**: 100% (保持) ✅
**P1 Core Features**: 60% → 68% (+8%)
**生产就绪度**: 82% → 87% (+5%)

**可以上线**:
- ✅ 内部测试环境
- ✅ Beta测试环境
- ✅ 生产环境（Reward功能完整）

---

## 📅 Week 18 时间线

```
Day 1: Reward Thread分析 + RewardService (6小时)
├── Task #1: Reward Thread分析 (2小时)
├── Task #2: RewardService实现 (4小时)

Day 2: RewardController集成 (2小时)
└── Task #4: 附件上传UI优化 (6小时)
    ├── 拖拽上传 (3小时)
    └── 批量上传 (3小时)

Day 3: 附件上传进度条 (2小时) + 性能优化 (4小时) + 文档 (2小时)
├── Task #4: 上传进度条 (2小时)
├── Task #5: 性能优化 (4小时)
└── 文档整理 (2小时)
```

---

**文档创建**: 2026-02-21
**创建者**: Week 18 规划团队
**状态**: ⏳ 待开始
**版本**: 1.0
