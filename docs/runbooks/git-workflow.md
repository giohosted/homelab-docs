# Git Workflow — homelab-infra

**Applies to:** docker-prod-01, auth-prod-01, immich-prod-01, observer-prod-01 (`/opt/stacks`)  
**Repo:** `homelab-infra` (private, GitHub)  
**Last Updated:** 2026-10-29

---

## Overview

`/opt/stacks` on each participating host is a clone of the private `homelab-infra` GitHub repository.

Docker Compose configuration files are maintained in this repository. Changes are committed and pushed to GitHub to preserve version history and keep the intended infrastructure configuration consistent across hosts.

`/opt/appdata` is not in Git — it contains persistent container state and application configuration.

Secrets are excluded from version control, including `.env` files containing API tokens, passwords, and other sensitive information.

### Repository Locations

|Host|Path|What lives there|Status|
|---|---|---|---|
|docker-prod-01|`/opt/stacks`|traefik, arr, books, torrent, dockman, cloudflared, adguardhome-sync, and other application stacks|Active|
|auth-prod-01|`/opt/stacks`|authentik|Active|
|immich-prod-01|`/opt/stacks`|immich|Active|
|observer-prod-01|`/opt/stacks`|proxmox-mcp|Active|

**GitHub is the source of truth for committed Compose configuration — not any individual host.**

However, GitHub does not contain the complete operational state of each service. Runtime data, secrets, host-level configuration, and installed system services must be managed and recovered separately.

### Host-Specific Management

The repository is shared, but each host manages its own deployed workloads.

- **docker-prod-01:** Docker Compose and Dockman.
    
- **auth-prod-01:** Docker Compose.
    
- **immich-prod-01:** Docker Compose.
    
- **observer-prod-01:** Docker Compose through SSH and CLI.
    

The observer does not use Dockman.

Its OpenAI Secure MCP Tunnel client is installed as a native systemd service rather than as a Docker Compose stack.

See `services/infra/ai-observer.md`.

---

## The Golden Rule

**Always synchronize your local repository before making changes, and inspect your working tree before pulling or committing.**

Because `homelab-infra` is cloned on multiple hosts, changes committed from one host must be incorporated into the others before additional work is committed.

A safe workflow is:

1. Check the working tree.
    
2. Synchronize with the remote repository if the working tree is clean.
    
3. Make the intended changes.
    
4. Review and stage only the appropriate files.
    
5. Commit locally.
    
6. Push to GitHub.
    
7. Synchronize other hosts before making changes there.
    

Use `git pull --ff-only` for routine synchronization. It refuses a pull that would require an automatic merge.

**Important:** If local files are already modified, do not blindly pull, reset, or overwrite them. Inspect and reconcile the changes first.

Git normally prevents a non-fast-forward push from overwriting newer remote commits. The principal risks are conflicts, stale deployments, and accidental replacement of configuration during manual conflict resolution.

Never use force-push as a routine way to resolve synchronization problems.

---

## Standard Workflow

Use this process when making a Compose configuration change on a participating host.

### Step 1 — Inspect and Synchronize

```
cd /opt/stacks

# Check the working tree before pulling
git status

# Synchronize when the working tree is clean
git pull --ff-only
```

If Git reports local modifications or a non-fast-forward situation, stop and investigate before continuing.

### Step 2 — Make Your Changes

Edit the appropriate Compose file or other version-controlled configuration.

For example:

```
nano /opt/stacks/<stack-name>/compose.yaml
```

Do not place credentials directly in the Compose file.

Use a Git-excluded `.env` file for sensitive environment variables when appropriate.

### Step 3 — Review Changes

```
cd /opt/stacks

# Show modified and untracked files
git status

# Review changes to a specific file
git diff -- <stack-name>/compose.yaml
```

Ensure that only intended configuration changes are present.

### Step 4 — Stage Changes

Prefer explicitly staging the files you intend to commit:

```
git add -- <stack-name>/compose.yaml
```

Review the staged changes:

```
git diff --cached
```

Verify that no credentials, sensitive values, or unrelated changes have been staged.

### Step 5 — Commit

```
git commit -m "brief description of what changed"
```

### Step 6 — Push

```
git push
```

