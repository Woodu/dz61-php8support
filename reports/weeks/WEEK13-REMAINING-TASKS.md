# 📋 Week 13 剩余任务 & 下周工作安排

**更新时间**: 2026-02-19
**基于**: 综合代码审查报告
**Week 13进度**: 50% (Day 1-3完成，Day 4-6待完成)

---

## 🎯 本周剩余任务 (Week 13 Day 4-6)

**预计完成**: 2026-02-21 (周四)
**剩余工时**: 18小时
**优先级**: P0 (阻塞发布)

### Day 4: E2E测试修复 (8小时)

**状态**: 🚫 Blocked (命名空间问题)
**预计工时**: 8小时

#### Task 1: 修复E2E测试基类 (2小时)

**问题**: `Class 'Tests\Feature\E2E\E2ETestCase' not found`

**诊断步骤**:
```bash
# 1. 检查E2ETestCase文件位置
find modern-php-migration-code/tests/ -name "E2ETestCase.php"

# 2. 检查命名空间声明
head -20 tests/Feature/E2E/E2ETestCase.php

# 3. 检查composer.json自动加载
grep -A 10 "autoload-dev" composer.json
```

**修复方案** (3选1):

**方案A**: 修改文件位置
```bash
# 移动文件到正确位置
mkdir -p tests/E2E/
mv tests/Feature/E2E/E2ETestCase.php tests/E2E/
mv tests/Feature/E2E/*JourneyTest.php tests/E2E/

# 更新命名空间
sed -i 's/Tests\\Feature\\E2E/Tests\\E2E/g' tests/E2E/*.php
```

**方案B**: 修改命名空间
```bash
# 修改基类命名空间
sed -i 's/namespace Tests\\Feature\\E2E/namespace Tests\\Feature/g' tests/Feature/E2E/E2ETestCase.php

# 修改测试类继承
sed -i 's/extends E2ETestCase/extends \\Tests\\Feature\\E2E\\E2ETestCase/g' tests/Feature/E2E/*JourneyTest.php
```

**方案C**: 修改composer.json
```json
{
  "autoload-dev": {
    "psr-4": {
      "Tests\\": "tests/",
      "Tests\\Feature\\E2E\\": "tests/Feature/E2E/"
    }
  }
}
```

**验收标准**:
- [ ] E2ETestCase类可被自动加载
- [ ] 所有E2E测试类无语法错误
- [ ] `composer dump-autoload`无错误

#### Task 2: 执行E2E测试 (4小时)

**测试场景** (50个):

**Journey 1: 用户注册旅程** (10 scenarios)
```bash
tests/E2E/UserRegistrationJourneyTest.php
- test_visitor_can_access_registration_page
- test_visitor_can_register_with_valid_data
- test_visitor_cannot_register_with_duplicate_username
- test_visitor_cannot_register_with_duplicate_email
- test_visitor_cannot_register_with_weak_password
- test_user_receives_activation_email
- test_user_can_activate_account
- test_user_can_login_after_activation
- test_user_can_view_profile
- test_user_can_logout
```

**Journey 2: 用户登录旅程** (8 scenarios)
```bash
tests/E2E/UserLoginJourneyTest.php
- test_visitor_can_access_login_page
- test_user_can_login_with_valid_credentials
- test_user_cannot_login_with_invalid_username
- test_user_cannot_login_with_invalid_password
- test_user_cannot_login_with_csrf_token_missing
- test_user_session_is_created_after_login
- test_user_can_access_protected_page_after_login
- test_user_can_logout
```

**Journey 3: 发帖旅程** (12 scenarios)
```bash
tests/E2E/PostCreationJourneyTest.php
- test_user_can_access_new_thread_page
- test_user_can_create_thread_with_title_and_content
- test_user_can_create_thread_with_bbcode
- test_user_can_create_thread_with_attachment
- test_user_can_create_poll_thread
- test_user_can_create_paid_thread
- test_user_cannot_create_thread_without_title
- test_user_cannot_create_thread_without_content
- test_user_can_edit_own_thread
- test_user_can_delete_own_thread
- test_user_can_reply_to_thread
- test_user_can_edit_own_reply
```

