# Legacy Discuz! 6.1F CSS分析报告

## 1. CSS文件结构

### 主要CSS文件列表
| 文件路径 | 用途 | 大小 |
|---------|------|------|
| `templates/default/css_common.htm` | 通用样式（主样式表） | 29.5 KB |
| `templates/default/css_viewthread.htm` | 主题页面专用样式 | 8.4 KB |
| `templates/default/css_editor.htm` | 编辑器专用样式 | 5.5 KB |
| `templates/default/css_special.htm` | 特殊主题（投票、悬赏、交易、辩论、视频）样式 | 8.8 KB |
| `templates/default/css_append.htm` | 附加样式 | 1.5 KB |
| `templates/default/css_calendar.htm` | 日历样式 | 1.4 KB |

## 2. 颜色系统

### 主色调
- 主色: #069 (链接高亮、重要元素)
- 辅助色: #FDB939 (按钮边框)、#FFFDEE (按钮背景)
- 背景色: #FFF (主背景)、#F7F7F7 (内容框背景)、#F5FAFE (交替背景1)、#F7F7F3 (交替背景2)
- 文字色: #000 (主要文字)、#666 (次要文字)、#999 (提示文字)、#090 (成功提示)
- 边框色: #CAD9EA (主边框)、#E8E8E8 (内容框边框)、#EDEDCE (提示框边框)

### 完整色板
| 颜色名 | Hex值 | RGB | 用途 |
|--------|-------|-----|------|
| LINK | #069 | (0, 105, 255) | 链接、高亮文字 |
| HIGHLIGHTLINK | #069 | (0, 105, 255) | 重要链接、高亮 |
| TABLETEXT | #000 | (0, 0, 0) | 主要文字 |
| TEXT | #666 | (102, 102, 102) | 次要文字 |
| LIGHTTEXT | #999 | (153, 153, 153) | 提示文字 |
| NOTICEBG | #FFFFF2 | (255, 255, 242) | 提示框背景 |
| NOTICEBORDER | #EDEDCE | (237, 237, 206) | 提示框边框 |
| NOTICETEXT | #090 | (0, 153, 0) | 提示文字 |
| TABLEBG | #FFF | (255, 255, 255) | 主背景 |
| ALTBG1 | #F5FAFE | (245, 250, 254) | 交替背景1 |
| ALTBG2 | #F7F7F3 | (247, 247, 243) | 交替背景2 |
| COMMONBOXBG | #F7F7F7 | (247, 247, 247) | 内容框背景 |
| BGCOLOR | #FFF | (255, 255, 255) | 按钮背景 |
| CATCOLOR | #E8F3FD | (232, 243, 253) | 分类标题背景 |
| CATBORDER | #CAD9EA | (202, 217, 234) | 分类边框 |
| BORDERCOLOR | #9DB3C5 | (157, 179, 197) | 主边框 |
| HEADERTEXT | #FFF | (255, 255, 255) | 头部文字 |
| BGBORDER | #CAD9EA | (202, 217, 234) | 边框背景 |
| COMMONBOXBORDER | #E8E8E8 | (232, 232, 232) | 内容框边框 |
| INPUTBORDER | #D7D7D7 | (215, 215, 215) | 输入框边框 |
| HEADERMENUTEXT | #333 | (51, 51, 51) | 菜单文字 |
| PORTALBOXBG | #FFF | (255, 255, 255) | 门户框背景 |

## 3. 布局系统

### 容器规范
- 主容器宽度: 1000px (.wrap)
- 侧边栏宽度: 18% (.side)
- 内容区域宽度: 80% (.content)
- 消息框最小高度: 表达式控制 (IE6)
- 间距规范: 10px (.boxspace)

### 布局结构
```
wrap (1000px, margin: 0 auto)
├── header (logo + ad)
├── menu (导航菜单)
├── foruminfo/forumdisplay/viewthread (主要内容)
│   ├── userinfo/nav (左侧信息)
│   └── headstats/search (右侧信息)
├── mainbox (主要内容框)
├── side (侧边栏 - 仅管理页面)
└── footer (页脚)
```

### 栅格系统
- 使用浮动布局 (float: left/right)
- 固定宽度容器 (1000px)
- 响应式百分比布局 (侧边栏18%, 内容80%)
- 清除浮动 (clear: both)

## 4. 排版系统

### 字体家族
- 中文字体: Microsoft YaHei (优先)
- 英文字体: Comic Sans MS, Tahoma, Helvetica, Arial, sans-serif
- 等宽字体: "Courier New", Courier, monospace

### 字号规范
| 元素 | 字号 | 行高 | 字重 |
|------|------|------|------|
| body | 12px | 1.6em | normal |
| 标题 (h1-h3) | 12px | 31px | normal |
| 消息内容 | 14px | 1.6em | normal |
| 小文字 | 12px | 1.6em | normal |
| 大文字 | 16px | 1.6em | normal |
| 提示文字 | 10px | - | normal |
| 页脚文字 | 0.83em | 1.5em | normal |

