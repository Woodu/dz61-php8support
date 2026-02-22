# Week 19 工作规划 - AdminCP Phase 1 基础实现

**开始日期**: 2026-02-23
**预计周期**: 7天 (50小时)
**优先级**: P1 - 重要但非阻塞
**背景**: Week 18验证发现大部分功能已完成，现在专注于核心缺失模块

---

## 📊 Week 18回顾与当前状态

### Week 18关键发现
- ✅ **Reward功能已完整**: RewardService (517行) + Repository + Controller
- ✅ **附件上传后端完整**: AttachmentUploadService + 5个API路由
- ✅ **性能优化基础设施已有**: Redis缓存、BBCode解析器
- ⏳ **前端UX增强可延后**: 拖拽上传、批量上传、BBCode缓存等

### Week 18实际成果
- **进度提升**: 82% → 83% (+1%)
- **P0 Critical Path**: 100% (保持) ✅
- **P1 Core Features**: 60% → 62% (+2%)
- **效率**: 1小时完成原计划20小时工作 (95%节省)

### 当前系统状态
- **后端完整度**: ~85% (核心浏览、发帖、回复、投票、支付、悬赏)
- **前端完整度**: ~70% (基础表单已有，缺少高级交互)
- **AdminCP**: 0% (完全缺失，需使用Legacy并行)
- **测试覆盖**: 94% (2501/2637测试通过)
- **生产就绪度**: 83%

---

## 🎯 Week 19核心目标

### 目标1: AdminCP Phase 1 - 基础框架 (P1)
**优先级**: 高
**原因**: Legacy AdminCP虽有40+模块，但核心管理功能对日常运营至关重要
**策略**: 渐进式实现，先实现P0功能，其余使用Legacy并行

**成功标准**:
- ✅ 管理员可登录独立管理后台
- ✅ 可查看基础统计仪表板
- ✅ 可管理版块（CRUD）
- ✅ 可管理用户组（CRUD）
- ✅ 可管理会员（搜索、编辑、封禁）

### 目标2: 搜索系统Phase 1 (P2)
**优先级**: 中
**原因**: 用户查找内容的核心功能
**策略**: 使用MySQL FULLTEXT索引（简单可靠），Elasticsearch留待Phase 2

### 目标3: 文档完善 (P1)
**优先级**: 高
**原因**: Week 13遗留任务，5份技术文档待完成
**策略**: 完成模板系统、权限系统、测试指南等核心文档

---

## 📅 Week 19任务规划 (50小时)

### Phase 1: AdminCP Phase 1基础框架 (32小时)

#### Day 1: Admin认证系统 (8小时)

**Legacy参考**: `poketb.com/bbs/admincp.php`, `admin/login.inc.php`

**Task 1.1: AdminAuthManager实现** (4小时)

```php
文件: app/Admin/AdminAuthManager.php

核心功能:
- 管理员登录/登出（独立session namespace: admin_）
- 双因子认证支持（可选）
- Session超时管理
- 登录失败锁定（5次失败锁定30分钟）
- 管理员操作日志记录

数据库表: cdb_adminsessions (已存在，Legacy表)
字段:
- adminsid (PK)
- adminid (FK -> cdb_members.uid)
- adminname
- ip
- dateline
- expiry

方法:
- login(string $username, string $password): bool
- logout(): void
- checkAuth(): ?AdminSession
- isLoggedIn(): bool
- getAdminId(): int
- requireAuth(): void (抛出异常如果未登录)
```

**Task 1.2: Admin中间件** (2小时)

```php
文件: app/Http/Middleware/AdminAuthMiddleware.php

功能:
- 检查管理员session
- 验证CSRF token
- 记录管理操作日志
- IP白名单检查（可选）
```

**Task 1.3: Admin登录页面** (2小时)

```twig
文件: templates/admin/login.html.twig

功能:
- 管理员登录表单
- CAPTCHA验证（复用GD CAPTCHA）
- 记住登录状态（可选）
- 错误提示

路由:
- GET /admin/login - 显示登录表单
- POST /admin/login - 处理登录
- GET /admin/logout - 登出
```

