# Git Workflow — homelab-infra

**Applies to:** docker-prod-01, auth-prod-01, immich-prod-01, observer-prod-01 (`/opt/stacks`)  
**Repo:** `homelab-infra` (private, GitHub)  
**Last Updated:** 2026-11-16

---

## Overview

`/opt/stacks` on each participating host is a clone of the private `homelab-infra` GitHub repository.

Docker Compose configuration files are maintained in this repository. Changes are committed and pushed to GitHub to preserve version history and keep the intended infrastructure configuration consistent across hosts.

`/opt/appdata` is not in Git — it contains persistent container state and application configuration.

Secrets are excluded from version control, including `.env` files containing API tokens, passwords, and other sensitive information.

### Repository Locations

| Host | Path | What lives there | Status |
|---|---|---|---|
| docker-prod-01 | `/opt/stacks` | traefik, arr, books, torrent, cloudflared, adguardhome-sync, and other application stacks | Active |
| auth-prod-01 | `/opt/stacks` | authentik, beszel-agent, local Komodo Periphery configuration | Active |
| immich-prod-01 | `/opt/stacks` | immich, beszel-agent, local Komodo Periphery configuration | Active |
| observer-prod-01 | `/opt/stacks` | proxmox-mcp, unraid-mcp, komodo-mcp, unifi-mcp; `observer/tunnel-client/` source templates | Active |

**GitHub is the source of truth for committed Compose configuration — not any individual host.**

However, GitHub does not contain the complete operational state of each service. Runtime data, secrets, host-level configuration, and installed system services must be managed and recovered separately.

### Host-Specific Management

The repository is shared, but each host manages its own deployed workloads.

- **docker-prod-01:** Docker Compose and Komodo (Dockman decommissioned November 2026).
- **auth-prod-01:** Docker Compose and Komodo.
- **immich-prod-01:** Docker Compose and Komodo.
- **observer-prod-01:** Docker Compose through SSH and CLI; Komodo integration where configured.

The observer is maintained through its documented SSH/CLI procedures; the former Dockman deployment has been removed from docker-prod-01.

Its four OpenAI Secure MCP Tunnel clients are installed as **native systemd services** rather than Docker Compose stacks. Their Git-tracked copies and sanitized examples are in `observer/tunnel-client/`, separate from active files under `/etc`.

Pulling the repository on another host does not install or start those services.

See `services/infra/ai-observer.md`.

---

## The Golden Rule

**Always synchronize your local repository before making changes, and inspect your working tree before pulling or committing.**

Because `homelab-infra` is cloned on multiple hosts, changes committed from one host must be incorporated into the others before additional work is committed.

A safe workflow is:

1. Check the working tree.
2. Fetch and inspect remote changes.
3. Reconcile local changes, untracked files, and incoming deletions.
4. Fast-forward when the local and remote histories permit it.
5. Make the intended configuration changes.
6. Review and stage only the appropriate files.
7. Commit locally.
8. Push to GitHub.
9. Synchronize other hosts before making changes there.

Use `git pull --ff-only` for routine synchronization after confirming that the working tree and incoming changes are safe.

**Important:** If local files are already modified, do not blindly pull, reset, or overwrite them.

Git normally prevents a non-fast-forward push from overwriting newer remote commits.

The principal risks are conflicts, stale deployments, accidental replacement of configuration during manual conflict resolution, and deletion of locally important files during a valid fast-forward.

Never use force-push as a routine way to resolve synchronization problems.

---

## Standard Workflow

Use this process when making a Compose configuration change on a participating host.

### Step 1 — Inspect and Synchronize

```bash
cd /opt/stacks

# Check the working tree
git status -sb

# Include individual untracked files
git status --short --untracked-files=all

# Fetch the current remote state
git fetch origin

# Check branch divergence
git status -sb

git log --oneline --left-right origin/main...HEAD

# Review incoming file changes
git diff --name-status HEAD..origin/main
```

Before proceeding, identify:

- Tracked files with local modifications.
- Staged but uncommitted changes.
- Untracked files or directories.
- Incoming files that collide with untracked files.
- Incoming deletions that affect persistent runtime state.
- Any locally committed changes that have not been pushed.

