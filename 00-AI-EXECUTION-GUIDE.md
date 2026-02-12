# AI Execution Guide - Instructions for Claude

**Purpose:** Claude's step-by-step execution instructions for Discuz! migration
**Target:** AI agent (Claude Code) executing the migration plan
**Critical:** READ THIS DOCUMENT FIRST before starting any work

---

## 🚨 CRITICAL: EXECUTION PROTOCOL

**Before you do ANYTHING, you MUST:**

1. **Read this entire document** (5 minutes)
2. **Read the Quick Start section below** (2 minutes)
3. **Follow the execution sequence EXACTLY** (no shortcuts)

---

## 📋 QUICK START (First Things First)

### Step 0: Initialization (DO THIS FIRST)

```bash
# Run these commands to prepare your workspace
cd /root/poketb-renew/modern-php-execution-plan

# Read the execution guide
cat 00-AI-EXECUTION-GUIDE.md

# Read your current task
cat 00-SUMMARY.md | head -100
```

---

## 🎯 EXECUTION SEQUENCE

### Phase 1: Understand Context (Read Order)

**Read these documents IN ORDER before any coding:**

```
1. 00-AI-EXECUTION-GUIDE.md (this file)
   ↓
2. 06-execution-sprints.md
   Purpose: Understand which Sprint we're on
   Focus on: Current Sprint tasks (Sprint 1-20)
   ↓
3. 01-p0-critical-path.md OR 02-p1-core-features.md OR ...
   Purpose: Understand the features for current Sprint
   Focus on: Code examples, architecture
   ↓
4. 07-file-migration-checklist.md
   Purpose: See which files need migration
   Focus on: Priority P0/P1/P2/P3 status
   ↓
5. 12-tdd-workflow.md
   Purpose: Understand HOW to migrate (TDD cycle)
   Focus on: RED→GREEN→REFACTOR workflow
   ↓
6. 11-file-level-test-plan.md
   Purpose: Get test cases for the file
   Focus on: Test examples to copy
   ↓
7. 08-gbk-to-utf8-detailed.md (if Sprint 1)
   Purpose: UTF-8 migration procedures
   Focus on: Conversion scripts, validation
```

**Total reading time:** ~30 minutes

**Do NOT start coding until you've read all relevant docs.**

---

### Phase 2: Determine Current Sprint

**Check which Sprint to execute:**

```bash
# Ask user or check project state
echo "Current Sprint: ___"

# Based on Sprint number, read the relevant section:
# Sprint 1-3  → Read 01-p0-critical-path.md
# Sprint 4-10 → Read 02-p1-core-features.md
# Sprint 11-14 → Read 03-p2-important-features.md
# Sprint 15-20 → Read 04-p3-enhanced-features.md
```

---

### Phase 3: Select Next File to Migrate

**Decision Tree:**

```
START
  │
  ├─ Are we in Sprint 1 (Week 1)?
  │   ├─ YES → Is GBK→UTF-8 migration done?
  │   │   ├─ NO → MIGRATE UTF-8 FIRST (see 08-gbk-to-utf8-detailed.md)
  │   │   └─ YES → Select next P0 file
  │   │
  │   └─ Next files in priority order:
  │       1. config.inc.php → config/app.php
  │       2. include/common.inc.php → bootstrap/app.php
  │       3. include/db_mysql.class.php → Database/Connection.php
  │
  ├─ Are we in Sprint 2-10?
  │   └─ Select next P1 file from 07-file-migration-checklist.md
  │       Priority: Forum → Thread → Post → User → PM
  │
  ├─ Are we in Sprint 11-14?
  │   └─ Select next P2 file from 07-file-migration-checklist.md
  │       Priority: ModCP → AdminCP → Search
  │
  └─ Are we in Sprint 15-20?
      └─ Select next P3 file from 07-file-migration-checklist.md
          Priority: Social → Gamification → Templates
```

**File Selection Rules:**

1. ✅ **Check status in 07-file-migration-checklist.md**
   - Only select files marked ⏳ (pending) or 🔄 (in progress)
   - Never select files marked ✅ (complete)