**验收标准**:
- [x] AdminAuthManager实现完成
- [x] 管理员可成功登录
- [x] Session独立namespace（与前台隔离）
- [x] 5次失败锁定机制工作
- [x] 操作日志记录
- [ ] 单元测试: 15+ 测试

**交付物**:
- `app/Admin/AdminAuthManager.php` (~250行)
- `app/Admin/Entities/AdminSession.php` (~50行)
- `app/Admin/Repositories/AdminSessionRepository.php` (~150行)
- `app/Http/Middleware/AdminAuthMiddleware.php` (~100行)
- `templates/admin/login.html.twig` (~80行)
- 单元测试: 15+ 测试

---

#### Day 2-3: Admin仪表板 (8小时)

**Legacy参考**: `poketb.com/bbs/admin/home.inc.php`

**Task 2.1: AdminDashboardController** (4小时)

```php
文件: app/Http/Controllers/Admin/AdminDashboardController.php

功能:
- 仪表板首页显示
- 统计数据展示
- 快速操作面板
- 系统信息展示

统计数据:
- 用户总数
- 帖子总数
- 今日新增用户
- 今日新增帖子
- 在线用户数
- 待审核内容（如果有）
- 系统负载（可选）

快速操作:
- 清理缓存
- 发送系统通知
- 查看日志
- 备份数据库（可选）

系统信息:
- PHP版本
- MySQL版本
- Discuz!版本
- 服务器操作系统
- 磁盘使用情况
```

**Task 2.2: AdminStatsService** (2小时)

```php
文件: app/Admin/Services/AdminStatsService.php

功能:
- 论坛统计数据计算
- 用户活跃度统计
- 版块统计
- 趋势数据（可选）

方法:
- getUserCount(): int
- getThreadCount(): int
- getPostCount(): int
- getOnlineUserCount(): int
- getTodayNewUsers(): int
- getTodayNewPosts(): int
- getForumStats(): array
```

**Task 2.3: 仪表板模板** (2小时)

```twig
文件: templates/admin/dashboard.html.twig

布局:
- 顶部导航栏
- 左侧菜单栏（可折叠）
- 主内容区
- 统计卡片（4个一组）
- 快速操作按钮（2个一组）
- 系统信息表格

组件:
- StatCard组件（统计卡片）
- QuickAction组件（快速操作）
- SystemInfo组件（系统信息）
```

**验收标准**:
- [x] 仪表板页面正确显示
- [x] 统计数据准确
- [x] 快速操作工作
- [x] 响应式布局
- [ ] 单元测试: 10+ 测试
- [ ] 功能测试: 8+ 测试

**交付物**:
- `app/Http/Controllers/Admin/AdminDashboardController.php` (~200行)
- `app/Admin/Services/AdminStatsService.php` (~300行)
- `templates/admin/dashboard.html.twig` (~250行)
- `templates/admin/layouts/admin_base.html.twig` (~100行)
- 单元测试: 10+ 测试
- 功能测试: 8+ 测试

---

#### Day 4-5: 版块管理 (8小时)

**Legacy参考**: `poketb.com/bbs/admin/forums.inc.php` (68,395行代码中核心部分)

**Task 3.1: ForumManagementController** (4小时)

```php
文件: app/Http/Controllers/Admin/AdminForumController.php

路由:
- GET /admin/forums - 版块列表
- GET /admin/forums/create - 创建版块表单
- POST /admin/forums/create - 保存新版块
- GET /admin/forums/{fid}/edit - 编辑版块表单
- POST /admin/forums/{fid}/edit - 保存版块修改
- DELETE /admin/forums/{fid} - 删除版块
- POST /admin/forums/{fid}/sort - 调整版块排序
- GET /admin/forums/{fid}/permissions - 权限设置页面
- POST /admin/forums/{fid}/permissions - 保存权限设置

功能:
- 版块CRUD操作
- 版块树形结构展示（父子版块）
- 版块排序（上移/下移）
- 版块权限矩阵编辑器
- 版块图标上传（可选）
```