When the working tree is safe and the branch can fast-forward:

```bash
git merge --ff-only origin/main
```

Alternatively:

```bash
git pull --ff-only
```

`git pull --ff-only` combines fetching and fast-forward synchronization.

**Do not use either command blindly when incoming changes might remove application runtime data or overwrite host-specific configuration.**

On observer-prod-01, authenticated Git operations use the credential-helper-disabled procedure described later in this document.

### Step 2 — Make Your Changes

Edit the appropriate Compose file or other version-controlled configuration.

For example:

```bash
nano /opt/stacks/<stack-name>/compose.yaml
```

Do not place credentials directly in the Compose file.

Use a Git-excluded `.env` file for sensitive environment variables when appropriate.

### Step 3 — Review Changes

```bash
cd /opt/stacks

# Show modified and untracked files
git status --short --untracked-files=all

# Review changes to a specific file
git diff -- <stack-name>/compose.yaml
```

Ensure that only intended configuration changes are present.

### Step 4 — Stage Changes

Prefer explicitly staging the files you intend to commit:

```bash
git add -- <stack-name>/compose.yaml
```

Review the staged changes:

```bash
git diff --cached

git diff --cached --check

git diff --cached --stat
```

Verify that no credentials, sensitive values, generated runtime files, or unrelated changes have been staged.

### Step 5 — Commit

```bash
git commit -m "brief description of what changed"
```

### Step 6 — Push

```bash
git push origin main
```

If the push is rejected because the remote repository has newer commits, inspect the divergence and reconcile the branches.

Do not force-push to bypass the rejection.

### Step 7 — Verify

```bash
git status -sb

git log --oneline -5
```

The expected result after a successful push is a synchronized branch with the intended commit present.

Host-specific untracked or ignored files may still exist.

**Important:** A successful Git push confirms that configuration was committed. It does not prove that a deployed container has been recreated or that the service is healthy.

Application changes require separate deployment and service verification.

---

## When to Use `git add .` vs `git add <path>`

| Situation | Recommended command |
|---|---|
| One specific Compose file changed | `git add -- <stack-name>/compose.yaml` |
| Multiple known files changed | `git add -- <path1> <path2>` |
| All changes were reviewed and should be committed | `git add .` |
| Unsure what changed | Run `git status` and `git diff` first |
| A secret or credential file appears | Do not stage it; investigate why it is not excluded |
| Generated application runtime data appears | Review and add an appropriate ignore rule |
| An untracked backup appears | Preserve or remove it deliberately; do not stage it automatically |

`git add .` stages changes throughout the current directory tree, including new files that are not ignored.

For a multi-host infrastructure repository, explicitly staging files is usually safer.

**Always review the staged changes before committing.**

---

## Daily Workflow

After creating or editing a Compose configuration:

```bash
cd /opt/stacks

# Check local state
git status -sb
git status --short --untracked-files=all

# Inspect incoming changes
git fetch origin
git log --oneline --left-right origin/main...HEAD
git diff --name-status HEAD..origin/main

# Synchronize only after confirming it is safe
git merge --ff-only origin/main

# Make the intended configuration changes

# Review modifications
git status
git diff

# Stage only the intended files
git add -- <path-to-changed-file>

# Inspect staged changes
git diff --cached
git diff --cached --check

# Commit and push
git commit -m "brief description of what changed"
git push origin main

# Verify
git status -sb
```

On `observer-prod-01`, use the credential-handling procedure in the dedicated section below for authenticated fetch, pull, and push operations.

### Good Commit Message Examples

- `add traefik compose`
- `add authentik compose`
- `add arr stack compose`
- `add immich compose`
- `add proxmox-mcp compose`
- `update sonarr-tv restart policy`
- `add gluetun killswitch to torrent stack`
- `fix immich compose - use official /data mount path`
- `update proxmox-mcp container limits`
- `Update Profilarr v2 Compose configuration`
- `Stop tracking host-specific Beszel agent state`
- `Update Authentik image fallback to 2026.8.5`
- `Switch Traefik to latest stable image`
- `Harden Traefik HTTP header alias handling`

Commit messages should describe the actual configuration change, not merely state that files were updated.

