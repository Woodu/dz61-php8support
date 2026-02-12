# File Migration Checklist - Complete File-by-File Status

**Purpose:** Track migration status of every PHP file
**Total Files:** 964 PHP files
**Last Updated:** 2026-02-13

**Status Legend:**
- ⏳ **Pending** - Not started
- 🔄 **In Progress** - Currently being migrated
- ✅ **Complete** - Migrated successfully
- ⚠️ **Blocked** - Blocked by dependency
- ❌ **Deprecated** - Will not migrate (obsolete)

---

## Root PHP Files (74 files)

| File | Lines | Priority | Status | Target Sprint | Notes |
|------|-------|----------|--------|---------------|-------|
| index.php | 300 | P1 | ⏳ | Sprint 4 | Forum Index |
| forumdisplay.php | 500 | P1 | ⏳ | Sprint 5 | Forum Display |
| viewthread.php | 600 | P1 | ⏳ | Sprint 6 | Thread View |
| post.php | 600 | P1 | ⏳ | Sprint 7 | Post Controller |
| logging.php | 200 | P0 | ⏳ | Sprint 3 | Login/Logout |
| member.php | 800 | P1 | ⏳ | Sprint 10 | User Management |
| memcp.php | 400 | P1 | ⏳ | Sprint 10 | User CP |
| pm.php | 600 | P2 | ⏳ | Sprint 12 | PM System |
| search.php | 500 | P2 | ⏳ | Sprint 12 | Search |
| admincp.php | 100 | P2 | ⏳ | Sprint 14 | AdminCP |
| modcp.php | 300 | P2 | ⏳ | Sprint 11 | ModCP |
| plugin.php | 50 | P3 | ⏳ | Sprint 19 | Plugin Handler |
| attachment.php | 400 | P2 | ⏳ | Sprint 13 | Attachments |
| ajax.php | 400 | P3 | ⏳ | Sprint 17 | AJAX |
| misc.php | 1000 | P3 | ⏳ | Sprint 17 | Misc Functions |
| rss.php | 200 | P3 | ⏳ | Sprint 15 | RSS Feed |
| sitemap.php | 300 | P3 | ⏳ | Sprint 15 | Sitemap |
| announcement.php | 200 | P3 | ⏳ | Sprint 15 | Announcements |
| faq.php | 300 | P3 | ⏳ | Sprint 15 | FAQ |
| seccode.php | 100 | P3 | ⏳ | Sprint 17 | CAPTCHA |
| space.php | 500 | P3 | ⏳ | Sprint 15 | User Space |
| stats.php | 400 | P3 | ⏳ | Sprint 15 | Statistics |
| ranklist.php | 300 | P3 | ⏳ | Sprint 15 | Rank List |
| digest.php | 300 | P3 | ⏳ | Sprint 15 | Digest |
| topic.php | 200 | P3 | ⏳ | Sprint 15 | Topics |
| redirect.php | 150 | P3 | ⏳ | Sprint 17 | Redirect |
| help.php | 200 | P3 | ⏳ | Sprint 17 | Help |
| frame.php | 200 | P3 | ⏳ | Sprint 18 | Frame |
| leftmenu.php | 150 | P3 | ⏳ | Sprint 18 | Left Menu |
| magic.php | 400 | P3 | ⏳ | Sprint 16 | Magic Items |
| medal.php | 300 | P3 | ⏳ | Sprint 16 | Medals |
| bank.php | 500 | P3 | ⏳ | Sprint 16 | Bank |
| bank2.php | 300 | P3 | ⏳ | Sprint 16 | Bank Alt |
| bank3.php | 300 | P3 | ⏳ | Sprint 16 | Bank Alt 2 |
| petcenter.php | 400 | P3 | ⏳ | Sprint 20 | Pet Center |
| dex.php | 300 | P3 | ⏳ | Sprint 20 | DEX |
| mdex.php | 300 | P3 | ⏳ | Sprint 20 | Mini DEX |
| campaign.php | 200 | P3 | ⏳ | Sprint 20 | Campaign |
| invite.php | 200 | P3 | ⏳ | Sprint 20 | Invite |
| repairpost.php | 150 | P3 | ⏳ | Sprint 20 | Repair Post |
| info.php | 100 | P3 | ⏳ | Sprint 20 | User Info |
| a.php | 100 | P3 | ⏳ | Sprint 20 | Custom A |
| inccode.php | 150 | P3 | ⏳ | Sprint 17 | Include Code |
| downloader.php | 200 | P3 | ⏳ | Sprint 20 | Downloader |
| dzuser.php | 200 | P3 | ⏳ | Sprint 20 | DZ User |
| facenter.php | 200 | P3 | ⏳ | Sprint 20 | FA Center |
| my.php | 300 | P3 | ⏳ | Sprint 20 | My Page |
| team.php | 200 | P3 | ⏳ | Sprint 20 | Team |
| ptbgood.php | 300 | P3 | ⏳ | Sprint 20 | PTB Good |
| eventdownload.php | 200 | P3 | ⏳ | Sprint 20 | Event DL |
| topicadmin.php | 300 | P2 | ⏳ | Sprint 11 | Topic Admin |
| 404.php | 50 | P3 | ⏳ | Sprint 17 | 404 Handler |
| phpinfo.php | 20 | P3 | ⏳ | Sprint 20 | PHP Info |
| test.php | 100 | P3 | ⏳ | Sprint 20 | Test |
| test2.php | 100 | P3 | ❌ | - | Test File (skip) |
| test3.php | 100 | P3 | ❌ | - | Test File (skip) |
| test_db.php | 100 | P3 | ❌ | - | Test DB (skip) |
| test_robot.php | 100 | P3 | ❌ | - | Test Robot (skip) |
| test_uc.php | 150 | P3 | ❌ | - | Test UC (skip) |
| dexajax.php | 150 | P3 | ⏳ | Sprint 20 | DEX AJAX |
| plantajax.php | 100 | P3 | ⏳ | Sprint 20 | Plant AJAX |
| dexAPI.inc.php | 200 | P3 | ⏳ | Sprint 20 | DEX API |
| discuz_version.php | 20 | P0 | ⏳ | Sprint 1 | Version |
| config.inc.php | 100 | P0 | ⏳ | Sprint 1 | Config |
| mysql_compat.php | 100 | P0 | ⏳ | Sprint 2 | MySQL Compat |
| mysql_compat.inc.php | 50 | P0 | ⏳ | Sprint 2 | MySQL Compat Inc |

