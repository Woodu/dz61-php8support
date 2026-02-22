# Legacy系统模板编译和CSS生成机制分析

**生成时间**: 2026-02-20  
**审查范围**: Legacy Discuz! 6.1F 模板编译、CSS生成、缓存刷新机制

---

## 一、Legacy系统机制概述

Legacy系统有一套完整的模板编译和CSS生成机制：

1. **模板编译**: `.htm` → `.tpl.php` (PHP文件)
2. **CSS编译**: CSS模板 `.htm` + 变量 → 静态 `.css` 文件
3. **自动刷新**: 检查文件修改时间，自动重新编译
4. **缓存删除**: 删除编译文件触发重新编译

---

## 二、模板编译机制

### 2.1 编译流程

```
模板文件 (.htm)
    ↓
parse_template() 函数
    ↓
编译后的PHP文件 (.tpl.php)
    ↓
template() 函数加载
    ↓
checktplrefresh() 检查是否需要重新编译
```

### 2.2 核心函数

#### `parse_template($tplfile, $templateid, $tpldir)`
**位置**: `include/template.func.php:14`

**功能**:
- 读取 `.htm` 模板文件
- 转换模板语法为PHP代码：
  - `{subtemplate header}` → 加载子模板
  - `{if $var}` → `<? if($var) { ?>`
  - `{loop $array $item}` → `<? foreach($array as $item) { ?>`
  - `{lang key}` → 语言变量
  - `{$var}` → `<?=$var?>`
- 生成编译后的 `.tpl.php` 文件
- 在文件头部插入 `checktplrefresh()` 检查代码

**输出位置**: `forumdata/templates/{templateid}_{filename}.tpl.php`

**示例**:
```php
// 编译后的文件头部
<? if(!defined('IN_DISCUZ')) exit('Access Denied'); 
0
|| checktplrefresh('/path/to/discuz.htm', '/path/to/header.htm', 1770914091, '1', './templates/default')
|| checktplrefresh('/path/to/discuz.htm', '/path/to/footer.htm', 1770914091, '1', './templates/default')
;?>
```

#### `checktplrefresh($maintpl, $subtpl, $timecompare, $templateid, $tpldir)`
**位置**: `include/global.func.php:106`

**功能**:
- 检查子模板文件修改时间是否比编译文件新
- 如果 `$tplrefresh == 1`，总是重新编译
- 如果 `$tplrefresh > 1`，按时间间隔检查
- 如果子模板更新，调用 `parse_template()` 重新编译主模板

**逻辑**:
```php
function checktplrefresh($maintpl, $subtpl, $timecompare, $templateid, $tpldir) {
    global $tplrefresh;
    // 如果timecompare为空（文件不存在）或tplrefresh=1，总是重新编译
    if(empty($timecompare) || $tplrefresh == 1 || ($tplrefresh > 1 && !($GLOBALS['timestamp'] % $tplrefresh))) {
        // 如果子模板比编译文件新，重新编译
        if(empty($timecompare) || @filemtime($subtpl) > $timecompare) {
            require_once DISCUZ_ROOT.'./include/template.func.php';
            parse_template($maintpl, $templateid, $tpldir);
            return TRUE;
        }
    }
    return FALSE;
}
```

#### `template($file, $templateid, $tpldir)`
**位置**: `include/global.func.php:940`

**功能**:
- 加载模板文件
- 调用 `checktplrefresh()` 检查是否需要重新编译
- 返回编译后的 `.tpl.php` 文件路径

**逻辑**:
```php
function template($file, $templateid = 0, $tpldir = '') {
    $tplfile = DISCUZ_ROOT.'./'.$tpldir.'/'.$file.'.htm';
    $objfile = DISCUZ_ROOT.'./forumdata/templates/'.$templateid.'_'.$file.'.tpl.php';
    
    // 检查是否需要重新编译
    @checktplrefresh($tplfile, $tplfile, filemtime($objfile), $templateid, $tpldir);
    
    return $objfile;
}
```

### 2.3 触发重新编译的方式

1. **删除编译文件**: 删除 `forumdata/templates/{templateid}_{file}.tpl.php`，下次访问时自动重新编译
2. **修改模板文件**: 修改 `.htm` 文件，`checktplrefresh()` 检测到文件更新，自动重新编译
3. **设置 `$tplrefresh = 1`**: 每次访问都重新编译（开发模式）
4. **调用 `updatecache('styles')`**: 更新样式缓存时，会重新编译CSS

