# P3 Enhanced Features - Detailed Migration Guide

**Purpose:** Nice-to-have features and third-party integrations
**Priority:** ENHANCED - Complete after P2, can be done incrementally
**Timeline:** Weeks 15-20 (30 working days)

---

## Overview

P3 features are enhanced functionality, plugins, and custom features that make the forum more engaging but aren't critical for basic operations. These can be migrated incrementally or deferred.

**P3 Features:**
1. Social Features (6 features)
2. Content Features (6 features)
3. Security & Anti-Spam (5 features)
4. Editor & Formatting (5 features)
5. Gamification (5 features)
6. Special Features (6 features)
7. Template & Theme System (4 features)
8. XML & API (2 features)
9. Archive & Mobile (2 features)
10. Plugin System (5 features)
11. Custom/PoketTB Specific (20 features)
12. UC Integration (4 features)
13. Testing & Debug (3 features)
14. Image Processing (3 features)

---

## 1. Social Features

### 1.1 User Space

**Current File:** `/root/poketb-renew/poketb.com/bbs/space.php`

**Lines:** 1-500

**Migration Effort:** 2 days

**Database Tables:**
- `cdb_members`
- `cdb_memberfields`
- `cdb_threads`
- `cdb_posts`

**Testing Checklist:**
- [ ] User space loads
- [ ] User info displays
- [ ] User's threads list
- [ ] User's posts list
- [ ] Privacy settings work
- [ ] Customization works

**Dependencies:**
- User Management (P1)

---

### 1.2 Buddy List

**Current File:** Various (in memcp.php)

**Migration Effort:** 1.5 days

**Database Tables:**
- `cdb_buddys`

**Testing Checklist:**
- [ ] Add buddy works
- [ ] Remove buddy works
- [ ] Buddy list displays
- [ ] Online status shows
- [ ] PM from buddy list works

**Dependencies:**
- User Space (1.1)

---

### 1.3 User Statistics

**Current File:** `/root/poketb-renew/poketb.com/bbs/stats.php`

**Lines:** 1-400

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Forum stats display
- [ ] Member stats display
- [ ] Post stats display
- [ ] Activity graphs work
- [ ] Trends calculate

**Dependencies:**
- Forum Display (P1)

---

### 1.4 Rank List

**Current File:** `/root/poketb-renew/poketb.com/bbs/ranklist.php`

**Lines:** 1-300

**Migration Effort:** 1.5 days

**Testing Checklist:**
- [ ] Member rankings work
- [ ] Thread rankings work
- [ ] Post rankings work
- [ ] Ranking criteria work
- [ ] Pagination works

**Dependencies:**
- User Statistics (1.3)

---

### 1.5 Digest Display

**Current File:** `/root/poketb-renew/poketb.com/bbs/digest.php`

**Lines:** 1-300

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Digest threads list
- [ ] Filtering by level works
- [ ] Pagination works

**Dependencies:**
- Forum Display (P1)

---

### 1.6 Tag System

**Current File:** Various (in misc.php)

**Migration Effort:** 2 days

**Database Tables:**
- `cdb_tags`
- `cdb_threadtags`

**Testing Checklist:**
- [ ] Tag creation works
- [ ] Tag editing works
- [ ] Tag deletion works
- [ ] Tag cloud displays
- [ ] Tag search works
- [ ] Hot tags show

**Dependencies:**
- Thread Management (P1)

---

## 2. Content Features

### 2.1 Announcements

**Current File:** `/root/poketb-renew/poketb.com/bbs/announcement.php`

**Lines:** 1-200

**Migration Effort:** 1 day

**Database Tables:**
- `cdb_announcements`

**Testing Checklist:**
- [ ] Announcement displays
- [ ] Date filtering works
- [ ] Group filtering works

**Dependencies:**
- Forum Display (P1)

---

### 2.2 FAQ System

**Current File:** `/root/poketb-renew/poketb.com/bbs/faq.php`

**Lines:** 1-300

**Migration Effort:** 1 day

**Database Tables:**
- `cdb_faqs`

**Testing Checklist:**
- [ ] FAQ categories load
- [ ] FAQ entries display
- [ ] FAQ search works

**Dependencies:**
- None

---

### 2.3 RSS Feed

**Current File:** `/root/poketb-renew/poketb.com/bbs/rss.php`

**Lines:** 1-200

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] RSS feed generates
- [ ] RSS validates
- [ ] Forum-specific feeds work
- [ ] Authentication works

**Dependencies:**
- Thread Viewing (P1)

---