---

## include/ Directory (42 files)

| File | Lines | Priority | Status | Target Sprint | Notes |
|------|-------|----------|--------|---------------|-------|
| common.inc.php | 500 | P0 | ⏳ | Sprint 1 | Bootstrap |
| global.func.php | 2000 | P0 | ⏳ | Sprint 1 | Global Functions |
| db_mysql.class.php | 400 | P0 | ⏳ | Sprint 2 | Database Layer |
| cache.func.php | 800 | P0 | ⏳ | Sprint 3 | Cache System |
| session.inc.php | 200 | P0 | ⏳ | Sprint 3 | Session |
| security.inc.php | 200 | P0 | ⏳ | Sprint 3 | Security |
| request.func.php | 300 | P0 | ⏳ | Sprint 1 | Request |
| db_mysql_error.inc.php | 50 | P0 | ⏳ | Sprint 2 | DB Error |
| forum.func.php | 1000 | P1 | ⏳ | Sprint 4 | Forum Functions |
| post.func.php | 1500 | P1 | ⏳ | Sprint 9 | Post Functions |
| discuzcode.func.php | 1200 | P1 | ⏳ | Sprint 9 | BBCode Parser |
| attachment.func.php | 800 | P2 | ⏳ | Sprint 13 | Attachments |
| editor.func.php | 600 | P3 | ⏳ | Sprint 17 | Editor |
| template.func.php | 600 | P3 | ⏳ | Sprint 18 | Templates |
| chinese.class.php | 400 | P3 | ⏳ | Sprint 18 | Chinese |
| image.class.php | 800 | P2 | ⏳ | Sprint 13 | Image Processing |
| seccode.class.php | 400 | P3 | ⏳ | Sprint 17 | CAPTCHA |
| xml.class.php | 400 | P3 | ⏳ | Sprint 19 | XML |
| xmlparser.class.php | 300 | P3 | ⏳ | Sprint 19 | XML Parser |
| diff.class.php | 500 | P3 | ⏳ | Sprint 17 | Diff |
| gifmerge.class.php | 400 | P3 | ⏳ | Sprint 13 | GIF Merge |
| magic.func.php | 800 | P3 | ⏳ | Sprint 16 | Magic Functions |
| misc.func.php | 1000 | P3 | ⏳ | Sprint 17 | Misc Functions |
| newthread.inc.php | 400 | P1 | ⏳ | Sprint 7 | New Thread |
| newreply.inc.php | 350 | P1 | ⏳ | Sprint 8 | New Reply |
| editpost.inc.php | 500 | P1 | ⏳ | Sprint 8 | Edit Post |
| moderation.inc.php | 500 | P2 | ⏳ | Sprint 11 | Moderation |
| category.inc.php | 200 | P1 | ⏳ | Sprint 4 | Categories |
| counter.inc.php | 200 | P3 | ⏳ | Sprint 15 | Counter |
| cron.func.php | 400 | P3 | ⏳ | Sprint 17 | Cron |
| advertisements.inc.php | 300 | P3 | ⏳ | Sprint 15 | Ads |
| promotion.inc.php | 200 | P3 | ⏳ | Sprint 17 | Promotion |
| sendmail.inc.php | 300 | P3 | ⏳ | Sprint 17 | Email |
| printable.inc.php | 150 | P3 | ⏳ | Sprint 15 | Print View |
| threadpay.inc.php | 150 | P1 | ⏳ | Sprint 6 | Thread Pay |
| viewthread_poll.inc.php | 200 | P1 | ⏳ | Sprint 6 | Polls |
| viewthread_special.inc.php | 300 | P1 | ⏳ | Sprint 6 | Special |
| xsecode.func.php | 300 | P3 | ⏳ | Sprint 17 | Custom BBCode |
| zpetrandomgoods.inc.php | 200 | P3 | ⏳ | Sprint 20 | Pet Random |
| ftp.func.php | 300 | P3 | ⏳ | Sprint 13 | FTP |
| ranklist.inc.php | 200 | P3 | ⏳ | Sprint 15 | Rank List |

