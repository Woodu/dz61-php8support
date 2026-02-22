# CSS编译系统设计文档

**生成时间**: 2026-02-20  
**状态**: ✅ 已完成

---

## 一、系统概述

Legacy Discuz! 6.1F使用CSS模板文件（.htm）包含变量（如`{BGCODE}`, `{TABLETEXT}`, `{FONTSIZE}`等），这些变量在运行时被替换为实际值，生成编译后的CSS文件。

Modern系统需要：
1. 保持Legacy CSS样式完全不变
2. 支持CSS变量编译
3. 生成静态CSS文件供Twig模板使用

---

## 二、实现方案

### 2.1 架构设计

```
Legacy CSS模板 (.htm)
    ↓
StyleCompilerService (读取模板 + 变量)
    ↓
编译后的CSS文件 (public/css/compiled/)
    ↓
Twig模板引用
```

### 2.2 核心组件

#### StyleCompilerService
- **位置**: `app/Style/Services/StyleCompilerService.php`
- **功能**:
  - 读取Legacy CSS模板文件
  - 加载Legacy style变量（从`style_{STYLEID}.php`缓存文件）
  - 替换CSS变量
  - 生成编译后的CSS文件

#### CSS编译脚本
- **位置**: `scripts/compile-css.php`
- **用法**: `php scripts/compile-css.php [style_id]`
- **功能**: 编译指定style ID的CSS文件

### 2.3 CSS文件类型

Legacy系统支持以下CSS类型：
- `_common`: 通用样式（css_common.htm + css_append.htm）
- `_editor`: 编辑器样式（css_editor.htm）
- `_special`: 特殊主题样式（css_special.htm）
- `_viewthread`: 主题查看样式（css_viewthread.htm）
- `_calendar`: 日历样式（css_calendar.htm）

### 2.4 文件结构

```
modern-php-migration-code/
├── resources/
│   ├── legacy-css/              # Legacy CSS模板文件
│   │   ├── css_common.htm
│   │   ├── css_append.htm
│   │   ├── css_editor.htm
│   │   ├── css_special.htm
│   │   ├── css_viewthread.htm
│   │   └── css_calendar.htm
│   └── legacy-style-cache/      # Legacy style变量缓存
│       └── style_1.php
├── public/
│   └── css/
│       └── compiled/             # 编译后的CSS文件
│           ├── style_1_common.css
│           ├── style_1_editor.css
│           ├── style_1_special.css
│           ├── style_1_viewthread.css
│           └── style_1_calendar.css
└── app/
    └── Style/
        └── Services/
            └── StyleCompilerService.php
```

---

## 三、使用方法

### 3.1 编译CSS

```bash
# 编译style ID为1的CSS
docker exec -i discuz_modern_php php scripts/compile-css.php 1

# 编译所有styles（传入0）
docker exec -i discuz_modern_php php scripts/compile-css.php 0
```

### 3.2 在Twig模板中使用

更新`templates/layouts/base.html.twig`：

```twig
{% block styles %}
{% set styleId = style_id|default(1) %}
<link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_common.css') }}">
{% if css_type is defined %}
    {% if css_type == 'editor' %}
        <link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_editor.css') }}">
    {% elseif css_type == 'viewthread' %}
        <link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_viewthread.css') }}">
        {% if thread_special is defined and thread_special %}
            <link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_special.css') }}">
        {% endif %}
    {% endif %}
{% endif %}
{% endblock %}
```

### 3.3 在Controller中设置CSS类型

```php
// 编辑器页面
return $this->render('post/edit.html.twig', [
    'css_type' => 'editor',
    'style_id' => 1,
    // ... other data
]);

// 主题查看页面
return $this->render('forum/thread.html.twig', [
    'css_type' => 'viewthread',
    'thread_special' => $thread->isSpecial(),
    'style_id' => 1,
    // ... other data
]);
```

---

## 四、CSS变量处理

### 4.1 变量来源

CSS变量从Legacy的`style_{STYLEID}.php`缓存文件中读取，包含：
- `BGCODE`: 背景代码
- `TABLETEXT`: 表格文本颜色
- `FONTSIZE`: 字体大小
- `FONT`: 字体族
- `LINK`: 链接颜色
- `IMGDIR`: 图片目录
- 等等...

### 4.2 变量替换

`StyleCompilerService::replaceVariables()`方法使用正则表达式替换：
- 模式: `/{([A-Z0-9]+)}/`
- 替换为: `$styleVars[strtolower($matches[1])]`

### 4.3 图片路径处理

如果`IMGDIR`不是绝对URL，自动转换为相对于public目录的路径：
- `url("{IMGDIR}/image.gif")` → `url("/images/default/image.gif")`

---

## 五、验证

### 5.1 编译验证

```bash
# 编译style 1
docker exec -i discuz_modern_php php scripts/compile-css.php 1

# 检查生成的文件
ls -la public/css/compiled/style_1_*.css
```

### 5.2 样式验证

1. 访问使用编译后CSS的页面
2. 检查浏览器开发者工具，确认CSS文件加载
3. 对比Legacy系统页面，确认样式一致

---

## 六、后续工作

1. ✅ CSS编译系统已完成
2. ⏳ 迁移P0模板时，确保使用编译后的CSS
3. ⏳ 根据需要编译其他style IDs的CSS
4. ⏳ 考虑添加CSS缓存失效机制（当Legacy CSS模板更新时）

---

## 七、注意事项

1. **保持样式不变**: 编译后的CSS必须与Legacy系统完全一致
2. **变量完整性**: 确保所有CSS变量都被正确替换
3. **路径处理**: 图片路径需要正确转换为Modern系统的路径结构
4. **性能**: 编译后的CSS是静态文件，性能优于运行时编译

---

**文档状态**: ✅ 完成  
**实现状态**: ✅ 已完成并测试通过