---

## 三、CSS编译机制

### 3.1 编译流程

```
CSS模板文件 (.htm)
    ↓
writetocsscache() 函数
    ↓
读取CSS模板 + 替换变量
    ↓
生成静态CSS文件 (.css)
```

### 3.2 核心函数

#### `writetocsscache($data)`
**位置**: `include/cache.func.php:266`

**功能**:
- 读取CSS模板文件（`css_common.htm`, `css_editor.htm` 等）
- 替换CSS变量（`{BGCODE}`, `{TABLETEXT}` 等）
- 生成编译后的CSS文件

**CSS类型**:
- `_common`: `css_common.htm` + `css_append.htm`
- `_editor`: `css_editor.htm`
- `_special`: `css_special.htm`
- `_viewthread`: `css_viewthread.htm`
- `_calendar`: `css_calendar.htm`

**输出位置**: `forumdata/cache/style_{styleid}_{type}.css`

**变量替换**:
```php
// 替换 {VARIABLE} 为实际值
$cssdata = preg_replace("/\{([A-Z0-9]+)\}/e", '\$data[strtolower(\'\1\')]', $cssdata);
```

### 3.3 CSS编译触发

#### `updatecache('styles')`
**位置**: `include/cache.func.php:79`

**功能**:
- 从数据库读取style变量
- 调用 `writetocsscache()` 生成CSS文件
- 调用 `writetocache()` 生成style变量PHP文件

**调用时机**:
- 管理员在后台修改样式设置
- 手动调用 `updatecache('styles')`
- 删除 `forumdata/cache/style_{styleid}.php` 后，下次访问自动重新生成

### 3.4 CSS文件引用

**在模板中引用**:
```html
<!-- header.htm -->
<link rel="stylesheet" type="text/css" href="forumdata/cache/style_{STYLEID}_common.css" />
<!--{if CURSCRIPT == 'viewthread'}-->
    <link rel="stylesheet" type="text/css" href="forumdata/cache/style_{STYLEID}_viewthread.css" />
<!--{/if}-->
<!--{if CURSCRIPT == 'post'}-->
    <link rel="stylesheet" type="text/css" href="forumdata/cache/style_{STYLEID}_editor.css" />
<!--{/if}-->
```

---

## 四、缓存刷新机制

### 4.1 模板刷新机制

**检查逻辑**:
1. 编译后的 `.tpl.php` 文件头部包含 `checktplrefresh()` 调用
2. 每次加载模板时，检查子模板文件修改时间
3. 如果子模板更新，自动重新编译主模板

**刷新条件**:
- `$tplrefresh == 1`: 总是重新编译（开发模式）
- `$tplrefresh > 1`: 按时间间隔检查（如每N次请求）
- 子模板文件修改时间 > 编译文件修改时间
- 编译文件不存在（`filemtime()` 返回false）

### 4.2 CSS刷新机制

**检查逻辑**:
- CSS文件是静态文件，不自动检查更新
- 需要手动调用 `updatecache('styles')` 重新生成
- 删除CSS文件不会自动重新生成（需要调用 `updatecache()`）

**刷新方式**:
1. 删除 `forumdata/cache/style_{styleid}.php` → 下次访问自动重新生成（包括CSS）
2. 调用 `updatecache('styles')` → 立即重新生成
3. 修改CSS模板文件 → 需要手动调用 `updatecache('styles')`

---

## 五、Modern系统需要实现的机制

### 5.1 模板编译机制（Twig不需要）

**说明**: Modern系统使用Twig模板引擎，不需要编译为PHP文件。Twig会自动处理模板缓存。

**但需要实现**:
- 模板文件修改检测
- 自动清除Twig缓存

### 5.2 CSS编译机制（需要实现）

**已实现**:
- ✅ `StyleCompilerService` - CSS编译服务
- ✅ `scripts/compile-css.php` - 编译脚本

**需要增强**:
- ⏳ 自动检测CSS模板文件修改
- ⏳ 删除编译CSS文件触发重新编译
- ⏳ 在Twig模板中自动引用编译后的CSS

### 5.3 缓存刷新机制（需要实现）

