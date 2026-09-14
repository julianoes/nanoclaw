# NanoClaw Migration Guide

Generated: 2026-07-22T10:05:59Z (updated 2026-09-15)
Base: `641963c1` (upstream at last upgrade)
HEAD at generation: `c74050f7` (post-upgrade)
Upstream at generation: `3f9ed607` (nanocoai/nanoclaw, branch `main`)

> **Remote note:** `origin` and `upstream` both point at `nanocoai/nanoclaw`
> (`upstream` added 2026-09-15); `julianoes` is the user's own fork.

> **Never touch:** `data/`, `groups/`, `.env` are gitignored local state holding the
> live DB, the `main` + `birdconfig` agents, and channel wiring. Code only.

Scope: Tier 2. 12 changed files, ~850 insertions vs base (mostly the Discord adapter + lockfile).
No skill-branch merges in local history.

---

## Applied Skills

- `add-discord` — **not** a `skill/*` branch merge. Upstream trunk ships no channel
  adapters; Discord is installed by running the `/add-discord` skill, which copies
  the adapter in from the `channels` branch and installs a pinned dependency.

  Reapply by running `/add-discord` on the clean upstream tree. This recreates:
  - `src/channels/discord.ts`
  - the `import './discord.js';` line in `src/channels/index.ts`
  - the `@chat-adapter/discord` dependency in `package.json` + lockfile

  Do **not** hand-copy these files from the old tree — let the skill install the
  current version.

No custom (user-authored) `.claude/skills/` directories. All 47 local skills match upstream.

## Skill Interactions

None. Only one channel skill is installed, and no provider skills.

---

## Modifications to Applied Skills

### add-discord: patch bare URLs to suppress Discord link embeds

**Intent:** When the agent sends a message containing a bare URL, the Discord
adapter renders it as `[url](url)`. This makes Discord expand a link preview/embed,
which is noisy in the dev channels. Rendering it as `<url>` suppresses the embed.

**How to apply:** Upstream `/add-discord` still pins **`4.29.0`** (as of `3f9ed607`),
matching the existing patch. After re-running `/add-discord`, copy the patch and its
`pnpm-workspace.yaml` entry from the main tree, then `pnpm install`:
```bash
mkdir -p "$WORKTREE/patches"
cp "$PROJECT_ROOT/patches/@chat-adapter__discord@4.29.0.patch" "$WORKTREE/patches/"
```
In `$WORKTREE/pnpm-workspace.yaml` add (merge into any existing `patchedDependencies`):
```yaml
patchedDependencies:
  '@chat-adapter/discord@4.29.0': patches/@chat-adapter__discord@4.29.0.patch
```

**If the pin has moved:** check whether the new version already emits `<url>` when
link text equals the URL (`grep -n "isLinkNode" node_modules/@chat-adapter/discord/dist/index.js`);
if not, regenerate with `pnpm patch @chat-adapter/discord@<ver>`, insert the early
return below in the `isLinkNode` branch of `dist/index.js`, then `pnpm patch-commit <dir>`:
```javascript
if (isLinkNode(node)) {
  const linkText = getNodeChildren(node).map((child) => this.nodeToDiscordMarkdown(child)).join("");
  if (linkText === node.url) {
    return `<${node.url}>`;
  }
  return `[${linkText}](${node.url})`;
}
```

**Files:** `patches/@chat-adapter__discord@<version>.patch`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`

**Reference — the original 4.26.0 patch (for intent only, do not apply verbatim):**
```diff
@@ -317,6 +317,9 @@
     }
     if (isLinkNode(node)) {
       const linkText = getNodeChildren(node).map((child) => this.nodeToDiscordMarkdown(child)).join("");
+      if (linkText === node.url) {
+        return `<${node.url}>`;
+      }
       return `[${linkText}](${node.url})`;
     }
