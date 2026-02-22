# 📋 Week 14 团队任务清单

**周次**: Week 14 - 质量保证与验证  
**日期范围**: 2026-02-20 ~ 2026-02-26  
**当前项目进度**: 75%  
**目标进度**: 80%  
**团队结构**: 设计团队 | 开发团队 | 测试团队 | Review团队

---

## 🎯 Week 14 总体目标

1. ✅ 修复所有阻塞性测试问题
2. ✅ 完成E2E测试全量验收
3. ✅ 补全未完成的控制器（Payment/Poll/Post）
4. ✅ 修复前端模板问题
5. ✅ 更新项目进度文档

---

## 👥 团队分工

| 团队 | 负责人 | 主要职责 | 预计工时 |
|------|--------|---------|---------|
| **设计团队** | 架构师 | 技术方案设计、问题分析 | 8小时 |
| **开发团队** | 后端开发 | 代码修复、功能补全 | 32小时 |
| **测试团队** | QA工程师 | 测试执行、问题验证 | 24小时 |
| **Review团队** | 技术负责人 | 代码审查、质量把关 | 16小时 |

---

## 📅 Day 1 (2026-02-20): 测试套件修复

### 🎨 设计团队任务

#### Task D1.1: 分析测试失败根因 (2小时)
**负责人**: 架构师  
**优先级**: 🔴 P0

**任务描述**:
分析所有失败的测试，识别根本原因并制定修复方案。

**分析范围**:
1. 集成测试依赖注入问题
2. E2E测试投票表引用问题
3. E2E测试基类缺失问题
4. Controller测试失败（9个）

**交付物**:
- `docs/week14-test-failure-analysis.md`
- 修复方案文档

**验收标准**:
- [ ] 所有测试失败原因已分析
- [ ] 修复方案已制定
- [ ] 技术方案已评审

---

### 💻 开发团队任务

#### Task DEV1.1: 修复集成测试依赖注入 (2小时)
**负责人**: 后端开发  
**优先级**: 🔴 P0

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

#### Task DEV1.2: 修复E2E测试投票表引用 (1小时)
**负责人**: 后端开发  
**优先级**: 🔴 P0

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

#### Task DEV1.3: 创建E2ETestCase基类 (1小时)
**负责人**: 后端开发  
**优先级**: 🔴 P0

**问题详情**:
```
错误: Class "Discuz\Tests\Feature\E2E\E2ETestCase" not found
影响: 18 个特性测试无法运行
```

**实现步骤**:
1. 检查现有E2ETestCase文件位置
2. 确认命名空间是否正确
3. 更新composer.json自动加载（如需要）
4. 验证所有特性测试可以运行

**验收标准**:
- [ ] E2ETestCase类可被自动加载
- [ ] 所有特性测试可以运行
- [ ] 测试数据库隔离正常工作

**相关文件**:
- `tests/Feature/E2E/E2ETestCase.php` (检查现有)
- `composer.json`

---

### 🧪 测试团队任务

#### Task QA1.1: 执行测试套件回归 (3小时)
**负责人**: QA工程师  
**优先级**: 🔴 P0

**执行步骤**:
```bash
# 1. 单元测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/unit/ \
  --testdox \
  > storage/logs/test-results-unit-2026-02-20.txt

# 2. 集成测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/Integration/ \
  --testdox \
  > storage/logs/test-results-integration-2026-02-20.txt

# 3. 特性测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/Feature/ \
  --testdox \
  > storage/logs/test-results-feature-2026-02-20.txt

# 4. E2E 测试
docker exec -i discuz_modern_php php vendor/bin/phpunit \
  tests/E2E/ \
  --testdox \
  > storage/logs/test-results-e2e-2026-02-20.txt
```

**验收标准**:
- [ ] 所有测试套件执行完成
- [ ] 测试结果报告生成
- [ ] 失败测试已记录

**交付物**:
- `storage/logs/test-results-*-2026-02-20.txt`
- 测试失败清单

---

### 📝 Review团队任务

#### Task REV1.1: 代码审查 - 测试修复 (2小时)
**负责人**: 技术负责人  
**优先级**: 🔴 P0

**审查范围**:
1. 集成测试依赖注入修复
2. E2E测试投票表引用修复
3. E2ETestCase基类实现

**审查重点**:
- 代码质量（PSR-12）
- 测试覆盖率
- Legacy兼容性
- 零改表原则合规性

**验收标准**:
- [ ] 所有修复代码已审查
- [ ] 无代码质量问题
- [ ] 符合项目规范

---

## 📅 Day 2 (2026-02-21): 测试执行与分析

