# Execution Sprints - Detailed Week-by-Week Plan

**Purpose:** Break down migration into manageable weekly sprints
**Total Duration:** 20 weeks (100 working days)
**Team:** 5-6 people

---

## Quick Reference

**Sprints:** 20 (1 week each)
**Total Days:** 100 working days
**Phases:**
- Phase 1: P0 Critical (Sprints 1-3, Days 1-15)
- Phase 2: P1 Core (Sprints 4-10, Days 16-50)
- Phase 3: P2 Important (Sprints 11-14, Days 51-70)
- Phase 4: P3 Enhanced (Sprints 15-20, Days 71-100)

---

## Sprint Structure

**Monday:** Planning (2h)
**Tuesday-Thursday:** Development & Testing
**Friday:** Review & Retrospective (2h)

**Daily Standup (15 min):**
- What did you do yesterday?
- What will you do today?
- Any blockers?

---

## Sprint Schedule

### Sprint 1: Configuration & Bootstrap (Days 1-5)

**Goal:** Application loads with modern config + UTF-8 database

**Tasks:**
- **GBK→UTF-8 Database Migration** (CRITICAL - see 08-gbk-to-utf8-detailed.md)
  - Full database backup
  - Convert database schema to utf8mb4
  - Convert all data from GBK to UTF-8
  - Validate data integrity
- Migrate config.inc.php → config/app.php
- Implement environment variables
- Migrate common.inc.php → bootstrap/app.php
- Set up Composer autoloader
- Add error handling and logging

**Deliverables:**
- ✅ Database converted to utf8mb4
- ✅ All data validated as UTF-8
- ✅ Application loads
- ✅ Configuration from .env
- ✅ Error logging works

---

### Sprint 2: Database Layer (Days 6-10)

**Goal:** PDO-based database layer

**Tasks:**
- Create Database/Connection.php
- Implement PDO connection
- Add query builder
- Implement prepared statements
- Add transaction support

**Deliverables:**
- ✅ Database connects via PDO
- ✅ CRUD operations work
- ✅ Query logging enabled

---

### Sprint 3: Authentication & Caching (Days 11-15)

**Goal:** Login and cache work

**Tasks:**
- Migrate logging.php → AuthService
- Implement password hashing (bcrypt)
- Create session management
- Add CSRF protection
- Implement cache repository
- Add Redis store

**Deliverables:**
- ✅ User can login/logout
- ✅ Session persists
- ✅ CSRF protection
- ✅ Cache works

**Milestone: P0 Complete!** 🎉

---

### Sprint 4: Forum Navigation (Days 16-20)

**Goal:** Forum index displays

**Tasks:**
- Migrate index.php → ForumIndexController
- Implement forum listing
- Add category grouping
- Display sub-forums
- Show statistics

**Deliverables:**
- ✅ Index page loads
- ✅ Forums display correctly

---

### Sprint 5: Forum Display (Days 21-25)

**Goal:** Forum pages work

**Tasks:**
- Migrate forumdisplay.php
- List threads with pagination
- Separate sticky/normal
- Check permissions
- Show moderators

**Deliverables:**
- ✅ Forum displays threads
- ✅ Pagination works

---

### Sprint 6: Thread Viewing (Days 26-30)

**Goal:** Users can read threads

**Tasks:**
- Migrate viewthread.php
- Load thread + posts
- Paginate posts
- Increment views
- Display user info

**Deliverables:**
- ✅ Thread loads correctly
- ✅ Posts display

---

### Sprint 7: New Thread (Days 31-35)

**Goal:** Create new threads

**Tasks:**
- Migrate newthread.inc.php
- Create thread + first post
- Update stats
- Add flood control
- Handle attachments

**Deliverables:**
- ✅ Threads create
- ✅ Stats update

---

### Sprint 8: Reply & Edit (Days 36-40)

**Goal:** Reply and edit posts

**Tasks:**
- Migrate newreply.inc.php
- Migrate editpost.inc.php
- Implement reply creation
- Implement editing
- Track edit history

**Deliverables:**
- ✅ Replies work
- ✅ Editing works

---

### Sprint 9: Post Processing (Days 41-45)

**Goal:** BBCode and uploads

**Tasks:**
- Migrate discuzcode.func.php
- Implement BBCode parser
- Add XSS protection
- Migrate attachment upload
- Implement file handling

**Deliverables:**
- ✅ BBCode renders
- ✅ Attachments upload

---

### Sprint 10: User Management (Days 46-50)

**Goal:** Profiles and registration

**Tasks:**
- Migrate member.php
- Implement profile viewing
- Migrate registration
- Create user CP
- Add avatar upload

**Deliverables:**
- ✅ Profiles work
- ✅ Registration works
- ✅ User CP functional