### 2.4 Sitemap

**Current File:** `/root/poketb-renew/poketb.com/bbs/sitemap.php`

**Lines:** 1-300

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Sitemap generates
- [ ] Sitemap validates
- [ ] Sitemap pings work

**Dependencies:**
- Forum Display (P1)

---

### 2.5 Print View

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/printable.inc.php`

**Lines:** 1-150

**Migration Effort:** 0.5 days

**Testing Checklist:**
- [ ] Print view displays
- [ ] Formatting is clean

**Dependencies:**
- Thread Viewing (P1)

---

### 2.6 Topic Display

**Current File:** `/root/poketb-renew/poketb.com/bbs/topic.php`

**Lines:** 1-200

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Topic list loads
- [ ] Topic filtering works
- [ ] Pagination works

**Dependencies:**
- Forum Display (P1)

---

## 3. Security & Anti-Spam

### 3.1 CAPTCHA/Seccode

**Current File:** `/root/poketb-renew/poketb.com/bbs/seccode.class.php`

**Lines:** 1-400

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] CAPTCHA displays
- [ ] CAPTCHA validates
- [ ] CAPTCHA refreshes
- [ ] Different CAPTCHA types work

**Dependencies:**
- Core Bootstrap (P0)

---

### 3.2 Seccode Display

**Current File:** `/root/poketb-renew/poketb.com/bbs/seccode.php`

**Lines:** 1-100

**Migration Effort:** 0.5 days

**Dependencies:**
- CAPTCHA System (3.1)

---

### 3.3 Anti-Flood Control

**Current File:** Various (in security.inc.php)

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Flood detection works
- [ ] Flood prevention works
- [ ] Time limits enforce
- [ ] Whitelist works

**Dependencies:**
- Core Bootstrap (P0)

---

### 3.4 Login Protection

**Current File:** Various (in logging.php)

**Migration Effort:** 1 day

**Database Tables:**
- `cdb_failedlogins`

**Testing Checklist:**
- [ ] Failed login tracking works
- [ ] Temporary ban works
- [ ] IP blocking works
- [ ] Cooldown works

**Dependencies:**
- User Login (P0)

---

### 3.5 IP Banning

**Current File:** Various (in common.inc.php)

**Migration Effort:** 1 day

**Database Tables:**
- `cdb_banned`

**Testing Checklist:**
- [ ] IP ban blocks
- [ ] IP range ban works
- [ ] Whitelist overrides
- [ ] Ban expiration works

**Dependencies:**
- Core Bootstrap (P0)

---

## 4. Editor & Formatting

### 4.1 BBCode Editor

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/editor.func.php`

**Lines:** 1-600

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] BBCode buttons work
- [ ] Preview works
- [ ] Toolbar displays
- [ ] Emoticons insert

**Dependencies:**
- DiscuzCode Parser (P1)

---

### 4.2 Rich Text Editor

**Current File:** Various (in editor.func.php)

**Migration Effort:** 3 days

**Testing Checklist:**
- [ ] WYSIWYG mode works
- [ ] Source mode works
- [ ] Toolbar works
- [ ] Image upload works
- [ ] Fullscreen mode works

**Dependencies:**
- BBCode Editor (4.1)

---

### 4.3 Code Highlighting

**Current File:** Various (in discuzcode.func.php)

**Migration Effort:** 1.5 days

**Testing Checklist:**
- [ ] Code blocks format
- [ ] Syntax highlighting works
- [ ] Line numbers work
- [ ] Different languages work

**Dependencies:**
- DiscuzCode Parser (P1)

---

### 4.4 Custom BBCode

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/xsecode.func.php`

**Lines:** 1-300

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Custom tags work
- [ ] Custom tag parameters work
- [ ] Tag nesting works
- [ ] Tag validation works

**Dependencies:**
- DiscuzCode Parser (P1)

---

### 4.5 Diff Engine

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/diff.class.php`

**Lines:** 1-500

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Diff generates correctly
- [ ] Diff displays properly
- [ ] Inline diff works
- [ ] Unified diff works

**Dependencies:**
- Edit Post (P1)

---

## 5. Gamification

### 5.1 Magic Items

**Current File:** `/root/poketb-renew/poketb.com/bbs/magic.php`

**Lines:** 1-400

**Migration Effort:** 3 days

**Database Tables:**
- `cdb_magics`
- `cdb_membermagic`

**Testing Checklist:**
- [ ] Magic shop works
- [ ] Magic use works
- [ ] Magic effects work
- [ ] Magic inventory works
- [ ] Magic transfer works