2. ✅ **Respect dependencies (see 05-dependency-graph.md)**
   - Can't migrate post.php if thread.php not done
   - Can't migrate thread.php if forum.php not done

3. ✅ **Ask user if uncertain**
   - "Found 5 pending files. Which should I migrate first?"
   - Wait for user input

---

### Phase 4: Migrate Single File (TDD Cycle)

**Follow 12-tdd-workflow.md EXACTLY:**

For each file, repeat this cycle:

```bash
# ====== ITERATION 1: First Function ======

# 🔴 STEP 1: RED (Write Failing Test)
# Time budget: 10-20 minutes

1. Read legacy code
   cat poketb.com/bbs/[FILE_PATH]

2. Read test template from 11-file-level-test-plan.md
   Find the section for your file
   Copy the test template

3. Write the test (DO NOT implement yet)
   vim tests/Unit/[TestFile].php
   # Paste test template
   # Customize for specific function

4. Run test → MUST FAIL
   vendor/bin/phpunit tests/Unit/[TestFile].php
   # Expected: Error - function not found

5. Commit RED
   git add tests/Unit/[TestFile].php
   git commit -m "test([feature]): add failing test for [function_name] (RED)"

# ====== 🟢 STEP 2: GREEN (Make Test Pass) ======
# Time budget: 20-40 minutes

6. Read legacy implementation
   Study the old code carefully

7. Create modern implementation
   vim src/[Path]/[ClassName].php
   # Migrate to PHP 8.3
   # Add namespaces, type hints
   # Keep it SIMPLE (don't optimize yet)

8. Run test → MUST PASS
   vendor/bin/phpunit tests/Unit/[TestFile].php
   # Expected: OK (1 test, 1 assertion)

9. Commit GREEN
   git add src/[Path]/[ClassName].php
   git commit -m "feat([feature]): implement [function_name] (GREEN)"

# ====== 🔵 STEP 3: REFACTOR (Clean Up) ======
# Time budget: 20-30 minutes

10. Review code quality
    vim src/[Path]/[ClassName].php
    # Extract magic values
    # Improve naming
    # Add missing type hints

11. Run tests after EACH change
    vendor/bin/phpunit tests/Unit/[TestFile].php
    # Must stay GREEN

12. Commit REFACTOR
    git add src/[Path]/[ClassName].php
    git commit -m "refactor([feature]): improve [design_aspect]"

# ====== REPEAT FOR NEXT FUNCTION ======
13. Go back to STEP 1 for next function in the file
```

---

### Phase 5: Validate & Integrate

**After all functions in file are migrated:**

```bash
# 1. Run full test suite
vendor/bin/phpunit

# 2. Check code style
vendor/bin/phpcs --standard=PSR12 src/

# 3. Type check
vendor/bin/phpstan analyse src/ --level=max

# 4. Update file status in checklist
vim 07-file-migration-checklist.md
# Change: 🔄 → ✅
# Add notes: "Migrated on YYYY-MM-DD, [hours] hours"

# 5. Report to user
echo "✅ File [filename] migration complete!"
echo "Tests: X/Y passing"
echo "Coverage: XX%"
echo "Next file: [filename]"
```

---

## 📊 DAILY EXECUTION RHYTHM

### Morning Start (09:00)

```bash
# 1. Check current Sprint
echo "Current Sprint: [SPRINT_NUMBER]"
echo "Day [DAY_NUMBER] of [TOTAL_DAYS]"

# 2. Check today's tasks
cat 06-execution-sprints.md | grep -A 20 "Sprint [SPRINT_NUMBER]"

# 3. Verify previous work
git log --oneline -5
# Check yesterday's commits

# 4. Plan today's work
echo "Today's goal: Migrate [X] files"
echo "Files to migrate:"
echo "  1. [FILE_1]"
echo "  2. [FILE_2]"
```

### During Work (Follow TDD Cycle)

```bash
# For EACH function, follow RED→GREEN→REFACTOR
# See Phase 4 above

# Every 60 minutes, report progress:
echo "Progress: [X]/[Y] functions done"
echo "Tests: [PASSING]/[TOTAL] passing"
```