---

## Adding a New Stack

Before creating a new stack, synchronize the repository:

```bash
cd /opt/stacks

git status -sb
git fetch origin
git diff --name-status HEAD..origin/main

# After confirming synchronization is safe
git merge --ff-only origin/main
```

Create the stack directory and Compose file:

```bash
mkdir -p /opt/stacks/<stack-name>

nano /opt/stacks/<stack-name>/compose.yaml
```

Create an appdata directory only when the application requires persistent host storage:

```bash
mkdir -p /opt/appdata/<app-name>
```

Create a separate `.env` file if sensitive configuration is required:

```bash
nano /opt/stacks/<stack-name>/.env
```

Confirm that `.env` is excluded from Git:

```bash
cd /opt/stacks

git check-ignore -v <stack-name>/.env
```

Review and commit only the intended configuration:

```bash
git status

git add -- <stack-name>/compose.yaml

git diff --cached

git commit -m "add <stack-name> compose"

git push origin main
```

### Host-Specific File Permissions

Docker hosts do not necessarily share the same file ownership requirements.

On `docker-prod-01`, the previously established `gio:service` ownership and group-writable Compose settings supported Dockman editing, but **Dockman has been retired**. Use permissions appropriate to the current repository operator and service; do not broaden access solely for a removed management UI.

See `runbooks/new-stack-checklist.md` for the existing baseline and per-stack cautions.

Do not apply docker-prod-01's `gio:service` ownership requirements or UID/GID 2000 assumptions automatically to `observer-prod-01`.

The observer is managed through the CLI, and its Proxmox MCP `.env` is restricted to the `gio` account.

#### Permission Issue Encountered on auth-prod-01 — 2026-11-14

During Git synchronization, Authentik's Compose file and parent directory were discovered to be owned by `root:root`.

The `gio` account could read the file but could not replace it during a Git stash operation.

Git created the stash successfully but failed to restore the file because of filesystem permissions.

The affected paths were:

- `/opt/stacks/authentik/`
- `/opt/stacks/authentik/compose.yaml`

The original Compose configuration was independently backed up before the issue was corrected.

Ownership was changed only for the affected directory and Compose file:

```bash
sudo chown gio:gio \
  /opt/stacks/authentik \
  /opt/stacks/authentik/compose.yaml
```

The Git staging area was then inspected separately from the working tree.

After verifying the staged changes and backups, the tracked file was restored to its committed state:

```bash
git restore --source=HEAD \
  --staged --worktree \
  -- authentik/compose.yaml
```

The repository subsequently fast-forwarded successfully.

**Important lessons:**

- A successful `git stash push` message does not guarantee that the working tree was fully restored.
- Always inspect `git status --short` after stashing.
- The first and second character positions in Git's short status represent staged and working-tree changes respectively.
- Changing directory ownership can affect application security and maintenance; restrict corrections to the verified paths.
- Do not recursively change ownership of an entire application directory merely to resolve one Git permissions problem.
- Do not use `sudo git` as an automatic workaround.
- Preserve `.env` file ownership and permissions independently from Compose-file ownership.

---

## Checking What Changed

```bash
cd /opt/stacks

# See files that are new, modified, or deleted
git status

# Compact status output
git status --short

# Include individual untracked files
git status --short --untracked-files=all

# See recent commits
git log --oneline -10

# Inspect changes to a specific file
git diff -- <file>

# Inspect all staged changes
git diff --cached

# Inspect unstaged changes
git diff

# Show files included in the next commit
git diff --cached --name-only

# Identify incoming changes
git diff --name-status HEAD..origin/main

# Confirm whether a file is tracked
git ls-files -- <path>

# Confirm whether a file is ignored
git check-ignore -v <path>
```

Use `git status` and `git diff` before staging.

Use `git diff --cached` after staging.

These are separate checks and both are useful for avoiding accidental changes.

### Inspecting Remote Changes

Fetching does not alter the working files:

```bash
git fetch origin
```

Inspect incoming commits:

```bash
git log --oneline HEAD..origin/main
```

Inspect local commits not on the remote:

```bash
git log --oneline origin/main..HEAD
```

Inspect incoming filenames:

