# Phase 5: Template Integration - Quick Reference Guide

**Date**: 2026-02-19
**Status**: ✅ COMPLETED

---

## Quick Lookup: Legacy Template Structure

### Creating a New Page (Legacy Style)

```twig
{# Extend base layout #}
{% extends 'layouts/base.html.twig' %}

{# Page title #}
{% block title %}Page Title - {{ app.name }}{% endblock %}

{# Content #}
{% block content %}
{# Breadcrumb #}
<div id="foruminfo">
    <div id="nav">
        <a href="{{ url('index.php') }}">Home</a> &raquo; Page Title
    </div>
</div>

{# Main content table #}
<div class="maintable">
    <div class="spaceborder">
        <table class="tableborder" cellspacing="1" cellpadding="4" width="100%">
            <tr class="header">
                <td>Page Title</td>
            </tr>
            <tr class="row">
                <td>Page content here</td>
            </tr>
        </table>
    </div>
</div>
{% endblock %}
```

---

## Common Legacy Patterns

### 1. Data Table Pattern

```twig
<table class="tableborder" cellspacing="1" cellpadding="4" width="100%">
    {# Header #}
    <tr class="header">
        <td colspan="X">Table Title</td>
    </tr>

    {# Category/Header Row #}
    <tr class="category">
        <th>Column 1</th>
        <th>Column 2</th>
        <th>Column 3</th>
    </tr>

    {# Data Rows #}
    {% for item in items %}
    <tr class="row" onMouseOver="this.className='row1'" onMouseOut="this.className='row'">
        <td>{{ item.col1 }}</td>
        <td>{{ item.col2 }}</td>
        <td>{{ item.col3 }}</td>
    </tr>
    {% endfor %}
</table>
```

### 2. Form Pattern (Legacy Style)

```twig
<div class="maintable">
    <div class="spaceborder">
        <form method="post" action="{{ url('index.php?route=action') }}">
            <input type="hidden" name="formhash" value="{{ csrf_token() }}" />

            <table class="tableborder" cellspacing="1" cellpadding="4">
                <tr class="header">
                    <td colspan="2">Form Title</td>
                </tr>

                {# Text Input #}
                <tr class="row">
                    <th width="30%" align="right">Field Label *</th>
                    <td>
                        <input type="text" name="field_name" size="25" value="{{ value }}" />
                    </td>
                </tr>

                {# Password Input #}
                <tr class="row">
                    <th align="right">Password *</th>
                    <td>
                        <input type="password" name="password" size="25" />
                    </td>
                </tr>

                {# Select Dropdown #}
                <tr class="row">
                    <th align="right">Select Label</th>
                    <td>
                        <select name="select_name">
                            <option value="1">Option 1</option>
                            <option value="2">Option 2</option>
                        </select>
                    </td>
                </tr>

                {# Checkbox #}
                <tr class="row">
                    <th align="right">&nbsp;</th>
                    <td class="smalltxt">
                        <label>
                            <input type="checkbox" name="check_name" value="1" />
                            Checkbox label
                        </label>
                    </td>
                </tr>

                {# Submit Button #}
                <tr class="row">
                    <th align="right">&nbsp;</th>
                    <td>
                        <button type="submit" name="submit" value="true" class="lightbutton">
                            Submit
                        </button>
                    </td>
                </tr>
            </table>
        </form>
    </div>
</div>
```

### 3. Section with Categories Pattern

```twig
<div class="maintable">
    <div class="spaceborder">
        <table class="tableborder">
            {# Main Header #}
            <tr class="header">
                <td colspan="2">Main Title</td>
            </tr>

            {# Category 1 #}
            <tr class="category">
                <td colspan="2">Category 1</td>
            </tr>
            <tr class="row">
                <th width="30%" align="right">Label 1</th>
                <td>Value 1</td>
            </tr>

            {# Category 2 #}
            <tr class="category">
                <td colspan="2">Category 2</td>
            </tr>
            <tr class="row">
                <th align="right">Label 2</th>
                <td>Value 2</td>
            </tr>
        </table>
    </div>
</div>
```

### 4. List Pattern (Using Table)