### 文字对齐
- 主要文本: text-align: left
- 导航菜单: text-align: center, float: right
- 页脚: text-align: center
- 表格: 根据内容设置

## 5. 组件样式

### 按钮
```css
/* 默认按钮 */
button {
    border: 1px solid #CAD9EA #E8E8E8 #E8E8E8 #CAD9EA;
    background: #E8F3FD;
    height: 2em;
    line-height: 2em;
    cursor: pointer;
}

/* 提交按钮 */
#postsubmit, button.submit {
    border-color: #FFFDEE #FDB939 #FDB939 #FFFDEE;
    background: #FFF8C5;
    color: #090;
    padding: 0 10px;
}
```

### 表单
```css
/* 输入框 */
input, textarea {
    border-width: 1px;
    background: #FFF;
    border-color: #D7D7D7;
    padding: 2px;
}

/* 选择框 */
select {
    border: 1px solid #CAD9EA;
    background: #F7F7F7;
}
```

### 表格
```css
/* 表格基础样式 */
table {
    empty-cells: show;
    border-collapse: collapse;
}

/* 表头样式 */
.mainbox thead th, .mainbox thead td {
    background: #E8F3FD;
    padding: 2px 5px;
    line-height: 22px;
    color: #000;
}

/* 斑马纹 */
.mainbox tbody tr:hover th, .mainbox tbody tr:hover td {
    background-color: #F7F7F3;
}
```

### 导航
```css
/* 主导航 */
#menu {
    height: 31px;
    border: 1px solid #CAD9EA;
    background: #E8F3FD url(...) repeat-x;
}

/* 面包屑 */
#nav {
    margin: 10px 5px;
}

/* 分页 */
.pages {
    float: left;
    border: 1px solid #CAD9EA;
    background: #F7F7F7;
    height: 24px;
    line-height: 26px;
    color: #999;
}
```

## 6. 页面布局

### 首页布局 (forumdisplay)
```html
<div class="wrap">
    <div id="header">
        <h2><a href="index.php">{BOARDLOGO}</a></h2>
        <div id="ad_headerbanner"></div>
    </div>
    <div id="menu">
        <!-- 用户信息 -->
        <span class="avataonline"></span>
        <!-- 导航菜单 -->
        <ul>
            <li><a href="...">消息</a></li>
            <li><a href="...">宠物中心</a></li>
            <li><a href="...">排行榜</a></li>
            <!-- 更多菜单项 -->
        </ul>
    </div>
    <div id="foruminfo">
        <div id="headsearch">
            <!-- 搜索和工具 -->
        </div>
        <div id="nav">
            <!-- 面包屑导航 -->
        </div>
    </div>
    <!-- 论坛列表/板块列表 -->
    <table class="mainbox forumlist">
        <thead>
            <tr>
                <th>板块名称</th>
                <th>主题</th>
                <th>回复</th>
                <th>最后发表</th>
            </tr>
        </thead>
        <tbody>
            <!-- 板块列表项 -->
        </tbody>
    </table>
</div>
```

### 主题页布局 (viewthread)
```html
<div class="wrap">
    <!-- header 和 menu 同上 -->
    <div id="threadinfo">
        <!-- 主题标题和作者信息 -->
    </div>
    <!-- 主题内容 -->
    <div class="mainbox viewthread">
        <!-- 帖子列表 -->
        <table>
            <!-- 帖子1 -->
            <tr class="threadlist">
                <td class="folder">
                    <div class="target"></div>
                </td>
                <td class="icon">
                    <img src="..." />
                </td>
                <td class="author">
                    <em>作者</em>
                    <cite><a href="...">作者名</a></cite>
                </td>
                <!-- 其他列 -->
            </tr>
            <!-- 更多帖子 -->
        </table>
        <!-- 回复表单 -->
        <div id="quickpost">
            <!-- 快速回复 -->
        </div>
    </div>
</div>
```

## 7. CSS类名规范

### 常用类名列表
| 类名 | 用途 | 定义位置 |
|------|------|----------|
| wrap | 主容器包装器 | css_common.htm |
| mainbox | 主要内容框 | css_common.htm |
| box | 通用内容框 | css_common.htm |
| forumlist | 论坛列表 | css_common.htm |
| threadlist | 主题列表 | css_common.htm |
| postmessage | 帖子内容 | css_viewthread.htm |
| postauthor | 帖子作者信息 | css_viewthread.htm |
| pages | 分页导航 | css_common.htm |
| tabs | 标签页 | css_common.htm |
| formbox | 表单框 | css_common.htm |
| headactions | 头部操作按钮 | css_common.htm |
| popupmenu_popup | 弹出菜单 | css_common.htm |
| side | 侧边栏 | css_common.htm |

