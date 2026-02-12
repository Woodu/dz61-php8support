# Complete Feature Inventory - Discuz! 6.1F Migration to PHP 8.x

**Generated:** 2026-02-13
**Total Files Analyzed:** 964 PHP files
**Legacy Path:** `/root/poketb-renew/poketb.com/bbs/`
**Goal:** Migrate to modern PHP 8.x with best practices

## Executive Summary

This inventory maps every legacy file to a feature, database tables, priority level, and dependencies. Features are prioritized as:

- **P0 (Critical):** Core authentication, session management, database connectivity
- **P1 (Core):** Forum listing, thread viewing, posting, user management
- **P2 (Important):** Search, moderation, attachments, PM system
- **P3 (Enhanced):** Plugins, social features, gamification, special features

---

## P0: CRITICAL INFRASTRUCTURE

### 1. Core Bootstrap & Configuration
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Global Bootstrap | `include/common.inc.php:1-500` | cdb_settings, cdb_sessions | P0 | None |
| Configuration | `config.inc.php:1-100` | None | P0 | None |
| Database Layer | `include/db_mysql.class.php:1-400` | All tables | P0 | None |
| Error Handling | `include/db_mysql_error.inc.php:1-50` | cdb_failedlogins | P0 | Database Layer |
| Global Functions | `include/global.func.php:1-2000` | Various | P0 | Core Bootstrap |
| Request Handling | `include/request.func.php:1-300` | None | P0 | Global Functions |
| Security System | `include/security.inc.php:1-200` | cdb_banned | P0 | Request Handling |

### 2. Authentication & Session Management
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| User Login | `logging.php:16-150`, `include/misc.func.php:100-300` | cdb_members, cdb_sessions, cdb_banned | P0 | Core Bootstrap, Database Layer |
| User Logout | `logging.php:16-35` | cdb_sessions | P0 | Session Management |
| Session Management | `include/common.inc.php:141-200` | cdb_sessions, cdb_memberfields | P0 | Database Layer |
| Cookie Management | `include/global.func.php:300-400` | None | P0 | Global Functions |
| Password Security | `logging.php:59-74` | cdb_members | P0 | User Login |
| UCenter Integration | `uc_client/client.php:1-2000` | uc_members, uc_sessions | P0 | User Login |

### 3. Caching System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Cache Core | `include/cache.func.php:1-800` | Multiple cache tables | P0 | Database Layer |
| Cache Settings | `forumdata/cache/cache_settings.php` | cdb_settings | P0 | Cache Core |
| Cache Forums | `forumdata/cache/cache_forums.php` | cdb_forums, cdb_forumfields | P0 | Cache Core |
| Cache Usergroups | `forumdata/cache/cache_usergroups.php` | cdb_usergroups | P0 | Cache Core |

---

## P1: CORE FORUM FEATURES

### 4. Forum Navigation & Display
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Forum Index | `index.php:1-300` | cdb_forums, cdb_forumfields, cdb_threads | P1 | Caching System, Authentication |
| Forum Category List | `include/category.inc.php:1-200` | cdb_forums | P1 | Forum Index |
| Forum Display | `forumdisplay.php:1-500` | cdb_forums, cdb_threads, cdb_posts | P1 | Forum Index, Permissions |
| Forum Functions | `include/forum.func.php:1-1000` | cdb_forums | P1 | Core Bootstrap |
| Sub-forum Navigation | `forumdisplay.php:200-300` | cdb_forums | P1 | Forum Display |

### 5. Thread Management
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| View Thread | `viewthread.php:1-600` | cdb_threads, cdb_posts, cdb_attachments | P1 | Forum Display, Permissions |
| New Thread | `include/newthread.inc.php:1-400` | cdb_threads, cdb_posts, cdb_moderates | P1 | Post Functions, Authentication |
| Thread Display Logic | `include/viewthread_special.inc.php:1-300` | cdb_threads | P1 | View Thread |
| Thread Polls | `include/viewthread_poll.inc.php:1-200` | cdb_threads, cdb_polloptions, cdb_polls | P1 | View Thread |
| Thread Payment | `include/threadpay.inc.php:1-150` | cdb_threads, cdb_creditslog | P1 | View Thread, Credits System |