```twig
<div class="maintable">
    <div class="spaceborder">
        <table class="tableborder">
            <tr class="header">
                <td>List Title</td>
            </tr>
            {% for item in items %}
            <tr class="row" onMouseOver="this.className='row1'" onMouseOut="this.className='row'">
                <td class="smalltxt">
                    <a href="{{ item.url }}">{{ item.title }}</a>
                    <br>
                    <em class="smalltxt" style="color: #999;">{{ item.description }}</em>
                </td>
            </tr>
            {% endfor %}
        </table>
    </div>
</div>
```

---

## Legacy CSS Classes Reference

### Containers
```html
<div class="wrap">           <!-- Main container -->
<div class="maintable">      <!-- Table container -->
<div class="spaceborder">    <!-- Border wrapper -->
<table class="tableborder">  <!-- Table with borders -->
```

### Sections (IDs)
```html
<div id="header">            <!-- Header section -->
<div id="menu">              <!-- Navigation menu -->
<div id="foruminfo">         <!-- Forum info -->
<div id="nav">               <!-- Breadcrumb -->
<div id="footer">            <!-- Footer -->
<div id="footlinks">         <!-- Footer links -->
<p id="copyright">           <!-- Copyright -->
<p id="debuginfo">          <!-- Debug info -->
```

### Table Rows
```html
<tr class="header">          <!-- Blue background, white text -->
<tr class="category">        <!-- Gray background -->
<tr class="row">             <!-- White background -->
<tr class="row1">            <!-- Hover state (JS) -->
```

### Forum/Thread Specific
```html
<table class="forumlist">    <!-- Forum list -->
<table class="threadlist">   <!-- Thread list -->
<div class="viewthread">     <!-- Thread view -->
<td class="postauthor">      <!-- Post author (180px) -->
<td class="postcontent">     <!-- Post content -->
<div class="postinfo">       <!-- Post metadata -->
<div class="postmessage">    <!-- Post content -->
<div class="signatures">     <!-- User signature -->
<div class="postactions">    <!-- Post actions -->
```

### Utilities
```html
<span class="smalltxt">      <!-- Small text (12px) -->
<button class="lightbutton"><!-- Lightweight button -->
<div class="pages">          <!-- Pagination wrapper -->
```

---

## Common Mistakes to Avoid

### ❌ Wrong: Using modern classes
```twig
<div class="container">
    <div class="card">
        <div class="card-body">Content</div>
    </div>
</div>
```

### ✅ Right: Using Legacy classes
```twig
<div class="wrap">
    <div class="maintable">
        <div class="spaceborder">
            <table class="tableborder">
                <tr class="row">
                    <td>Content</td>
                </tr>
            </table>
        </div>
    </div>
</div>
```

### ❌ Wrong: Using semantic HTML5
```twig
<main class="content">
    <section class="post">
        <article class="post-content">...</article>
    </section>
</main>
```

### ✅ Right: Using Legacy structure
```twig
<div class="wrap">
    <div class="maintable">
        <table class="tableborder">
            <tr class="row">
                <td class="postcontent">...</td>
            </tr>
        </table>
    </div>
</div>
```

### ❌ Wrong: Inline styles
```twig
<div style="background: #fff; border: 1px solid #ddd; padding: 10px;">
    Content
</div>
```

### ✅ Right: External CSS
```twig
<div class="maintable">
    <div class="spaceborder">
        <table class="tableborder">
            <tr class="row">
                <td>Content</td>
            </tr>
        </table>
    </div>
</div>
```

---

## Twig Tips for Legacy Templates

### 1. Always use `|escape` for user data
```twig
{{ user.username|escape }}
{{ thread.subject|escape }}
{{ post.message|escape }}
```

### 2. Use `|number_format` for statistics
```twig
{{ stats.threads|number_format }}
{{ stats.posts|number_format }}
{{ thread.views|number_format }}
```

### 3. Use `format_date()` for dates
```twig
{{ format_date(post.dateline) }}
{{ format_date(user.regdate, 'Y-m-d') }}
```

