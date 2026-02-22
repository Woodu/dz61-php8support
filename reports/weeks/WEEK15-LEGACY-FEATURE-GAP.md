# 🔍 Week 15 Legacy功能差距分析

**分析日期**: 2026-02-21  
**分析范围**: Legacy发帖功能 vs Modern实现  
**目的**: 确保功能完整迁移

---

## 📊 功能对比总览

| 功能模块 | Legacy实现 | Modern实现 | 差距 | 优先级 |
|---------|-----------|-----------|------|--------|
| **基本表单** | ✅ | ✅ | 0% | - |
| **BBCode编辑器** | ✅ | ✅ | 0% | - |
| **主题图标** | ✅ | ❌ | 100% | 🔴 P0 |
| **投票功能** | ✅ | ❌ | 100% | 🔴 P0 |
| **支付功能** | ✅ | ❌ | 100% | 🔴 P0 |
| **附件上传** | ✅ | ⚠️ | 60% | 🔴 P0 |
| **表情符号** | ✅ | ❌ | 100% | 🟡 P1 |
| **编辑器选项** | ✅ | ❌ | 100% | 🟡 P1 |
| **高级选项** | ✅ | ❌ | 100% | 🟡 P1 |

---

## 🔴 P0 必须实现功能

### 1. 主题图标选择

**Legacy实现** (`post_newthread.htm` Line 210-212):
```html
<th>{lang icon}</th>
<td>
    <label>
        <input class="radio" type="radio" name="iconid" value="0" checked="checked" />
        {lang none}
    </label>
    $icons
</td>
```

**Legacy图标数据来源**:
- 文件系统: `images/icons/icon1.gif` ~ `icon9.gif`
- PHP生成: `$icons`变量（包含9个图标的radio buttons）

**Modern需要实现**:
- [ ] 添加图标选择区域到`new_thread.html.twig`
- [ ] 从文件系统加载图标列表（icon1.gif ~ icon9.gif）
- [ ] 生成radio buttons
- [ ] 默认选择"无图标"（iconid=0）

**实现位置**: `templates/post/new_thread.html.twig`（在"发帖选项"区域之前）

---

### 2. 投票功能

**Legacy实现** (`post_newthread.htm` Line 131-144):
```html
<!--{if $special == 1 && $allowpostpoll}-->
<tr>
    <th><label for="expiration">{lang poll_days_valid}</label></th>
    <td>
        <input type="text" name="expiration" id="expiration" value="0" size="6" />
        <em class="tips">({lang post_zero_is_nopermission})</em>
    </td>
</tr>
<tr>
    <th valign="top">
        {lang post_poll_options}<br />
        {lang post_poll_comment}<br /><br />
        <input type="checkbox" name="visiblepoll" value="1" />
        {lang poll_submit_after}<br />
        <input type="checkbox" name="multiplepoll" value="1" 
               onclick="this.checked?$('maxchoicescontrol').style.display='':$('maxchoicescontrol').style.display='none';" />
        {lang post_poll_allowmultiple}<br />
        <span id="maxchoicescontrol" style="display: none">
            {lang poll_max_options}: 
            <input type="text" name="maxchoices" value="$maxpolloptions" size="5"><br />
        </span>
    </th>
    <td>
        <textarea rows="8" name="polloptions" style="width: 600px; word-break: break-all">
            $polloptions
        </textarea>
    </td>
</tr>
<!--{/if}-->
```

**功能说明**:
- `expiration`: 投票有效期（天数，0=永久）
- `polloptions`: 投票选项（textarea，每行一个选项）
- `visiblepoll`: 投票前可见（checkbox）
- `multiplepoll`: 允许多选（checkbox）
- `maxchoices`: 最大选择数（多选时显示）

**Modern需要实现**:
- [ ] 添加"添加投票"checkbox（控制显示/隐藏投票区域）
- [ ] 投票有效期输入框
- [ ] 投票选项textarea（每行一个选项）
- [ ] 投票设置checkboxes（可见性、多选）
- [ ] 最大选择数输入（多选时显示）
- [ ] JavaScript控制显示/隐藏逻辑

**实现位置**: `templates/post/new_thread.html.twig`（在内容编辑器之后）

---

### 3. 支付/付费主题

**Legacy实现** (`post_newthread.htm` Line 199-205):
```html
<!--{if $maxprice && !$special}-->
<tr>
    <th><label for="price">{lang price}({$extcredits[$creditstrans][title]})</label></th>
    <td>
        <input type="text" name="price" id="price" size="6" value="$price" />
        <em class="tips">
            {$extcredits[$creditstrans][unit]} 
            ({lang post_price_comment}
            <!--{if $maxincperthread}-->{lang post_price_income_comment}<!--{/if}-->
            <!--{if $maxchargespan}-->{lang post_price_charge_comment}<!--{/if}-->
            )
            {lang post_price_free_comment}
        </em>
    </td>
</tr>
<!--{/if}-->
```