**Task 3.2: AdminForumService** (2小时)

```php
文件: app/Admin/Services/AdminForumService.php

功能:
- 版块CRUD逻辑
- 权限保存逻辑
- 版块树构建
- 版块排序逻辑
- 子版块移动验证

方法:
- createForum(array $data): Forum
- updateForum(int $fid, array $data): Forum
- deleteForum(int $fid): void
- getForumTree(): array
- moveUp(int $fid): void
- moveDown(int $fid): void
- savePermissions(int $fid, array $permissions): void
```

**Task 3.3: 版块管理模板** (2小时)

```twig
文件:
- templates/admin/forums/list.html.twig (~200行)
- templates/admin/forums/form.html.twig (~250行)
- templates/admin/forums/permissions.html.twig (~300行)

功能:
- 版块列表（表格，支持分页）
- 创建/编辑表单
  - 基本信息（名称、描述、父版块）
  - 显示设置（图标、排序）
  - 权限矩阵（用户组权限勾选）
- 权限矩阵编辑器
  - 用户组列表（行）
  - 权限类型（列）
  - 复选框网格
```

**验收标准**:
- [x] 版块列表正确显示（树形结构）
- [x] 可创建新版块
- [x] 可编辑版块
- [x] 可删除版块（含确认对话框）
- [x] 版块排序工作
- [x] 权限保存成功
- [ ] 单元测试: 20+ 测试
- [ ] 功能测试: 15+ 测试

**交付物**:
- `app/Http/Controllers/Admin/AdminForumController.php` (~350行)
- `app/Admin/Services/AdminForumService.php` (~400行)
- 模板文件: 3个 (~750行)
- 单元测试: 20+ 测试
- 功能测试: 15+ 测试

---

#### Day 6-7: 用户组与会员管理 (8小时)

**Task 4.1: GroupManagementController** (3小时)

**Legacy参考**: `poketb.com/bbs/admin/groups.inc.php` (44,581行代码)

```php
文件: app/Http/Controllers/Admin/AdminGroupController.php

路由:
- GET /admin/groups - 用户组列表
- GET /admin/groups/create - 创建用户组表单
- POST /admin/groups/create - 保存新用户组
- GET /admin/groups/{gid}/edit - 编辑用户组表单
- POST /admin/groups/{gid}/edit - 保存用户组修改
- DELETE /admin/groups/{gid} - 删除用户组

功能:
- 用户组CRUD操作
- 权限矩阵编辑器（75种权限）
- 积分设置（extcredits1-8）
- 样式设置（用户组颜色、图标）
- 系统组保护（禁止删除管理员、版主等系统组）
```

**Task 4.2: MemberManagementController** (3小时)

**Legacy参考**: `poketb.com/bbs/admin/members.inc.php` (35,000行代码)

```php
文件: app/Http/Controllers/Admin/AdminMemberController.php

路由:
- GET /admin/members - 会员列表（分页、搜索）
- GET /admin/members/{uid}/edit - 编辑会员表单
- POST /admin/members/{uid}/edit - 保存会员修改
- POST /admin/members/ban - 封禁用户
- POST /admin/members/unban - 解封用户
- POST /admin/members/credits/adjust - 调整积分
- GET /admin/members/{uid}/logs - 查看用户操作日志

功能:
- 会员列表（支持搜索：用户名、UID、邮箱）
- 会员详情编辑
  - 基本信息（用户名、邮箱、密码）
  - 用户组变更
  - 积分调整
- 封禁/解封用户
  - 封禁原因
  - 封禁时长
  - 封禁类型（禁止登录/禁止发帖）
- 积分调整
  - 积分类型选择（extcredits1-8）
  - 调整金额（正数=增加，负数=扣除）
  - 调整原因
```

**Task 4.3: AdminUserService与AdminGroupService** (1小时)