### 🎨 设计团队任务

#### Task D2.1: 制定测试修复优先级 (2小时)
**负责人**: 架构师  
**优先级**: 🟡 P1

**任务描述**:
根据测试结果，制定测试修复优先级和计划。

**分析内容**:
1. 失败测试分类（P0/P1/P2）
2. 修复难度评估
3. 修复时间估算
4. 依赖关系分析

**交付物**:
- `docs/week14-test-fix-priority.md`
- 修复计划时间表

---

### 💻 开发团队任务

#### Task DEV2.1: 修复失败的Controller测试 (4小时)
**负责人**: 后端开发  
**优先级**: 🔴 P0

**已知失败测试**:
- AuthController: 3个失败
- RegistrationController: 2个失败
- ModerationController: 0/9 (0%)

**修复步骤**:
1. 分析每个失败测试的原因
2. 修复代码逻辑或测试断言
3. 重新运行测试验证
4. 确保所有测试通过

**验收标准**:
- [ ] AuthController测试通过率 ≥ 95%
- [ ] RegistrationController测试通过率 ≥ 95%
- [ ] ModerationController测试通过率 ≥ 80%

---

#### Task DEV2.2: 补充AttachmentController测试 (2小时)
**负责人**: 后端开发  
**优先级**: 🟡 P1

**任务描述**:
为AttachmentController创建完整的测试套件。

**测试范围**:
- 附件上传
- 附件下载
- 附件删除
- 权限检查
- 文件验证

**验收标准**:
- [ ] AttachmentController测试覆盖率 ≥ 80%
- [ ] 所有测试通过

---

### 🧪 测试团队任务

#### Task QA2.1: 分析测试失败并分类 (3小时)
**负责人**: QA工程师  
**优先级**: 🔴 P0

**任务描述**:
分析所有测试失败，分类并记录详细原因。

**分类标准**:
- **P0 (阻塞)**: 影响核心功能
- **P1 (重要)**: 影响次要功能
- **P2 (可选)**: 不影响功能

**交付物**:
- `storage/logs/test-failures-analysis-2026-02-21.md`
- 失败测试分类清单

---

#### Task QA2.2: E2E测试全量验收 (4小时)
**负责人**: QA工程师  
**优先级**: 🔴 P0

**测试范围**:
- 用户注册旅程 (10 scenarios)
- 用户登录旅程 (8 scenarios)
- 发帖旅程 (12 scenarios)
- 版主管理旅程 (15 scenarios)
- 附件上传旅程 (10 scenarios)

**验收标准**:
- [ ] 所有E2E场景执行完成
- [ ] 通过率 ≥ 90%
- [ ] 失败场景有详细日志

**交付物**:
- `storage/logs/e2e-test-results-2026-02-21.md`
- E2E测试报告

---

### 📝 Review团队任务

#### Task REV2.1: 测试结果审查 (2小时)
**负责人**: 技术负责人  
**优先级**: 🔴 P0

**审查内容**:
1. 测试覆盖率分析
2. 失败测试根因分析
3. 测试质量评估
4. 改进建议

**交付物**:
- `docs/week14-test-review-report.md`

---

## 📅 Day 3 (2026-02-22): 控制器补全

### 🎨 设计团队任务

#### Task D3.1: PaymentController补全设计 (2小时)
**负责人**: 架构师  
**优先级**: 🟡 P1

**任务描述**:
设计PaymentController缺失功能的实现方案。

**缺失功能**:
- 支付历史查询接口
- 支付退款处理
- 支付回调验证增强

**交付物**:
- `docs/week14-payment-controller-design.md`
- API设计文档

---

### 💻 开发团队任务

#### Task DEV3.1: PaymentController补全 (3小时)
**负责人**: 后端开发  
**优先级**: 🟡 P1

**实现内容**:
1. 支付历史查询接口
2. 支付退款处理
3. 支付回调验证增强

**验收标准**:
- [ ] PaymentController功能完整
- [ ] 单元测试覆盖率 ≥ 90%
- [ ] 集成测试通过

---

#### Task DEV3.2: PollController补全 (2小时)
**负责人**: 后端开发  
**优先级**: 🟡 P1

**缺失功能**:
- 投票结果导出
- 投票截止提醒
- 多选投票验证增强

**验收标准**:
- [ ] PollController功能完整
- [ ] 单元测试覆盖率 ≥ 90%
- [ ] 集成测试通过

---

#### Task DEV3.3: PostController补全 (3小时)
**负责人**: 后端开发  
**优先级**: 🟡 P1

