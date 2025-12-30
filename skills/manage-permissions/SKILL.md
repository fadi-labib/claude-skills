---
name: manage-permissions
description: Use when a tool call is denied by the user, when the user asks about permissions, or when you notice repeated permission prompts during a session. Classifies commands into allow/deny/ask categories and collects them for a batch update at the end of the session.
---

# Permission Manager

When you encounter a permission prompt or denial during a session, classify the command and **collect it silently**. Do NOT update `.claude/settings.local.json` immediately. Instead, keep a mental note of all new permission patterns encountered during the session and propose them as a batch update when the user is finishing up (e.g., before a commit, at the end of a task, or when explicitly asked).

**During the session**: When a command is prompted or denied, briefly note the classification (one line, e.g., "Noted: `pnpm audit` → safe to auto-allow"). Continue with the task.

**At session end**: Present a single summary table of all collected permissions and ask for confirmation before updating the settings file.

## Classification Rules

### AUTO-ALLOW (safe, local, reversible)

These patterns should be added to the `allow` list:

**Git (read-only + local writes):**
- `git status`, `git diff`, `git log`, `git show`, `git branch`, `git stash`
- `git add`, `git commit`, `git fetch`, `git remote`, `git rev-parse`, `git merge-base`
- `git tag` (listing only), `git blame`, `git shortlog`, `git reflog`

**Package manager (build/test/lint):**
- `pnpm install`, `pnpm dev`, `pnpm build`, `pnpm test`, `pnpm lint`
- `pnpm format`, `pnpm typecheck`, `pnpm run`, `pnpm --filter`, `pnpm ls`
- `pnpm exec`, `pnpm -r`, `pnpm -w`, `pnpm why`
- `npm install`, `npm test`, `npm run`, `yarn install`, `yarn test`, `yarn run`
- `bun install`, `bun test`, `bun run`

**Known safe npx tools:**
- `npx prisma generate`, `npx prisma db push`, `npx prisma migrate`
- `npx prisma studio`, `npx prisma format`, `npx prisma validate`
- `npx husky`, `npx lint-staged`, `npx turbo`, `npx nest`, `npx tsc`
- `npx jest`, `npx playwright`, `npx vitest`, `npx eslint`, `npx prettier`

**Database (non-destructive):**
- `pnpm db:generate`, `pnpm db:push`, `pnpm db:migrate`, `pnpm db:studio`, `pnpm db:seed`

**Docker (read + dev lifecycle):**
- `docker compose up`, `docker compose down`, `docker compose ps`, `docker compose logs`
- `docker ps`, `docker logs`

**Shell (read-only + safe utilities):**
- `ls`, `tree`, `du`, `wc`, `which`, `diff`, `mkdir`, `cp`
- `ps aux`, `lsof -i`, `head`, `tail`, `find`, `grep`, `echo`
- `node --version`, `node -v`

**GitHub CLI (read-only):**
- `gh pr view`, `gh pr list`, `gh pr status`, `gh pr diff`, `gh pr checks`
- `gh issue list`, `gh issue view`, `gh issue status`
- `gh repo view`, `gh api`, `gh auth status`, `gh run list`, `gh run view`

**MCP tools (read + edit — same risk as Edit/Write):**
- All `serena` read tools: `read_file`, `find_file`, `list_dir`, `search_for_pattern`, `get_symbols_overview`, `find_symbol`, `find_referencing_symbols`
- All `serena` write tools: `create_text_file`, `replace_content`, `replace_symbol_body`, `insert_after_symbol`, `insert_before_symbol`, `rename_symbol`
- All `serena` memory tools: `write_memory`, `read_memory`, `list_memories`, `delete_memory`, `edit_memory`, `rename_memory`
- `context7` tools: `resolve-library-id`, `query-docs`

### AUTO-DENY (destructive, irreversible, or escape hatches)

These patterns should be added to the `deny` list:

**File destruction:**
- `rm -rf`, `rm -r`, `shred`

**Git destruction:**
- `git push --force`, `git push -f`, `git push origin main --force`
- `git reset --hard`, `git clean -f`, `git clean -fd`
- `git checkout .`, `git checkout -- .`, `git restore .`
- `git branch -D`

**Escape hatches (bypass other rules):**
- `node -e`, `node --eval`
- `python -c`, `python3 -c`
- `bash -c`, `sh -c`
- `eval`

**Database destruction:**
- `pnpm db:reset`, `npx prisma db reset`, `npx prisma migrate reset`
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
- Any broad `npx:*` or `node:*` pattern

## Process

### During the session (collect silently)

When you detect a denied or prompted permission:

1. **Classify** the command using the rules above
2. **Note it briefly** in your response (one line): "Noted: `<command>` → allow/deny/ask-first"
3. **Continue with the task** — do NOT stop to update settings

### At session end (batch update)

When the user is wrapping up, or when `/update-permissions` is invoked:

1. **Read** the current `.claude/settings.local.json`
2. **Present a summary table** of all collected permissions:

| Command | Pattern | Classification | Reason |
|---------|---------|---------------|--------|
| `pnpm audit` | `Bash(pnpm audit:*)` | Allow | Safe, read-only |
| `node -e "..."` | `Bash(node -e:*)` | Deny | Escape hatch |
| `git push` | — | Ask first | Affects remote |

3. **Ask for confirmation**: "Want me to add these to your settings?"
4. **On approval**: Read the current settings, add the new patterns, write the updated file
5. **Skip duplicates**: Don't add patterns already in the list
6. **Report what changed**: Show the user exactly what was added

## Pattern Format

Use the Claude Code permission pattern format:

- Bash commands: `Bash(<command prefix>:*)`
- MCP tools: `mcp__<server>__<tool>`
- Built-in tools: `Read`, `Edit`, `Write`, `Glob`, `Grep`, `Task`, etc.

Examples:
- `Bash(git status:*)` — matches `git status`, `git status --short`, etc.
- `Bash(pnpm test:*)` — matches `pnpm test`, `pnpm test:unit`, etc.
- `Bash(npx prisma generate:*)` — matches only prisma generate, not other npx commands

## Important

- **Never add broad wildcards** like `Bash(npx:*)`, `Bash(node:*)`, or `Bash(bash:*)` — these are escape hatches
- **Prefer specific prefixes** — `Bash(npx prisma generate:*)` over `Bash(npx prisma:*)`
- **Keep deny rules tight** — include both long and short flag variants (e.g., both `--force` and `-f`)
- **Check for duplicates** before adding — don't add patterns already in the list
- **Preserve existing entries** — never remove entries the user already has