```bash
git diff --name-status HEAD..origin/main
```

Inspect a particular file's remote contents:

```bash
git show origin/main:<path>
```

Do not print credential-bearing files or secrets to the terminal unnecessarily.

### Working with Untracked Files

Git may refuse to check out a remotely tracked file when an untracked local file already exists at the same path.

Before resolving a collision:

1. Identify the local file.
2. Compare it with the incoming tracked file.
3. Determine whether it contains host-specific data.
4. Preserve unique local content.
5. Move or reconcile only the confirmed collision.
6. Synchronize Git.
7. Verify the resulting files and application behavior.

Avoid broad deletion commands or indiscriminate stashing of untracked directories.

### Git Stashes

A stash may be useful for preserving specific local tracked modifications during synchronization.

Example:

```bash
git stash push \
  -m "pre-sync local Compose changes" \
  -- <stack-name>/compose.yaml
```

Verify:

```bash
git status --short
git stash list
```

If the stash operation reports a permissions error, inspect both the staged and unstaged state before continuing.

A stash is not a substitute for a verified backup of critical application configuration.

Do not automatically apply obsolete stashes after synchronizing to a newer configuration.

---

## Multi-Host Gotcha — Stale Files

Because the repository exists on multiple hosts, it is possible to work from an outdated local checkout.

For example:

1. You update `traefik/compose.yaml` on docker-prod-01 and push it to GitHub.
2. You SSH into auth-prod-01, whose repository has not been synchronized.
3. You modify another file using the older local checkout.
4. You attempt to commit and push the change.
5. Git may reject the push because the remote branch has advanced.

This is expected behavior and protects the remote branch from accidental non-fast-forward updates.

The danger comes from attempting to resolve the problem by force-pushing, discarding remote changes, or incorrectly resolving merge conflicts.

### How to Avoid It

- Synchronize before editing.
- Keep the tracked working tree clean between tasks.
- Use fast-forward-only synchronization for routine updates.
- Commit related changes together.
- Do not force-push routine infrastructure changes.
- Review conflicts carefully if divergent edits occur.
- Inspect incoming deletions and untracked file collisions.
- Preserve persistent runtime state before potentially destructive Git operations.

### Important Deployment Distinction

Pulling newer Compose configuration onto a host does not automatically recreate its running containers.

Git synchronization and Docker deployment are separate activities.

Only deploy a stack when the relevant service change is intended and the appropriate maintenance procedure has been followed.

For example, synchronizing Traefik's Compose file to auth-prod-01 does not upgrade Traefik because Traefik runs on docker-prod-01.

---

## Multi-Host Incident — Beszel Agent Runtime State

**Identified and corrected:** 2026-11-14

### Problem

The shared repository was tracking:

```text
beszel-agent/beszel_agent_data/fingerprint
```

The Beszel agent Compose configuration mounts:

```yaml
- ./beszel_agent_data:/var/lib/beszel-agent
```

The fingerprint is generated runtime state associated with the agent.

It should remain local to each host rather than being distributed through the shared Git repository.

The tracked fingerprint also created a risk during synchronization: Git could delete or overwrite a host's local runtime data.

### Repository Correction

The shared `.gitignore` was updated to include:

```gitignore
# Beszel agent runtime state (host-specific)
/beszel-agent/beszel_agent_data/
```

The fingerprint was removed from Git tracking using:

```bash
git rm --cached -- \
  beszel-agent/beszel_agent_data/fingerprint
```

The `--cached` option removed the file from the Git index without deleting it locally on the host executing that command.

The correction was committed and pushed:

**Commit:** `10848f0` — `Stop tracking host-specific Beszel agent state`

**Important:** Removing a tracked file from Git does not remove its contents from existing Git history.

If a tracked runtime file contained credentials or other sensitive information, additional assessment and possible credential rotation would be required. Removing it from the current repository tree alone would not be sufficient.

### Synchronization — auth-prod-01

The local Beszel fingerprint differed from the previously tracked GitHub copy.

It was backed up outside the repository and verified using matching SHA-256 hashes.

The locally untracked `beszel-agent/compose.yaml` was byte-for-byte identical to the incoming tracked file.

