## MODIFIED Requirements

### Requirement: Windows hosts get a working in-container identity and git

A Windows host has no POSIX ownership to preserve: under Git Bash, `id -u` /
`id -g` return synthetic values derived from the Windows SID (e.g.
`197609`/`197121`), and engine bind mounts of NTFS paths show files as owned by
container root regardless of who wrote them. Under an MSYS/MINGW/Cygwin shell,
`run.sh` SHALL therefore:

- forward `HOST_UID=1000` and `HOST_GID=1000` instead of the synthetic IDs, so
  the entrypoint's privilege drop targets a UID that fits the default
  subordinate-ID range of rootless engines (65536 IDs). The drop itself is
  unchanged — the agent SHALL still run as a non-root UID with no usable
  capabilities;
- carry the host repository's effective `core.autocrlf` (as reported by the
  host's `git config --get core.autocrlf` for that workspace, which includes
  Git for Windows' system-level default) into that workspace's container-only
  `.git/config` overlay, so in-container git agrees with the host on line
  endings.

The root-owned NTFS mounts would trip git's ownership check; the image's
system-wide `safe.directory` (requirement *In-container git trusts only the
mounted workspaces*) covers them, so `run.sh` passes no git configuration
through the environment.

Off MSYS/MINGW/Cygwin, `run.sh` SHALL forward `id -u` / `id -g` unchanged and
SHALL NOT add the `core.autocrlf` overlay entry.

#### Scenario: Git Bash forwards a fixed non-root UID

- **GIVEN** `run.sh` runs under Git Bash, where `id -u` prints `197609`
- **WHEN** the wrapper builds the engine argv
- **THEN** it passes `HOST_UID=1000` and `HOST_GID=1000`
- **AND** the agent process inside the container runs as UID 1000, not 0

#### Scenario: in-container git works on a Windows workspace

- **GIVEN** `run.sh` runs under Git Bash and the workspace is a git repo on NTFS
- **WHEN** the agent runs `git status` in `/workspaces/<name>`
- **THEN** git does not fail with `detected dubious ownership`
- **AND** no `GIT_CONFIG_*` variable is passed to the engine

#### Scenario: host line-ending policy reaches the container

- **GIVEN** `run.sh` runs under Git Bash and the host's effective `core.autocrlf` for the workspace is `true`
- **WHEN** the agent runs `git config --get core.autocrlf` in `/workspaces/<name>`
- **THEN** it prints `true`
- **AND** `git status` does not report CRLF-checked-out files as modified

#### Scenario: Linux and macOS are unaffected

- **GIVEN** `run.sh` runs under a Linux or macOS shell
- **WHEN** the wrapper builds the engine argv
- **THEN** `HOST_UID` / `HOST_GID` equal the host's `id -u` / `id -g`
- **AND** no `GIT_CONFIG_*` variable is passed and the `.git/config` overlay gains no `core.autocrlf` entry

## ADDED Requirements

### Requirement: In-container git trusts only the mounted workspaces

On every host OS, the image's system git config (`/etc/gitconfig`) SHALL set
`safe.directory = /workspaces/*` and no other `safe.directory` value, so git's
ownership check never refuses a mounted workspace or a worktree nested inside
one. `run.sh` SHALL NOT pass `safe.directory` through the environment.

#### Scenario: an ownership mismatch under /workspaces is accepted

- **GIVEN** a git repository at `/workspaces/<name>`, or a worktree under `/workspaces/<name>/.claude/worktrees/`, whose owner git sees as differing from the current user
- **WHEN** the agent runs `git rev-parse --show-toplevel` there
- **THEN** git prints the top-level path and does not fail with `detected dubious ownership`

#### Scenario: an exported GIT_CONFIG set does not drop the trust

- **GIVEN** the same repository, and a session that sets its own git config through the environment (system scope is read regardless)
- **WHEN** the agent runs git with `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=safe.directory GIT_CONFIG_VALUE_0=/elsewhere` exported
- **THEN** git still accepts the repository

#### Scenario: a repository outside /workspaces is still refused

- **GIVEN** a git repository under `/tmp` whose owner git sees as differing from the current user
- **WHEN** the agent runs git there
- **THEN** git fails with `detected dubious ownership`

#### Scenario: the system scope trusts only /workspaces

- **WHEN** the agent runs `git config --system --get-all safe.directory`
- **THEN** it prints exactly `/workspaces/*`