### End of Day (17:00)

```bash
# 1. Verify all tests pass
vendor/bin/phpunit

# 2. Commit everything
git add -A
git commit -m "daily: [SUMMARY OF WORK DONE]"

# 3. Push to GitHub
git push origin [BRANCH_NAME]

# 4. Update checklist
vim 07-file-migration-checklist.md
# Mark files as ✅

# 5. Daily report to user
echo "=== DAILY REPORT ===="
echo "Files completed: [LIST]"
echo "Commits: [NUMBER]"
echo "Tests passing: [PERCENTAGE]%"
echo "Blockers: [DESCRIPTION]"
echo "Tomorrow's plan: [PLAN]"
```

---

## 🚨 CRITICAL RULES

### ✅ MUST DO

1. **READ BEFORE YOU CODE**
   - Always read the relevant docs first
   - Never assume you know what to do
   - Documented approach is battle-tested

2. **FOLLOW TDD CYCLE**
   - RED → GREEN → REFACTOR → NEXT
   - No skipping steps
   - Each phase = separate commit

3. **RESPECT FILE DEPENDENCIES**
   - Check 05-dependency-graph.md
   - Migrate dependencies first
   - Don't break existing code

4. **KEEP TESTS PASSING**
   - Run tests after every change
   - Never commit failing tests
   - If tests fail, fix immediately

5. **COMMIT FREQUENTLY**
   - Each TDD phase = 1 commit
   - Don't batch changes
   - Make commits reversible

6. **UPDATE TRACKING**
   - Mark files in 07-file-migration-checklist.md
   - Update after each file completion
   - Keep progress visible

### ❌ NEVER DO

1. **Don't skip reading docs**
   - Docs contain critical info
   - "I know TDD" is not acceptable
   - Read the specific guidance for THIS migration

2. **Don't implement without tests**
   - No "I'll write tests later"
   - Tests MUST come first
   - No exceptions

3. **Don't batch multiple TDD cycles**
   - One RED, one GREEN, one REFACTOR
   - Not "I did 3 functions today"
   - Each phase separately

4. **Don't ignore UTF-8**
   - All code must handle UTF-8
   - Use mb_* functions
   - Test with Chinese characters

5. **Don't break dependencies**
   - Check what depends on current file
   - Don't break existing features
   - Run integration tests

6. **Don't guess, ask**
   - If uncertain, stop and ask
   - Don't make assumptions
   - User prefers questions over mistakes

---

## 📋 DECISION CHECKLIST

### Before Starting Work (Daily)

- [ ] Read 00-AI-EXECUTION-GUIDE.md (this file)
- [ ] Read current Sprint tasks from 06-execution-sprints.md
- [ ] Read feature docs (01-04 depending on Sprint)
- [ ] Check 07-file-migration-checklist.md for pending files
- [ ] Verify UTF-8 migration done (if Sprint 1)

### Before Migrating a File (Per File)

- [ ] Selected file from checklist (⏳ or 🔄 status)
- [ ] Checked dependencies in 05-dependency-graph.md
- [ ] Read test plan from 11-file-level-test-plan.md
- [ ] Read legacy code completely
- [ ] Created test file

### During Migration (Per Function)

- [ ] 🔴 RED: Test written and failing
- [ ] 🔴 RED: Test committed separately
- [ ] 🟢 GREEN: Implementation passes test
- [ ] 🟢 GREEN: Implementation committed separately
- [ ] 🔵 REFACTOR: Code improved
- [ ] 🔵 REFACTOR: Tests still passing
- [ ] 🔵 REFACTOR: Refactoring committed separately

### After File Complete (Per File)

- [ ] All tests passing
- [ ] Code style compliant
- [ ] Type check passing
- [ ] Integration tests passing
- [ ] Status updated to ✅ in checklist
- [ ] Reported completion to user

---

## 🆘 TROUBLESHOOTING

### Problem: Don't Know Which File to Migrate Next