**需要实现**:
1. **CSS模板修改检测**:
   - 检查 `resources/legacy-css/*.htm` 文件修改时间
   - 如果比编译后的CSS文件新，自动重新编译

2. **删除缓存触发重新编译**:
   - 删除 `public/css/compiled/style_{styleid}_{type}.css`
   - 下次访问时自动重新编译

3. **Twig模板中的自动引用**:
   - 根据页面类型自动加载对应的CSS文件
   - 检查CSS文件是否存在，不存在则自动编译

---

## 六、Modern系统实现方案

### 6.1 CSS编译增强

#### 方案1: 运行时自动编译（推荐）

在Twig模板加载时，检查CSS文件是否存在或需要更新：

```php
// 在Controller或Middleware中
public function beforeRender($template, $data) {
    $styleId = $data['style_id'] ?? 1;
    $cssType = $data['css_type'] ?? 'common';
    
    $compiler = new StyleCompilerService(...);
    
    // 检查CSS文件是否存在
    if (!$compiler->compiledCssExists($styleId, $cssType)) {
        $compiler->compileStyle($styleId);
    }
    
    // 检查CSS模板是否更新
    if ($compiler->isCssTemplateUpdated($styleId, $cssType)) {
        $compiler->compileStyle($styleId);
    }
}
```

#### 方案2: 命令行编译 + 手动触发

保持当前实现，但添加：
- 检测CSS模板修改时间的命令
- 删除缓存后自动重新编译的机制

### 6.2 Twig模板自动引用

在 `base.html.twig` 中实现自动CSS加载：

```twig
{% block styles %}
{% set styleId = style_id|default(1) %}
{% set cssType = css_type|default('common') %}

{# 确保CSS文件存在 #}
{% if not css_compiled[styleId][cssType] %}
    {# 触发编译 #}
    {{ compile_css(styleId, cssType) }}
{% endif %}

<link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_common.css') }}">
{% if cssType == 'editor' %}
    <link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_editor.css') }}">
{% elseif cssType == 'viewthread' %}
    <link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_viewthread.css') }}">
    {% if thread_special is defined and thread_special %}
        <link rel="stylesheet" href="{{ asset('css/compiled/style_' ~ styleId ~ '_special.css') }}">
    {% endif %}
{% endif %}
{% endblock %}
```

### 6.3 缓存删除触发重新编译

实现一个Middleware或Service：

```php
class CssCacheMiddleware {
    public function handle($request, $next) {
        // 检查CSS文件是否存在
        $styleId = $this->getStyleId($request);
        $cssType = $this->getCssType($request);
        
        $cssFile = "public/css/compiled/style_{$styleId}_{$cssType}.css";
        
        if (!file_exists($cssFile)) {
            // 自动编译
            $compiler = new StyleCompilerService(...);
            $compiler->compileStyle($styleId);
        }
        
        return $next($request);
    }
}
```

---

## 七、实现建议

### 7.1 立即实现

1. **增强 `StyleCompilerService`**:
   - 添加 `isCssTemplateUpdated()` 方法
   - 添加 `getCssTemplateMtime()` 方法
   - 添加 `getCompiledCssMtime()` 方法

2. **实现CSS自动编译Middleware**:
   - 检查CSS文件是否存在
   - 检查CSS模板是否更新
   - 自动触发编译

3. **更新Twig模板**:
   - 在 `base.html.twig` 中实现自动CSS加载
   - 根据页面类型自动选择CSS类型

### 7.2 后续优化

1. **性能优化**:
   - 缓存CSS模板修改时间
   - 批量编译多个style IDs

2. **开发体验**:
   - 添加开发模式：总是重新编译
   - 添加编译日志

---

## 八、总结

Legacy系统的机制：
- ✅ 模板编译：`.htm` → `.tpl.php`（Modern不需要，Twig自动处理）
- ✅ CSS编译：CSS模板 + 变量 → 静态CSS（Modern已实现基础功能）
- ✅ 自动刷新：检查文件修改时间（Modern需要实现）
- ✅ 删除缓存触发重新编译（Modern需要实现）

Modern系统需要：
1. 增强CSS编译服务，支持自动检测更新
2. 实现Middleware自动编译缺失的CSS
3. 在Twig模板中自动引用编译后的CSS

---

**文档状态**: ✅ 完成  
**下一步**: 实现CSS自动编译和刷新机制