---

## admin/ Directory (37 files)

| File | Lines | Priority | Status | Target Sprint | Notes |
|------|-------|----------|--------|---------------|-------|
| main.inc.php | 200 | P2 | ⏳ | Sprint 14 | Admin Index |
| login.inc.php | 150 | P2 | ⏳ | Sprint 14 | Admin Login |
| forums.inc.php | 800 | P2 | ⏳ | Sprint 14 | Forums |
| groups.inc.php | 600 | P2 | ⏳ | Sprint 14 | User Groups |
| members.inc.php | 700 | P2 | ⏳ | Sprint 14 | Members |
| database.inc.php | 1000 | P2 | ⏳ | Sprint 14 | Database |
| settings.inc.php | 1500 | P2 | ⏳ | Sprint 14 | Settings |
| templates.inc.php | 600 | P2 | ⏳ | Sprint 18 | Templates |
| plugins.inc.php | 500 | P2 | ⏳ | Sprint 19 | Plugins |
| announcements.inc.php | 300 | P2 | ⏳ | Sprint 14 | Announcements |
| threads.inc.php | 400 | P2 | ⏳ | Sprint 14 | Threads |
| attachments.inc.php | 300 | P2 | ⏳ | Sprint 13 | Attachments |
| moderate.inc.php | 300 | P2 | ⏳ | Sprint 11 | Moderation |
| counter.inc.php | 400 | P2 | ⏳ | Sprint 14 | Statistics |
| prune.inc.php | 400 | P2 | ⏳ | Sprint 11 | Prune |
| logs.inc.php | 300 | P2 | ⏳ | Sprint 14 | Logs |
| smilies.inc.php | 300 | P2 | ⏳ | Sprint 14 | Smilies |
| faq.inc.php | 300 | P2 | ⏳ | Sprint 15 | FAQ |
| advertisements.inc.php | 600 | P2 | ⏳ | Sprint 14 | Ads |
| creditwizard.inc.php | 500 | P2 | ⏳ | Sprint 16 | Credits |
| medals.inc.php | 400 | P2 | ⏳ | Sprint 16 | Medals |
| magics.inc.php | 500 | P2 | ⏳ | Sprint 16 | Magic |
| jswizard.inc.php | 300 | P2 | ⏳ | Sprint 18 | JS Wizard |
| runwizard.inc.php | 200 | P2 | ⏳ | Sprint 14 | Run Wizard |
| tools.inc.php | 400 | P2 | ⏳ | Sprint 14 | Tools |
| project.inc.php | 300 | P2 | ⏳ | Sprint 14 | Project |
| quickqueries.inc.php | 200 | P2 | ⏳ | Sprint 14 | Quick Queries |
| recyclebin.inc.php | 400 | P2 | ⏳ | Sprint 14 | Recycle Bin |
| checktools.inc.php | 300 | P2 | ⏳ | Sprint 14 | Check Tools |
| styles.inc.php | 500 | P2 | ⏳ | Sprint 18 | Styles |
| threadtypes.inc.php | 400 | P2 | ⏳ | Sprint 14 | Thread Types |
| home.inc.php | 200 | P2 | ⏳ | Sprint 14 | Home |
| video.inc.php | 300 | P2 | ⏳ | Sprint 14 | Video |
| menu.inc.php | 200 | P2 | ⏳ | Sprint 14 | Menu |
| misc.inc.php | 500 | P2 | ⏳ | Sprint 14 | Misc |

---

## modcp/ Directory (13 files)

