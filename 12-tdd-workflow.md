# TDD Workflow & Standard Operating Procedures

**Purpose:** Step-by-step TDD workflow for every file migration
**Method:** Red-Green-Refactor cycle
**Goal:** Zero bugs, 100% test coverage, no regressions

---

## Executive Summary

**Why This Document Matters:**

When migrating 964 files, you need a **repeatable, reliable process** that:
- ✅ Prevents bugs before they happen
- ✅ Ensures every change is tested
- ✅ Makes progress visible (red → green)
- ✅ Catches regressions immediately
- ✅ Creates safety net for refactoring

**This document provides:**
- ✅ Standard TDD cycle (Red-Green-Refactor)
- ✅ Step-by-step SOP for each file
- ✅ Exit criteria for each step
- ✅ Troubleshooting guide
- ✅ Daily workflow checklists
- ✅ Pair programming guide

---

## Part 1: The TDD Cycle (Red-Green-Refactor)

### The Core Loop

```
┌─────────────────────────────────────────────────────────────┐
│                    TDD CYCLE                            │
├─────────────────────────────────────────────────────────────┤
│                                                           │
│  🔴 RED          Write failing test                      │
│     ↓            (Test describes desired behavior)          │
│                                                           │
│  ✅ GREEN       Make test pass (minimal code)           │
│     ↓            (Just enough to pass)                   │
│                                                           │
│  🔵 REFACTOR    Clean up code                         │
│     ↓            (Improve design, keep tests green)      │
│                                                           │
│  ➡️  NEXT        Move to next requirement                 │
│                  (Back to RED)                           │
│                                                           │
└─────────────────────────────────────────────────────────────┘
```

---

### Step 1: 🔴 RED - Write Failing Test

**Goal:** Describe the desired behavior before implementing it

**Checklist:**
- [ ] Read legacy code to understand current behavior
- [ ] Write test case for expected behavior
- [ ] Run test → **MUST FAIL** ✗
- [ ] See error message (this is expected)
- [ ] Commit test (separate commit from implementation)

**Time Budget:** 10-20 minutes per test

**Example:**

```bash
# 1. Read legacy code
cat poketb.com/bbs/include/global.func.php | grep -A 10 "function authcode"

# 2. Create test file
touch tests/Unit/Utils/AuthCodeTest.php

# 3. Write failing test
cat > tests/Unit/Utils/AuthCodeTest.php << 'EOF'
<?php
declare(strict_types=1);

namespace Tests\Unit\Utils;

use PHPUnit\Framework\TestCase;

class AuthCodeTest extends TestCase
{
    public function testAuthCodeEncryptsString()
    {
        // Arrange
        $input = 'teststring';
        $key = 'secretkey';

        // Act
        $encrypted = authcode($input, 'ENCODE', $key);

        // Assert
        $this->assertNotEmpty($encrypted);
        $this->assertNotEquals($input, $encrypted);
    }
}
EOF

# 4. Run test → WILL FAIL
vendor/bin/phpunit tests/Unit/Utils/AuthCodeTest.php

# Expected output:
# Error: Call to undefined function authcode()
```

**Exit Criteria:**
✅ Test written and committed
✅ Test fails with expected error
✅ Error message understood

---

### Step 2: 🟢 GREEN - Make Test Pass

**Goal:** Write simplest code that makes test pass

**Checklist:**
- [ ] Read legacy implementation
- [ ] Migrate to modern PHP (add types, namespaces)
- [ ] Copy minimal logic to make test pass
- [ ] Run test → **MUST PASS** ✓
- [ ] No refactoring yet (just make it work)
- [ ] Commit implementation

**Time Budget:** 20-40 minutes

**Rules:**
- ✅ Do simplest thing that works
- ✅ Don't worry about code quality yet
- ✅ Don't optimize prematurely
- ❌ NO refactoring in this step

**Example:**