### 6. Posting System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Post Controller | `post.php:1-600` | cdb_threads, cdb_posts | P1 | Authentication, Forum Display |
| New Reply | `include/newreply.inc.php:1-350` | cdb_threads, cdb_posts, cdb_attachments | P1 | Post Functions, Thread Management |
| Edit Post | `include/editpost.inc.php:1-500` | cdb_posts, cdb_threads | P1 | Post Controller, Permissions |
| Post Functions | `include/post.func.php:1-1500` | cdb_threads, cdb_posts, cdb_attachments | P1 | Global Functions |
| Attachment Upload | `include/attachment.func.php:1-800` | cdb_attachments | P1 | Post Functions |
| DiscuzCode Parser | `include/discuzcode.func.php:1-1200` | None | P1 | Global Functions |

### 7. User Management
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Member Profile | `member.php:1-800` | cdb_members, cdb_memberfields | P1 | Authentication |
| Member List | `member.php:54-200` | cdb_members | P1 | User Management |
| Online Users | `member.php:24-52`, `include/counter.inc.php:1-200` | cdb_sessions | P1 | Session Management |
| User CP | `memcp.php:1-400` | cdb_members, cdb_memberfields | P1 | Authentication |
| User Registration | `register.php:1-600` (in misc.php) | cdb_members, cdb_memberfields | P1 | UCenter Integration |
| Avatar System | `uc_client/avatar.php:1-200` | cdb_memberfields | P1 | User Management |

---

## P2: IMPORTANT FEATURES

### 8. Moderation System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| ModCP Index | `modcp.php:1-300`, `modcp/index.php:1-100` | cdb_moderators, cdb_threads, cdb_posts | P2 | Authentication, Permissions |
| ModCP Login | `modcp/login.inc.php:1-100` | cdb_moderators | P2 | Authentication |
| Thread Moderation | `modcp/moderate.inc.php:1-400` | cdb_threads, cdb_moderates | P2 | ModCP Index |
| Post Moderation | `include/moderation.inc.php:1-500` | cdb_posts, cdb_moderates | P2 | Thread Moderation |
| Forum Access Control | `modcp/forumaccess.inc.php:1-200` | cdb_access | P2 | ModCP Index |
| Member Banning | `modcp/members.inc.php:1-300` | cdb_banned, cdb_members | P2 | ModCP Index |
| Report System | `modcp/report.inc.php:1-200` | cdb_reports | P2 | ModCP Index |
| ModCP Logs | `modcp/logs.inc.php:1-150` | cdb_moderatorlogs | P2 | ModCP Index |
| Prune Threads | `modcp/prune.inc.php:1-300` | cdb_threads, cdb_posts | P2 | ModCP Index |
| Edit Post (Mod) | `modcp/editpost.inc.php:1-200` | cdb_posts | P2 | ModCP Index |

### 9. Search System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Search Controller | `search.php:1-500` | cdb_threads, cdb_posts, cdb_searchindex | P2 | Forum Display |
| Fulltext Search | `search.php:100-300` | cdb_threads, cdb_posts | P2 | Search Controller |
| Search Cache | `forumdata/cache/cache_search.php` | cdb_searchindex | P2 | Caching System |

### 10. Private Messaging
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| PM System | `pm.php:1-600` | cdb_pms, cdb_pmfields | P2 | User Management |
| PM Sending | `pm.php:200-400` | cdb_pms | P2 | PM System |
| PM Folder Management | `pm.php:400-550` | cdb_pms | P2 | PM System |
| PM Blacklist | `pm.php:550-600` | cdb_pms | P2 | PM System |

### 11. Attachment System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Attachment Handler | `attachment.php:1-400` | cdb_attachments, cdb_attachmentfields | P2 | Posting System |
| Thumbnail Generation | `include/image.class.php:1-600` | None | P2 | Attachment Handler |
| Image Processing | `include/image.class.php:100-500` | None | P2 | Attachment Handler |