### ID命名规范
| ID | 用途 | 定义位置 |
|----|------|----------|
| header | 页头 | header.htm |
| menu | 导航菜单 | header.htm |
| foruminfo | 论坛信息区 | forumdisplay.htm |
| nav | 面包屑导航 | forumdisplay.htm |
| footer | 页脚 | footer.htm |
| postsubmit | 提交按钮 | post_*.htm |
| quickpost | 快速回复区 | viewthread.htm |
| smilieslist | 表情列表 | css_common.htm |

## 8. 响应式设计
- 遗憾的是，Discuz! 6.1F 没有现代意义上的响应式设计
- 使用固定宽度布局 (1000px)
- 主要兼容 IE6+ 浏览器
- 使用 CSS hack 处理浏览器兼容性

## 9. 兼容性说明
- **IE6 兼容处理**: 使用 expression() 进行动态计算
- **IE7 兼容处理**: 使用 * html 选择器
- **PNG 透明**: 使用 DD_belatedPNG.js 修复 IE6 PNG 透明
- **浮动布局**: 使用 float 布局，IE6 需要额外处理高度

## 10. 迁移建议

### 可以直接复用的元素
1. **颜色系统**: 完整的色板可以直接使用
2. **字体设置**: 字体家族和字号规范
3. **基础布局**: wrap、mainbox、box 等基础容器
4. **表格样式**: 论坛列表和帖子列表样式
5. **导航组件**: 菜单和面包屑样式

### 需要现代化处理
1. **布局系统**: 从固定宽度改为响应式
2. **CSS选择器**: 使用语义化类名替代 .mainbox 等泛用类名
3. **CSS变量**: 将硬编码的颜色值提取为 CSS 变量
4. **浏览器兼容**: 移除 IE6/7 hack，使用现代CSS特性
5. **盒模型**: 使用 border-box 替代 content-box

### 应该废弃
1. **Expression()**: 完全移除 IE6 的 expression()
2. **CSS hack**: 移除 * html、*+html 等 hack
3. **内联样式**: 移除模板中的 style 属性
4. **图片精灵**: 考虑使用 SVG 或 icon font 替代

## 11. 特殊功能样式

### 图标系统
- **主题图标**: icon1.gif ~ icon9.gif 位于 `images/icons/`
- **状态图标**: 在线状态、离线状态图标位于 `images/common/`
- **功能图标**: 编辑、删除、加精等图标使用 bb_*.gif

### 特殊主题样式
- **投票**: `.pollpanel` - 投票面板样式
- **悬赏**: `.specialthread_3` - 悬赏主题样式
- **交易**: `.tradethread` - 交易主题样式
- **辩论**: `.specialthread_5` - 辩论主题样式
- **视频**: `.videobox` - 视频播放器样式

### 编辑器样式
- **工具栏**: `.editortoolbar` - 编辑器工具栏
- **文本区域**: `.editor_text` - 编辑器文本域
- **按钮**: `.editor_button` - 编辑器按钮

## 12. 图片资源位置

### 主要图标目录
```
images/
├── icons/           # 主题图标 (icon1.gif ~ icon9.gif)
├── common/         # 通用图标
│   ├── bb_*.gif    # 编辑器按钮图标
│   ├── *.gif       # 状态图标
│   └── editor.gif   # 编辑器精灵图
├── avatars/        # 用户头像
├── smilies/        # 表情图标
├── attachicons/    # 附件图标
└── default/        # 默认主题图片
```

### 关键图片资源
- **编辑器图标**: `images/common/editor.gif` (包含所有编辑器按钮)
- **状态图标**: `images/common/online_*.gif` (在线状态指示器)
- **导航图标**: `images/common/menu_*.gif` (菜单图标)
- **箭头图标**: `images/common/arrow_right.gif` (通用箭头)

## 13. 性能优化建议

### CSS优化
1. **压缩CSS**: 移除注释和多余空格
2. **合并选择器**: 减少重复声明
3. **使用简写**: 使用 margin、padding、background 等简写属性
4. **减少嵌套**: 避免过深的CSS选择器嵌套

### 图片优化
1. **Sprite技术**: 合并小图标为精灵图
2. **WebP格式**: 考虑使用WebP格式减小文件大小
3. **懒加载**: 对于图片资源实现懒加载
4. **CDN加速**: 将图片资源托管到CDN

## 14. 浏览器支持范围

### 完全支持
- IE8+
- Firefox 3.6+
- Chrome
- Safari
- Opera 10.5+

### 需要修复
- IE6 (已过时，建议放弃支持)
- IE7 (基本功能支持，但可能需要额外测试)

### 不支持
- IE5 及以下版本

---

*本报告基于 Discuz! 6.1F 版本分析，包含完整的CSS样式系统和HTML结构分析。适用于现代PHP项目的CSS回滚和样式迁移参考。*