**Dependencies:**
- User Management (P1)

---

### 5.2 Medal System

**Current File:** `/root/poketb-renew/poketb.com/bbs/medal.php`

**Lines:** 1-300

**Migration Effort:** 2 days

**Database Tables:**
- `cdb_medals`
- `cdb_memberfields`

**Testing Checklist:**
- [ ] Medal display works
- [ ] Medal granting works
- [ ] Medal revoking works
- [ ] Medal images work

**Dependencies:**
- User Management (P1)

---

### 5.3 Credit System

**Current File:** Various (in misc.php)

**Migration Effort:** 2 days

**Database Tables:**
- `cdb_creditslog`
- `cdb_settings`

**Testing Checklist:**
- [ ] Credits earn correctly
- [ ] Credits spend correctly
- [ ] Credit transfers work
- [ ] Credit logs show
- [ ] Exchange rates work

**Dependencies:**
- User Management (P1)

---

### 5.4 Bank System

**Current File:** `/root/poketb-renew/poketb.com/bbs/bank.php`

**Lines:** 1-500

**Migration Effort:** 3 days

**Database Tables:**
- Custom tables

**Testing Checklist:**
- [ ] Deposit works
- [ ] Withdraw works
- [ ] Transfer works
- [ ] Interest accrues
- [ ] Balance shows

**Dependencies:**
- Credit System (5.3)

---

### 5.5 Reward System

**Current File:** Various

**Migration Effort:** 1.5 days

**Testing Checklist:**
- [ ] Rewards grant correctly
- [ ] Reward notifications send
- [ ] Reward history shows

**Dependencies:**
- Credit System (5.3)

---

## 6. Special Features

### 6.1 AJAX Handler

**Current File:** `/root/poketb-renew/poketb.com/bbs/ajax.php`

**Lines:** 1-400

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] AJAX requests work
- [ ] JSON responses work
- [ ] Error handling works
- [ ] CSRF protection works

**Dependencies:**
- Core Bootstrap (P0)

---

### 6.2 Redirect System

**Current File:** `/root/poketb-renew/poketb.com/bbs/redirect.php`

**Lines:** 1-150

**Migration Effort:** 0.5 days

**Testing Checklist:**
- [ ] Redirects work
- [ ] External links redirect
- [ ] Referrer tracking works

**Dependencies:**
- Core Bootstrap (P0)

---

### 6.3 Help System

**Current File:** `/root/poketb-renew/poketb.com/bbs/help.php`

**Lines:** 1-200

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Help pages load
- [ ] Help categories work
- [ ] Help search works

**Dependencies:**
- None

---

### 6.4 Misc Functions

**Current File:** `/root/poketb-renew/poketb.com/bbs/misc.php`

**Lines:** 1-1000

**Migration Effort:** 3 days

**Testing Checklist:**
- [ ] All misc actions work
- [ ] Subscribe works
- [ ] Bookmark works
- [ ] Share works
- [ ] Print works

**Dependencies:**
- Multiple

---

### 6.5 Promotion System

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/promotion.inc.php`

**Lines:** 1-200

**Migration Effort:** 1.5 days

**Database Tables:**
- `cdb_promotions`

**Testing Checklist:**
- [ ] Promotions apply
- [ ] Promo codes work
- [ ] Rewards grant

**Dependencies:**
- User Management (P1)

---

### 6.6 Cron System

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/cron.func.php`

**Lines:** 1-400

**Migration Effort:** 2 days

**Database Tables:**
- `cdb_crons`

**Testing Checklist:**
- [ ] Cron jobs run
- [ ] Cron schedules work
- [ ] Cron logging works
- [ ] Manual execution works

**Dependencies:**
- Core Bootstrap (P0)

---

## 7. Template & Theme System

### 7.1 Template Engine

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/template.func.php`

**Lines:** 1-600

**Migration Effort:** 3 days

**Database Tables:**
- `cdb_templates`

**Testing Checklist:**
- [ ] Templates compile
- [ ] Template caching works
- [ ] Template inheritance works
- [ ] Template variables work

**Dependencies:**
- Caching System (P0)

---

### 7.2 Chinese Language

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/chinese.class.php`

**Lines:** 1-400

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Character conversion works
- [ ] UTF-8 support works
- [ ] Traditional/Simplified works

**Dependencies:**
- None

---

### 7.3 Multi-byte Support

**Current File:** Various (in chinese.class.php)

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] String functions work
- [ ] Truncation works
- [ ] Length calculation works