**Journey 4: 版主管理旅程** (15 scenarios)
```bash
tests/E2E/ModeratorManagementJourneyTest.php
- test_moderator_can_access_moderation_panel
- test_moderator_can_view_pending_threads
- test_moderator_can_approve_thread
- test_moderator_can_delete_thread
- test_moderator_can_move_thread
- test_moderator_can_stick_thread
- test_moderator_can_highlight_thread
- test_moderator_can_close_thread
- test_moderator_can_digest_thread
- test_moderator_can_ban_user
- test_moderator_can_view_moderation_logs
- test_regular_user_cannot_access_moderation_panel
- test_moderator_actions_are_logged
- test_moderator_can_batch_operate_threads
- test_moderator_can_restore_deleted_thread
```

**Journey 5: 附件上传旅程** (10 scenarios)
```bash
tests/E2E/AttachmentUploadJourneyTest.php
- test_user_can_upload_image_attachment
- test_user_can_upload_document_attachment
- test_user_cannot_upload_exe_file
- test_user_cannot_upload_file_exceeding_size_limit
- test_user_can_preview_image_attachment
- test_user_can_download_attachment
- test_user_can_delete_attachment
- test_attachment_thumbnail_is_generated
- test_attachment_counts_towards_user_quota
- test_attachment_is_associated_with_post
```

**执行命令**:
```bash
# 运行所有E2E测试
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/E2E/

# 运行单个Journey测试
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/E2E/UserRegistrationJourneyTest.php

# 生成测试报告
docker exec -i discuz_modern_php php vendor/bin/phpunit tests/E2E/ --testdox > storage/logs/e2e-test-results-$(date +%Y%m%d).md
```

**验收标准**:
- [ ] 50个E2E场景全部执行
- [ ] 通过率≥90%
- [ ] 生成测试报告
- [ ] 失败场景有详细错误日志

#### Task 3: E2E测试报告 (2小时)

**报告内容**:
1. 执行摘要
   - 总场景数: 50
   - 通过数: X/Y
   - 失败数: X/Y
   - 通过率: X%

2. 场景覆盖度
   - 用户注册: 10/10 scenarios
   - 用户登录: 8/8 scenarios
   - 发帖功能: 12/12 scenarios
   - 版主管理: 15/15 scenarios
   - 附件上传: 10/10 scenarios

3. 失败场景分析
   - 场景名称
   - 失败原因
   - 错误堆栈
   - 修复建议

4. 性能指标
   - 平均场景执行时间
   - 最慢场景Top 5
   - 页面加载时间统计

**输出文件**: `storage/logs/e2e-test-results-2026-02-19.md`

---

---

## 📅 下周工作安排 (Week 14)

**开始日期**: 2026-02-24 (周一)
**预计工时**: 50小时 (1周)
**优先级**: P0 (阻塞发布)

### Day 66: CSS回滚到Legacy (8小时)

**目标**: 将现代CSS回滚到Legacy CSS，保持用户体验一致

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 66

### Day 67-68: 发帖表单 (16小时)

**目标**: 用户可以发布新主题

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 67-68

### Day 69: 回复表单 (8小时)

**目标**: 用户可以回复帖子

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 69

### Day 70: 附件上传UI (6小时)

**目标**: 用户可以上传附件

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 70

### Day 71: 投票UI (4小时)

**目标**: 用户可以创建和参与投票

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 71

### Day 72: 支付UI (4小时)

**目标**: 用户可以购买付费内容

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 72

### Day 73: 编辑功能和集成 (8小时)

**目标**: 完善所有表单功能

**详细任务**: 见 `EXECUTION-PLAN-COMPLETE.md` Week 14 Day 73

---

## ✅ Week 13完成标准

### 测试标准
- [ ] E2E测试: 50个场景全部执行，通过率≥90%

### 质量标准
- [ ] E2E测试报告生成
- [ ] Week 13完成报告

---

**文档版本**: 2.0 (已简化)
**创建时间**: 2026-02-19
**预计完成**: 2026-02-21
**责任人**: 开发团队