**功能说明**:
- `price`: 主题价格（积分）
- 价格说明：显示积分单位、价格说明、免费说明

**Modern需要实现**:
- [ ] 添加"设置为付费主题"checkbox（控制显示/隐藏支付区域）
- [ ] 价格输入框
- [ ] 价格说明文字
- [ ] JavaScript控制显示/隐藏逻辑

**实现位置**: `templates/post/new_thread.html.twig`（在投票选项之后，或在高级选项中）

---

### 4. 附件上传增强

**Legacy实现** (`post_editor.htm` Line 453-511):
- 附件上传表格
- 每个附件：文件选择、阅读权限、价格、描述
- 附件大小限制显示
- 允许的文件扩展名显示
- 图片水印选项（setwater checkbox）

**Modern当前状态**:
- ⚠️ 基本文件上传存在
- ❌ 缺少附件列表UI
- ❌ 缺少附件描述输入
- ❌ 缺少附件权限设置
- ❌ 缺少附件价格设置

**Modern需要实现**:
- [ ] 附件上传表格UI
- [ ] 附件列表显示
- [ ] 附件描述输入
- [ ] 附件删除功能
- [ ] 图片预览功能

---

## 🟡 P1 重要功能

### 5. 表情符号选择器

**Legacy实现** (`post_smilies.htm`):
- 表情符号分类标签
- 表情符号网格显示
- 点击插入表情代码
- 分页支持

**Modern需要实现**:
- [ ] 表情符号选择器UI
- [ ] 表情符号分类
- [ ] 点击插入功能
- [ ] 集成到BBCode工具栏

---

### 6. 编辑器选项

**Legacy实现** (`post_editor.htm` Line 93-99):
```html
<li>
    <label>
        <input type="checkbox" name="parseurloff" id="parseurloff" value="1" />
        {lang disable} {lang post_parseurl}
    </label>
</li>
<li>
    <label>
        <input type="checkbox" name="smileyoff" id="smileyoff" value="1" />
        {lang disable} {faq smilies}
    </label>
</li>
<li>
    <label>
        <input type="checkbox" name="bbcodeoff" id="bbcodeoff" value="1" />
        {lang disable} {faq discuzcode}
    </label>
</li>
```

**功能说明**:
- `parseurloff`: 禁用URL自动解析
- `smileyoff`: 禁用表情符号
- `bbcodeoff`: 禁用BBCode解析

**Modern需要实现**:
- [ ] 添加编辑器选项区域
- [ ] 禁用URL解析checkbox
- [ ] 禁用表情checkbox
- [ ] 禁用BBCode checkbox

---

### 7. 高级选项折叠区域

**Legacy实现** (`post_newthread.htm` Line 173-215):
- "其他信息"折叠区域（adv）
- 包含：阅读权限、价格、图标选择

**Modern需要实现**:
- [ ] 添加"高级选项"折叠区域
- [ ] 阅读权限输入（readperm）
- [ ] 价格设置（price，与支付功能合并）
- [ ] 图标选择（iconid，与主题图标合并）

---

## 📋 Week 15 任务更新

基于Legacy审查，需要更新以下任务：

### Day 1 新增任务

- [ ] **Task DEV1.3: 添加主题图标选择** (2小时)
  - 参考: Legacy Line 210-212
  - 实现: 图标radio buttons
  - 图标来源: 文件系统（icon1.gif ~ icon9.gif）

- [ ] **Task DEV1.4: 添加投票功能** (3小时)
  - 参考: Legacy Line 131-144
  - 实现: 投票选项textarea、投票设置checkboxes

- [ ] **Task DEV1.5: 添加支付功能** (2小时)
  - 参考: Legacy Line 199-205
  - 实现: 价格输入框、价格说明

### Day 2 新增任务

- [ ] **Task DEV2.4: 添加高级选项折叠区域** (2小时)
  - 参考: Legacy Line 173-215
  - 实现: 折叠区域、阅读权限输入

### Day 3 新增任务

- [ ] **Task DEV3.4: 添加回复选项** (2小时)
  - 参考: Legacy post_editor.htm Line 93-115
  - 实现: 禁用URL解析、禁用表情、禁用BBCode、显示签名、邮件通知

### Day 4 新增任务

- [ ] **Task DEV4.4: 添加表情符号选择器** (3小时)
  - 参考: Legacy post_smilies.htm
  - 实现: 表情符号UI、分类、插入功能

---

**报告生成时间**: 2026-02-21  
**下一步**: 开始实现P0功能