**缺失功能**:
- 草稿保存功能
- 自动保存功能
- 编辑历史记录

**验收标准**:
- [ ] PostController功能完整
- [ ] 单元测试覆盖率 ≥ 90%
- [ ] 集成测试通过

---

### 🧪 测试团队任务

#### Task QA3.1: 控制器功能测试 (4小时)
**负责人**: QA工程师  
**优先级**: 🟡 P1

**测试范围**:
- PaymentController新功能
- PollController新功能
- PostController新功能

**验收标准**:
- [ ] 所有新功能测试通过
- [ ] 测试覆盖率达标

---

### 📝 Review团队任务

#### Task REV3.1: 控制器代码审查 (2小时)
**负责人**: 技术负责人  
**优先级**: 🟡 P1

**审查重点**:
- 代码质量
- 安全性
- 性能
- Legacy兼容性

---

## 📅 Day 4 (2026-02-23): 前端修复

### 🎨 设计团队任务

#### Task D4.1: 前端问题分析 (2小时)
**负责人**: 架构师  
**优先级**: 🟡 P1

**问题范围**:
1. BBCode渲染问题
2. 附件显示问题
3. 分页导航问题

**交付物**:
- `docs/week14-frontend-issues-analysis.md`
- 修复方案

---

### 💻 开发团队任务

#### Task DEV4.1: 修复BBCode渲染 (2小时)
**负责人**: 前端开发  
**优先级**: 🟡 P1

**问题描述**:
BBCode渲染不完整，部分标签未正确渲染。

**修复步骤**:
1. 检查BBCode解析器
2. 修复渲染逻辑
3. 添加测试用例
4. 验证渲染结果

**验收标准**:
- [ ] 所有BBCode标签正确渲染
- [ ] 无XSS漏洞
- [ ] 渲染性能正常

---

#### Task DEV4.2: 修复附件显示 (2小时)
**负责人**: 前端开发  
**优先级**: 🟡 P1

**问题描述**:
附件缩略图不显示，附件URL路径错误。

**修复步骤**:
1. 检查附件URL生成逻辑
2. 修复路径问题
3. 验证缩略图生成
4. 测试附件显示

**验收标准**:
- [ ] 附件正常显示
- [ ] 缩略图正常生成
- [ ] 附件下载正常

---

#### Task DEV4.3: 修复分页导航 (1小时)
**负责人**: 前端开发  
**优先级**: 🟡 P1

**问题描述**:
分页链接不工作，查询参数丢失。

**修复步骤**:
1. 检查分页URL生成
2. 修复查询参数处理
3. 验证分页功能

**验收标准**:
- [ ] 分页链接正常工作
- [ ] 查询参数正确传递
- [ ] 分页状态正确显示

---

### 🧪 测试团队任务

#### Task QA4.1: 前端功能测试 (3小时)
**负责人**: QA工程师  
**优先级**: 🟡 P1

**测试范围**:
- BBCode渲染测试
- 附件显示测试
- 分页导航测试
- 浏览器兼容性测试

**验收标准**:
- [ ] 所有前端功能正常
- [ ] 浏览器兼容性通过
- [ ] 响应式设计正常

---

### 📝 Review团队任务

#### Task REV4.1: 前端代码审查 (2小时)
**负责人**: 技术负责人  
**优先级**: 🟡 P1

**审查重点**:
- 代码质量
- 安全性（XSS防护）
- 性能
- 可维护性

---

## 📅 Day 5 (2026-02-24): 代码质量与文档

### 🎨 设计团队任务

#### Task D5.1: 代码质量分析 (2小时)
**负责人**: 架构师  
**优先级**: 🟢 P2

**分析内容**:
1. PSR-12合规性检查
2. 代码复杂度分析
3. 技术债务识别
4. 改进建议

**交付物**:
- `docs/week14-code-quality-report.md`

---

### 💻 开发团队任务

#### Task DEV5.1: 修复PSR-12代码风格 (2小时)
**负责人**: 后端开发  
**优先级**: 🟢 P2

**执行步骤**:
```bash
docker exec -i discuz_modern_php composer lint:fix
```

**验收标准**:
- [ ] 代码风格问题修复完成
- [ ] PSR-12合规性 ≥ 95%

---

### 🧪 测试团队任务

#### Task QA5.1: 测试覆盖率分析 (2小时)
**负责人**: QA工程师  
**优先级**: 🟡 P1

**分析内容**:
1. 单元测试覆盖率
2. 集成测试覆盖率
3. E2E测试覆盖率
4. 覆盖率报告生成

