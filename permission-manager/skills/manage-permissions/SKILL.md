---
name: manage-permissions
description: Use when a tool call is denied by the user, when the user asks about permissions, or when you notice repeated permission prompts during a session. Classifies commands into allow/deny/ask categories and collects them for a batch update at the end of the session.
---

# Permission Manager

When you encounter a permission prompt or denial during a session, classify the command and **collect it silently**. Do NOT update settings immediately. Instead, keep a mental note of all new permission patterns encountered during the session and propose them as a batch update when the user is finishing up (e.g., before a commit, at the end of a task, or when explicitly asked).

**During the session**: When a command is prompted or denied, briefly note the classification (one line, e.g., "Noted: `pnpm audit` → safe to auto-allow"). Continue with the task.

**At session end**: Present a single summary table of all collected permissions and ask for confirmation before updating the settings file.

## Decision Framework

When you encounter an unknown command, classify it by asking these questions **in order**:

1. **Can it destroy data or state that can't be recovered?** → **DENY**
   - Deletes files, drops databases, force-pushes, overwrites history
2. **Can it bypass other permission rules?** → **DENY**
   - Inline code execution (`node -e`, `python -c`, `bash -c`, `eval`)
   - These can run arbitrary commands that sidestep allow/deny lists
3. **Does it affect shared state visible to others?** → **ASK FIRST**
   - Pushes code, creates/closes PRs or issues, sends messages, modifies remote resources
4. **Does it change local working context in hard-to-undo ways?** → **ASK FIRST**
   - Switches branches (loses uncommitted work context), merges, rebases
   - Moves or renames files (can break references)
   - Kills processes, modifies system config
5. **Is it read-only, local-only, or easily reversible via git?** → **ALLOW**
   - File reads, searches, listing, status checks
   - Local git operations (add, commit, stash, diff, log)
   - Build, test, lint, format commands
   - File creation and copying (tracked by git)

**When uncertain**: default to ASK FIRST. It's always safer to prompt once and then allow than to auto-allow something risky.

## Known Patterns

These are common commands pre-classified for quick reference. For anything not listed, use the decision framework above.

### AUTO-ALLOW (safe, local, reversible)

**Git (read-only + local writes):**
- `git status`, `git diff`, `git log`, `git show`, `git branch`, `git stash`
- `git add`, `git commit`, `git fetch`, `git remote`, `git rev-parse`, `git merge-base`
- `git tag` (listing only), `git blame`, `git shortlog`, `git reflog`

**Package managers (build/test/lint) — any package manager follows the same pattern:**
- Install: `pnpm install`, `npm install`, `yarn install`, `bun install`, `cargo fetch`, `pip install`, `mix deps.get`
- Build/test/lint: `pnpm build`, `npm test`, `yarn lint`, `bun run`, `cargo build`, `cargo test`, `mix test`, `go build`, `go test`, `make`
- Workspace commands: `pnpm --filter`, `pnpm ls`, `pnpm -r`, `pnpm -w`, `pnpm why`

**Known safe npx/tool runners:**
- `npx husky`, `npx lint-staged`, `npx turbo`, `npx tsc`
- `npx jest`, `npx playwright`, `npx vitest`, `npx eslint`, `npx prettier`
- `npx prisma generate`, `npx prisma db push`, `npx prisma migrate dev`, `npx prisma studio`, `npx prisma format`, `npx prisma validate`
- `npx nest` (NestJS CLI)

**Docker (dev lifecycle):**
- `docker compose up`, `docker compose down`, `docker compose ps`, `docker compose logs`
- `docker ps`, `docker logs`, `docker images`

**Shell (read-only + safe utilities):**
- `ls`, `tree`, `du`, `wc`, `which`, `diff`, `mkdir`, `cp`, `touch`
- `ps aux`, `lsof -i`, `head`, `tail`, `find`, `grep`, `echo`, `cat`
- `node --version`, `node -v`, `python --version`, `ruby --version`

**GitHub CLI (read-only):**
- `gh pr view`, `gh pr list`, `gh pr status`, `gh pr diff`, `gh pr checks`
- `gh issue list`, `gh issue view`, `gh issue status`
- `gh repo view`, `gh api`, `gh auth status`, `gh run list`, `gh run view`

**MCP tools — apply the same decision framework:**
- Read/search/list tools from any MCP server → ALLOW (same risk as the `Read`/`Grep` built-in tools)
- Edit/write/create tools from any MCP server → ALLOW (same risk as `Edit`/`Write` built-in tools)
- Tools that send messages, create external resources, or call external APIs → ASK FIRST

### AUTO-DENY (destructive, irreversible, or escape hatches)

**File destruction:**
- `rm -rf`, `rm -r`, `shred`

**Git destruction:**
- `git push --force`, `git push -f`
- `git reset --hard`
- `git clean -f`, `git clean -fd`
- `git checkout .`, `git checkout -- .`, `git restore .`
- `git branch -D`

**Escape hatches (bypass other rules):**
- `node -e`, `node --eval`
- `python -c`, `python3 -c`
- `bash -c`, `sh -c`
- `eval`

**Database destruction:**
- Any command containing `reset` for database tools (e.g., `prisma db reset`, `prisma migrate reset`, `db:reset`)
- `DROP DATABASE`, `DROP TABLE`

**System-level:**
- `chmod 777`, `chown`, `sudo`

### ASK FIRST (shared state, judgment required)

