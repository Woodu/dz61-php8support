# P0 Critical Path - Migration Progress

**Project**: Discuz! 6.1F → PHP 8.3 Migration
**Phase**: P0 Critical Path
**Timeline**: Weeks 1-3 (15 working days)
**Status**: Week 1 - Foundation

---

## Progress Overview

| Week | Focus | Status | Completion |
|------|-------|--------|------------|
| Week 1 | Foundation | ⏳ In Progress | 0% |
| Week 2 | Authentication | ⏳ Not Started | 0% |
| Week 3 | Caching | ⏳ Not Started | 0% |

---

## Week 1: Foundation (Days 1-5)

### Day 1: GBK→UTF-8 Database Migration

**Status**: ⏳ Pending

**Tasks**:
- [ ] Backup production database
- [ ] Create test database
- [ ] Convert database schema (latin1 → utf8mb4)
- [ ] Convert all tables to utf8mb4
- [ ] Verify data integrity
- [ ] Create migration rollback plan

**Documentation**: `08-gbk-to-utf8-detailed.md` Phase 1

**Estimated Time**: 8 hours

---

### Day 2: Configuration System (1.1)

**Status**: ⏳ Pending

**Legacy File**: `/root/poketb-renew/poketb.com/bbs/config.inc.php`

**Modern File**: `config/app.php`

**Tasks**:
- [ ] 🔴 RED: Write configuration loading test
- [ ] 🟢 GREEN: Implement Config class
- [ ] 🔵 REFACTOR: Add validation
- [ ] 🔴 RED: Write environment variable test
- [ ] 🟢 GREEN: Implement dotenv loading
- [ ] 🔵 REFACTOR: Add type safety

**Database Tables**: `cdb_settings`

**Estimated Time**: 6 hours

---

### Day 3-4: Core Bootstrap (1.2)

**Status**: ⏳ Pending

**Legacy File**: `/root/poketb-renew/poketb.com/bbs/include/common.inc.php`

**Modern File**: `app/Bootstrap.php`

**Tasks**:
- [ ] 🔴 RED: Write autoloader test
- [ ] 🟢 GREEN: Implement Composer PSR-4 autoloading
- [ ] 🔵 REFACTOR: Optimize autoloader
- [ ] 🔴 RED: Write constant definition test
- [ ] 🟢 GREEN: Define application constants
- [ ] 🔵 REFACTOR: Organize constants
- [ ] 🔴 RED: Write error handler test
- [ ] 🟢 GREEN: Implement error handling
- [ ] 🔵 REFACTOR: Add structured logging
- [ ] 🔴 RED: Write request initialization test
- [ ] 🟢 GREEN: Implement Request object
- [ ] 🔵 REFACTOR: Add security middleware

**Database Tables**: `cdb_settings`, `cdb_sessions`

**Estimated Time**: 12 hours

---

### Day 5: Database Layer (1.3)

**Status**: ⏳ Pending

**Legacy File**: `/root/poketb-renew/poketb.com/bbs/include/db_mysql.class.php`

**Modern File**: `app/Database/Connection.php`

**Tasks**:
- [ ] 🔴 RED: Write PDO connection test
- [ ] 🟢 GREEN: Implement Connection class
- [ ] 🔵 REFACTOR: Add connection pooling
- [ ] 🔴 RED: Write prepared statement test
- [ ] 🟢 GREEN: Implement query execution
- [ ] 🔵 REFACTOR: Add query logging
- [ ] 🔴 RED: Write transaction test
- [ ] 🟢 GREEN: Implement transaction support
- [ ] 🔵 REFACTOR: Add savepoint support
- [ ] 🔴 RED: Write query builder test
- [ ] 🟢 GREEN: Implement Query Builder
- [ ] 🔵 REFACTOR: Add complex where clauses

**Database Tables**: All tables (database layer)

**Estimated Time**: 8 hours

---

## Week 2: Authentication (Days 6-10)

### Day 6-7: Session Management (2.2)

**Status**: ⏳ Not Started

**Legacy File**: `/root/poketb-renew/poketb.com/bbs/include/common.inc.php` (lines 141-200)

**Modern File**: `app/Session/SessionManager.php`

**Tasks**:
- [ ] 🔴 RED: Write session creation test
- [ ] 🟢 GREEN: Implement SessionManager
- [ ] 🔵 REFACTOR: Add session regeneration
- [ ] 🔴 RED: Write session storage test
- [ ] 🟢 GREEN: Implement database session storage
- [ ] 🔵 REFACTOR: Add session cleanup
- [ ] 🔴 RED: Write cookie handling test
- [ ] 🟢 GREEN: Implement secure cookies
- [ ] 🔵 REFACTOR: Add SameSite support

**Database Tables**: `cdb_sessions`

**Estimated Time**: 10 hours

---

### Day 8-9: User Login (2.1)

**Status**: ⏳ Not Started