**验收标准**:
- [ ] 覆盖率报告生成
- [ ] 覆盖率 ≥ 85%

---

### 📝 Review团队任务

#### Task REV5.1: 更新项目进度文档 (4小时)
**负责人**: 技术负责人  
**优先级**: 🔴 P0

**需要更新的文件**:
1. `modern-php-migration-code/PROGRESS-REPORT.md`
2. `modern-php-migration-code/TASK-TRACKER.md`
3. `modern-php-migration-code/EXECUTION-PLAN-COMPLETE.md`
4. `modern-php-execution-plan/reports/weeks/WEEK14-COMPLETE.md`

**更新内容**:
- Week 14完成情况
- 测试结果统计
- 代码质量数据
- Week 15准备情况

**验收标准**:
- [ ] 所有进度文档更新完成
- [ ] 文档准确率100%
- [ ] 数据一致性验证通过

---

## 📅 Day 6-7 (2026-02-25 ~ 2026-02-26): Week 14 收尾

### 📝 Review团队任务

#### Task REV6.1: Week 14 验收与总结 (4小时)
**负责人**: 技术负责人  
**优先级**: 🔴 P0

**验收清单**:
```markdown
## Week 14 完成清单

### 测试修复 ✅
- [ ] 集成测试通过率 ≥ 95%
- [ ] E2E 测试通过率 ≥ 90%
- [ ] 所有 P0 测试失败修复

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

### 文档更新 ✅
- [ ] PROGRESS-REPORT.md 更新
- [ ] TASK-TRACKER.md 更新
- [ ] EXECUTION-PLAN-COMPLETE.md 更新
- [ ] WEEK14-COMPLETE.md 生成
```

**交付物**:
- `modern-php-execution-plan/reports/weeks/WEEK14-COMPLETE.md`
- Week 14 总结报告

---

## 📊 每日站会检查点

### Day 1 站会 (2026-02-20)
- [ ] 设计团队：测试失败分析完成
- [ ] 开发团队：集成测试修复进度
- [ ] 测试团队：测试套件执行进度
- [ ] Review团队：代码审查进度
- [ ] 阻塞问题

### Day 2 站会 (2026-02-21)
- [ ] 设计团队：测试修复优先级制定
- [ ] 开发团队：Controller测试修复进度
- [ ] 测试团队：E2E测试验收进度
- [ ] Review团队：测试结果审查进度
- [ ] 阻塞问题

### Day 3 站会 (2026-02-22)
- [ ] 设计团队：PaymentController设计完成
- [ ] 开发团队：控制器补全进度
- [ ] 测试团队：控制器功能测试进度
- [ ] Review团队：控制器代码审查进度
- [ ] 阻塞问题

### Day 4 站会 (2026-02-23)
- [ ] 设计团队：前端问题分析完成
- [ ] 开发团队：前端修复进度
- [ ] 测试团队：前端功能测试进度
- [ ] Review团队：前端代码审查进度
- [ ] 阻塞问题

### Day 5 站会 (2026-02-24)
- [ ] 设计团队：代码质量分析完成
- [ ] 开发团队：代码风格修复进度
- [ ] 测试团队：覆盖率分析进度
- [ ] Review团队：文档更新进度
- [ ] 阻塞问题

---

## 🚨 风险与应对

### 高风险 🔴
1. **测试修复耗时超出预期**
   - 应对: 预留缓冲时间，优先修复P0问题
   - 备选: 延后P1/P2测试修复

2. **E2E测试通过率不达标**
   - 应对: 深入分析失败原因，制定针对性修复方案
   - 备选: 降低通过率要求，延后部分场景

### 中风险 🟡
1. **控制器补全功能复杂度超出预期**
   - 应对: 简化功能实现，分阶段完成
   - 备选: 延后非关键功能

2. **前端修复影响现有功能**
   - 应对: 充分测试，使用特性分支
   - 备选: 回滚修复，重新设计

---

## 📈 成功指标

### 测试指标
- ✅ 集成测试通过率: ≥ 95%
- ✅ E2E测试通过率: ≥ 90%
- ✅ 测试覆盖率: ≥ 85%

### 代码质量指标
- ✅ PSR-12合规性: ≥ 95%
- ✅ 代码审查通过率: 100%
- ✅ 零改表原则合规性: 100%

### 进度指标
- ✅ Week 14完成度: 100%
- ✅ 项目总体进度: 75% → 80%
- ✅ 文档准确率: 100%

---

**文档版本**: 1.0  
**创建时间**: 2026-02-20  
**预计完成**: 2026-02-26  
**责任人**: 各团队负责人