### 4. Always include CSRF token in forms
```twig
<input type="hidden" name="formhash" value="{{ csrf_token() }}" />
```

### 5. Use proper URL functions
```twig
{{ url('index.php') }}
{{ url('index.php?route=forum&fid=' ~ forum.fid) }}
{{ asset('images/common/star.gif') }}
```

---

## Migration Checklist

When migrating a modern template to Legacy:

- [ ] Replace modern containers with `.wrap`
- [ ] Replace flex/grid layouts with tables
- [ ] Use `.maintable` > `.spaceborder` > `.tableborder` structure
- [ ] Use `.header`, `.category`, `.row` for table rows
- [ ] Replace modern buttons with `.lightbutton` class
- [ ] Add `onMouseOver`/`onMouseOut` to `.row` elements
- [ ] Use `<th>` for labels, `<td>` for values
- [ ] Add `.smalltxt` class for small text
- [ ] Remove all inline styles
- [ ] Test with sample data
- [ ] Verify Twig logic works

---

## Quick Conversion Examples

### Button
```html
{# Before #}
<button class="btn btn-primary">Submit</button>

{# After #}
<button class="lightbutton">Submit</button>
```

### Small Text
```html
{# Before #}
<span class="text-muted small">Text</span>

{# After #}
<span class="smalltxt">Text</span>
```

### Container
```html
{# Before #}
<div class="container">Content</div>

{# After #}
<div class="wrap">Content</div>
```

### Badge/Metric
```html
{# Before #}
<div class="badge">123</div>

{# After #}
<em>123</em>
```

---

## Useful Code Snippets

### Pagination (Legacy Style)
```twig
<div class="pages">
    {% if pagination.current_page > 1 %}
        <a href="{{ url('page=' ~ (pagination.current_page - 1)) }}">&laquo;</a>
    {% endif %}

    {% for page in range(1, pagination.total_pages) %}
        {% if page == pagination.current_page %}
            <strong>{{ page }}</strong>
        {% else %}
            <a href="{{ url('page=' ~ page) }}">{{ page }}</a>
        {% endif %}
    {% endfor %}

    {% if pagination.current_page < pagination.total_pages %}
        <a href="{{ url('page=' ~ (pagination.current_page + 1)) }}">&raquo;</a>
    {% endif %}
</div>
```

### Empty State
```twig
<div class="maintable">
    <div class="spaceborder">
        <table class="tableborder">
            <tr class="header">
                <td>Title</td>
            </tr>
            <tr class="row">
                <td style="text-align: center; padding: 30px;">
                    <p style="color: #999;">No data available</p>
                </td>
            </tr>
        </table>
    </div>
</div>
```

### Error Message
```twig
{% if error %}
<tr class="row">
    <td colspan="2" style="color: red; text-align: center;">
        {{ error }}
    </td>
</tr>
{% endif %}
```

---

## Resources

### Documentation
- `PHASE5-EXECUTIVE-SUMMARY.md` - Overall summary
- `PHASE5-TEMPLATE-INTEGRATION-COMPLETE.md` - Detailed report
- `PHASE5-BEFORE-AFTER-COMPARISON.md` - Visual comparison

### Reference Files
- `templates/layouts/base.html.twig` - Base layout
- `templates/components/header.html.twig` - Header component
- `templates/components/menu.html.twig` - Menu component
- `templates/components/footer.html.twig` - Footer component
- `templates/components/post.html.twig` - Post component

### Legacy Reference
- `/root/poketb-renew/poketb.com/bbs/templates/default/` - Legacy Discuz! templates

---

## Quick Help

### Problem: Template not rendering correctly
**Solution**: Check that you're using `.wrap` container and proper table structure

### Problem: Styles not applying
**Solution**: Verify you're using correct Legacy class names (`.maintable`, `.tableborder`, etc.)

### Problem: Twig logic not working
**Solution**: Ensure you're using correct Twig syntax (`{{ }}` for output, `{% %}` for logic)

### Problem: Form not submitting
**Solution**: Check that CSRF token is included: `{{ csrf_token() }}`

---

**Last Updated**: 2026-02-19
**Maintained by**: Claude Code (Migration Agent)