**Dependencies:**
- Chinese Language (7.2)

---

### 7.4 Frame Support

**Current File:** `/root/poketb-renew/poketb.com/bbs/frame.php`

**Lines:** 1-200

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Frames load
- [ ] Frame navigation works
- [ ] Frame-breaking protection works

**Dependencies:**
- Template Engine (7.1)

---

## 8. XML & API

### 8.1 XML Core

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/xml.class.php`

**Lines:** 1-400

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] XML parsing works
- [ ] XML generation works
- [ ] XML validation works

**Dependencies:**
- None

---

### 8.2 XML Parser

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/xmlparser.class.php`

**Lines:** 1-300

**Migration Effort:** 1 day

**Dependencies:**
- XML Core (8.1)

---

## 9. Archive & Mobile

### 9.1 Archiver

**Current File:** `/root/poketb-renew/poketb.com/bbs/archiver/` directory

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Archive pages generate
- [ ] Archive navigation works
- [ ] SEO URLs work

**Dependencies:**
- Forum Display (P1)

---

### 9.2 WAP Support

**Current File:** `/root/poketb-renew/poketb.com/bbs/wap/` directory

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] WAP pages work
- [ ] Mobile optimization works
- [ ] Device detection works

**Dependencies:**
- Forum Display (P1)

---

## 10. Plugin System

### 10.1 Plugin Handler

**Current File:** `/root/poketb-renew/poketb.com/bbs/plugin.php`

**Lines:** 1-50

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Plugins load
- [ ] Plugin permissions work
- [ ] Plugin routing works

**Dependencies:**
- Core Bootstrap (P0)

---

### 10.2 Bank Plugin

**Current File:** `/root/poketb-renew/poketb.com/bbs/plugins/bank/` directory

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Bank features work
- [ ] Integration works

**Dependencies:**
- Plugin Handler (10.1)

---

### 10.3 Google Sitemap Plugin

**Current File:** `/root/poketb-renew/poketb.com/bbs/plugins/googlesitemap/` directory

**Migration Effort:** 1 day

**Testing Checklist:**
- [ ] Sitemap generation works
- [ ] Sitemap submission works

**Dependencies:**
- Plugin Handler (10.1)

---

### 10.4 Admin Plugins

**Current File:** `/root/poketb-renew/poketb.com/bbs/plugins/admin/` directory

**Migration Effort:** Varies

**Dependencies:**
- Plugin Handler (10.1)

---

### 10.5 MOC Plugin

**Current File:** `/root/poketb-renew/poketb.com/bbs/plugins/moc/` directory

**Migration Effort:** Varies

**Dependencies:**
- Plugin Handler (10.1)

---

## 11. Custom/PoketTB Specific

### 11.1 Pet System

**Current File:** `/root/poketb-renew/poketb.com/bbs/pet/`, `zpet/`, `mdex/` directories

**Migration Effort:** 5 days

**Database Tables:**
- Custom tables

**Testing Checklist:**
- [ ] Pet creation works
- [ ] Pet feeding works
- [ ] Pet battles work
- [ ] Pet stats work
- [ ] Pet items work

**Dependencies:**
- User Management (P1)
- Credit System (5.3)

---

### 11.2 DEX System

**Current File:** `/root/poketb-renew/poketb.com/bbs/dex.php`, `mdex.php`

**Migration Effort:** 3 days

**Testing Checklist:**
- [ ] DEX trading works
- [ ] DEX items work
- [ ] DEX API works

**Dependencies:**
- Credit System (5.3)

---

### 11.3 Helper System

**Current File:** `/root/poketb-renew/poketb.com/bbs/helper*.php` files

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Helper features work
- [ ] Helper permissions work

**Dependencies:**
- User Management (P1)

---

### 11.4 Campaign System

**Current File:** `/root/poketb-renew/poketb.com/bbs/campaign.php`

**Migration Effort:** 2 days

**Testing Checklist:**
- [ ] Campaigns create
- [ ] Campaigns display
- [ ] Campaign tracking works

**Dependencies:**
- User Management (P1)

---

### 11.5 Family System

**Current File:** `/root/poketb-renew/poketb.com/bbs/family/` directory

**Migration Effort:** 3 days

**Testing Checklist:**
- [ ] Family creation works
- [ ] Family members work
- [ ] Family features work

**Dependencies:**
- User Management (P1)

---

### 11.6 Event Download

**Current File:** `/root/poketb-renew/poketb.com/bbs/eventdownload.php`

**Migration Effort:** 1.5 days