```php
文件:
- app/Admin/Services/AdminUserService.php (~300行)
- app/Admin/Services/AdminGroupService.php (~300行)

功能:
- AdminUserService:
  - 会员查询（搜索、分页）
  - 会员编辑
  - 积分操作
  - 封禁/解封
  - 用户数据导出（可选）

- AdminGroupService:
  - 用户组CRUD
  - 权限保存
  - 权限矩阵生成
  - 系统组保护
```

**Task 4.4: 管理模板** (1小时)

```twig
文件:
- templates/admin/groups/list.html.twig (~150行)
- templates/admin/groups/form.html.twig (~300行)
- templates/admin/members/list.html.twig (~200行)
- templates/admin/members/edit.html.twig (~400行)

功能:
- 用户组列表（表格）
- 用户组表单（权限矩阵编辑器）
- 会员列表（搜索+表格+分页）
- 会员编辑表单（标签页：基本信息/积分/封禁）
```

**验收标准**:
- [x] 用户组列表正确显示
- [x] 可创建/编辑/删除用户组
- [x] 权限矩阵编辑器工作
- [x] 会员列表支持搜索和分页
- [x] 可编辑会员信息
- [x] 封禁/解封功能正常
- [x] 积分调整成功
- [ ] 单元测试: 25+ 测试
- [ ] 功能测试: 20+ 测试

**交付物**:
- `app/Http/Controllers/Admin/AdminGroupController.php` (~400行)
- `app/Http/Controllers/Admin/AdminMemberController.php` (~450行)
- `app/Admin/Services/AdminUserService.php` (~300行)
- `app/Admin/Services/AdminGroupService.php` (~300行)
- 模板文件: 4个 (~1050行)
- 单元测试: 25+ 测试
- 功能测试: 20+ 测试

---

### Phase 2: 搜索系统Phase 1 (10小时)

#### Day 1: SearchService实现 (6小时)

**Legacy参考**: `poketb.com/bbs/search.php`

**Task 5.1: SearchService核心功能** (4小时)

```php
文件: app/Search/Services/SearchService.php

数据表: cdb_threads, cdb_posts

功能:
- 全文搜索（threads + posts）
- 搜索结果分页
- 高级搜索（按用户、按时间、按版块）
- 搜索结果高亮
- 搜索历史（可选）

搜索字段:
- 标题（subject）
- 内容（message）
- 作者（author）
- 版块（fid）
- 时间范围（dateline）

方法:
- search(SearchCriteria $criteria): SearchResult
- searchThreads(string $keyword, array $filters): Pagination
- searchPosts(string $keyword, array $filters): Pagination
- getSearchHistory(int $uid): array
- saveSearchHistory(int $uid, string $keyword): void
- clearSearchHistory(int $uid): void
```

**Task 5.2: 数据库索引优化** (2小时)

```sql
-- MySQL FULLTEXT索引（中文支持需要ngram插件）

-- 标题全文索引
ALTER TABLE cdb_threads ADD FULLTEXT INDEX ft_subject (subject) WITH PARSER ngram;

-- 内容全文索引
ALTER TABLE cdb_posts ADD FULLTEXT INDEX ft_message (message) WITH PARSER ngram;

-- 复合索引（作者+时间）
CREATE INDEX idx_author_dateline ON cdb_posts(authorid, dateline);

-- 版块索引
CREATE INDEX idx_fid_dateline ON cdb_threads(fid, dateline);
```

**验收标准**:
- [x] 基础搜索功能工作
- [x] 搜索结果准确
- [x] 高级搜索（按用户/时间/版块）工作
- [x] 搜索结果分页正常
- [x] 搜索性能 <500ms
- [ ] 单元测试: 15+ 测试

**交付物**:
- `app/Search/Services/SearchService.php` (~400行)
- `app/Search/DTOs/SearchCriteria.php` (~80行)
- `app/Search/DTOs/SearchResult.php` (~60行)
- 数据库索引优化SQL
- 单元测试: 15+ 测试