| File | Lines | Priority | Status | Target Sprint | Notes |
|------|-------|----------|--------|---------------|-------|
| index.php | 100 | P2 | ⏳ | Sprint 11 | ModCP Index |
| login.inc.php | 100 | P2 | ⏳ | Sprint 11 | ModCP Login |
| home.inc.php | 100 | P2 | ⏳ | Sprint 11 | ModCP Home |
| moderate.inc.php | 400 | P2 | ⏳ | Sprint 11 | Moderate |
| editpost.inc.php | 200 | P2 | ⏳ | Sprint 11 | Edit Post |
| forumaccess.inc.php | 200 | P2 | ⏳ | Sprint 11 | Forum Access |
| forums.inc.php | 100 | P2 | ⏳ | Sprint 11 | Forums |
| members.inc.php | 300 | P2 | ⏳ | Sprint 11 | Members |
| logs.inc.php | 150 | P2 | ⏳ | Sprint 11 | Logs |
| report.inc.php | 200 | P2 | ⏳ | Sprint 11 | Reports |
| prune.inc.php | 300 | P2 | ⏳ | Sprint 11 | Prune |
| announcements.inc.php | 100 | P2 | ⏳ | Sprint 11 | Announcements |
| noperm.inc.php | 50 | P2 | ⏳ | Sprint 11 | No Perm |

---

## Plugins (150+ files)

### Core Plugins
| Plugin | Files | Priority | Status | Target Sprint | Notes |
|--------|--------|----------|--------|---------------|-------|
| Bank | 20 files | P3 | ⏳ | Sprint 16 | Bank System |
| Google Sitemap | 5 files | P3 | ⏳ | Sprint 15 | Sitemap |
| Admin Plugins | 30 files | P2 | ⏳ | Sprint 14 | Admin |
| MOC | 10 files | P3 | ⏳ | Sprint 20 | Custom |

### Custom/PoketTB Plugins
| Plugin | Files | Priority | Status | Target Sprint | Notes |
|--------|--------|----------|--------|---------------|-------|
| Pet System | 50 files | P3 | ⏳ | Sprint 20 | Pets |
| DEX System | 30 files | P3 | ⏳ | Sprint 20 | DEX |
| Helper | 15 files | P3 | ⏳ | Sprint 20 | Helper |
| Family | 20 files | P3 | ⏳ | Sprint 20 | Family |
| Campaign | 10 files | P3 | ⏳ | Sprint 20 | Campaign |
| Other Custom | 40 files | P3 | ⏳ | Sprint 20 | Various |

---

## UC Client (100+ files)

| Component | Files | Priority | Status | Target Sprint | Notes |
|-----------|--------|----------|--------|---------------|-------|
| Client Core | 50 files | P3 | ⏳ | Sprint 19 | UC Client |
| Avatar | 10 files | P3 | ⏳ | Sprint 19 | Avatar |
| PM | 20 files | P3 | ⏳ | Sprint 19 | PM Sync |
| User | 20 files | P3 | ⏳ | Sprint 19 | User Sync |

---

## Template Files (500+ files)

| Type | Count | Priority | Status | Target Sprint | Notes |
|------|-------|----------|--------|---------------|-------|
| .htm Templates | 400+ | P1-P3 | ⏳ | Sprint 18 | All |
| CSS Files | 50+ | P3 | ⏳ | Sprint 18 | Styles |
| JS Files | 30+ | P3 | ⏳ | Sprint 18 | Scripts |
| Images | 1000+ | P3 | ⏳ | - | Assets |

---

## Summary Statistics

**By Status:**
- ⏳ Pending: 964 files (100%)
- 🔄 In Progress: 0 files (0%)
- ✅ Complete: 0 files (0%)
- ⚠️ Blocked: 0 files (0%)
- ❌ Deprecated: 5 files (<1%)

**By Priority:**
- P0: 9 files (1%)
- P1: 22 files (2%)
- P2: 50 files (5%)
- P3: 883 files (92%)

**By Sprint:**
- Sprint 1: 2 files
- Sprint 2: 2 files
- Sprint 3: 3 files
- Sprint 4: 2 files
- Sprint 5: 1 file
- Sprint 6: 2 files
- Sprint 7: 1 file
- Sprint 8: 2 files
- Sprint 9: 2 files
- Sprint 10: 2 files
- Sprint 11: 7 files
- Sprint 12: 2 files
- Sprint 13: 3 files
- Sprint 14: 20+ files
- Sprint 15: 10+ files
- Sprint 16: 5 files
- Sprint 17: 8 files
- Sprint 18: 5 files
- Sprint 19: 10+ files
- Sprint 20: 100+ files

---

## Tracking Notes

**Update this document after each sprint:**
1. Mark completed files as ✅
2. Update in-progress files to 🔄
3. Note any blocked files
4. Add any newly discovered files
5. Update completion percentage

**Completion Criteria:**
- File migrated to PHP 8.3
- No PHP 4/5 code remaining
- Type hints added
- Tests written
- Code reviewed
- Documentation updated

---

**Document Status:** ✅ Complete
**Next Document:** 08-gbk-to-utf8-detailed.md