```

**Supply-chain note:** do not add `minimumReleaseAgeExclude` or
`onlyBuiltDependencies` entries to work around install failures — both require
explicit human approval (see CLAUDE.md).

---

## Customizations

### 1. Container git identity (BanSiesta) + OneCLI CA for git

**Intent:** The agent commits and pushes to GitHub as "BanSiesta", which is the
identity invited as a collaborator on the user's repos (e.g. `julianoes/birdconfig`).
Without this, container-side `git commit` fails or uses a wrong/unset identity.

Second line (added 2026-08-04, committed `8615ae0e`): `@onecli-sh/sdk`'s
`applyContainerConfig` never sets `GIT_SSL_CAINFO`, so git can't verify the OneCLI
gateway's intercepted TLS ("server certificate verification failed. CAfile: none").
Pointing git at the combined CA bundle the gateway mounts fixes it. Before applying,
confirm the gateway still mounts it at that path:
`grep -rn onecli-combined-ca.pem src/gateway-providers/` (true as of `3f9ed607`).

**Files:** `container/Dockerfile`

**How to apply:** Insert this block **after** the `RUN mkdir -p /workspace/group ...`
/ `chown` block and **immediately before** the `USER node` line:

```dockerfile
RUN git config --system user.name "BanSiesta" && \
    git config --system user.email "bansiesta@gmail.com" && \
    git config --system http.sslCAInfo /tmp/onecli-combined-ca.pem
```

Verify after `./container/build.sh`:
`docker run --rm --entrypoint git nanoclaw-agent:latest config --system --get http.sslCAInfo`

It must come before `USER node` because `git config --system` writes to
`/etc/gitconfig` and requires root.

### 2. Custom container skills — `capabilities` and `status`

**Intent:** Two read-only introspection skills the user wrote, loaded into every
agent container. `/capabilities` reports installed skills, available tools and
system info; `/status` is a quick health check (session context, workspace mounts,
tool availability, task snapshot). Neither exists upstream.

**Files:** `container/skills/capabilities/SKILL.md`, `container/skills/status/SKILL.md`
(88 lines each)

**How to apply:** Copy verbatim from the pre-migration state — do not rewrite:
```bash
cp -r "$PROJECT_ROOT/container/skills/capabilities" "$PROJECT_ROOT/container/skills/status" "$WORKTREE/container/skills/"
```

Upstream `container/skills/` contains `agent-browser`, `frontend-engineer`,
`onecli-gateway`, `self-customize`, `welcome` — no name collision.

### 3. Local migration notes doc

**Intent:** Historical record of the user's v1→v2 migration. User chose to keep it.

**Files:** `docs/migration-2026-05-20.md`

**How to apply:**
```bash
cp "$PROJECT_ROOT/docs/migration-2026-05-20.md" "$WORKTREE/docs/"
```

---

## Explicitly dropped (do NOT reapply)

### Dockerfile base image `node:22-trixie-slim` + `libasound2t64`

The fork changed `FROM node:22-slim` → `node:22-trixie-slim`, which forced
`libasound2` → `libasound2t64` (Debian trixie's 64-bit `time_t` rename). The two
changes are coupled — reapplying one without the other breaks the image build.

**Decision (2026-07-22): dropped.** The user could not recall the reason and chose
to take upstream's `node:22-slim` + `libasound2`.

**If `./container/build.sh` fails after upgrade** with an apt error about
`libasound2` or a missing package, that is the original reason resurfacing.
Reinstate BOTH lines together in `container/Dockerfile`:
```dockerfile
FROM node:22-trixie-slim
# ...and in the apt install list:
        libasound2t64 \
```

### `groups/main/CLAUDE.md` and `groups/global/CLAUDE.md` deletions

The fork deleted these tracked files. Upstream has since untracked them too, so
this is a **no-op** on a clean upstream checkout. Nothing to do. The live agent
content lives in the gitignored `groups/` directory and is untouched by the upgrade.

---

## Post-upgrade validation

1. `pnpm install` — must succeed; a failure here usually means the patch version
   is wrong (see the version-drift note above).
2. `pnpm run build` and `pnpm test`.
3. `pnpm exec tsc -p container/agent-runner/tsconfig.json --noEmit` (container typecheck).
4. `./container/build.sh` — required, since `container/Dockerfile` is customized.
5. Confirm Discord still works: the host must connect as `BanSiesta` and both
   `dev-mavsdk` and `dev-birdconfig` must get exactly **one** reply per message.
6. Restart the service: `systemctl --user restart nanoclaw`
   (Linux/systemd, **user** unit — there is deliberately no system-level unit;
   see the note below.)

## Install-specific notes (not code, do not migrate)

- Service is a **user** systemd unit (`systemctl --user`), enabled to start at login.
  A duplicate **system** unit at `/etc/systemd/system/nanoclaw.service` was removed
  on 2026-07-22 because it caused duplicate message delivery. Do not recreate it.
- `loginctl` linger is **off** deliberately: the home directory is encrypted and
  not mounted until login, so a lingering/boot-time start cannot work and broke
  the user's audio stack.