### 12. Admin Control Panel
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| AdminCP Login | `admin/login.inc.php:1-150` | cdb_members, cdb_adminsessions | P2 | Authentication |
| AdminCP Index | `admin/main.inc.php:1-200` | Multiple | P2 | AdminCP Login |
| Forum Management | `admin/forums.inc.php:1-800` | cdb_forums, cdb_forumfields | P2 | AdminCP Index |
| User Group Management | `admin/groups.inc.php:1-600` | cdb_usergroups | P2 | AdminCP Index |
| Member Management | `admin/members.inc.php:1-700` | cdb_members, cdb_memberfields | P2 | AdminCP Index |
| Database Management | `admin/database.inc.php:1-1000` | All tables | P2 | AdminCP Index |
| Settings Management | `admin/settings.inc.php:1-1500` | cdb_settings | P2 | AdminCP Index |
| Template Management | `admin/templates.inc.php:1-600` | cdb_templates | P2 | AdminCP Index |
| Plugin Management | `admin/plugins.inc.php:1-500` | cdb_plugins | P2 | AdminCP Index |
| Announcement Mgmt | `admin/announcements.inc.php:1-300` | cdb_announcements | P2 | AdminCP Index |
| Thread Management | `admin/threads.inc.php:1-400` | cdb_threads | P2 | AdminCP Index |
| Attachment Mgmt | `admin/attachments.inc.php:1-300` | cdb_attachments | P2 | AdminCP Index |
| Moderation Queue | `admin/moderate.inc.php:1-300` | cdb_moderates | P2 | AdminCP Index |
| Statistics | `admin/counter.inc.php:1-400` | Multiple | P2 | AdminCP Index |
| Prune Tools | `admin/prune.inc.php:1-400` | cdb_threads, cdb_posts | P2 | AdminCP Index |
| Logs Management | `admin/logs.inc.php:1-300` | cdb_adminlogs | P2 | AdminCP Index |
| Smilies Management | `admin/smilies.inc.php:1-300` | cdb_smilies | P2 | AdminCP Index |
| FAQ Management | `admin/faq.inc.php:1-300` | cdb_faqs | P2 | AdminCP Index |
| Advertisement Mgmt | `admin/advertisements.inc.php:1-600` | cdb_advertisements | P2 | AdminCP Index |
| Credit System | `admin/creditwizard.inc.php:1-500` | cdb_settings | P2 | AdminCP Index |
| Medals Management | `admin/medals.inc.php:1-400` | cdb_medals, cdb_memberfields | P2 | AdminCP Index |
| Magic Items | `admin/magics.inc.php:1-500` | cdb_magics, cdb_membermagic | P2 | AdminCP Index |
| JS/CSS Wizard | `admin/jswizard.inc.php:1-300` | cdb_settings | P2 | AdminCP Index |
| Run Wizard | `admin/runwizard.inc.php:1-200` | cdb_settings | P2 | AdminCP Index |
| Tools | `admin/tools.inc.php:1-400` | Multiple | P2 | AdminCP Index |
| Project/Update | `admin/project.inc.php:1-300` | None | P2 | AdminCP Index |
| Quick Queries | `admin/quickqueries.inc.php:1-200` | Multiple | P2 | AdminCP Index |
| Recycle Bin | `admin/recyclebin.inc.php:1-400` | cdb_threads, cdb_posts | P2 | AdminCP Index |
| Check Tools | `admin/checktools.inc.php:1-300` | None | P2 | AdminCP Index |
| Styles Management | `admin/styles.inc.php:1-500` | cdb_styles | P2 | AdminCP Index |
| Thread Types | `admin/threadtypes.inc.php:1-400` | cdb_threadtypes | P2 | AdminCP Index |
| Home Page Config | `admin/home.inc.php:1-200` | cbd_settings | P2 | AdminCP Index |
| Video Management | `admin/video.inc.php:1-300` | cdb_threads | P2 | AdminCP Index |
| Menu Management | `admin/menu.inc.php:1-200` | None | P2 | AdminCP Index |
| Misc Settings | `admin/misc.inc.php:1-500` | cdb_settings | P2 | AdminCP Index |

---

## P3: ENHANCED FEATURES

### 13. Social Features
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| User Space | `space.php:1-500` | cdb_members, cdb_memberfields | P3 | User Management |
| User Profile View | `space.php:100-400` | cdb_memberfields | P3 | User Space |
| Buddy List | `memcp.php:200-300` | cdb_buddys | P3 | User Management |
| User Statistics | `stats.php:1-400` | cdb_threads, cdb_posts, cdb_members | P3 | Forum Index |
| Rank List | `ranklist.php:1-300` | cdb_members, cdb_threads | P3 | User Management |
| Digest Display | `digest.php:1-300` | cdb_threads | P3 | Forum Index |

### 14. Content Features
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Announcements | `announcement.php:1-200` | cdb_announcements | P3 | Forum Index |
| FAQ System | `faq.php:1-300` | cdb_faqs | P3 | None |
| RSS Feed | `rss.php:1-200` | cdb_threads, cdb_posts | P3 | Forum Display |
| Sitemap | `sitemap.php:1-300` | cdb_threads | P3 | Forum Index |
| Print View | `include/printable.inc.php:1-150` | cdb_threads, cdb_posts | P3 | View Thread |
| Topic Display | `topic.php:1-200` | cdb_threads | P3 | Forum Display |