**Legacy File**: `/root/poketb-renew/poketb.com/bbs/logging.php`

**Modern File**: `app/Auth/AuthService.php`, `app/Auth/Controllers/AuthController.php`

**Tasks**:
- [ ] 🔴 RED: Write login test
- [ ] 🟢 GREEN: Implement AuthService
- [ ] 🔵 REFACTOR: Add bcrypt hashing
- [ ] 🔴 RED: Write password verification test
- [ ] 🟢 GREEN: Implement password hash check
- [ ] 🔵 REFACTOR: Add argon2 support
- [ ] 🔴 RED: Write remember me test
- [ ] 🟢 GREEN: Implement remember tokens
- [ ] 🔵 REFACTOR: Add token expiration
- [ ] 🔴 RED: Write logout test
- [ ] 🟢 GREEN: Implement logout
- [ ] 🔵 REFACTOR: Add session cleanup

**Database Tables**: `cdb_members`, `cdb_memberfields`, `cdb_sessions`, `cdb_banned`, `cdb_failedlogins`

**Estimated Time**: 12 hours

---

### Day 10: Testing & Bug Fixes

**Status**: ⏳ Not Started

**Tasks**:
- [ ] Run full test suite
- [ ] Fix failing tests
- [ ] Update documentation
- [ ] Code review
- [ ] Performance profiling

**Estimated Time**: 8 hours

---

## Week 3: Caching & Testing (Days 11-15)

### Day 11-12: Caching System (3.1)

**Status**: ⏳ Not Started

**Legacy File**: `/root/poketb-renew/poketb.com/bbs/include/cache.func.php`

**Modern File**: `app/Cache/CacheRepository.php`

**Tasks**:
- [ ] 🔴 RED: Write cache interface test
- [ ] 🟢 GREEN: Implement CacheRepository
- [ ] 🔵 REFACTOR: Add cache tagging
- [ ] 🔴 RED: Write Redis store test
- [ ] 🟢 GREEN: Implement RedisStore
- [ ] 🔵 REFACTOR: Add connection pooling
- [ ] 🔴 RED: Write file store test
- [ ] 🟢 GREEN: Implement FileStore
- [ ] 🔵 REFACTOR: Add file locking
- [ ] 🔴 RED: Write system cache test
- [ ] 🟢 GREEN: Implement system caches
- [ ] 🔵 REFACTOR: Add cache warming

**Database Tables**: `cdb_settings`, `cdb_forums`, `cdb_forumfields`, `cdb_usergroups`, `cdb_smilies`, `cdb_announcements`, `cache`

**Estimated Time**: 12 hours

---

### Day 13-14: Integration Testing

**Status**: ⏳ Not Started

**Tasks**:
- [ ] Write integration tests
- [ ] Test authentication flow
- [ ] Test session persistence
- [ ] Test cache invalidation
- [ ] Test error handling
- [ ] Load testing

**Estimated Time**: 12 hours

---

### Day 15: Performance Testing

**Status**: ⏳ Not Started

**Tasks**:
- [ ] Benchmark database queries
- [ ] Optimize slow queries
- [ ] Cache optimization
- [ ] Session optimization
- [ ] Memory profiling
- [ ] Create performance report

**Estimated Time**: 8 hours

---

## P0 Success Criteria

✅ **Week 1 Completion**:
- [ ] GBK→UTF-8 migration complete
- [ ] Configuration loads without errors
- [ ] Bootstrap initializes correctly
- [ ] Database layer works with PDO
- [ ] All tests pass

✅ **Week 2 Completion**:
- [ ] User can login successfully
- [ ] Session persists across requests
- [ ] Remember me works
- [ ] Logout clears session
- [ ] All tests pass

✅ **Week 3 Completion**:
- [ ] Cache speeds up page loads
- [ ] System caches update correctly
- [ ] Integration tests pass
- [ ] Performance benchmarks met
- [ ] No security vulnerabilities

---

## Notes

### Critical Dependencies

1. **String Functions**: Must migrate to mb_* variants during Week 1
   - `strlen()` → `mb_strlen()`
   - `strpos()` → `mb_strpos()`
   - `substr()` → `mb_substr()`
   - etc. (see `08-gbk-to-utf8-detailed.md`)

2. **Database Charset**: Must use utf8mb4 everywhere
   - Schema: `utf8mb4`
   - Collation: `utf8mb4_unicode_ci`
   - PDO charset: `utf8mb4`

3. **Security**: No shortcuts
   - All queries must use prepared statements
   - All forms must have CSRF protection
   - All cookies must be HttpOnly + SameSite

### Testing Requirements

- **Minimum Coverage**: 80%
- **All Tests Must Pass**: No exceptions
- **TDD Workflow**: Strict RED→GREEN→REFACTOR

---

**Last Updated**: 2026-02-13
**Next Review**: After Day 1 completion