**Testing Checklist:**
- [ ] Event downloads work
- [ ] File permissions work

**Dependencies:**
- Attachment System (P2)

---

### 11.7 Other Custom Features

**Files:** `ptbgood.php`, `team.php`, `my.php`, `facenter.php`, `invite.php`, `repairpost.php`, `info.php`, `a.php`, `inccode.php`, `downloader.php`, `dzuser.php`

**Migration Effort:** 5-7 days (combined)

**Dependencies:**
- Various

---

## 12. UC Integration

### 12.1 UC Client

**Current File:** `/root/poketb-renew/poketb.com/bbs/uc_client/` directory

**Migration Effort:** 4 days

**Database Tables:**
- uc_* tables

**Testing Checklist:**
- [ ] User sync works
- [ ] PM sync works
- [ ] Friend sync works
- [ ] Avatar sync works

**Dependencies:**
- User Management (P1)
- PM System (P2)

---

### 12.2 PM Sync

**Migration Effort:** 1 day

**Dependencies:**
- UC Client (12.1)

---

### 12.3 Avatar Sync

**Migration Effort:** 1 day

**Dependencies:**
- UC Client (12.1)

---

### 12.4 Member Sync

**Migration Effort:** 1 day

**Dependencies:**
- UC Client (12.1)

---

## 13. Testing & Debug

### 13.1 PHP Info

**Current File:** `/root/poketb-renew/poketb.com/bbs/phpinfo.php`

**Migration Effort:** 0.5 days

**Dependencies:**
- None

---

### 13.2 Test DB

**Current File:** `/root/poketb-renew/poketb.com/bbs/test_db.php`

**Migration Effort:** 0.5 days

**Dependencies:**
- Database Layer (P0)

---

### 13.3 Test Robot

**Current File:** `/root/poketb-renew/poketb.com/bbs/test_robot.php`

**Migration Effort:** 0.5 days

**Dependencies:**
- Security System (P3)

---

## 14. Image Processing

### 14.1 Image Class

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/image.class.php`

**Lines:** 1-800

**Migration Effort:** 3 days

**Testing Checklist:**
- [ ] Image loading works
- [ ] Image resizing works
- [ ] Image cropping works
- [ ] Watermarking works
- [ ] Format conversion works

**Dependencies:**
- None (GD/Imagick)

---

### 14.2 GIF Merge

**Current File:** `/root/poketb-renew/poketb.com/bbs/include/gifmerge.class.php`

**Lines:** 1-400

**Migration Effort:** 1.5 days

**Dependencies:**
- Image Class (14.1)

---

### 14.3 Thumbnailer

**Current File:** Various (in attachment.func.php)

**Migration Effort:** Already covered in P2

---

## Summary: P3 Enhanced Features

**Total Features:** 150+ sub-features
**Estimated Time:** 4-6 weeks (30 working days)
**Dependencies:** P2 complete

### Week 15: Social + Content (Days 81-85)
- Day 81-82: User Space + Buddies
- Day 83: User Stats + Ranks
- Day 84: Digest + Tags
- Day 85: Announcements + FAQ + RSS

### Week 16: Security + Editor (Days 86-90)
- Day 86-87: CAPTCHA + Anti-Flood
- Day 88: Editor + BBCode
- Day 89-90: Rich Text Editor + Code Highlighting

### Week 17: Gamification (Days 91-95)
- Day 91-93: Magic Items
- Day 94: Medals + Credits
- Day 95: Bank System

### Week 18: Special Features (Days 96-100)
- Day 96: AJAX + Redirect
- Day 97: Misc Functions
- Day 98: Promotion + Cron
- Day 99-100: Template Engine + Language

### Week 19: XML + Archive + Plugins (Days 101-105)
- Day 101: XML Processing
- Day 102-103: Archiver + WAP
- Day 104-105: Plugin System + Core Plugins

### Week 20: Custom Features + UC (Days 106-120)
- Day 106-110: Pet System
- Day 111-113: DEX System
- Day 114-116: Helper + Campaign + Family
- Day 117-120: UC Integration + Testing

### Success Criteria
✅ Social features work
✅ Gamification works
✅ Templates can be customized
✅ Plugins load and work
✅ Mobile/archiver versions work
✅ Custom PoketTB features work
✅ UC integration works (if needed)
✅ All P3 tests pass

**Notes:**
- P3 features can be done incrementally
- Some custom features may be deprecated
- UC integration can be replaced with native auth
- Plugin system may need modernization

**Migration Complete:** All P0-P3 features migrated to PHP 8.x