### 15. Security & Anti-Spam
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| CAPTCHA/Seccode | `seccode.class.php:1-400` | None | P3 | Core Bootstrap |
| Seccode Display | `seccode.php:1-100` | None | P3 | CAPTCHA System |
| Anti-Flood Control | `include/security.inc.php:50-150` | cdb_banned | P3 | Security System |
| Login Protection | `logging.php:52-74` | cdb_failedlogins | P3 | User Login |
| IP Banning | `include/common.inc.php:200-250` | cdb_banned | P3 | Core Bootstrap |

### 16. Editor & Formatting
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| BBCode Editor | `include/editor.func.php:1-600` | None | P3 | DiscuzCode Parser |
| Rich Text Editor | `include/editor.func.php:100-500` | None | P3 | BBCode Editor |
| Code Highlighting | `include/discuzcode.func.php:800-1200` | None | P3 | DiscuzCode Parser |
| Custom BBCode | `include/xsecode.func.php:1-300` | None | P3 | DiscuzCode Parser |
| Diff Engine | `include/diff.class.php:1-500` | None | P3 | Global Functions |

### 17. Gamification
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Magic Items | `magic.php:1-400` | cdb_magics, cdb_membermagic | P3 | User Management |
| Magic Functions | `include/magic.func.php:1-800` | cdb_magics | P3 | Magic Items |
| Medal System | `medal.php:1-300` | cdb_medals, cdb_memberfields | P3 | User Management |
| Credit System | `misc.php:150-300` | cdb_creditslog | P3 | User Management |
| Bank System | `bank.php:1-500`, `bank2.php:1-300`, `bank3.php:1-300` | Custom tables | P3 | Credit System |

### 18. Special Features
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| AJAX Handler | `ajax.php:1-400` | Various | P3 | Core Bootstrap |
| Redirect System | `redirect.php:1-150` | None | P3 | Global Functions |
| Help System | `help.php:1-200` | None | P3 | None |
| Misc Functions | `misc.php:1-1000` | Various | P3 | Multiple Features |
| Promotion System | `include/promotion.inc.php:1-200` | cdb_promotions | P3 | User Management |
| Cron System | `include/cron.func.php:1-400` | cdb_crons | P3 | Caching System |
| Email Sending | `include/sendmail.inc.php:1-300` | None | P3 | Global Functions |

### 19. Template & Theme System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Template Engine | `include/template.func.php:1-600` | cdb_templates | P3 | Caching System |
| Chinese Language | `include/chinese.class.php:1-400` | None | P3 | Template Engine |
| Multi-byte Support | `include/chinese.class.php:50-300` | None | P3 | Chinese Language |
| Frame Support | `frame.php:1-200` | None | P3 | Template Engine |
| Left Menu | `leftmenu.php:1-150` | None | P3 | Template Engine |

### 20. XML & API
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| XML Core | `include/xml.class.php:1-400` | None | P3 | None |
| XML Parser | `include/xmlparser.class.php:1-300` | None | P3 | XML Core |
| API Handler | `api/` directory files | Various | P3 | Authentication |

### 21. Archive & Mobile
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Archiver | `archiver/` directory files | cdb_threads, cdb_posts | P3 | Forum Display |
| WAP Support | `wap/` directory files | Various | P3 | Forum Display |
| Mobile Access | `archiver/index.php:1-200` | cdb_forums | P3 | Archiver |

### 22. Plugin System
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Plugin Handler | `plugin.php:1-50` | cdb_plugins | P3 | Core Bootstrap |
| Bank Plugin | `plugins/bank/` directory | Custom tables | P3 | Credit System |
| Google Sitemap | `plugins/googlesitemap/` directory | None | P3 | Sitemap |
| Admin Plugins | `plugins/admin/` directory | Various | P3 | AdminCP Index |
| MOC Plugin | `plugins/moc/` directory | Custom tables | P3 | None |