**Milestone: P1 Complete!** 🎉

---

### Sprint 11: Moderation (Days 51-55)

**Goal:** Moderators can moderate

**Tasks:**
- Migrate modcp.php
- Implement thread moderation
- Implement post moderation
- Add banning
- Create moderation logs

**Deliverables:**
- ✅ ModCP works
- ✅ Content moderation

---

### Sprint 12: Search & PM (Days 56-60)

**Goal:** Search and messages

**Tasks:**
- Migrate search.php
- Implement fulltext search
- Migrate pm.php
- Implement PM system
- Add PM folders

**Deliverables:**
- ✅ Search finds results
- ✅ Private messages work

---

### Sprint 13: Attachments (Days 61-65)

**Goal:** Full attachment support

**Tasks:**
- Migrate attachment.php
- Implement downloads
- Add permission checks
- Generate thumbnails
- Handle image processing

**Deliverables:**
- ✅ Attachments download
- ✅ Thumbnails work

---

### Sprint 14: AdminCP (Days 66-70)

**Goal:** Admin panel functional

**Tasks:**
- Migrate admincp.php
- Implement forum management
- Implement user group management
- Add settings management
- Create member management

**Deliverables:**
- ✅ AdminCP accessible
- ✅ Can manage forums/users

**Milestone: P2 Complete!** 🎉

---

### Sprint 15: Social Features (Days 71-75)

**Goal:** User profiles, buddies

**Tasks:**
- Migrate space.php
- Implement buddy system
- Migrate stats.php
- Add rank lists
- Create digest view

**Deliverables:**
- ✅ User spaces work
- ✅ Buddy lists

---

### Sprint 16: Gamification (Days 76-80)

**Goal:** Credits, medals, magic

**Tasks:**
- Implement credit system
- Migrate medal.php
- Migrate magic.php
- Create bank system
- Add rewards

**Deliverables:**
- ✅ Credits work
- ✅ Medals work
- ✅ Magic items work

---

### Sprint 17: Editor & Security (Days 81-85)

**Goal:** WYSIWYG and CAPTCHA

**Tasks:**
- Migrate editor.func.php
- Implement WYSIWYG editor
- Migrate seccode.class.php
- Add modern CAPTCHA
- Implement anti-flood

**Deliverables:**
- ✅ Rich text editor
- ✅ CAPTCHA works

---

### Sprint 18: Templates (Days 86-90)

**Goal:** Template system

**Tasks:**
- Migrate template.func.php
- Implement modern template engine
- Add theme support
- Migrate Chinese language
- Add multi-byte support

**Deliverables:**
- ✅ Templates work
- ✅ Themes can be customized

---

### Sprint 19: Plugins & API (Days 91-95)

**Goal:** Plugin system

**Tasks:**
- Migrate plugin.php
- Modernize plugin hooks
- Migrate XML classes
- Implement REST API (optional)
- Add plugin manager

**Deliverables:**
- ✅ Plugins load
- ✅ API works

---

### Sprint 20: Custom Features (Days 96-100)

**Goal:** PoketTB-specific

**Tasks:**
- Migrate pet system
- Migrate DEX system
- Migrate other custom features
- Final integration testing
- Performance optimization
- Documentation

**Deliverables:**
- ✅ Custom features work
- ✅ All tests pass

**Milestone: P3 Complete! Migration Done!** 🎉🎉🎉

---

## Sprint Review Checklist

**Demo (15 min):**
- Show completed features
- Demonstrate functionality
- Highlight technical improvements

**Metrics (10 min):**
- Stories completed: __/__
- Test coverage: __%
- Bugs found: __
- Bugs fixed: __

**Discussion (20 min):**
- What went well?
- What didn't go well?
- Action items for next sprint

**Definition of Done:**
- [ ] Migrated to PHP 8.3
- [ ] No legacy code
- [ ] Tests written (80%+ coverage)
- [ ] Code reviewed
- [ ] Documentation updated

---

## Risk Management

**Common Risks:**
1. Legacy code complexity → Add buffer time
2. GBK conversion issues → Test early
3. Hidden dependencies → Update graph
4. Performance issues → Benchmark often

**Mitigation:**
- 20% time buffer in each sprint
- Spike solutions for complex areas
- Continuous integration
- Automated testing

---

## Success Metrics

**Velocity:**
- Target: 5-8 story points/sprint
- Track actual vs planned

**Quality:**
- Test coverage: >80%
- Critical bugs: 0
- Major bugs: <5 per sprint

**Timeline:**
- On-time: >90%
- Buffer usage: <20%

---

**Document Status:** ✅ Complete
**Next Document:** 07-file-migration-checklist.md