That Compose file was temporarily moved outside the repository to avoid an untracked-file collision.

After resolving the separate Authentik Compose permissions issue, Git fast-forwarded successfully.

Verification confirmed:

- The tracked Beszel Compose file matched its original backup.
- The local fingerprint remained unchanged.
- Git ignored the fingerprint using the new `.gitignore` rule.
- The local Komodo Periphery configuration remained untouched.

### Synchronization — immich-prod-01

Immich's existing Git checkout still tracked the fingerprint.

The original fingerprint was backed up and hash-verified before synchronization.

After the fast-forward, Git removed the tracked fingerprint and its now-empty runtime directory.

The fingerprint was restored from its verified backup after recreating the directory.

However, Docker's running Beszel agent still referenced the original bind-mounted directory.

The host and container directory device/inode values differed, and the container could not see the restored fingerprint.

The agent was recovered by recreating only its Compose service:

```bash
cd /opt/stacks/beszel-agent

sudo docker compose up -d \
  --no-deps \
  --pull never \
  --force-recreate \
  beszel-agent
```

Subsequent verification confirmed:

- Matching host and container directory device/inode values.
- The container could access the original fingerprint.
- The container remained running with no restart loop.
- Beszel hub connectivity was established.
- The Beszel dashboard showed current metrics for immich-prod-01.

No Immich application containers were deliberately restarted during this recovery.

### Synchronization — observer-prod-01

Observer's checkout also tracked the fingerprint, but no Beszel agent container was running there.

The file was backed up and hash-verified before synchronization.

The repository then fast-forwarded, and the original fingerprint was restored.

Verification confirmed:

- The original fingerprint matched its backup.
- The new Git ignore rule applied.
- The working tree was clean.
- No Observer containers needed recreation.

### Important Operational Lessons

1. Generated runtime state should not be tracked by Git.
2. An incoming deletion can remove a live Docker bind-mounted directory.
3. Restoring the directory on the host does not necessarily repair an existing container's stale bind mount.
4. A container reporting `running` does not guarantee that its mounts or application state are valid.
5. Compare mount identities and verify container-visible files when a bind-mounted directory has been replaced.
6. Recreate only the affected container when its bind mount needs to be reestablished.
7. Verify application health through the actual monitoring interface before considering recovery complete.

### Future Beszel Git Synchronization

The fingerprint is now ignored by Git.

Do not manually stage it or remove the ignore rule.

Keep host-specific runtime state out of the shared repository.

On existing hosts, verify:

```bash
cd /opt/stacks

git check-ignore -v \
  beszel-agent/beszel_agent_data/fingerprint
```

If a future Git change adds, removes, or relocates a bind-mounted runtime directory, assess the container impact before synchronizing it.

See `services/monitoring/beszel.md`.

---

## Multi-Host Synchronization — November 14, 2026

All four participating Docker hosts were synchronized to commit `042bbd5` during the reviewed maintenance session.

| Host | Verified Commit | Result |
|---|---|---|
| docker-prod-01 | `042bbd5` | Synchronized |
| auth-prod-01 | `042bbd5` | Synchronized; local configurations preserved |
| immich-prod-01 | `042bbd5` | Synchronized; Beszel bind mount repaired |
| observer-prod-01 | `042bbd5` | Synchronized; local fingerprint preserved |

The synchronization included:

- Profilarr v2 Compose changes.
- AI Observer tunnel-client templates.
- Komodo-related stack definitions.
- Beszel runtime-data Git cleanup.
- Authentik Compose fallback update to `2026.8.5`.

Afterward, further Traefik changes were committed and pushed from docker-prod-01:

| Commit | Change |
|---|---|
| `9348c68` | Switch Traefik to latest stable image |
| `8153e1c` | Harden Traefik HTTP header alias handling |
| `d06dac8` | Restrict encoded backslashes and null characters in Traefik |

**Latest verified repository state at documentation time:**

- docker-prod-01: synchronized with GitHub at `d06dac8`.
- auth-prod-01: last verified at `042bbd5`.
- immich-prod-01: last verified at `042bbd5`.
- observer-prod-01: last verified at `042bbd5`.