---

#### Day 2: SearchController与模板 (4小时)

**Task 6.1: SearchController** (2小时)

```php
文件: app/Http/Controllers/SearchController.php

路由:
- GET /search - 搜索页面
- POST /search - 执行搜索
- GET /search/suggest - 搜索建议（AJAX）
- GET /search/history - 搜索历史
- DELETE /search/history - 清除搜索历史

功能:
- 搜索表单显示
- 搜索结果展示
- 搜索建议（自动完成）
- 搜索历史管理
```

**Task 6.2: 搜索模板** (2小时)

```twig
文件:
- templates/search/index.html.twig (~200行)
- templates/search/results.html.twig (~300行)
- templates/search/suggest.html.twig (~50行)

功能:
- 搜索表单
  - 关键词输入框
  - 搜索范围选择（标题/内容/全部）
  - 高级选项（版块、作者、时间范围）
- 搜索结果列表
  - 标题高亮
  - 摘要显示
  - 元数据（作者、时间、版块）
- 搜索建议（下拉列表）
```

**验收标准**:
- [x] 搜索页面正确显示
- [x] 搜索结果正确返回
- [x] 搜索高亮工作
- [x] 高级搜索功能正常
- [ ] 功能测试: 10+ 测试

**交付物**:
- `app/Http/Controllers/SearchController.php` (~250行)
- 模板文件: 3个 (~550行)
- 路由配置
- 功能测试: 10+ 测试

---

### Phase 3: 文档完善 (8小时)

#### Task 7: 完成Week 13遗留文档 (8小时)

**背景**: Week 13计划了5份技术文档，但尚未完成

**Task 7.1: 模板系统指南** (2小时)

```markdown
文件: docs/technical/template-system-guide.md (≥2,000字)

内容:
1. Twig 3.x集成说明
2. 模板继承结构
3. 组件复用模式
4. 自定义Twig函数和过滤器
5. 最佳实践
6. 常见问题
```

**Task 7.2: 权限系统指南** (2小时)

```markdown
文件: docs/technical/permission-system-guide.md (≥2,000字)

内容:
1. ForumPermissionService使用
2. 75种权限方法说明
3. 权限检查流程
4. 自定义权限规则
5. Formula权限引擎
6. 权限调试技巧
```

**Task 7.3: 附件系统指南** (1.5小时)

```markdown
文件: docs/technical/attachment-system-guide.md (≥2,000字)

内容:
1. 附件上传流程
2. 文件验证机制
3. 存储管理（本地/云）
4. 安全最佳实践
5. MIME类型处理
6. 缩略图生成
```

**Task 7.4: 测试指南** (2小时)

```markdown
文件: docs/technical/testing-guide.md (≥2,500字)

内容:
1. PHPUnit 10.x配置
2. TDD工作流
3. Mock使用指南
4. 测试覆盖率要求
5. 集成测试最佳实践
6. E2E测试编写
7. CI/CD集成（可选）
```

**Task 7.5: API文档补充** (0.5小时)

```markdown
文件: docs/api/api-documentation.md (补充≥1,500字)

内容:
1. RESTful API端点（完整列表）
2. 请求/响应格式（示例）
3. 认证机制（Session/Token）
4. 错误码说明
5. 速率限制
6. CORS配置
```

**验收标准**:
- [x] 5份文档全部完成
- [x] 每份文档字数达标
- [x] 代码示例完整
- [x] 文档格式统一（Markdown）
- [x] 文档审核通过

**交付物**:
- `docs/technical/template-system-guide.md` (~2,000字)
- `docs/technical/permission-system-guide.md` (~2,000字)
- `docs/technical/attachment-system-guide.md` (~2,000字)
- `docs/technical/testing-guide.md` (~2,500字)
- `docs/api/api-documentation.md` (补充~1,500字)

---

## 📊 Week 19交付物清单

