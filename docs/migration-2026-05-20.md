# v1 → v2 migration — 2026-05-20

Source: `migrate-v2.sh` + `/migrate-from-v1` skill.

## Source install

- v1 path: `/home/julianoes/src/my/nanoclaw`
- v1 version: 1.2.26
- Channels: discord (only)

## Handoff snapshot

```
overall_status: partial
onecli_healthy: false        (resolved during /migrate-from-v1)
service_switched: false      (resolved during /migrate-from-v1)
```

### Step results from migrate-v2.sh

| Step | Status |
|------|--------|
| 1a-env | success |
| 1b-db | success |
| 1c-groups | success |
| 1d-sessions | success |
| 1e-tasks | success |
| 2b-channel-auth | success |
| 2c-install-discord | success |
| 3b-onecli | **failed** — `denied: denied` from ghcr.io on `ghcr.io/onecli/onecli:1.23.0`. Resolved by `docker login ghcr.io` with the existing `GITHUB_TOKEN` (any authenticated GitHub user can pull). |
| 3e-build | success |

## Phase 0: service switch

- v1 service `nanoclaw.service` was active and intercepting Discord. Stopped + disabled.
- v2 service `nanoclaw-v2-08f09bdf.service` enabled + started.
- Discord adapter (`@chat-adapter/discord`) required two values v1 didn't:
  - `DISCORD_PUBLIC_KEY` — added to `.env`.
  - `DISCORD_APPLICATION_ID` — added to `.env`.
- Smoke test passed (BanSiesta replied on Clawmock channel).

## Phase 1: owner + access

- Owner role granted to `discord:786767812248600597` (JulianOes), global scope.
- Access policy: Clawmock channel kept at `public`. New stranger DMs default to `request_approval` per `src/router.ts:192` — unknown senders trigger approval to the owner before getting through.

## Phase 2: CLAUDE.local.md

- `groups/main/CLAUDE.local.md` was a verbatim copy of v1 template (329 lines, no customizations).
- Replaced with a minimal identity-only version (3 lines: `# BanSiesta` + one-paragraph description). Composed fragments in `groups/main/CLAUDE.md` cover the rest.

## Phase 3: container.json

- `additionalMounts` empty in both v1 and v2. No-op.

## Phase 4: fork customizations

v1 fork was ~92 commits ahead of `upstream/main` — almost all upstream-branch merges. Real user-authored customizations: 3 commits.

| v1 commit | Action |
|-----------|--------|
| `31f4a0d` git identity in container | **Ported** — added to `container/Dockerfile`: `git config --system user.name "BanSiesta" / user.email "bansiesta@gmail.com"`. |
| `6ede57e` Trixie base + `CLAUDE_MODEL` env | **Partially ported.** Base image switched `node:22-slim` → `node:22-trixie-slim` and `libasound2` → `libasound2t64`. `CLAUDE_MODEL` env passthrough not ported — v2 uses `ncl groups config update --model …` instead. |
| `3cf33bb` build tools + `GITHUB_TOKEN` passthrough | **Build tools ported** via `ncl groups config add-package --apt` on the `main` group: `gh`, `cmake`, `ninja-build`, `g++`, `clang-format`, `libssl-dev`, `python3`, `python3-pip`. `GITHUB_TOKEN` passthrough **not** ported — replaced by OneCLI vault entry. |

## Credentials migrated to OneCLI vault

| Secret | Type | Host pattern |
|--------|------|--------------|
| Anthropic | anthropic | `api.anthropic.com` |
| GitHub | generic | `api.github.com` (header: `Authorization: token {value}`) |

`CLAUDE_CODE_OAUTH_TOKEN` and `GITHUB_TOKEN` removed from `.env`. Vault is sole source of truth.

## Verify

```
SERVICE: running
CONTAINER_RUNTIME: docker
CREDENTIALS: configured
CONFIGURED_CHANNELS: discord
CHANNEL_AUTH: {"discord":"configured"}
REGISTERED_GROUPS: 1
MOUNT_ALLOWLIST: configured
STATUS: success
```