**Pending maintenance:** Fast-forward auth-prod-01, immich-prod-01, and observer-prod-01 to the latest GitHub `main` after reviewing their current working trees.

No application containers should be redeployed merely because Git synchronization completes.

### Known Host-Specific Files

During the November 14 maintenance:

- auth-prod-01 had an untracked `komodo-periphery/compose.yaml`.
- immich-prod-01 had an untracked `komodo-periphery/compose.yaml`.
- The directory contents were intentionally preserved during synchronization.
- Their future version-control treatment was not resolved as part of this maintenance.

Do not automatically stage or discard these files without reviewing their contents and intended management model.

---

## Per-Host Git Identity

Each host needs Git author information before creating commits.

The existing `homelab-infra` identity is:

- **Name:** `giohosted`
- **Email:** `delgadogiovanny@gmail.com`

To configure identity for the repository without changing the system-wide Git configuration:

```bash
cd /opt/stacks

git config user.name "giohosted"
git config user.email "delgadogiovanny@gmail.com"
```

Verify:

```bash
git config --get user.name
git config --get user.email
```

The original workflow used global Git identity configuration. That remains valid where already configured.

Repository-local identity is preferable when a host may eventually use different repositories or Git identities.

**Git identity is not GitHub authentication.**

Setting `user.name` and `user.email` does not grant permission to pull or push a private repository.

---

## What Is and Isn't in Git

| Path | In Git | Notes |
|---|---|---|
| `/opt/stacks/*/compose.yaml` | Generally yes | Intended infrastructure configuration; must not contain secrets |
| `/opt/stacks/beszel-agent/compose.yaml` | Yes | Shared Beszel agent deployment definition |
| `/opt/stacks/beszel-agent/beszel_agent_data/` | No | Host-specific runtime state; explicitly gitignored |
| `/opt/stacks/komodo-periphery/compose.yaml` | Not tracked on reviewed auth/Immich hosts | Local files preserved pending review |
| `/opt/stacks/unifi-mcp/{Dockerfile,.dockerignore,http_server.py}` | Yes | Reproducible UniFi local image and Streamable HTTP launcher |
| `/opt/stacks/unifi-mcp/source` | Yes, as submodule | Pinned upstream Git commit; submodule working tree must be initialized on a fresh clone |
| `/opt/stacks/observer/tunnel-client/config/*.yaml.example` | Yes | Four sanitized examples with placeholder tunnel IDs/channels and credential file paths only |
| `/opt/stacks/observer/tunnel-client/systemd/*.service` | Yes | Four copied systemd units; inert inside the Git working tree |
| `/opt/stacks/*/.env` | No | Gitignored; may contain passwords or API tokens |
| `/opt/stacks/*/.env.example` | Optional | Placeholder values only; never real credentials |
| `/opt/appdata/` | No | Persistent application configuration and runtime state |
| `/etc/tunnel-client/config.yaml` | No | Observer host-level tunnel configuration, outside the repository |
| `/etc/tunnel-client/openai-runtime.key` | No | Protected OpenAI credential |
| `/etc/tunnel-client/mcp-authorization` | No | Protected MCP bearer credential |
| `/etc/systemd/system/tunnel-client*.service` | No | Four installed systemd units outside Git; reviewed copies are tracked under `observer/tunnel-client/systemd/` |
| `/opt/tunnel-client/` | No | Installed tunnel-client binaries, outside the repository |

### Secrets

`.env` files are excluded from Git because they may contain sensitive values.

Use `.env.example` files with placeholders when documenting which environment variables are needed.

Do not commit:

- Proxmox API token secrets.
- OpenAI API keys.
- MCP bearer tokens.
- GitHub personal access tokens.
- Application passwords.
- Private SSH keys.
- TLS private keys.
- Application-generated credential files.

A private GitHub repository is still not an appropriate secret store.

### Backup Considerations

The established Docker-host rsync backup procedures cover stacks and appdata on:

- docker-prod-01
- auth-prod-01
- immich-prod-01

These backups are maintained separately from Git.

**Observer exception:** Do not assume that observer-prod-01 has an equivalent rsync backup or that its `/etc/tunnel-client` files are protected by the existing Docker-host jobs.

