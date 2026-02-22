# 📋 Week 15 Legacy模板审查报告

**审查日期**: 2026-02-21  
**审查范围**: Legacy系统发帖相关模板  
**审查目的**: 确保功能完整迁移

---

## 🔍 Legacy模板文件清单

### 核心模板文件

1. **post_newthread.htm** - 新主题表单模板
2. **post_newreply.htm** - 回复表单模板
3. **post_editor.htm** - 编辑器组件（被前两者引用）
4. **post_js.htm** - JavaScript功能（被前两者引用）
5. **post_preview.htm** - 预览功能
6. **post_smilies.htm** - 表情符号选择器

---

## 📊 Legacy功能清单

### post_newthread.htm 功能分析

#### ✅ 已实现功能（Modern系统）

1. **基本表单字段**
   - ✅ 用户名显示（已登录用户）
   - ✅ 主题标题输入（subject）
   - ✅ 内容编辑器（message）
   - ✅ CSRF保护（formhash）

2. **BBCode编辑器**
   - ✅ 工具栏按钮（通过post_editor.htm）
   - ✅ 预览功能（通过post_preview.htm）
   - ✅ 自动保存草稿（restoredata链接）

3. **发帖选项**
   - ✅ 匿名发帖（anonymous）
   - ✅ 订阅主题（subscribe）

#### ⚠️ 缺失功能（Modern系统需要补充）

1. **主题图标选择** ❌
   - Legacy: `<input type="radio" name="iconid" value="0">` + `$icons`
   - Modern: 未实现
   - **位置**: Line 210-212 in Legacy
   - **优先级**: 🔴 P0

2. **投票功能** ❌
   - Legacy: 
     - 投票有效期（expiration）
     - 投票选项（polloptions textarea）
     - 投票前可见（visiblepoll checkbox）
     - 允许多选（multiplepoll checkbox）
     - 最大选择数（maxchoices）
   - Modern: 未实现
   - **位置**: Line 131-144 in Legacy
   - **优先级**: 🔴 P0

3. **支付/付费主题** ❌
   - Legacy: 
     - 价格输入（price）
     - 价格说明（post_price_comment）
   - Modern: 未实现
   - **位置**: Line 199-205 in Legacy
   - **优先级**: 🔴 P0

4. **主题类型选择** ❌
   - Legacy: `$typeselect` (Line 93)
   - Modern: 未实现
   - **优先级**: 🟡 P1

5. **标签功能** ❌
   - Legacy: 
     - 标签输入（tags）
     - 搜索标签按钮（relatekw）
   - Modern: 未实现
   - **位置**: Line 151-159 in Legacy
   - **优先级**: 🟡 P1

6. **高级选项（折叠）** ❌
   - Legacy: "其他信息"折叠区域（adv）
     - 阅读权限（readperm）
     - 价格设置（price）
     - 图标选择（iconid）
   - Modern: 未实现
   - **位置**: Line 173-215 in Legacy
   - **优先级**: 🟡 P1

7. **特殊主题类型** ❌
   - Legacy支持：
     - special=1: 投票主题
     - special=3: 悬赏主题（reward）
     - special=5: 辩论主题（debate）
     - special=6: 视频主题
   - Modern: 未实现
   - **优先级**: 🟢 P2（可延后）

---

### post_newreply.htm 功能分析

#### ✅ 已实现功能（Modern系统）

1. **基本表单字段**
   - ✅ 用户名显示
   - ✅ 回复标题（subject，可选）
   - ✅ 内容编辑器（message）
   - ✅ CSRF保护

2. **主题预览**
   - ✅ 显示原主题内容（post_thread_review）
   - ✅ 显示帖子列表（postlist loop）

3. **BBCode编辑器**
   - ✅ 工具栏按钮
   - ✅ 预览功能
   - ✅ 自动保存草稿

#### ⚠️ 缺失功能（Modern系统需要补充）

1. **引用功能** ⚠️ 部分实现
   - Legacy: 通过post_editor.htm实现
   - Modern: 有引用显示，但缺少点击引用按钮自动插入功能
   - **优先级**: 🔴 P0

2. **回复选项** ❌
   - Legacy: 通过post_editor.htm提供
   - Modern: 未实现
   - **优先级**: 🟡 P1

---