**Solution:**
```bash
# 1. Check current Sprint
echo "Current Sprint: [SPRINT_NUMBER]"

# 2. Read checklist
vim 07-file-migration-checklist.md

# 3. Find first ⏳ or 🔄 file
# Search for "⏳ [P0]" or "⏳ [P1]" etc.

# 4. If still unsure, ask user
echo "Found [X] pending files. Which should I migrate first?"
```

### Problem: File Seems Too Complex

**Solution:**
```bash
# 1. Break into smaller functions
# Don't migrate entire file at once
# Do one function at a time

# 2. Ask for help
echo "File [filename] has [X] complex functions"
echo "Estimated time: [hours] hours"
echo "Should I proceed or skip to simpler file?"
```

### Problem: Tests Keep Failing

**Solution:**
```bash
# 1. Stop and read test carefully
vim tests/Unit/[TestFile].php

# 2. Read legacy code again
cat poketb.com/bbs/[LegacyFile]

# 3. Spike (research)
git checkout -b spike-[feature-name]
# Try anything to understand
# After 30 min, report findings

# 4. Ask for help
echo "Test [test_name] keeps failing"
echo "Tried: [ATTEMPTS]"
echo "Need help understanding [ASPECT]"
```

### Problem: Dependencies Are Unclear

**Solution:**
```bash
# 1. Check dependency graph
vim 05-dependency-graph.md

# 2. Search for file name
# See what it depends on

# 3. Verify dependencies are done
vim 07-file-migration-checklist.md
# Check if dependent files are ✅

# 4. If not, migrate dependencies first
echo "File [filename] depends on [dependencies]"
echo "Dependencies status: [STATUS]"
echo "Migrating dependencies first..."
```

---

## 📈 PROGRESS REPORTING

### Format for Daily Updates

```markdown
## Migration Progress: YYYY-MM-DD

**Sprint:** [X]
**Day:** [Y] of [Z]

### Files Completed Today
1. ✅ [filename]
   - Functions: [X]
   - Tests: [Y]/[Y] passing
   - Coverage: [XX]%
   - Time: [X hours]

2. ✅ [filename]
   - Functions: [X]
   - Tests: [Y]/[Y] passing
   - Coverage: [XX]%
   - Time: [X hours]

### Files In Progress
1. 🔄 [filename]
   - Progress: [X]%
   - Blocked by: [description]

### Tomorrow's Plan
1. Complete [filename]
2. Start [filename2]
3. Start [filename3]

### Blockers
- [Description of blocker]
- [Who can help]
- [ETA for resolution]
```

---

## 🔍 QUALITY GATES

### Before Marking File Complete (Must Pass ALL)

```bash
# Gate 1: Unit Tests
vendor/bin/phpunit tests/Unit/[TestFile].php
# ✅ Must see: "OK (X tests, Y assertions)"

# Gate 2: Code Coverage
vendor/bin/phpunit --coverage-text=coverage.txt
# ✅ Must see: >80% for new code

# Gate 3: Code Style
vendor/bin/phpcs --standard=PSR12 src/[Path]/[File].php
# ✅ Must see: No errors

# Gate 4: Type Safety
vendor/bin/phpstan analyse src/[Path]/[File].php --level=max
# ✅ Must see: No errors

# Gate 5: Integration
vendor/bin/phpunit tests/Integration/
# ✅ Must see: All tests passing

# Gate 6: UTF-8 Validation
php scripts/validate-utf8.php
# ✅ Must see: All files valid UTF-8
```

---

## 🎯 SUCCESS METRICS

### Daily Targets

- ✅ **2-4 files** migrated (depending on complexity)
- ✅ **100%** of tests passing
- ✅ **0** regressions in existing code
- ✅ **All** commits follow TDD pattern
- ✅ **Checklist** updated for completed files

### Weekly Targets

- ✅ **Sprint complete** (all tasks done)
- ✅ **P0/P1 files** for that sprint: 100% migrated
- ✅ **Integration tests** all passing
- ✅ **Demo** working features
- ✅ **Retrospective** completed

---

## 📚 DOCUMENT REFERENCE MATRIX