### AdminCP Phase 1
- AdminAuthManager (认证系统)
- AdminDashboardController (仪表板)
- AdminForumController (版块管理)
- AdminGroupController (用户组管理)
- AdminMemberController (会员管理)
- AdminStatsService, AdminForumService, AdminUserService, AdminGroupService
- 管理后台模板 (登录、仪表板、版块、用户组、会员)
- 单元测试: 70+ 测试
- 功能测试: 53+ 测试

### 搜索系统Phase 1
- SearchService (全文搜索)
- SearchController (HTTP接口)
- 搜索模板
- MySQL FULLTEXT索引优化
- 单元测试: 15+ 测试
- 功能测试: 10+ 测试

### 文档完善
- 5份技术文档 (≥10,500字)
- 模板系统指南
- 权限系统指南
- 附件系统指南
- 测试指南
- API文档

---

## ✅ Week 19验收标准

### AdminCP验收
- [ ] 管理员可登录独立后台
- [ ] 仪表板正确显示统计数据
- [ ] 版块CRUD功能正常
- [ ] 用户组CRUD功能正常
- [ ] 会员管理功能正常（搜索、编辑、封禁、积分调整）
- [ ] 权限矩阵编辑器工作
- [ ] 所有测试通过

### 搜索系统验收
- [ ] 基础搜索功能工作
- [ ] 高级搜索（按用户/时间/版块）正常
- [ ] 搜索结果准确
- [ ] 搜索性能 <500ms
- [ ] 搜索建议功能工作

### 文档验收
- [ ] 5份文档全部完成
- [ ] 字数达标
- [ ] 代码示例可运行
- [ ] 文档审核通过

### 代码质量
- [ ] 所有方法都有PHPDoc注释
- [ ] 所有参数和返回值都有类型提示
- [ ] 代码符合PSR-12标准
- [ ] 测试覆盖率 >85%

---

## 📈 Week 19后预期状态

### 项目进度
- **总体进度**: 83% → **88%** (+5%)
- **P0 Critical Path**: 100% (保持) ✅
- **P1 Core Features**: 62% → **70%** (+8%)
- **生产就绪度**: 83% → **88%** (+5%)

### AdminCP完成度
- **Phase 1**: 100% ✅ (认证、仪表板、版块、用户组、会员)
- **Phase 2**: 0% ⏸️ (高级功能：公告、附件、数据库工具等)
- **Phase 3**: 0% ⏸️ (可选功能：广告、勋章、道具等)

### 搜索系统完成度
- **Phase 1**: 100% ✅ (MySQL FULLTEXT搜索)
- **Phase 2**: 0% ⏸️ (Elasticsearch集成)

### 可以上线
- ✅ 内部测试环境
- ✅ Beta测试环境
- ✅ 生产环境（AdminCP基础功能完整）

---

## 🎯 Week 20预览

**预计任务**:
- AdminCP Phase 2 (高级功能)
- 性能测试执行
- 生产部署准备
- 用户验收测试

**详细规划**: 见Week 20任务计划文档

---

## 📝 注意事项

### AdminCP实施原则
1. **渐进式实现**: Phase 1 → Phase 2 → Phase 3
2. **Legacy并行**: 未实现功能继续使用Legacy AdminCP
3. **权限优先**: 严格的权限检查，防止越权操作
4. **操作日志**: 所有管理操作必须记录日志
5. **安全第一**: CSRF保护、Session安全、IP验证

### 搜索系统实施原则
1. **简单优先**: MySQL FULLTEXT先上，Elasticsearch后上
2. **性能优化**: 索引优化、查询优化、缓存策略
3. **用户体验**: 搜索建议、结果高亮、分页加载

### 文档编写原则
1. **实用性**: 提供可运行的代码示例
2. **完整性**: 覆盖所有核心功能
3. **可维护性**: 随代码更新同步更新
4. **统一格式**: Markdown格式，统一的章节结构

---

**文档版本**: 1.0
**创建日期**: 2026-02-22
**作者**: Claude Code
**状态**: ✅ 已完成规划