### post_editor.htm 功能分析

**关键功能**（需要审查完整文件）:
- BBCode工具栏
- 附件上传
- 表情符号
- 字数统计
- 自动保存

---

## 🎯 Week 15 迁移任务优先级

### P0 (必须实现)

1. **主题图标选择**
   - 实现位置: `templates/post/new_thread.html.twig`
   - 需要添加: 图标选择UI（radio buttons）
   - 参考: Legacy Line 210-212

2. **投票功能**
   - 实现位置: `templates/post/new_thread.html.twig`
   - 需要添加:
     - 投票有效期输入
     - 投票选项textarea
     - 投票设置（可见性、多选、最大选择数）
   - 参考: Legacy Line 131-144

3. **支付/付费主题**
   - 实现位置: `templates/post/new_thread.html.twig`
   - 需要添加:
     - 价格输入框
     - 价格说明
   - 参考: Legacy Line 199-205

4. **引用功能增强**
   - 实现位置: `templates/post/reply.html.twig`
   - 需要添加: 点击引用按钮自动插入BBCode
   - JavaScript: `public/js/reply-thread.js`

### P1 (重要，可延后)

1. **主题类型选择**
2. **标签功能**
3. **高级选项折叠区域**

### P2 (可选，延后)

1. **特殊主题类型**（投票、悬赏、辩论、视频）

---

## 📝 详细对比表

| 功能 | Legacy位置 | Modern状态 | 优先级 | 备注 |
|------|-----------|-----------|--------|------|
| 主题标题 | Line 90-95 | ✅ 已实现 | - | - |
| 内容编辑器 | post_editor.htm | ✅ 已实现 | - | - |
| BBCode工具栏 | post_editor.htm | ✅ 已实现 | - | - |
| 预览功能 | post_preview.htm | ✅ 已实现 | - | - |
| 自动保存草稿 | Line 222 | ✅ 已实现 | - | restoredata链接 |
| 主题图标 | Line 210-212 | ❌ 缺失 | 🔴 P0 | 需要添加 |
| 投票功能 | Line 131-144 | ❌ 缺失 | 🔴 P0 | 需要添加 |
| 支付功能 | Line 199-205 | ❌ 缺失 | 🔴 P0 | 需要添加 |
| 主题类型 | Line 93 | ❌ 缺失 | 🟡 P1 | 可延后 |
| 标签功能 | Line 151-159 | ❌ 缺失 | 🟡 P1 | 可延后 |
| 高级选项 | Line 173-215 | ❌ 缺失 | 🟡 P1 | 可延后 |
| 引用功能 | post_editor.htm | ⚠️ 部分 | 🔴 P0 | 需要增强 |

---

## 🔧 需要审查的Legacy文件

### 必须审查

1. ✅ `post_newthread.htm` - 已审查
2. ✅ `post_newreply.htm` - 已审查
3. ⏳ `post_editor.htm` - 需要详细审查
4. ⏳ `post_js.htm` - 需要详细审查
5. ⏳ `post_preview.htm` - 需要详细审查
6. ⏳ `post_smilies.htm` - 需要详细审查

### 相关PHP文件

1. ⏳ `include/newthread.inc.php` - 新主题处理逻辑
2. ⏳ `include/newreply.inc.php` - 回复处理逻辑
3. ⏳ `post.php` - 发帖入口文件

---

## 📋 Week 15 任务更新

基于Legacy审查，需要更新Week 15任务清单：

### Day 1 新增任务

- [ ] **Task DEV1.3: 添加主题图标选择** (2小时)
  - 参考Legacy Line 210-212
  - 实现图标radio buttons
  - 图标数据从文件系统加载（icon1.gif ~ icon9.gif）

### Day 2 新增任务

- [ ] **Task DEV2.3: 添加投票功能到新主题表单** (3小时)
  - 参考Legacy Line 131-144
  - 实现投票选项textarea
  - 实现投票设置（有效期、可见性、多选）

### Day 2 新增任务

- [ ] **Task DEV2.4: 添加支付功能到新主题表单** (2小时)
  - 参考Legacy Line 199-205
  - 实现价格输入框
  - 实现价格说明显示

---

**报告生成时间**: 2026-02-21  
**下一步**: 详细审查post_editor.htm和post_js.htm
