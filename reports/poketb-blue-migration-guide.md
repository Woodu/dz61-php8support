# PokeTB-Blue 风格迁移指南

**日期**: 2026-02-20
**主题**: PokeTB-Blue (Legacy默认风格)
**风格ID**: 1

---

## 风格概述

PokeTB-Blue是Legacy论坛的默认风格，具有以下特征：

### 颜色方案

| 元素 | 颜色值 | 说明 |
|------|--------|------|
| `altbg1` | `#F5FAFE` | 淡蓝色背景（主要特征） |
| `altbg2` | `#F7F7F3` | 灰白色背景 |
| `bordercolor` | `#9DB3C5` | 蓝灰色边框 |
| `catcolor` | `#E8F3FD` | 分类标题背景（淡蓝） |
| `highlightlink` | `#069` | 深蓝色链接 |
| `headermenu` | `menu_bg.gif` | 菜单背景图 |

### 布局参数

- `maintablewidth`: 1000px
- `fontsize`: 12px
- `tablespace`: 1px
- `boxspace`: 10px

---

## 已完成的工作

### 1. PokeTB-Blue CSS文件

**文件**: `public/css/compiled/style_1_poketb_blue.css`

**生成方式**: 从PokeTB模板提取，替换所有样式变量

**关键样式**:
```css
/* PokeTB-Blue特征 - 淡蓝色背景 */
.forumlist tbody th, .forumlist tbody td {
    background-color: #F5FAFE;  /* altbg1 */
}

/* 分类标题淡蓝色背景 */
.mainbox thead.category th {
    background: #E8F3FD url("/images/default/cat_bg.gif");
}

/* 蓝灰色边框 */
border-color: #9DB3C5;
```

### 2. 图片资源

**位置**: `public/images/default/`

**关键图片**:
- `menu_bg.gif` - 菜单背景
- `cat_bg.gif` - 分类标题背景
- `head.gif` - 标题背景
- `portalbox_bg.gif` - 内容框背景

**状态**: ✅ 全部已复制（146个文件）

---

## 使用PokeTB-Blue风格

### 方式1: 直接使用编译后的CSS

在模板中直接引用PokeTB-Blue CSS：

```twig
{% block styles %}
<link rel="stylesheet" href="/css/compiled/style_1_poketb_blue.css">
{% endblock %}
```

### 方式2: 替换现有CSS文件（推荐）

替换当前的style_1_common.css：

```bash
# 备份当前文件
mv public/css/compiled/style_1_common.css \
   public/css/compiled/style_1_common_default.css

# 使用PokeTB-Blue版本
cp public/css/compiled/style_1_poketb_blue.css \
   public/css/compiled/style_1_common.css
```

### 方式3: 创建新的layout模板（推荐）

创建 `templates/layouts/base-poketb.html.twig`:

```twig
{% extends 'layouts/base.html.twig' %}

{% block styles %}
    {# 使用PokeTB-Blue风格 #}
    <link rel="stylesheet" href="{{ asset('css/compiled/style_1_poketb_blue.css') }}">

    {# 额外的PokeTB-Blue特定样式 #}
    <style>
        /* PokeTB-Blue特征: 淡蓝色表格背景 */
        .forumlist tbody th,
        .forumlist tbody td {
            background-color: #F5FAFE !important;
        }

        /* 悬停效果 */
        .forumlist tbody tr:hover th,
        .forumlist tbody tr:hover td {
            background-color: #F7F7F3 !important;
        }
    </style>
{% endblock %}
```

---

## 验证PokeTB-Blue样式

### 检查清单

- [ ] 淡蓝色背景 `#F5FAFE` 应用于表格行
- [ ] 分类标题有淡蓝色背景 `#E8F3FD`
- [ ] 链接是深蓝色 `#069`
- [ ] 边框是蓝灰色 `#9DB3C5`
- [ ] 菜单背景图正确显示
- [ ] Logo正确显示

### 测试命令

```bash
# 访问论坛首页
curl -s http://localhost:8083/ | grep "style_1"

# 检查CSS是否加载
curl -s -I http://localhost:8083/css/compiled/style_1_poketb_blue.css

# 查看页面HTML中的样式表链接
curl -s http://localhost:8083/ | grep -o 'stylesheet[^"]*"[^"]*"'
```

---

## PokeTB-Blue vs Default对比

| 特征 | Default | PokeTB-Blue |
|------|---------|-------------|
| 表格背景 | 白色 | 淡蓝 `#F5FAFE` |
| 边框颜色 | `#CAD9EA` | `#9DB3C5` |
| 分类标题 | `#E8F3FD` | `#E8F3FD` (相同) |
| 链接颜色 | 黑色 `#000` | 黑色 `#000` |
| 高亮链接 | `#069` | `#069` (相同) |

---

## 下一步

1. **测试PokeTB-Blue CSS**
   - 访问论坛首页
   - 验证表格行背景色
   - 检查所有图片加载

2. **调整布局（如需要）**
   - 微调颜色值
   - 调整间距
   - 修复任何显示问题

3. **应用到其他页面**
   - 帖子列表页
   - 帖子详情页
   - 用户资料页

---

## 相关文件

- `public/css/compiled/style_1_poketb_blue.css` - PokeTB-Blue CSS
- `public/css/compiled/style_1_common.css` - 当前使用的CSS
- `public/images/default/` - 图片资源目录
- `poketb.com/bbs/templates/poketb/` - Legacy PokeTB模板

---

**迁移状态**: ✅ CSS已生成，待集成到模板