Do NOT auto-add these — they should remain as "ask first":

- `git push` (any non-force push — affects remote)
- `git checkout <branch>`, `git switch` (changes working context)
- `git merge`, `git rebase` (can create conflicts)
- `gh pr create`, `gh issue create`, `gh issue close` (visible to others)
- `mv` (can break references)
- `rm <file>` (single file deletion — reversible via git but worth confirming)
- `kill`, `pkill` (process management)
- Any network request tool (`curl`, `wget`, `WebFetch`)
- Any broad wildcard pattern (`npx:*`, `node:*`, `bash:*`)

## Process

### During the session (collect silently)

When you detect a denied or prompted permission:

1. **Classify** the command using the decision framework (or the known patterns list for quick matches)
2. **Note it briefly** in your response (one line): "Noted: `<command>` → allow/deny/ask-first"
3. **Continue with the task** — do NOT stop to update settings

Also note when the user explicitly approves a prompted command ("allow once") — this is a signal the command is safe for them and a candidate for the allow list.

### Compound commands

If the command uses `&&`, `||`, or `;` to chain multiple commands, classify each part separately. For example:

- `git add . && git commit -m "msg"` → both parts are safe → ALLOW both `Bash(git add:*)` and `Bash(git commit:*)`
- `pnpm build && git push` → build is safe, push affects remote → ALLOW `Bash(pnpm build:*)`, leave `git push` as ask-first

### At session end (batch update)

When the user is wrapping up, or when `/update-permissions` is invoked:

1. **Read** the current `.claude/settings.local.json` (project-level). If it doesn't exist, check `~/.claude/settings.json` (global)
2. **Audit existing patterns** for security issues:
   - Flag overly broad wildcards (e.g., `Bash(git:*)`, `Bash(npx:*)`, `Bash(node:*)`) that match destructive commands
   - For each broad pattern, propose specific replacements using the known patterns list
   - Present these as a separate "Security Audit" section before the new permissions table
3. **Scan the conversation** for any permission denials or prompts you may have missed (don't rely solely on memory — context compression may have dropped earlier notes)
4. **Present a summary table** of all collected permissions:

| Command | Pattern | Classification | Reason |
|---------|---------|---------------|--------|
| `pnpm audit` | `Bash(pnpm audit:*)` | Allow | Read-only, local |
| `node -e "..."` | `Bash(node -e:*)` | Deny | Escape hatch |
| `git push` | — | Ask first | Affects remote |

5. **Ask which settings file** to update: "Add to project settings (`.claude/settings.local.json`) or global (`~/.claude/settings.json`)?" Default to project-level if one exists.
6. **Ask for confirmation**: "Want me to add these to your settings?"
7. **On approval**: Read the current settings, add the new patterns, write the updated file
8. **Skip duplicates**: Don't add patterns already in the list
9. **Report what changed**: Show the user exactly what was added

## Pattern Format

Use the Claude Code permission pattern format:

- Bash commands: `Bash(<command prefix>:*)`
- MCP tools: `mcp__<server>__<tool>`
- Built-in tools: `Read`, `Edit`, `Write`, `Glob`, `Grep`, `Task`, etc.

Examples:
- `Bash(git status:*)` — matches `git status`, `git status --short`, etc.
- `Bash(pnpm test:*)` — matches `pnpm test`, `pnpm test:unit`, etc.
- `Bash(npx prisma generate:*)` — matches only prisma generate, not other npx commands

## Security Audit: Broad Wildcards

When auditing existing settings, flag any pattern that matches an entire command namespace. These are dangerous because they auto-allow destructive subcommands:

| Broad Pattern | Risk | Safe Replacements |
|---------------|------|-------------------|
| `Bash(git:*)` | Matches `git push --force`, `git reset --hard`, `git clean -f` | Use individual safe git commands (status, diff, log, show, add, commit, fetch, stash, branch, rev-parse, merge-base, remote, blame, shortlog, reflog, tag, worktree) |
| `Bash(npx:*)` | Matches any npx command including arbitrary code execution | Use specific tool patterns like `Bash(npx prisma generate:*)`, `Bash(npx tsc:*)` |
| `Bash(node:*)` | Matches `node -e` (escape hatch) | Use `Bash(node --version:*)` or specific script patterns |
| `Bash(npm:*)` | Matches `npm exec` which can run arbitrary code | Use `Bash(npm install:*)`, `Bash(npm test:*)`, `Bash(npm run:*)` |
| `Bash(docker:*)` | Matches `docker rm`, `docker rmi --force` | Use specific docker compose/ps/logs patterns |

When you find a broad wildcard, **always recommend replacing it** with the specific safe subcommands from the Known Patterns section. Present the replacement as a table showing what's being expanded and what stays as ask-first.

## Important

- **Never add broad wildcards** like `Bash(npx:*)`, `Bash(node:*)`, or `Bash(bash:*)` — these are escape hatches that bypass all other rules
- **Prefer specific prefixes** — `Bash(npx prisma generate:*)` over `Bash(npx prisma:*)` — narrower patterns are safer
- **Keep deny rules tight** — include both long and short flag variants (e.g., both `--force` and `-f`)
- **Check for duplicates** before adding — don't add patterns already in the list
- **Preserve existing entries** — never remove entries the user already has
- **When in doubt, ASK FIRST** — it's always safer to prompt once than to auto-allow something risky