| When you need... | Read this document | Section |
|-----------------|-------------------|----------|
| To know which Sprint | 06-execution-sprints.md | Current Sprint |
| To select file | 07-file-migration-checklist.md | Priority sections |
| To understand dependencies | 05-dependency-graph.md | Visual maps |
| To see code examples | 01-p0-critical-path.md (or 02/03/04) | Feature sections |
| To write tests | 11-file-level-test-plan.md | Test templates |
| To know HOW to migrate | 12-tdd-workflow.md | SOP sections |
| To handle UTF-8 | 08-gbk-to-utf8-detailed.md | Migration scripts |
| To run tests | 09-testing-strategy.md | Test execution |
| To validate quality | 12-tdd-workflow.md | Quality gates |

---

## 🚀 QUICK REFERENCE CARD

**Print this and keep visible:**

```
═══════════════════════════════════════════════════════
  CLAUDE'S DAILY EXECUTION CHECKLIST
═══════════════════════════════════════════════════════

MORNING:
☐ Read current Sprint tasks
☐ Select files to migrate today
☐ Verify dependencies are met
☐ Plan work: "I will migrate X files today"

PER FILE:
☐ Read test template from 11-file-level-test-plan.md
☐ 🔴 RED: Write failing test
☐ 🔴 RED: Commit test
☐ 🟢 GREEN: Implement feature
☐ 🟢 GREEN: Commit implementation
☐ 🔵 REFACTOR: Clean up code
☐ 🔵 REFACTOR: Commit refactor
☐ Run full test suite
☐ Update checklist to ✅

END OF DAY:
☐ All tests passing
☐ Code style clean
☐ Types correct
☐ Push to GitHub
☐ Daily report to user

QUALITY GATES (All must pass):
☐ Unit tests: 100% pass
☐ Coverage: >80%
☐ Style: No errors
☐ Types: No errors
☐ Integration: All pass
☐ UTF-8: Valid

═══════════════════════════════════════════════════════
```

---

## 🎓 EXAMPLE EXECUTION SESSION

### What a Perfect Day Looks Like

```bash
# 09:00 - Start
# Claude reads execution guide
# Claude reads Sprint 1 tasks
# Claude checks 07-file-migration-checklist.md

# 09:15 - File 1: config.inc.php
# Claude: "Starting config.inc.php migration"
# Reads test template
# 🔴 RED: Writes ConfigTest::testConfigLoadsFromArray
# Runs test → FAIL (expected)
# Commits: "test(config): add failing test (RED)"

# 09:35 - GREEN phase
# Reads legacy config.inc.php
# Migrates to config/app.php
# Runs test → PASS
# Commits: "feat(config): implement array config (GREEN)"

# 10:00 - REFACTOR phase
# Extracts constants
# Adds environment variable support
# Runs tests after each change
# Commits: "refactor(config): extract DATABASE_* constants"

# 10:30 - File 1 complete
# Runs full test suite
# Updates 07-file-migration-checklist.md: config.inc.php → ✅
# Reports: "✅ config.inc.php complete (1 hour, 15 min)"

# 10:35 - File 2: common.inc.php
# Repeats RED → GREEN → REFACTOR cycle
# Completes by 12:30

# 12:30 - Lunch

# 13:30 - File 3: db_mysql.class.php
# More complex file, takes 2.5 hours
# Multiple RED → GREEN → REFACTOR iterations
# Completes by 16:00

# 16:00 - Integration
# Runs full test suite
# All tests passing

# 16:30 - End of day
# Commits all work
# Pushes to GitHub
# Reports: "3 files migrated today: config, common, db_mysql"
```

---

## ✅ EXECUTION READY CHECKLIST

**Before user says "GO":**

- [ ] You have read this entire document (00-AI-EXECUTION-GUIDE.md)
- [ ] You understand the TDD cycle (RED→GREEN→REFACTOR)
- [ ] You know which Sprint to execute
- [ ] You know where to find file checklists
- [ ] You know where to find test templates
- [ ] You know quality gates to pass
- [ ] You know reporting format to use

**When all checked, you're ready to execute!**

---

**Document Status:** ✅ Complete
**Priority:** READ THIS FIRST - This is your instruction manual
**Next Step:** When user says "start migration", begin with Phase 1