### 23. Custom/PoketTB Specific
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Pet System | `pet/`, `zpet/`, `mdex/` directories | Custom tables | P3 | User Management |
| Pet Center | `petcenter.php:1-400` | Custom tables | P3 | Pet System |
| DEX System | `dex.php:1-300`, `mdex.php:1-300` | Custom tables | P3 | Pet System |
| DEX API | `dexAPI.inc.php:1-200`, `dexajax.php:1-150` | Custom tables | P3 | DEX System |
| Helper System | `helper*.php` files | Custom tables | P3 | User Management |
| Campaign | `campaign.php:1-200` | Custom tables | P3 | User Management |
| Family System | `family/` directory | Custom tables | P3 | User Management |
| Event Download | `eventdownload.php:1-200` | Custom tables | P3 | Forum System |
| Ptb Good | `ptbgood.php:1-300` | Custom tables | P3 | Forum System |
| Team | `team.php:1-200` | Custom tables | P3 | User Management |
| My Page | `my.php:1-300` | Custom tables | P3 | User Management |
| FA Center | `facenter.php:1-200` | Custom tables | P3 | User Management |
| Invite System | `invite.php:1-200` | Custom tables | P3 | User Management |
| Repair Post | `repairpost.php:1-150` | cdb_posts | P3 | Posting System |
| User Info | `info.php:1-100` | cdb_members | P3 | User Management |
| A.php | `a.php:1-100` | Custom tables | P3 | Custom |
| Inccode | `inccode.php:1-150` | None | P3 | Security |
| Downloader | `downloader.php:1-200` | cdb_attachments | P3 | Attachment System |
| Frame/Menu | `frame.php`, `leftmenu.php` | None | P3 | Template System |
| DZ User | `dzuser.php:1-200` | cdb_members | P3 | User Management |
| Test Files | `test*.php` | Various | P3 | None |
| 404 Handler | `404.php:1-50` | None | P3 | Core Bootstrap |

### 24. UC Integration
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| UC Client | `uc_client/` directory (1000+ lines) | uc_* tables | P3 | Authentication |
| UCenter | `ucenter/` directory | uc_* tables | P3 | UC Client |
| PM Sync | `uc_client/client.php:1000-1500` | uc_pms | P3 | PM System |
| Avatar Sync | `uc_client/avatar.php` | uc_memberfields | P3 | User Management |

### 25. Testing & Debug
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| PHP Info | `phpinfo.php:1-20` | None | P3 | None |
| Test DB | `test_db.php:1-100` | cdb_settings | P3 | Database Layer |
| Test Robot | `test_robot.php:1-100` | None | P3 | Security System |
| Test UC | `test_uc.php:1-150` | uc_* tables | P3 | UC Integration |

### 26. Image Processing
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Image Class | `include/image.class.php:1-800` | None | P3 | Attachment System |
| GIF Merge | `include/gifmerge.class.php:1-400` | None | P3 | Image Processing |
| Thumbnailer | `include/attachment.func.php:200-400` | None | P3 | Image Class |

### 27. Advanced Features
| Feature | Files | Tables | Priority | Dependencies |
|---------|--------|---------|----------|--------------|
| Tag System | `misc.php:400-600` | cdb_tags, cdb_threadtags | P3 | Thread Management |
| Subscription | `misc.php:600-800` | cdb_subscriptions | P3 | User Management |
| Report System | `misc.php:800-1000` | cdb_reports | P3 | Moderation |
| Friend System | `memcp.php:300-400` | cdb_buddys | P3 | Social Features |
| Block List | `memcp.php:400-500` | cdb_blocklist | P3 | PM System |
| Custom Avatar | `memcp.php:500-600` | cdb_memberfields | P3 | User Management |

---

## Summary Statistics

**Total Files:** 964
- Root PHP: 74 files
- Include PHP: 42 files
- Admin modules: 37 files
- ModCP modules: 13 files
- Plugins: 150+ files
- UC Client: 100+ files
- Template/cache: 500+ files

**Total Features:** 27 major feature groups
- P0 (Critical): 3 groups (9 features)
- P1 (Core): 4 groups (18 features)
- P2 (Important): 5 groups (40 features)
- P3 (Enhanced): 18 groups (150+ features)

**Database Tables:** 80+ core tables

**Estimated Migration Effort:**
- P0: 2-3 weeks
- P1: 4-5 weeks
- P2: 3-4 weeks
- P3: 4-6 weeks
- Testing: 2-3 weeks
- **Total:** 15-21 weeks

**Next Steps:**
1. Review P0 critical path (Document 01)
2. Plan P1 core features (Document 02)
3. Schedule P2 important features (Document 03)
4. Plan P3 enhanced features (Document 04)
5. Create dependency graph (Document 05)
6. Define execution sprints (Document 06)
