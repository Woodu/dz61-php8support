# 前端平行迁移到Twig - 执行进度报告

**日期**: 2026-02-20
**目标**: 将Discuz! 6.1F模板平行迁移到Twig，保持UI/UX完整
**原则**: PHP后端和Twig前端可独立开发

---

## ✅ 已完成任务

### 1. 翻译系统框架

**创建文件**: `app/Twig/TranslationExtension.php`

**功能**:
- ✅ 实现 `|trans` 过滤器
- ✅ 实现 `lang()` 函数
- ✅ 加载Legacy语言文件 (`resources/lang/{locale}/*.lang.php`)
- ✅ 支持参数替换: `{{ 'welcome_user'|trans({user: 'John'}) }}`
- ✅ 多域支持: templates, emails, pms, etc.

**使用方式**:
```twig
{# 过滤器方式 #}
{{ 'login'|trans }}
{{ 'welcome_user'|trans({user: username}) }}

{# 函数方式 #}
{{ lang('login') }}
{{ lang('logout') }}
```

### 2. 静态资源迁移

**CSS文件**:
- ✅ 复制所有Legacy CSS: `style_1_common.css`, `style_1_editor.css`, `style_1_viewthread.css`, `style_1_special.css`, `style_1_calendar.css`
- ✅ 修正图片路径: `../../images/` → `/images/default/`
- ✅ 验证路径替换完成

**图片资源**:
- ✅ 复制Legacy图片: 146个图片文件
- ✅ 位置: `public/images/default/`
- ✅ 包含: logo, icons, 背景图, 按钮等

### 3. 基础组件完善

**Header组件** (`templates/components/header.html.twig`):
- ✅ 保持Legacy HTML结构
- ✅ 支持LOGO变量 `{BOARDLOGO}`
- ✅ 支持广告横幅 `$advlist[headerbanner]`
- ✅ CSS类名完全匹配

**Menu组件** (`templates/components/menu.html.twig`):
- ✅ 用户登录状态显示
- ✅ 记住用户但未激活状态
- ✅ 访客状态
- ✅ 菜单项: PM, 会员列表, 搜索, 用户控制面板, 统计, 管理面板, FAQ
- ✅ 支持active_menu高亮

**Footer组件** (`templates/components/footer.html.twig`):
- ✅ 广告横幅占位
- ✅ 风格切换器
- ✅ 版权信息
- ✅ 调试信息
- ✅ 正确关闭HTML标签

---

## 📋 模板迁移对照表

### 核心布局组件

| Legacy文件 | Twig文件 | 状态 | 优先级 |
|-----------|---------|------|--------|
| header.htm | components/header.html.twig | ✅ 完成 | P0 |
| footer.htm | components/footer.html.twig | ✅ 完成 | P0 |
| (header中) | components/menu.html.twig | ✅ 完成 | P0 |
| - | layouts/base.html.twig | ✅ 已有 | P0 |

### P1 - 核心功能页面 (本周)

| Legacy文件 | Twig文件 | 状态 | 优先级 |
|-----------|---------|------|--------|
| discuz.htm | forum/index.html.twig | ❌ 待创建 | P1 |
| forumdisplay.htm | forum/list.html.twig | ❌ 待创建 | P1 |
| viewthread.htm | thread/view.html.twig | ❌ 待创建 | P1 |
| logging.htm | auth/login.html.twig | ✅ 已有 | P1 |
| register.htm | auth/register.html.twig | ✅ 已有 | P1 |

### P2 - 用户功能 (下周)

| Legacy文件 | Twig文件 | 状态 | 优先级 |
|-----------|---------|------|--------|
| viewprofile.htm | user/profile.html.twig | ❌ 待创建 | P2 |
| memcp.htm | user/controlpanel.html.twig | ❌ 待创建 | P2 |
| memberlist.htm | user/list.html.twig | ❌ 待创建 | P2 |

### P3 - 高级功能 (后续)

| Legacy文件 | Twig文件 | 状态 | 优先级 |
|-----------|---------|------|--------|
| search.htm | search/index.html.twig | ❌ 待创建 | P3 |
| post.htm | post/create.html.twig | ❌ 待创建 | P3 |
| pm.htm | message/index.html.twig | ❌ 待创建 | P3 |

---

## 🎯 下一步任务

### Task 1: 注册Twig TranslationExtension (30分钟)

需要在Twig初始化时注册翻译扩展:

```php
// 在Bootstrap或Twig初始化代码中
use Discuz\Twig\TranslationExtension;

$translationExtension = new TranslationExtension('zh');
$twig->addExtension($translationExtension);
```