If the push is rejected because the remote repository has newer commits, inspect the divergence and reconcile the branches.

Do not force-push to bypass the rejection.

### Step 7 — Verify

```
git status

git log --oneline -5
```

The expected result after a successful push is a clean working tree with the intended commit present.

**Note:** A successful Git push confirms that configuration was committed. It does not prove that a deployed container has been recreated or that the service is healthy.

Application changes require separate deployment and service verification.

---

## When to Use `git add .` vs `git add <path>`

|Situation|Recommended command|
|---|---|
|One specific Compose file changed|`git add -- <stack-name>/compose.yaml`|
|Multiple known files changed|`git add -- <path1> <path2>`|
|All changes were reviewed and should be committed|`git add .`|
|Unsure what changed|Run `git status` and `git diff` first|
|A secret or credential file appears|Do not stage it; investigate why it is not excluded|

`git add .` stages changes throughout the current directory tree, including new files that are not ignored.

For a multi-host infrastructure repository, explicitly staging files is usually safer.

**Always review the staged changes before committing.**

---

## Daily Workflow

After creating or editing a Compose configuration:

```
cd /opt/stacks

# Check local state and synchronize before editing
git status
git pull --ff-only

# Make the intended configuration changes

# Review modifications
git status
git diff

# Stage only the intended files
git add -- <path-to-changed-file>

# Inspect the staged changes
git diff --cached

# Commit and push
git commit -m "brief description of what changed"
git push

# Verify
git status
```

On `observer-prod-01`, use the credential-handling procedure in the dedicated section below for authenticated pull and push operations.

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
    

Commit messages should describe the actual configuration change, not merely state that files were updated.

---

## Adding a New Stack

Before creating a new stack, synchronize the repository:

```
cd /opt/stacks
git status
git pull --ff-only
```

Create the stack directory and Compose file:

```
mkdir -p /opt/stacks/<stack-name>

nano /opt/stacks/<stack-name>/compose.yaml
```

Create an appdata directory only when the application requires persistent host storage:

```
mkdir -p /opt/appdata/<app-name>
```

Create a separate `.env` file if sensitive configuration is required:

```
nano /opt/stacks/<stack-name>/.env
```

Confirm that `.env` is excluded from Git:

```
cd /opt/stacks
git check-ignore -v <stack-name>/.env
```

Review and commit only the intended configuration:

```
git status

git add -- <stack-name>/compose.yaml

git diff --cached

git commit -m "add <stack-name> compose"

git push
```

### Host-Specific File Permissions

Docker hosts do not necessarily share the same file ownership requirements.

On `docker-prod-01`, Dockman requires specific ownership and permissions so it can modify Compose files.

See `runbooks/new-stack-checklist.md`.

Do not apply docker-prod-01's `gio:service` ownership requirements or UID/GID 2000 assumptions automatically to `observer-prod-01`.

The observer is managed through the CLI, and its Proxmox MCP `.env` is restricted to the `gio` account.

---

## Checking What Changed

```
cd /opt/stacks

# See files that are new, modified, or deleted
git status

# Compact status output
git status --short

# See recent commits
git log --oneline -10

# Inspect changes to a specific file
git diff -- <file>

# Inspect all staged changes
git diff --cached

# Show files included in the next commit
git diff --cached --name-only

# Confirm whether a file is ignored
git check-ignore -v <path>
```

Use `git status` and `git diff` before staging.

Use `git diff --cached` after staging.

These are separate checks and both are useful for avoiding accidental changes.

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
    
- Keep the working tree clean between tasks.
    
- Use `git pull --ff-only` for routine updates.
    
- Commit related changes together.
    
- Do not force-push routine infrastructure changes.
    
- Review conflicts carefully if divergent edits occur.
    

### Important Deployment Distinction

Pulling newer Compose configuration onto a host does not automatically recreate its running containers.

Git synchronization and Docker deployment are separate activities.

Only deploy a stack when the relevant service change is intended and the appropriate maintenance procedure has been followed.

---

## Per-Host Git Identity

Each host needs Git author information before creating commits.

The existing `homelab-infra` identity is:

- **Name:** `giohosted`
    
- **Email:** `delgadogiovanny@gmail.com`
    

To configure identity for the repository without changing the system-wide Git configuration:

