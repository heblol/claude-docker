## Why

On a macOS host, in-container git refused a workspace with `fatal: detected dubious ownership in repository at '/workspaces/<name>'`, although the workspace, its `.git` and every worktree were owned by the UID git ran as. Only Windows hosts get `safe.directory=/workspaces/*`, so on Linux and macOS a failed ownership check stops git outright.

The failure is costly well beyond git itself. Agents work around it by appending `safe.directory` entries to `~/.gitconfig`, which lives in the persistent `claude-code-root` volume and so accumulates across sessions and workspaces. They also `export GIT_CONFIG_COUNT=1 …`, which replaces any `GIT_CONFIG_*` set that `run.sh` passed. Scripts that start git with a cleaned environment drop the Windows env-based setting as well, and still fail.

## What Changes

- The image's system git config (`/etc/gitconfig`) carries `safe.directory = /workspaces/*` on every host. Git honours `safe.directory` at system scope, a `GIT_CONFIG_*` export that adds entries (the agents' workaround) doesn't remove it, and git reads it regardless of the environment it runs in.
- `run.sh` no longer passes `GIT_CONFIG_COUNT` / `GIT_CONFIG_KEY_0` / `GIT_CONFIG_VALUE_0` on Windows hosts; the image setting replaces it.
- The scope is unchanged: only paths under `/workspaces` are trusted. A repository elsewhere in the container (e.g. under `/tmp`) whose owner differs is still refused.

## Capabilities

- `container-runtime`: MODIFIED *Windows hosts get a working in-container identity and git* (the `safe.directory` bullet moves out); ADDED *In-container git trusts only the mounted workspaces*.

## Impact

- Linux/macOS: git no longer refuses a workspace on an ownership mismatch. Every path under `/workspaces` is a directory the user passed to `claude-docker`, owned by the user or writable by the container (git already trusted a `--ro` workspace: it's owned by the UID the agent runs as), so trusting its owner adds no access the agent lacks.
- Windows: same behaviour, delivered by the image instead of the environment.
- Existing `safe.directory` entries in a warm `claude-code-root` volume are left alone. They become redundant, and the docs tell the user to delete them.

## Open question

This change doesn't explain the macOS failure itself: `stat` showed uid 502 throughout, the same uid git ran as. One candidate is the VM's file-sharing layer reporting a different owner, or failing a stat, which git treats as "not owned", for the worktree or common dir. The image setting makes git accept the workspace whatever the cause.
