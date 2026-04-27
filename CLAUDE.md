# CLAUDE.md — Team AI Development Guide

> Drop this file in the root of any Git repo. Claude reads it automatically at the start of every session.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Parallel Git Worktrees](#parallel-git-worktrees)
3. [Plan Mode & Staff Review](#plan-mode--staff-review)
4. [Prompting Best Practices](#prompting-best-practices)
5. [Self-Improvement via @.claude Tags](#self-improvement-via-claude-tags)
6. [Verification Checklists](#verification-checklists)
7. [Slash Commands](#slash-commands)
8. [Style Conventions](#style-conventions)
9. [Common Pitfalls](#common-pitfalls)
10. [Team Tips & Tribal Knowledge](#team-tips--tribal-knowledge)

---

## Project Overview

```
# Fill this in for each repo:
Project:      <name>
Stack:        <e.g., TypeScript / Node.js / PostgreSQL / React>
Test runner:  <e.g., vitest, jest, pytest>
Lint:         <e.g., eslint --fix, ruff>
Build:        <e.g., pnpm build>
Deploy:       <e.g., GitHub Actions → Fly.io>
Docs:         <link to internal wiki or README>
```

Claude should read this section first to orient itself to the project's shape before touching any code.

---

## Parallel Git Worktrees

Use worktrees to run multiple Claude sessions simultaneously without branch conflicts. Each worktree is an independent checkout — Claude can work on `feature/auth` while you review `feature/payments` in another terminal.

### Setup

```bash
# Create a worktree for a new feature branch
git worktree add ../project-auth feature/auth

# Open a second Claude session in that directory
cd ../project-auth && claude

# List all active worktrees
git worktree list

# Remove when done
git worktree remove ../project-auth
```

### Naming Convention

```
../project-<short-slug>   →  e.g., ../project-auth, ../project-billing
```

### Rules for Worktree Sessions

- Each Claude session should **only operate within its own worktree directory**. It should never reach into a sibling worktree.
- Shared config files (`.env`, `docker-compose.yml`) live in the primary worktree. Symlink if needed.
- Run tests in the worktree, not the primary. Avoid touching `main` or `staging` from inside a worktree session.
- When a worktree task is complete, open a PR from that branch. Then remove the worktree.

### Recommended Workflow

```
main worktree  →  review, merge, deploy
worktree-A     →  Claude working on feature A
worktree-B     →  Claude working on hotfix B
worktree-C     →  Claude working on refactor C
```

This is the core pattern for parallelizing AI-assisted development safely.

---

## Plan Mode & Staff Review

Before writing any code on a non-trivial task, Claude should enter **plan mode** — producing a structured plan for human review before touching files.

### When to Use Plan Mode

Use it whenever the task involves:

- New files or modules
- Schema changes
- API surface changes
- Multi-step refactors across more than 2 files
- Anything touching auth, payments, or data integrity
- Features with unclear requirements

### How to Trigger Plan Mode

Add `--plan` to your prompt, or start with:

```
Before writing any code, give me a plan.
```

Claude should respond with:

```markdown
## Plan

**Goal:** <one-sentence summary>

**Files affected:**
- `src/auth/middleware.ts` — add JWT validation
- `src/routes/user.ts` — protect user endpoint

**Steps:**
1. ...
2. ...
3. ...

**Risks / open questions:**
- Does this change affect session handling for OAuth users?

**Ready to proceed?** Respond with "go" or ask to revise.
```

### Staff Review Gate

For high-impact tasks, plans should be reviewed by a senior engineer **before** Claude proceeds. Mark these with:

```
[STAFF REVIEW REQUIRED]
```

Claude should not continue past a plan flagged with this marker until a human responds with explicit approval.

### Revision Loop

If the plan needs changes, say:

```
Revise step 2. The endpoint already exists — just add the middleware guard.
```

Claude updates the plan and waits for re-approval. This loop catches ambiguity early, before code sprawl sets in.

---

## Prompting Best Practices

These patterns produce better results across the team. Follow them consistently.

### 1. Be Specific About Scope

**Weak:**
```
Fix the login bug.
```

**Strong:**
```
In `src/auth/login.ts`, the `authenticateUser` function returns 200 on failed
password attempts when the user account is locked. Fix the status code — it
should return 403 with the message "Account locked."
```

### 2. Give Claude the Full Context It Needs

```
Here is the current schema for `users` table: <paste>
Here is the failing test: <paste>
Fix the implementation so the test passes without changing the test.
```

Claude has no memory between sessions. Load context explicitly.

### 3. Anchor to Specific Files

```
Only touch `src/payments/webhook.ts`. Don't refactor adjacent files unless I ask.
```

Without this, Claude may "helpfully" clean up nearby code — introducing scope creep.

### 4. Use Positive Constraints, Not Just Negative

Instead of: `Don't use any libraries.`  
Use: `Implement this using only the Node.js standard library.`

### 5. Ask for Step-by-Step Reasoning on Hard Problems

```
Before answering, think through the edge cases in this race condition.
Walk me through your reasoning before writing code.
```

### 6. One Task Per Message on Complex Work

Break large tasks into sequential messages. Finish step 1, verify it, then ask for step 2. Don't dump 10 requirements in one shot.

### 7. Use "Check Your Work" at the End

```
After implementing, re-read your changes and check for:
- Type errors
- Missing null checks
- Unhandled promise rejections
```

### 8. Cite the Docs When Relevant

```
According to our API contract in `/docs/api.md`, the `id` field must be a UUID.
Make sure the implementation matches.
```

Claude will follow documented conventions more reliably when you point to them directly.

---

## Self-Improvement via @.claude Tags

The `@.claude` tag system lets the team annotate code and documentation so Claude can find and improve its own instructions over time.

### How It Works

Tag any comment in the codebase with `@.claude` to flag it for Claude's attention during future sessions.

```typescript
// @.claude PITFALL: This function silently swallows errors on timeout.
// If you refactor this, make sure timeouts throw explicitly.
async function fetchWithRetry(url: string) { ... }
```

```python
# @.claude CONVENTION: All database writes go through the repo layer.
# Never call the ORM directly from a route handler.
def create_user(data: UserCreate) -> User:
    return user_repo.create(data)
```

### Tag Types

| Tag | Purpose |
|-----|---------|
| `@.claude PITFALL:` | Warn Claude about something that commonly breaks |
| `@.claude CONVENTION:` | Enforce a team pattern Claude should follow |
| `@.claude TODO:` | Work Claude could take on in a future session |
| `@.claude CONTEXT:` | Background Claude needs to understand this code |
| `@.claude REVIEW:` | Flag for human review before shipping |

### Self-Improvement Loop

At the end of any session where Claude encounters unexpected issues:

1. Claude should suggest a new `@.claude` tag to add near the relevant code.
2. A human approves and commits the tag.
3. Future Claude sessions read it automatically and avoid the same mistake.

Example prompt to trigger this:

```
Based on what you just debugged, add an @.claude tag near the problematic
code so future sessions know about this edge case.
```

This turns individual debugging sessions into accumulated team knowledge.

---

## Verification Checklists

Claude should run through the relevant checklist before considering any task complete.

### ✅ Code Change Checklist

```
[ ] Tests pass: <run command>
[ ] Linter passes: <run command>
[ ] Build succeeds: <run command>
[ ] No new TypeScript / type errors
[ ] No hardcoded secrets or credentials
[ ] No console.log / debug statements left in
[ ] Edge cases handled (null, empty, large input)
[ ] Error paths handled explicitly (no silent failures)
[ ] New functions have docstrings or inline comments for non-obvious logic
[ ] Changes are scoped to files listed in the plan
```

### ✅ PR Readiness Checklist

```
[ ] Branch name matches convention: <type>/<short-slug>
[ ] Commit messages are clean and descriptive
[ ] PR description explains what changed and why
[ ] Relevant tests added or updated
[ ] No unrelated files changed
[ ] Migration scripts included if schema changed
[ ] Breaking changes documented in PR description
[ ] Linked to issue or ticket if applicable
```

### ✅ Database / Schema Change Checklist

```
[ ] Migration is reversible (down migration exists)
[ ] Tested on a copy of production data if possible
[ ] No column renames without a multi-step migration plan
[ ] Indexes added for any new query patterns
[ ] Reviewed by a second engineer before merging
```

### ✅ Security-Sensitive Checklist

```
[ ] Input is validated and sanitized
[ ] SQL queries use parameterized statements
[ ] Auth checks are present and tested
[ ] Sensitive data is not logged
[ ] Dependencies checked against known vulnerabilities
```

---

## Slash Commands

These are team-standard slash commands Claude can execute in any session. Define them here so the whole team uses consistent language.

### `/pr`

Generate a pull request description for the current branch.

```
/pr

Output:
## What changed
<summary of changes>

## Why
<motivation or linked issue>

## How to test
<steps to verify locally>

## Checklist
- [ ] Tests pass
- [ ] Lint passes
- [ ] Reviewed by second engineer
```

### `/commit`

Generate a commit message following Conventional Commits format.

```
/commit

Output:
feat(auth): add JWT refresh token rotation

- Rotates refresh token on each use to prevent replay attacks
- Stores token hash in Redis with 7-day TTL
- Returns new refresh token in HTTP-only cookie

Closes #142
```

**Commit types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`

### `/plan`

Produce a structured plan before writing code (see Plan Mode section above).

```
/plan <task description>
```

### `/review`

Ask Claude to review the current diff or a pasted code block.

```
/review

Claude will check for:
- Logic errors
- Missing error handling
- Security issues
- Style violations
- Anything that should be flagged for a human reviewer
```

### `/explain`

Get a plain-English explanation of a function, file, or concept.

```
/explain src/queue/processor.ts
```

### `/test`

Generate tests for a given function or module.

```
/test src/utils/parseDate.ts

Claude will:
- Write unit tests covering happy path, edge cases, and failure cases
- Use the team's test runner and conventions
- Not modify the source file
```

### `/checklist`

Run through the relevant verification checklist for the current task.

```
/checklist pr
/checklist schema
/checklist security
```

---

## Style Conventions

These are enforced by Claude in every session. Update this section when team conventions change.

### General

- Prefer explicit over implicit. Clear beats clever.
- Functions do one thing. If it needs a conjunction in the name, split it.
- No magic numbers — use named constants.
- Errors are thrown, not returned (unless the pattern is already established).

### TypeScript

```typescript
// ✅ Use explicit return types on public functions
export function parseUserId(raw: string): UserId { ... }

// ✅ Use `const` by default, `let` only when reassignment is needed
const config = loadConfig();

// ✅ Use discriminated unions over boolean flags
type Result = { ok: true; data: User } | { ok: false; error: string };

// ❌ Avoid `any`. Use `unknown` and narrow.
function process(input: unknown) { ... }

// ❌ Avoid barrel exports that make tree-shaking harder
// import { everything } from './index'
```

### File Structure

```
src/
  features/          # Domain-grouped feature modules
    auth/
      auth.routes.ts
      auth.service.ts
      auth.test.ts
  shared/            # Cross-cutting utilities
  config/            # Environment and app config
  db/                # Database client and migrations
```

### Naming

| Thing | Convention | Example |
|-------|-----------|---------|
| Files | kebab-case | `user-service.ts` |
| Functions | camelCase | `getUserById` |
| Types / Interfaces | PascalCase | `UserProfile` |
| Constants | SCREAMING_SNAKE | `MAX_RETRY_COUNT` |
| DB tables | snake_case | `user_sessions` |

### Tests

- Test file lives next to source: `auth.service.test.ts`
- Test names: `describe('functionName') > it('should <behavior> when <condition>')`
- No mocking unless necessary. Prefer real implementations with test databases.
- One assertion concept per test.

### Git Branches

```
feat/<slug>       →  new feature
fix/<slug>        →  bug fix
refactor/<slug>   →  non-behavioral change
chore/<slug>      →  tooling, deps, config
hotfix/<slug>     →  production emergency
```

---

## Common Pitfalls

Patterns that have caused real problems on this team. Claude should actively avoid these.

### 1. Scope Creep

**What happens:** Claude "cleans up" adjacent code while completing a task, introducing unreviewed changes.

**Avoid by:** Anchoring Claude to specific files in the prompt. Use `/plan` to agree on scope upfront.

```
Only modify the files listed in the plan. If you see something worth improving
elsewhere, add an @.claude TODO tag and mention it — don't change it.
```

---

### 2. Silent Error Swallowing

**What happens:** Errors are caught and logged but execution continues, hiding failures.

```typescript
// ❌ This hides errors
try {
  await sendEmail(user);
} catch (err) {
  console.log(err);
}

// ✅ This surfaces them
try {
  await sendEmail(user);
} catch (err) {
  logger.error({ err, userId: user.id }, 'Failed to send welcome email');
  throw err; // or handle with intent
}
```

---

### 3. Missing Null / Undefined Guards

**What happens:** Claude assumes data is present. Production breaks when it isn't.

**Pattern to use:**

```typescript
const user = await getUser(id);
if (!user) throw new NotFoundError(`User ${id} not found`);
// Now it's safe to use user
```

---

### 4. Skipping the Plan on "Small" Changes

**What happens:** A small-looking change touches six files and breaks two unrelated things.

**Rule:** If you're not sure if it's small, use `/plan`. The plan takes 30 seconds. The debugging takes 3 hours.

---

### 5. Treating Claude's First Draft as Final

Claude produces good first drafts. It doesn't produce perfect production code automatically.

**Always:**
- Read the diff before accepting it
- Run tests
- Check the verification checklist
- Ask Claude to review its own work before you do

---

### 6. Overloading a Single Prompt

Giving Claude 10 requirements at once leads to code that partially satisfies all of them and fully satisfies none.

**Better:** Sequential messages. Confirm step 1 before asking for step 2.

---

### 7. Not Providing Enough Context

Claude has no memory between sessions. If you don't paste the relevant schema, interface, or function signature, Claude will guess — and guesses are wrong.

**Always paste:**
- The function or file being modified
- Relevant type definitions
- Failing test or error message
- Any constraints from other parts of the system

---

### 8. Letting Claude Rename or Restructure Without Explicit Approval

Claude may propose renaming functions or reorganizing files as part of a refactor. Unless the plan explicitly included restructuring, push back.

```
Don't rename existing functions. Keep the current public interface intact.
```

---

## Team Tips & Tribal Knowledge

Accumulated from team sessions. Add to this list whenever something saves real time.

---

**💡 Ask Claude to check its own work**

After any implementation, prompt:
```
Re-read what you just wrote. Check for missing error handling, type issues,
and anything that diverges from what we agreed in the plan.
```
This catches 20–30% of issues before you even run tests.

---

**💡 Use Claude for PR descriptions, not just code**

Once a feature is done:
```
/pr
```
Claude writes a clean, structured PR description in seconds. Edit it, don't skip it.

---

**💡 Paste error messages verbatim**

Don't paraphrase. Paste the full stack trace. Claude pattern-matches on exact error text.

---

**💡 Use worktrees for exploratory spikes**

When you're not sure how to approach something, spin up a worktree, let Claude explore, then throw it away. Keep your main branch clean.

---

**💡 Tag edge cases as you discover them**

When you find a non-obvious edge case during code review:
```
// @.claude PITFALL: This function returns null for users created before 2022-01-01.
// Always check for null before calling `.preferences`.
```

---

**💡 Give Claude a persona for style-sensitive tasks**

```
You are a senior backend engineer on this team. Write this as if you've been
working on this codebase for two years and know all our conventions.
```

Output quality improves noticeably on review tasks and documentation.

---

**💡 Use /explain before touching unfamiliar code**

Before editing a file you haven't worked in before:
```
/explain src/payments/reconcile.ts
```
Claude will summarize what it does, why it exists, and what to be careful about — before you touch it.

---

**💡 Commit plans as comments in large PRs**

Paste the Claude-generated plan into the PR description under a `## Plan` header. It helps reviewers understand intent, not just diff.

---

**💡 End sessions with a handoff note**

Before closing a long session:
```
Summarize what we built, what's still open, and what a new Claude session
would need to know to pick this up tomorrow.
```
Paste that summary into a comment or scratch file. Start the next session by showing it to Claude.

---

*Last updated: <!-- update this when you revise -->*  
*Maintainer: <!-- team or individual -->`*  
*Questions: open an issue or ping `#engineering` in Slack*
