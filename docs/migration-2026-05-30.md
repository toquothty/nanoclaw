# NanoClaw v1 → v2 migration — 2026-05-30

Completed via `/migrate-from-v1` after `bash migrate-v2.sh`. v1 install: `/Users/tq-mac-mini/git/nanoclaw` (v1.2.14, a customized fork of qwibitai/nanoclaw).

## Outcome

v2 routing real messages on **WhatsApp** and **Discord**. Agent identity **Marvin** restored across all three agent groups. Owner + `strict` access policy seeded. Daily Obsidian task restored.

## Blockers found & fixed (not handled by the deterministic script)

1. **Channel code never installed.** `migrate-v2.sh` and the `/add-*` skills fetch the `channels` branch from `origin`, but this install's `origin` is the user's fork (`toquothty/nanoclaw`); the branch only exists on `upstream` (`qwibitai/nanoclaw`). Fixed by copying `discord.ts` / `whatsapp.ts` / `github.ts` + `setup/{whatsapp-auth,groups}.ts` from `upstream/channels`, wiring imports, installing pinned deps. Also aligned `chat` core to `4.27.0` (adapters needed it; trunk had 4.26.0).

2. **Claude "Invalid API key".** The credential is a **subscription OAuth token** (`sk-ant-oat…`), but the migration stored it as a OneCLI `anthropic`-type secret, which injects `x-api-key`. An OAuth token must be sent as `Authorization: Bearer`. Fix:
   - Re-stored as a OneCLI `generic` secret: `--host-pattern api.anthropic.com --header-name Authorization --value-format "Bearer {value}"`.
   - Set `ANTHROPIC_BASE_URL=https://api.anthropic.com` in `.env` and loaded the claude provider (`src/providers/index.ts` → `import './claude.js'`) so the container gets `ANTHROPIC_AUTH_TOKEN=placeholder` and the SDK emits `Authorization: Bearer` for the gateway to overwrite.
   - **Code fix** in `src/container-runner.ts`: OneCLI also injects `ANTHROPIC_API_KEY=placeholder`, so the SDK additionally sent `x-api-key: placeholder` → Anthropic rejects on x-api-key first (even with a valid Bearer header). In Bearer mode (provider sets `ANTHROPIC_AUTH_TOKEN`), we now blank `ANTHROPIC_API_KEY` *after* the OneCLI env so only the Bearer header goes out.

3. **WhatsApp keystore wiped on every restart (upstream adapter bug).** In `src/channels/whatsapp.ts`, the connection-close handler treated *any* non-reconnect close (including a clean SIGTERM shutdown, `reason=undefined`) as a logout and deleted `store/auth/`. **Code fix:** only clear auth on a genuine `DisconnectReason.loggedOut` (401); a clean-shutdown close now keeps the keystore. Re-copied v1's Baileys keystore (the device link was still valid).

4. **Security:** bumped `@whiskeysockets/baileys` `7.0.0-rc.9` → `rc13` (GHSA-qvv5-jq5g-4cgg message-spoofing advisory; matches what v1 already ran).

## Identity / access

- **Marvin** personality (dry Hitchhiker's-Guide wit; family assistant for Thomas & Rachel) was lost by the migration — it lived in v1 `groups/global/CLAUDE.md` (included via `@./.claude-global.md`), which v2's startup cutover removed. The migration only kept a stale `groups/main` "Andy" file. Recovered from v1 `groups/global/CLAUDE.md`, stripped of v1 boilerplate (Managing Groups / Sender Allowlist / IPC / Container Mounts / Task Scripts / Message Formatting — all v2-fragment-handled), and written to `CLAUDE.local.md` in all three agent groups. Orphan `groups/main` (Andy) removed.
- Owner role: `whatsapp:17575613438` (Thomas) **and** `discord:195346768752410626` (tqizzle).
- Member: `discord:478908094626267166` (Rachel / WBmare) → Discord #marvin.
- `unknown_sender_policy = strict` on all three messaging groups.
- Obsidian vault mount (`~/git/obsidian` → `/workspace/extra/obsidian`) present on all three groups (added to discord_daily-update).

## Scheduled tasks

- 4 completed one-shots: correctly skipped.
- 1 active recurring daily Obsidian briefing (`0 9 * * *` → Discord #daily-update): the migration skipped it (Discord resolver). Recreated directly in the daily-update agent-group session (id `7358c4ac-…`), preserving the original Marvin-voiced prompt.

## Fork customizations (per user decision)

- Marvin personality ✓ and task-capture instructions ✓ recovered.
- v1 source fixes (resilient startup, Docker-wait) and Claude GitHub Actions workflows: **skipped** — v2 handles startup resilience natively and the CI isn't needed.

## Uncommitted local code changes (decide whether to commit to the fork)

- `src/channels/{discord,whatsapp,github}.ts`, `setup/{groups,whatsapp-auth}.ts`, `src/channels/index.ts` — channel installs.
- `src/container-runner.ts` — ANTHROPIC_API_KEY blank-in-Bearer-mode fix (general; worth upstreaming).
- `src/channels/whatsapp.ts` — auth-clear-only-on-loggedOut fix (general upstream bug; worth upstreaming).
- `src/providers/index.ts` — claude provider import (needed for OAuth Bearer auth).
- `package.json` / `pnpm-lock.yaml` — channel deps, `chat@4.27.0`, `baileys@7.0.0-rc13`.
- Deleted `groups/main` (orphan "Andy").