### Task 2: 迁移论坛首页 discuz.htm (4小时)

**分析Legacy模板**:
- 使用 `{subtemplate header}` 包含header
- 用户信息区: `#foruminfo #userinfo`
- 统计信息: `#forumstats`
- 公告区: `#announcement` (使用marquee)
- 分类循环: `<!--{loop $catlist}-->`
- 论坛列表: 嵌套循环显示版块
- 友情链接: `#forumlinks`

**创建Twig模板**: `templates/forum/index.html.twig`

### Task 3: 创建辅助组件 (2小时)

需要创建的可重用组件:
- `components/foruminfo.html.twig` - 用户信息和统计
- `components/announcement.html.twig` - 滚动公告
- `components/pagination.html.twig` - 分页
- `components/forum_category.html.twig` - 论坛分类

---

## 📁 目录结构

```
modern-php-migration-code/
├── app/
│   └── Twig/
│       └── TranslationExtension.php    ✅ 新建
├── public/
│   ├── css/compiled/
│   │   ├── style_1_common.css          ✅ 已更新路径
│   │   ├── style_1_editor.css          ✅ 已复制
│   │   ├── style_1_viewthread.css      ✅ 已复制
│   │   ├── style_1_special.css         ✅ 已复制
│   │   └── style_1_calendar.css        ✅ 已复制
│   └── images/default/                 ✅ 146个图片
├── resources/lang/
│   ├── zh/                             ✅ 已有语言文件
│   └── en/                             ✅ 已有语言文件
└── templates/
    ├── layouts/
    │   └── base.html.twig              ✅ 已有
    ├── components/
    │   ├── header.html.twig            ✅ 已完善
    │   ├── menu.html.twig              ✅ 已完善
    │   └── footer.html.twig            ✅ 已完善
    ├── forum/
    │   └── index.html.twig             ❌ 待创建
    └── auth/
        ├── login.html.twig             ✅ 已有
        └── register.html.twig          ✅ 已有
```

---

## 🔧 技术要点

### Twig与Legacy语法对照

| 功能 | Legacy语法 | Twig语法 |
|------|-----------|---------|
| 变量输出 | `$varname` | `{{ varname }}` |
| 条件判断 | `<!--{if}-->` | `{% if %}` |
| 循环 | `<!--{loop}-->` | `{% for %}` |
| 模板包含 | `{subtemplate header}` | `{{ include() }}` |
| 翻译 | `{lang key}` | `{{ 'key'\|trans }}` |
| 注释 | `<!--{注释}-->` | `{# 注释 #}` |
| 默认值 | `$var\|default` | `{{ var\|default('') }}` |

### 数据传递约定

Controller传递给Twig的数据结构:

```php
return $this->twig->render('forum/index.html.twig', [
    // 应用信息
    'app' => [
        'name' => 'PokeTB 口袋社区',
        'version' => '6.1F',
        'url' => 'http://localhost:8083',
    ],

    // 当前用户
    'current_user' => [
        'user_id' => 1,
        'username' => 'admin',
        'admin_id' => 1,
    ],

    // 论坛数据
    'catlist' => $categories,  // 分类列表
    'forumlist' => $forums,     // 版块列表
    'threads' => $threadCount,
    'posts' => $postCount,

    // UI状态
    'active_menu' => 'index',
    'style_id' => 1,
    'debug' => false,
]);
```

---

## ⚠️ 注意事项

1. **保持HTML结构不变**: 所有CSS类名、ID必须与Legacy一致
2. **图片路径统一**: 使用 `{{ asset('default/filename.gif') }}`
3. **翻译字符串优先**: 使用 `{lang key}` 或 `{{ 'key'|trans }}`
4. **链接生成**: 使用 `{{ url('route') }}` 辅助函数
5. **安全输出**: 默认自动转义，需要原始HTML时使用 `|raw`

---

## 📊 进度统计

- **核心组件**: 3/3 (100%)
- **CSS文件**: 5/5 (100%)
- **图片资源**: 146/146 (100%)
- **翻译系统**: 1/1 (100%)
- **页面模板**: 2/221 (1%)

**总体进度**: P0阶段完成，进入P1阶段

---

## 📚 相关文档

- [前端Twig迁移策略](../modern-php-migration-plan/design/frontend-twig-migration-strategy.md)
- [样式问题分析](../puppeteer-test/STYLE-PROBLEM-ANALYSIS.md)
- [样式分析报告](../puppeteer-test/STYLE-ANALYSIS-REPORT.md)

---

**报告生成时间**: 2026-02-20 15:30 UTC
