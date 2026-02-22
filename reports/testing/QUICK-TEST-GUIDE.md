# Quick Test Guide - CSS Rollback Phase 4

## Test Page Location

### File Path
```
/root/poketb-renew/modern-php-migration-code/public/test-legacy-css.html
```

### How to Open

#### Option 1: Direct File (Recommended for Quick Testing)
```bash
# Simply open the file in your browser
# File URL: file:///root/poketb-renew/modern-php-migration-code/public/test-legacy-css.html
```

#### Option 2: PHP Built-in Server
```bash
cd /root/poketb-renew/modern-php-migration-code/public
docker exec -i discuz_modern_php php -S localhost:8000
# Then visit: http://localhost:8000/test-legacy-css.html
```

#### Option 3: Docker Web Server (if configured)
```
http://localhost:8000/test-legacy-css.html
```

## What You'll See

The test page contains **12 sections** testing all rolled-back CSS components:

1. **Color Variables** - Visual swatches of all CSS variables
2. **Typography** - Headings, paragraphs, lists
3. **Buttons** - Default, submit, cancel, disabled states
4. **Form Elements** - Inputs, textareas, selects, checkboxes, radios
5. **Tables (Forum Style)** - Forum listing table
6. **Tables (Thread Style)** - Thread listing table
7. **Pagination** - Page navigation
8. **Notice Boxes** - Info, success, warning, error messages
9. **Tabs** - Tab navigation
10. **Post Layout** - Forum post with author sidebar
11. **Layout Containers** - Boxes and grid layouts
12. **Utility Classes** - Helper classes for common patterns

## Quick Verification Checklist

When you open the test page, check:

- [ ] All colors display correctly (no missing var() references)
- [ ] All text is readable with proper fonts
- [ ] Buttons have proper styling and hover states
- [ ] Form elements are styled consistently
- [ ] Tables have proper borders and alternating row colors
- [ ] Pagination displays correctly
- [ ] Notice boxes have appropriate colors
- [ ] Post layout shows sidebar and content areas
- [ ] Images load without 404 errors
- [ ] Layout is responsive on mobile viewport

## Expected Results

### Colors
- Blue links (not default browser blue)
- Proper table header gradients
- Alternating table row colors
- Notice boxes with appropriate background colors

### Images
- All background images load (no 404s in browser console)
- Forum icons display
- Arrow indicators show
- Menu backgrounds render

### Layout
- Proper spacing between sections
- Boxes with borders and backgrounds
- Grid layouts display correctly
- Responsive behavior on smaller screens

## Troubleshooting

### Images Not Loading
```bash
# Verify images exist
ls -lh /root/poketb-renew/modern-php-migration-code/public/images/default/*.gif

# Check permissions
chmod 644 /root/poketb-renew/modern-php-migration-code/public/images/default/*.gif
```

### CSS Not Applying
```bash
# Verify CSS files exist
ls -lh /root/poketb-renew/modern-php-migration-code/public/assets/css/legacy-*.css

# Check browser console for 404 errors
# Verify file paths in test-legacy-css.html
```

### Styling Looks Wrong
- Clear browser cache
- Hard refresh (Ctrl+Shift+R / Cmd+Shift+R)
- Check browser console for CSS errors
- Verify all 10 CSS files are linked

## Next Steps After Testing

If everything looks good:
1. ✅ Phase 4 is complete
2. 📝 Proceed to Phase 5: Documentation
3. 🚀 Start using rolled-back CSS in actual templates

If you find issues:
1. 🐛 Note the problems
2. 🔧 Fix in respective CSS file
3. ✅ Re-test until everything works

## Files Referenced

- Test Page: `public/test-legacy-css.html`
- CSS Files: `public/assets/css/legacy-*.css` (10 files)
- Images: `public/images/default/*.gif` (146 files)
- Full Report: `CSS-ROLLBACK-TEST-REPORT.md`
- Quick Summary: `CSS-ROLLBACK-PHASE4-SUMMARY.md`

---

**Last Updated**: 2026-02-19
**Phase**: 4/5 (80% Complete)
**Status**: ✅ Ready for Manual Testing