```
cd /opt/stacks

git config user.name "giohosted"
git config user.email "delgadogiovanny@gmail.com"
```

Verify:

```
git config --get user.name
git config --get user.email
```

The original workflow used global Git identity configuration. That remains valid where already configured.

Repository-local identity is preferable when a host may eventually use different repositories or Git identities.

**Git identity is not GitHub authentication.**

Setting `user.name` and `user.email` does not grant permission to pull or push a private repository.

---

## What Is and Isn't in Git

|Path|In Git|Notes|
|---|---|---|
|`/opt/stacks/*/compose.yaml`|Yes|Intended infrastructure configuration; must not contain secrets|
|`/opt/stacks/*/.env`|No|Gitignored; may contain passwords or API tokens|
|`/opt/stacks/*/.env.example`|Optional|Placeholder values only; never real credentials|
|`/opt/appdata/`|No|Persistent application configuration and runtime state|
|`/etc/tunnel-client/config.yaml`|No|Observer host-level tunnel configuration, outside the repository|
|`/etc/tunnel-client/openai-runtime.key`|No|Protected OpenAI credential|
|`/etc/tunnel-client/mcp-authorization`|No|Protected MCP bearer credential|
|`/etc/systemd/system/tunnel-client.service`|No|Observer systemd service unit, outside the repository|
|`/opt/tunnel-client/`|No|Installed tunnel-client binaries, outside the repository|

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
**Managed stack:** `proxmox-mcp`  
**Administration:** SSH and CLI  
**GitHub token storage:** Bitwarden; not deliberately persisted through Git's plaintext credential helper.

During initial deployment, authenticated Git operations were performed with the credential helper disabled for the individual command.

This avoided saving the token through `credential.helper store`.

### Synchronize Without Persistent Credential Storage

```
cd /opt/stacks

# Inspect local changes
git status

# Pull without using configured Git credential helpers
git -c credential.helper= pull --ff-only
```

If Git prompts for HTTPS credentials, supply the authorized GitHub username and token interactively.

Do not place a token directly in the command, remote URL, or Git configuration.

### Commit an Observer Configuration Change

```
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

```
cd /opt/stacks

git -c credential.helper= push

git status
```

This disables Git's configured credential helpers for the individual operation. It does not modify the host's permanent Git configuration.

**Security note:** Avoid placing credentials in shell arguments or URLs, which can expose them through command history or process inspection.

### Files Excluded From Git

The observer's credential-bearing files include:

- `/opt/stacks/proxmox-mcp/.env`
    
- `/etc/tunnel-client/openai-runtime.key`
    
- `/etc/tunnel-client/mcp-authorization`
    

The host-level tunnel configuration and service unit also remain outside the Compose repository.

A Git clone or pull alone is insufficient to rebuild the complete AI Observer.

See `services/infra/ai-observer.md` for implementation details.

---

## Notes

- Synchronize the repository before making configuration changes.
    
- Inspect local modifications before pulling.
    
- Prefer explicit file staging over `git add .`.
    
- Never use `git push --force` as a routine synchronization workaround.
    
- Commit intended configuration changes before making major deployment changes where practical, providing a rollback reference.
    
- `.env` files remain excluded from Git.
    
- Git version control is not a substitute for backups.
    
- A successful commit or push does not verify the corresponding service deployment.
    
- The observer's tunnel client is not deployed through Docker Compose.
    
- Any new VM hosting Compose configuration should have Git installed, the correct repository cloned, an appropriate Git identity configured, and a secure authentication method established.
    

---

## Related Documentation

- `runbooks/new-stack-checklist.md` — Creating new stacks on docker-prod-01.
    
- `runbooks/updating-containers.md` — Updating Docker Compose containers.
    
- `runbooks/backup-schedule.md` — Backup jobs and current coverage.
    
- `services/infra/dockman.md` — Docker Compose management on docker-prod-01.
    
- `services/infra/ai-observer.md` — Observer architecture, deployment, and credentials.
    
- `runbooks/ai-observer-operations.md` — Observer operations and recovery
    

**Related repositories:**

- `homelab-infra` — Infrastructure Compose configuration.
    
- `homelab-docs` — Obsidian documentation and operational runbooks.