Observer recovery depends on its separately verified VM backups, configuration recovery plan, and credentials retained securely in Bitwarden.

The observer's complete backup and restoration procedure remains pending verification.

See `services/infra/ai-observer.md` and `runbooks/backup-schedule.md`.

---

## Credentials

### GitHub Authentication

Accessing the private `homelab-infra` repository requires an authorized GitHub identity.

Authentication should use the minimum permissions required for the intended operation.

For HTTPS-based Git operations, supported options include:

- A short-lived or narrowly scoped GitHub personal access token.
- A secure credential manager or credential helper.
- Interactive authentication without persistent token storage.

For new credentials, prefer a fine-grained GitHub token restricted to `homelab-infra` with repository Contents permissions appropriate to the required read or write operations.

A classic token with broad `repo` scope should not be the default for new deployments when a suitable fine-grained token can be used.

### Historical Credential Storage

The original Git workflow recommended:

`git config --global credential.helper store`

That helper persists credentials in plaintext in a local credential file, commonly `~/.git-credentials`.

The current credential-helper configurations on docker-prod-01, auth-prod-01, and immich-prod-01 have not been independently audited as part of this documentation update.

Do not assume those hosts have already migrated to a different authentication method.

Credential storage on existing hosts should be reviewed separately before making changes.

---

## observer-prod-01 — Git Workflow

### Deployment Decision

The observer uses the same private `homelab-infra` repository as the other Docker hosts, but it follows a more restrictive credential-handling approach.

**Repository:** `homelab-infra`  
**Local path:** `/opt/stacks`  
**Managed stacks:** `proxmox-mcp`, `unraid-mcp`, `komodo-mcp`, `unifi-mcp`; plus `observer/tunnel-client/` recovery templates  
**Administration:** SSH and CLI  
**GitHub token storage:** Bitwarden; not deliberately persisted through Git's plaintext credential helper.

During initial deployment, authenticated Git operations were performed with the credential helper disabled for the individual command.

This avoided saving the token through `credential.helper store`.

### Synchronize Without Persistent Credential Storage

```bash
cd /opt/stacks

# Inspect local changes
git status -sb
git status --short --untracked-files=all

# Fetch without using configured credential helpers
git -c credential.helper= fetch origin

# Inspect branch divergence and incoming changes
git status -sb
git log --oneline --left-right origin/main...HEAD
git diff --name-status HEAD..origin/main

# After reviewing changes, fast-forward
git merge --ff-only origin/main
```

Alternatively, when the working tree and incoming changes are known to be safe:

```bash
git -c credential.helper= pull --ff-only
```

If Git prompts for HTTPS credentials, supply the authorized GitHub username and token interactively.

Do not place a token directly in the command, remote URL, or Git configuration.

### Commit an Observer Configuration Change

```bash
cd /opt/stacks

# Inspect changes
git status
git diff -- proxmox-mcp/compose.yaml

# Stage only the intended file
git add -- proxmox-mcp/compose.yaml

# Review staged content
git diff --cached

# Commit
git commit -m "update proxmox-mcp configuration"
```

### Push Without Persistent Credential Storage

```bash
cd /opt/stacks

git -c credential.helper= push

git status
```

This disables Git's configured credential helpers for the individual operation. It does not modify the host's permanent Git configuration.

**Security note:** Avoid placing credentials in shell arguments or URLs, which can expose them through command history or process inspection.

### Files Excluded From Git

The observer's credential-bearing files include all four MCP `.env` files and protected tunnel runtime keys.

The examples in Git retain references to files, not the referenced secret values.

In addition to the paths listed below, protect `/opt/stacks/unraid-mcp/.env`, `/opt/stacks/komodo-mcp/.env`, `/opt/stacks/unifi-mcp/.env`, every `/etc/tunnel-client*/openai-runtime.key`, and the two existing `mcp-authorization` files.

- `/opt/stacks/proxmox-mcp/.env`
- `/etc/tunnel-client/openai-runtime.key`
- `/etc/tunnel-client/mcp-authorization`

The active tunnel YAML configurations and installed systemd units remain outside the repository.

On November 6, 2026, sanitized YAML examples and verified copies of all four systemd units were committed under `observer/tunnel-client/` and pushed as `9887296`.

