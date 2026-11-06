# Operational Lessons

## 2026-09-08: Stop Exploring, Start Executing

**Problem:** Spent 31+ minutes on a simple config change because I kept running exploratory commands instead of applying fixes.

**Root causes:**
1. Used `timeout` instead of `timeoutSeconds` — invalid parameter, immediate error
2. Ran `openclaw logs --follow` — streaming command that never exits, guaranteed timeout
3. Tried multiple non-existent CLI commands (`delivery list`, `queue list`, `messages list`) — each waited 10s before failing
4. Attempted SQLite queries without checking available tools — two more failures

**Fix:**
- When you have enough info to act, **act immediately**. Don't gather "one more piece" of data.
- Before running any command, verify the parameter name (`timeoutSeconds`, not `timeout`).
- Never run streaming commands (`--follow`, `tail -f`) without `background=true`.
- If a command fails twice, stop. Don't try a third variation.
- Show diffs first, then apply. Don't explore and apply simultaneously.

**Rule:** If you can't complete a task in 3 commands, escalate to the user with what you have.

---

## 2026-11-06: Agent-DB Integrity Gate

- Symptoms (`couldn't finish this request`, `disk I/O error`, `domain is closed`) trace to the agent-DB integrity gate, not the model.
- **Fix was a version upgrade (2026.11.1).** Full runbook + dead ends: `knowledge/openclaw-agent-db-integrity-gate.md`.
- Never propose trimming the store: 280 MB is small and the check is not the bottleneck.
- Always read the installed version live (`node -p "require('$(npm root -g)/openclaw/package.json').version"`). I once reported a stale version and wasted a round trip.

## 2026-03-27: Config Changes

- ALWAYS show DT a diff before applying. Never blind-apply.
- `.bashrc` doesn't propagate to systemd. Secrets go in `.env`.

## 2026-04-15: Destructive Changes

- Never make permanent/destructive changes without explicit approval.
- Stay inside `/home/grimani/clawd` unless DT asks.

## 2026-07-27: Chromium PDF Caching

- Chromium print-to-pdf caches HTML. `rm` target PDF before re-rendering.
- Verify freshness by page count, not text grep.

## 2026-03-25: Git

- Always `git fetch origin && git rebase origin/main` before pushing.
- Verify diff against remote tip.