```bash
# 1. Create modern implementation
mkdir -p src/Utils
cat > src/Utils/AuthCode.php << 'EOF'
<?php
declare(strict_types=1);

namespace App\Utils;

function authcode(string $string, string $operation, string $key): string
{
    // Simplest implementation to make test pass
    if ($operation === 'ENCODE') {
        return base64_encode($string . $key); // Simplified
    }
    return '';
}
EOF

# 2. Run test → SHOULD PASS
vendor/bin/phpunit tests/Unit/Utils/AuthCodeTest.php

# Expected output:
# OK (1 test, 1 assertion)
```

**Exit Criteria:**
✅ Test passes
✅ Implementation committed
✅ Code may be ugly (that's OK)

---

### Step 3: 🔵 REFACTOR - Clean Up Code

**Goal:** Improve design while keeping tests green

**Checklist:**
- [ ] Run full test suite → **ALL MUST PASS**
- [ ] Review code quality
- [ ] Extract magic values to constants
- [ ] Improve names if unclear
- [ ] Add type hints if missing
- [ ] Run tests again after each change
- [ ] Commit refactoring (separate commit)

**Time Budget:** 20-30 minutes

**Rules:**
- ✅ Tests must stay green
- ✅ Small, incremental changes
- ✅ Run tests after each change
- ❌ NO new features in this step

**Example:**

```bash
# 1. Review current code
cat src/Utils/AuthCode.php

# 2. Identify improvements
# - Extract algorithm
# - Add constants
# - Improve error handling

# 3. Refactor step-by-step

# Step 3.1: Extract constants
cat > src/Utils/AuthCode.php << 'EOF'
<?php
declare(strict_types=1);

namespace App\Utils;

class AuthCode
{
    private const CIPHER_METHOD = 'AES-256-CBC';
    private const HASH_ALGO = 'sha256';

    public static function encode(string $string, string $key): string
    {
        // Better implementation
        $iv = random_bytes(16);
        $encrypted = openssl_encrypt(
            $string,
            self::CIPHER_METHOD,
            $key,
            0,
            $iv
        );
        return base64_encode($encrypted . '::' . base64_encode($iv));
    }
}
EOF

# Step 3.2: Run tests
vendor/bin/phpunit tests/Unit/Utils/AuthCodeTest.php

# If green → commit
# If red → fix immediately, don't continue

# Expected output:
# OK (1 test, 1 assertion)
```

**Exit Criteria:**
✅ All tests pass
✅ Code is clean and readable
✅ Refactoring committed

---

### Step 4: ➡️ NEXT - Move to Next Requirement

**Checklist:**
- [ ] Mark task as complete in tracker
- [ ] Update progress in `07-file-migration-checklist.md`
- [ ] Move to next test case
- [ ] Start RED step again

**Back to Step 1** → Repeat for each requirement

---

## Part 2: Standard Operating Procedure (SOP)

### SOP: Migrating a Single File

**Total Time Estimate:** 2-4 hours per file (varies by complexity)

#### Phase 1: Preparation (15 minutes)

**Step 1.1: Read Legacy Code (5 min)**

```bash
# Open legacy file
vim poketb.com/bbs/include/global.func.php

# Identify:
# - What functions exist?
# - What are the dependencies?
# - What encoding is used (GBK/UTF-8)?
# - Any security issues?
```

**Step 1.2: Create Test File (5 min)**

```bash
# Create matching test file
touch tests/Unit/GlobalFunctionsTest.php

# Add boilerplate
cat > tests/Unit/GlobalFunctionsTest.php << 'EOF'
<?php
declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

class GlobalFunctionsTest extends TestCase
{
    // Tests will go here
}
EOF
```

**Step 1.3: Mark in Checklist (5 min)**

```bash
# Update migration status
vim 07-file-migration-checklist.md

# Change status: ⏳ → 🔄
```

**Exit Criteria:**
✅ Legacy code understood
✅ Test file created
✅ Status marked as in-progress

---

#### Phase 2: Test-Driven Migration (2-3 hours)

**For EACH function/feature in the file:**

**Iteration 1: RED → GREEN → REFACTOR**

```bash
# ====== RED PHASE ======
# 1. Write test first
vim tests/Unit/GlobalFunctionsTest.php

# Add test:
public function testStrlenWorks()
{
    $this->assertEquals(4, mb_strlen('测试', 'UTF-8'));
}

# 2. Run test → MUST FAIL
vendor/bin/phpunit tests/Unit/GlobalFunctionsTest.php
# Expected: Call to undefined function mb_strlen()

# 3. Commit RED
git add tests/Unit/GlobalFunctionsTest.php
git commit -m "test: add failing test for mb_strlen (RED)"

# ====== GREEN PHASE ======
# 4. Migrate function to modern PHP
vim src/Utils/StringFunctions.php

# Add implementation:
namespace App\Utils;

function mb_strlen_wrapper(string $string): int
{
    return mb_strlen($string, 'UTF-8');
}

# 5. Run test → MUST PASS
vendor/bin/phpunit tests/Unit/GlobalFunctionsTest.php
# Expected: OK (1 test, 1 assertion)

# 6. Commit GREEN
git add src/Utils/StringFunctions.php
git commit -m "feat: add mb_strlen wrapper (GREEN)"

# ====== REFACTOR PHASE ======
# 7. Refactor for quality
vim src/Utils/StringFunctions.php

# Improve:
# - Add class structure
# - Add constants
# - Add documentation

# 8. Run tests after each change
vendor/bin/phpunit tests/Unit/GlobalFunctionsTest.php
# Must stay GREEN

# 9. Commit REFACTOR
git add src/Utils/StringFunctions.php
git commit -m "refactor: improve StringFunctions design"
```

**Repeat for each function:**
- `testStrlenWorks()` → implement → refactor → **NEXT**
- `testSubstrWorks()` → implement → refactor → **NEXT**
- `testStrposWorks()` → implement → refactor → **NEXT**

**Exit Criteria:**
✅ All functions have tests
✅ All tests pass
✅ Code is refactored
✅ Commits show RED-GREEN-REFACTOR cycle

---

#### Phase 3: Integration (30 minutes)

**Step 3.1: Run Full Test Suite**

```bash
# Run all tests to ensure no breakage
vendor/bin/phpunit

# Expected: All tests pass (including old ones)
```

**Step 3.2: Run Linter**

```bash
# Check code style
vendor/bin/phpcs --standard=PSR12 src/Utils/StringFunctions.php

# Fix any issues
vendor/bin/phpcbf --standard=PSR12 src/Utils/StringFunctions.php
```

**Step 3.3: Type Check**

```bash
# Verify type safety
vendor/bin/phpstan analyse src/Utils/StringFunctions.php --level=max

# Fix any type errors
```

**Exit Criteria:**
✅ Full test suite passes
✅ Code style compliant
✅ No type errors

---

#### Phase 4: Documentation (15 minutes)

**Step 4.1: Update Migration Checklist**

```bash
vim 07-file-migration-checklist.md

# Change: 🔄 → ✅
# Add notes about any challenges
```

**Step 4.2: Update File List**

```bash
# Mark file as migrated
vim 00-complete-feature-inventory.md

# Add migration notes
```

**Exit Criteria:**
✅ Checklist updated
✅ File marked complete
✅ Notes documented

---

## Part 3: Daily Workflow

### Standard Workday

**09:00 - Daily Standup (15 min)**
```
Round table: "Yesterday, Today, Blockers"

Example:
- Alice: "Yesterday migrated authcode(), today migrating authcode(), no blockers"
- Bob: "Yesterday tests passed, today starting StringHelper, blocked by charset issue"
- You: "Yesterday green on strlen(), today RED phase for substr(), no blockers"
```

**09:15 - Start RED Phase (until 11:00)**

```
Focus: Write failing tests
- Don't implement
- Just describe behavior
- Commit each RED test
```

**11:00 - GREEN Phase (until 12:30)**

```
Focus: Make tests pass
- Simplest implementation
- Don't refactor yet
- Commit each GREEN
```

**12:30 - Lunch Break**

**13:30 - REFACTOR Phase (until 15:00)**

```
Focus: Clean up code
- Improve design
- Keep tests green
- Commit each refactor
```

**15:00 - Integration Phase (until 16:00)**

```
Focus: Make it work together
- Run full test suite
- Fix any breakages
- Update checklists
```

**16:00 - Code Review (30 min)**

```
Review teammate's work:
- Check RED-GREEN-REFACTOR commits
- Verify tests exist
- Check test coverage
- Approve or request changes
```

**16:30 - Retro/Planning (30 min)**

```
Daily retrospective:
- What went well?
- What didn't?
- Action items for tomorrow
```

---

## Part 4: Pair Programming Workflow

### Driver-Navigator Pattern

**Roles:**

**Driver (at keyboard):**
- Types code
- Runs tests
- Focuses on implementation
- Asks for help when stuck

**Navigator (reviewing):**
- Reads legacy code
- Reviews test approach
- Catches issues early
- Thinks about next step

**Switch every 20 minutes** or after each TDD cycle

---

### Pair Programming SOP

**Session Start (5 min)**

```
1. Choose file to migrate
2. Set up git branch
3. Open test file
4. Identify first function
5. Decide who drives first
```

**TDD Cycle Together (30-45 min)**

```
RED Phase (Navigator leads):
- Navigator: "Let's write a test for strlen()"
- Driver: Writes test
- Navigator: "Good, commit it"

GREEN Phase (Driver leads):
- Driver: "Now I'll implement it"
- Navigator: "Watch out for encoding"
- Driver: Writes code
- Navigator: "Tests green, commit"

REFACTOR Phase (Together):
- Navigator: "Let's extract this constant"
- Driver: Types refactor
- Both: Run tests together
- Navigator: "Good commit"

Switch roles and repeat.
```

**Session End (5 min)**

```
1. Run full test suite
2. Create pull request
3. Mark tasks complete
4. Update checklist
```

---

## Part 5: Troubleshooting Guide

### Problem: Test Won't Fail (Red Phase)

**Symptoms:**
- Test passes when it should fail
- Function already exists
- Legacy code already works

**Solutions:**

```bash
# 1. Verify test is actually testing the new code
vim tests/Unit/GlobalFunctionsTest.php

# Make sure namespace is correct
namespace Tests\Unit;

use App\Utils\StringFunctions; // ← Must import

# 2. Delete old implementation
rm src/Utils/StringFunctions.php

# 3. Run test again
vendor/bin/phpunit tests/Unit/GlobalFunctionsTest.php
# Should fail now
```

---

### Problem: Test Won't Pass (Green Phase)

**Symptoms:**
- Test keeps failing
- Can't make it work
- Stuck for >30 min

**Solutions:**

```bash
# 1. Spike solution (research time)
# Create spike branch
git checkout -b spike-solution

# Try different approaches
# - Read legacy code more carefully
# - Search for similar implementations
# - Ask teammate for help

# 2. Use debugger
php -dxdebug.mode=debug vendor/bin/phpunit ...

# 3. Simplify test
# Maybe test is too complex?
# Break into smaller tests

# 4. Timebox: 45 min max
# If still failing → ask for help
# Don't waste whole day
```

---

### Problem: Tests Failing During Refactor

**Symptoms:**
- Was GREEN, now RED after refactor
- Refactor broke something

**Solutions:**

```bash
# IMMEDIATE ACTION:
git revert HEAD  # Undo the refactor

# Then:
# 1. Run tests to confirm GREEN
vendor/bin/phpunit

# 2. Refactor in smaller steps
# Instead of big change, do tiny changes

# Step 2.1: Extract constant
# Test
git commit -m "refactor: extract MAGIC_NUMBER"

# Step 2.2: Rename variable
# Test
git commit -m "refactor: rename x to totalCount"

# 3. Run tests after EACH step
```

---

### Problem: Integration Tests Fail

**Symptoms:**
- Unit tests pass, integration fail
- Works alone, breaks with others

**Solutions:**

```bash
# 1. Check for global state
# Did you use singletons?
# Did you modify $_SESSION directly?

# 2. Check for dependency issues
# Are you importing the right class?
# Is namespace correct?

# 3. Run with verbose output
vendor/bin/phpunit --verbose

# 4. Use debugger
# Set breakpoint in integration test
# Step through to find issue
```

---

## Part 6: Quality Gates

### Must Pass Before Moving On

**Gate 1: Red Phase**
✅ Test written and committed separately
✅ Test fails with expected error
✅ Test name describes behavior clearly

**Gate 2: Green Phase**
✅ Implementation passes the test
✅ Implementation committed separately
✅ No extra code beyond what's needed

**Gate 3: Refactor Phase**
✅ All tests still pass
✅ Code style compliant
✅ Types correct (no PHP errors)
✅ Refactoring committed separately

**Gate 4: File Complete**
✅ All functions have tests
✅ Test coverage >80%
✅ Integration tests pass
✅ No PHP deprecation warnings
✅ File checklist marked complete

---

## Part 7: Sprint Planning with TDD

### Sprint Planning Meeting (1 hour)

**Step 1: Select Files (15 min)**

```
From 07-file-migration-checklist.md, pick:
- 3-5 P0/P1 files
- Prioritize by dependencies
- Consider complexity

Example Sprint 2:
☐ config/app.php (2 hours)
☐ bootstrap/app.php (3 hours)
☐ Database/Connection.php (5 hours)
Total: ~10-12 hours → 2 days
```

**Step 2: Assign Tests (15 min)**

```
For each file, list test cases:

config/app.php:
- ☐ testConfigLoadsFromArray
- ☐ testEnvironmentVariablesOverride
- ☐ testUtf8CharsetIsSet
- ☐ testConfigThrowsOnMissingRequiredKey

Total: 4 tests → ~2 hours RED phase
```

**Step 3: Estimate Time (15 min)**

```
File                  | RED | GREEN | REFACTOR | Total
---------------------|------|--------|-----------|-------
config/app.php        | 2h   | 1h     | 1h        | 4h
bootstrap/app.php      | 3h   | 2h     | 1h        | 6h
Database/Connection.php| 5h   | 3h     | 2h        | 10h
---------------------|------|--------|-----------|-------
TOTAL                 | 10h  | 6h     | 4h        | 20h
```

**Step 4: Pair Assignment (15 min)**

```
Assign pairs for efficiency:

Day 1-2:
- Alice + Bob: config/app.php
- Carol + You: bootstrap/app.php

Day 3-4:
- Alice + Carol: Database/Connection.php (harder)
- Bob + You: Write tests for Connection
```

---

## Part 8: Commit Message Standards

### TDD Commit Pattern

**Each TDD cycle = 3 commits:**

```bash
# Commit 1: RED
git commit -m "test(authcode): add failing test for encryption (RED)

- Test expects authcode() to encrypt string
- Test verifies encrypted output differs from input
- Currently fails: Call to undefined function authcode()

Related: #123"
```

```bash
# Commit 2: GREEN
git commit -m "feat(authcode): implement encryption function (GREEN)

- Migrate authcode() to App\Utils\AuthCode class
- Implement using OpenSSL AES-256-CBC
- Simplest implementation to make test pass

Closes: #123"
```

```bash
# Commit 3: REFACTOR
git commit -m "refactor(authcode): extract constants and improve naming

- Extract CIPHER_METHOD and HASH_ALGO to class constants
- Rename $str to $plaintext for clarity
- Improve error handling with exceptions
- All tests still passing

Refines: #123"
```

---

## Part 9: Physical Workspace Setup

### Ideal Monitor Layout

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Left Monitor (30%)         │  Right Monitor (70%)           │
│  ┌──────────────┐          │  ┌──────────────────────────┐   │
│  │ Legacy Code   │          │  │ Modern Code + Tests     │   │
│  │ (Read only)   │          │  │ - Writing tests here   │   │
│  │               │          │  │ - Implementing here   │   │
│  │ gVim/poketb/ │          │  │ - Refactoring here     │   │
│  │               │          │  │                       │   │
│  └──────────────┘          │  └──────────────────────────┘   │
│                                   │                       │
│  Bottom Left: Terminal     │  Bottom Right: Test Results    │
│  ┌──────────────┐          │  ┌──────────────────────────┐   │
│  │ $ git status  │          │  │ PHPUnit Output          │   │
│  │ $ vim tests/ │          │  │ OK (15 tests, 45... │   │
│  └──────────────┘          │  └──────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Terminal Tabs

```bash
# Tab 1: Legacy Code
cd ~/poketb.com/bbs
vim include/global.func.php

# Tab 2: Modern Code
cd ~/modern-php/app
vim src/Utils/StringFunctions.php

# Tab 3: Tests
cd ~/modern-php
vim tests/Unit/Utils/StringFunctionsTest.php

# Tab 4: Git/Tests
cd ~/modern-php
watch -n 2 'vendor/bin/phpunit --color=always'

# Tab 5: Server/Logs
cd ~/modern-php
tail -f /var/log/php-fpm/error.log
```

---

## Part 10: Progress Tracking

### Daily Progress Template

**Create daily progress file:**

```markdown
# Progress: YYYY-MM-DD

## Files Migrated Today

### config/app.php ✅
- Started: 09:15
- Finished: 11:30
- Tests written: 4
- Test coverage: 100%
- Commits: 12 (4 RED + 4 GREEN + 4 REFACTOR)
- Challenges: Environment variable precedence
- Notes: Works correctly, UTF-8 validated

### bootstrap/app.php 🔄 (in progress)
- Started: 13:30
- Current status: GREEN phase (2/6 tests passing)
- Blocked by: Session handling unclear
- ETA: Tomorrow 10:00

## Tomorrow's Plan
1. Complete bootstrap/app.php
2. Start Database/Connection.php
3. Write integration tests for both

## Blockers
- Need clarification on session_start() placement

## Learnings Today
- TDD cycle really does prevent bugs
- Pair programming speeds up RED phase
```

---

## Part 11: Emergency Procedures

### When Tests Take Too Long

**Symptom:** Stuck in RED or GREEN >60 minutes

**Action:**

```bash
# 1. Stop and pair
# Get fresh eyes on problem

# 2. Create spike branch
git checkout -b spike-<function-name>

# 3. Timebox: 30 minutes
# Try anything, throw away code
# Just figure out the approach

# 4. Document findings
vim notes/<function-name>-spike.md

# 5. Return to main branch
git checkout main
git merge spike-<function-name>
```

---

### When Tests Keep Failing

**Symptom:** Can't get to GREEN

**Action:**

```bash
# 1. Revert to last known good
git log --oneline
git checkout <last-green-commit>

# 2. Analyze why it fails
# - Is test wrong?
# - Is legacy code misunderstood?
# - Is environment different?

# 3. Ask for help
# Don't waste >2 hours

# 4. Maybe skip and come back
# Mark as blocked, move to next file
```

---

## Success Criteria

✅ **Every file migrated via TDD cycle**
✅ **100% test coverage on P0/P1 files**
✅ **All commits follow RED-GREEN-REFACTOR pattern**
✅ **Zero regressions in integration tests**
✅ **Team follows standard workflow daily**

---

## Quick Reference Card

**Print this and keep on desk:**

```
═══════════════════════════════════════════════════════
  TDD CYCLE - RED → GREEN → REFACTOR
═══════════════════════════════════════════════════════

🔴 RED (Write Test First)
  ☐ Read legacy code
  ☐ Write test for expected behavior
  ☐ Run test → MUST FAIL
  ☐ Commit: "test(...): ... (RED)"

🟢 GREEN (Make Test Pass)
  ☐ Write simplest implementation
  ☐ Run test → MUST PASS
  ☐ Don't refactor yet
  ☐ Commit: "feat(...): ... (GREEN)"

🔵 REFACTOR (Clean Up)
  ☐ Improve code quality
  ☐ Keep tests green
  ☐ Run tests after each change
  ☐ Commit: "refactor(...): ..."

➡️ NEXT
  ☐ Mark task complete
  ☐ Move to next test
  ☐ Start RED again

═══════════════════════════════════════════════════════
```

---

**Document Status:** ✅ Complete
**Related Documents:**
- 09-testing-strategy.md (General testing approach)
- 11-file-level-test-plan.md (File-by-file test cases)
- 06-execution-sprints.md (Weekly planning)