The Komodo/UniFi MCP deployment definitions, UniFi local image build files, custom launcher and pinned `source/` submodule were pushed earlier as `6502c59`.

The local `observer-prod-01` `main` branch was verified synchronized with GitHub after both commits.

On November 14, 2026, Observer was subsequently fast-forwarded to `042bbd5` as part of the four-host Git synchronization.

A `git pull` on docker-prod-01, auth-prod-01, or immich-prod-01 obtains these inert files but does not run them.

The active OpenAI runtime keys, protected MCP bearer credentials, MCP `.env` files, live tunnel configs and installed binary remain outside Git.

A Git clone or pull alone is insufficient to rebuild the complete AI Observer.

See `services/infra/ai-observer.md` for implementation details.

---

### UniFi Git Submodule and Multi-Host Divergence

The parent repository tracks `unifi-mcp/source` as a Git submodule at revision `e13d0e22dce1b6a498edc5982d78bd589073d44c` (upstream `ry-ops/unifi-mcp-server`).

A fresh clone requires explicit submodule initialization to obtain the working tree:

```bash
cd /opt/stacks

git submodule status -- unifi-mcp/source

git submodule update --init --recursive -- unifi-mcp/source
```

These commands are for an authorized checkout or rebuild.

Do not casually change permissions of the current root-owned UniFi stack/source or use `sudo git` without assessing ownership and credential implications.

Git includes the submodule reference, not an immutable container image, a key, or a live deployment.

On November 6, a push from Observer was initially rejected because another host had advanced `main` with `61d7ed9` (`komodo/compose.yaml`).

The operator fetched and inspected the changed files, then rebased the unpublished observer commit without conflicts, producing `9887296`, and pushed successfully.

When local and remote have diverged, do not assume a blind `git pull --ff-only` will work or use `git push --force`.

Inspect:

```bash
git log --oneline HEAD..origin/main

git log --oneline origin/main..HEAD

git diff --name-status HEAD..origin/main
```

Review changes from the merge base when determining whether local and remote modifications overlap.

Resolve conflicts carefully and select an appropriate reviewed merge or rebase strategy.

Never routinely rebase shared published history.

---

## Notes

- Synchronize the repository before making configuration changes.
- Inspect local modifications and untracked files before pulling.
- Review incoming deletions, especially under directories used as Docker bind mounts.
- Prefer explicit file staging over `git add .`.
- Never use `git push --force` as a routine synchronization workaround.
- Commit intended configuration changes before making major deployment changes where practical, providing a rollback reference.
- `.env` files remain excluded from Git.
- Generated Beszel agent runtime state must remain host-specific and ignored by Git.
- Host-specific Komodo Periphery files observed during synchronization were preserved pending separate review.
- Git version control is not a substitute for backups.
- A successful commit or push does not verify the corresponding service deployment.
- A successful Git synchronization does not imply that services should be redeployed.
- A running container does not guarantee that its bind mounts remain valid after filesystem changes.
- The observer's tunnel client is not deployed through Docker Compose.
- Any new VM hosting Compose configuration should have Git installed, the correct repository cloned, an appropriate Git identity configured, and a secure authentication method established.

---

## Related Documentation

- `runbooks/new-stack-checklist.md` — Creating new stacks on docker-prod-01.
- `runbooks/updating-containers.md` — Updating Docker Compose containers and controlled image upgrades.
- `runbooks/backup-schedule.md` — Backup jobs and current coverage.
- `services/monitoring/beszel.md` — Beszel agent configuration and host-specific runtime data.
- `services/infra/traefik.md` — Reverse proxy, security hardening, and verified November 2026 upgrade.
- `services/infra/authentik.md` — Authentik deployment and version management.
- `services/infra/dockman.md` — Retired Dockman deployment and November 2026 decommission record.
- `services/infra/ai-observer.md` — Observer architecture, deployment, and credentials.
- `runbooks/ai-observer-operations.md` — Observer operations and recovery.

**Related repositories:**

- `homelab-infra` — Infrastructure Compose configuration.
- `homelab-docs` — Obsidian documentation and operational runbooks.
