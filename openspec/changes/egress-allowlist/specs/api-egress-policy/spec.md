# Spec Delta

## ADDED Requirements

### Requirement: --egress-allowlist needs --egress-lock

`--egress-allowlist` and `--egress-allowlist=FILE` SHALL turn on the allowlist mode of the egress proxy. Without `--egress-lock`, `run.sh` SHALL exit 1 before creating any container resource. `--egress-allow=ENTRY` without `--egress-allowlist`, or with an empty value, SHALL also exit 1. Without `--egress-allowlist`, `CLAUDE_DOCKER_EGRESS_ALLOWLIST` SHALL NOT be read or checked, and the session SHALL be unaffected.

#### Scenario: Allowlist without the lock

- **WHEN** the user runs `claude-docker --api --egress-allowlist ~/repo`
- **THEN** `run.sh` exits 1 with `--egress-allowlist needs --egress-lock`, and no container starts

#### Scenario: Entry without the allowlist

- **WHEN** the user runs `claude-docker --api --egress-lock --egress-allow=pypi.org ~/repo`
- **THEN** `run.sh` exits 1 with `--egress-allow needs --egress-allowlist`, and no container starts

#### Scenario: Empty entry

- **WHEN** the user passes `--egress-allow=` with `--egress-allowlist`
- **THEN** `run.sh` exits 1, and no container starts

#### Scenario: The lock alone ignores the env var

- **GIVEN** `CLAUDE_DOCKER_EGRESS_ALLOWLIST=/nonexistent`
- **WHEN** an `--api --egress-lock` session without `--egress-allowlist` requests `https://example.org/` through the proxy
- **THEN** the session starts and the request succeeds

#### Scenario: The lock alone generates the same config

- **WHEN** `run.sh` generates the squid config for an `--egress-lock` session without `--egress-allowlist`
- **THEN** it is byte-identical to the config generated before this change
- **AND** the saved `.meta` has the same keys as before this change, and no `.conf` file is saved

#### Scenario: With --api-direct

- **WHEN** the user runs `claude-docker --api-direct --egress-lock --egress-allowlist ~/repo`
- **THEN** the session starts in allowlist mode

### Requirement: Under --egress-allowlist, unlisted destinations are refused

Under `--egress-allowlist`, the proxy's last rule SHALL deny every request. Before it, the proxy SHALL allow only the `ANTHROPIC_BASE_URL` endpoint, the `--api` sidecar's model-proxy name, and the listed entries. No credential flag SHALL add a host. The metadata, link-local, loopback, provider-host and port denies SHALL come before every allow. A private-range destination SHALL be reachable only through a listed name.

#### Scenario: Listed and unlisted hosts

- **GIVEN** an `--api --egress-lock --egress-allowlist --egress-allow=example.com` session
- **WHEN** the agent requests `https://example.com/` and `https://example.org/` through the proxy
- **THEN** the first succeeds and the proxy answers `CONNECT example.org:443` with 403

#### Scenario: Plain HTTP to an unlisted host

- **GIVEN** an `--egress-allowlist` session that doesn't list `example.org`
- **WHEN** the agent requests `http://example.org/` through the proxy
- **THEN** the proxy answers 403

#### Scenario: Endpoint without any list

- **GIVEN** an `--api --egress-lock --egress-allowlist` session with no allowlist file and no `--egress-allow`, and `ANTHROPIC_BASE_URL=https://gw.example.eu:8443`
- **WHEN** Claude Code sends a model request
- **THEN** it reaches the endpoint

#### Scenario: Sidecar name on another port

- **GIVEN** an `--api --egress-lock --egress-allowlist` session
- **WHEN** a request for `http://claude-docker-api-proxy:9999/` is sent through the proxy
- **THEN** the proxy answers 403

#### Scenario: Credential flag adds nothing

- **GIVEN** an `--api --tfe --egress-lock --egress-allowlist` session with no entries
- **WHEN** the agent requests `https://app.terraform.io/`
- **THEN** the proxy answers 403

#### Scenario: Provider host in another spelling

- **GIVEN** an `--egress-allowlist` session
- **WHEN** requests for `https://API.Anthropic.com/` and `https://api.anthropic.com./` are sent through the proxy
- **THEN** the proxy answers both with 403

#### Scenario: IP-literal requests

- **GIVEN** an `--egress-allowlist` session
- **WHEN** requests for `http://10.0.0.5/`, `http://2130706433/` and `http://[2001:db8::1]/` are sent through the proxy
- **THEN** the proxy answers each with 403

#### Scenario: Listed on-prem name

- **GIVEN** an `--egress-allowlist` session listing `git.internal.example`, which resolves to `10.20.1.2`
- **WHEN** a request for `https://git.internal.example/` is sent through the proxy
- **THEN** it is allowed

#### Scenario: Listed name resolving to metadata

- **GIVEN** an `--egress-allowlist` session listing a name that resolves to `169.254.169.254`
- **WHEN** a request for that name is sent through the proxy
- **THEN** the proxy answers 403

### Requirement: --gh and --glab need their sidecar's hosts listed

Under `--egress-allowlist`, `run.sh` SHALL exit 1 before creating any container resource when `--gh` is passed and any of `github.com`, `api.github.com` or `uploads.github.com` is not covered by an entry, or when `--glab` is passed and the proxied GitLab host is not covered by an entry, naming the missing hosts. An exact entry or a covering `.suffix`, with or without a port, SHALL count.

#### Scenario: --gh without GitHub hosts

- **WHEN** the user runs `claude-docker --api --gh --egress-lock --egress-allowlist ~/repo` with no entries
- **THEN** `run.sh` exits 1 naming `github.com`, `api.github.com` and `uploads.github.com`, and no container starts

#### Scenario: --gh with GitHub hosts listed

- **GIVEN** entries `github.com`, `api.github.com`, `uploads.github.com` and `codeload.github.com`
- **WHEN** an `--api --gh --egress-lock --egress-allowlist` session clones a repository over HTTPS
- **THEN** the clone succeeds through the sidecar

#### Scenario: --glab with a suffix entry

- **GIVEN** `--glab` for `gitlab.internal.example:8443` and the entry `.internal.example`
- **WHEN** the user starts an `--egress-allowlist` session
- **THEN** the session starts

#### Scenario: --gh-direct needs no check

- **WHEN** the user runs `claude-docker --api --gh-direct --egress-lock --egress-allowlist ~/repo` with no entries
- **THEN** the session starts, and requests to `https://github.com/` are answered with 403

### Requirement: Allowlist ports

Under `--egress-allowlist`, a host or `.suffix` entry with `:port` SHALL make that port reachable for that entry only, in addition to 80 and 443. Every other destination SHALL stay on 80 and 443, with `CONNECT` only to 443.

#### Scenario: Listed host on its own port

- **GIVEN** an `--egress-allowlist` session with `--egress-allow=git.example.com:8443`
- **WHEN** the agent requests `https://git.example.com:8443/`, `https://git.example.com/` and `https://example.com:8443/` through the proxy
- **THEN** the first two are allowed and the proxy answers `CONNECT example.com:8443` with 403

#### Scenario: Suffix with a port

- **GIVEN** an `--egress-allowlist` session with `--egress-allow=.internal.example:8443`
- **WHEN** the agent requests `https://git.internal.example:8443/` and `https://git.internal.example:9443/`
- **THEN** the first is allowed and the proxy answers the second with 403

#### Scenario: Loopback listed with a port

- **GIVEN** an `--egress-allowlist` session with `--egress-allow=localhost:8080`
- **WHEN** a request for `http://localhost:8080/` is sent through the proxy
- **THEN** the proxy answers 403

### Requirement: Allowlist sources

The effective allowlist SHALL be the union of the endpoint and the `--api` sidecar name, at most one allowlist file and every `--egress-allow=ENTRY`. The file SHALL be `FILE` from `--egress-allowlist=FILE`, else the path in `CLAUDE_DOCKER_EGRESS_ALLOWLIST`, else none. A leading `~/` in either SHALL be expanded to `$HOME/`. No other file SHALL be read.

#### Scenario: File from the env var

- **GIVEN** `CLAUDE_DOCKER_EGRESS_ALLOWLIST=$HOME/team.allow` listing `pypi.org`
- **WHEN** an `--egress-allowlist` session requests `https://pypi.org/`
- **THEN** the request succeeds

#### Scenario: Flag file wins

- **GIVEN** `CLAUDE_DOCKER_EGRESS_ALLOWLIST` names a file listing `pypi.org`, and `$HOME/other.allow` lists `example.com`
- **WHEN** an `--egress-allowlist=$HOME/other.allow` session requests `https://pypi.org/`
- **THEN** the proxy answers 403

#### Scenario: File and flags combine

- **GIVEN** `$HOME/team.allow` listing `pypi.org`
- **WHEN** an `--egress-allowlist=$HOME/team.allow --egress-allow=example.com` session requests `https://pypi.org/` and `https://example.com/`
- **THEN** both succeed

#### Scenario: Tilde in the flag value

- **GIVEN** `$HOME/team.allow` listing `pypi.org`
- **WHEN** the user passes `--egress-allowlist=~/team.allow` with the `~` unexpanded by the shell
- **THEN** `run.sh` reads `$HOME/team.allow`

#### Scenario: No fixed fallback file

- **GIVEN** no flag file, no `CLAUDE_DOCKER_EGRESS_ALLOWLIST`, and `$XDG_CONFIG_HOME/claude-docker/egress-allowlist` listing `pypi.org`
- **WHEN** an `--egress-allowlist` session requests `https://pypi.org/`
- **THEN** the proxy answers 403

### Requirement: The allowlist file is out of the agent's reach

A path given by `--egress-allowlist=FILE` or `CLAUDE_DOCKER_EGRESS_ALLOWLIST` that is not a readable regular file SHALL make `run.sh` exit 1 before creating any container resource, naming the path and its source. With symlinks resolved on both sides, the file SHALL NOT lie inside any workspace the session mounts, read-only or not, compared on a `/` boundary; if it does, `run.sh` SHALL exit 1 naming the workspace.

#### Scenario: Missing file

- **WHEN** the user runs `claude-docker --api --egress-lock --egress-allowlist=/nonexistent ~/repo`
- **THEN** `run.sh` exits 1 with `not a readable regular file` and `/nonexistent`, and no container starts

#### Scenario: Directory as file

- **WHEN** the user passes `--egress-allowlist=$HOME`
- **THEN** `run.sh` exits 1 with `not a readable regular file`

#### Scenario: File inside the workspace

- **WHEN** the user runs `claude-docker --api --egress-lock --egress-allowlist=$HOME/repo/allow ~/repo`
- **THEN** `run.sh` exits 1 naming `~/repo`, and no container starts

#### Scenario: Symlinked workspace

- **GIVEN** `~/work` is a symlink to `/data/work`
- **WHEN** the user runs a session on `~/work` with `--egress-allowlist=/data/work/allow`
- **THEN** `run.sh` exits 1 naming the workspace

#### Scenario: Symlink into the workspace

- **GIVEN** `$HOME/team.allow` is a symlink to `$HOME/repo/allow`
- **WHEN** the user runs a session on `~/repo` with `--egress-allowlist=$HOME/team.allow`
- **THEN** `run.sh` exits 1 naming the workspace

#### Scenario: Read-only second workspace

- **WHEN** the user runs a session on `~/repo` and `--ro ~/docs` with `--egress-allowlist=$HOME/docs/allow`
- **THEN** `run.sh` exits 1 naming `~/docs`

#### Scenario: Sibling directory

- **WHEN** the user runs a session on `~/repo` with `--egress-allowlist=$HOME/repo2/allow`
- **THEN** the file is accepted

#### Scenario: File given with a space

- **WHEN** the user runs `claude-docker --api --egress-lock --egress-allowlist $HOME/team.allow ~/repo`
- **THEN** `run.sh` exits 1 and suggests `--egress-allowlist=FILE`

### Requirement: Allowlist entry grammar

Every allowlist entry SHALL be a hostname or a `.suffix` of at least two labels, either with an optional `:port` (1-65535). A host SHALL use only letters, digits, `-` and `.`, be at most 253 characters, have labels of 1-63 characters that neither start nor end with `-`, have no trailing dot, and be lowercased. In the file, blank lines, whole-line and trailing `#` comments SHALL be ignored and a trailing CR stripped. Exact duplicates SHALL be dropped.

#### Scenario: Every entry form

- **GIVEN** an allowlist file with the lines `# team list`, `PyPI.org  # index`, `.example.org`, `git.example.com:8443`, `.internal.example:8443` and `pypi.org`
- **WHEN** the user starts an `--egress-allowlist` session
- **THEN** the session starts, and the saved `.conf` lists `pypi.org` once, `.example.org`, `git.example.com` with port 8443 and `.internal.example` with port 8443

### Requirement: An invalid allowlist entry is fatal

An entry outside the grammar, an IPv4 address, an entry equal to or under an `EGRESS_MODEL_PROVIDERS` domain in any case, or an entry covered by a listed `.suffix` SHALL make `run.sh` exit 1 before creating any container resource, naming its source and, for the file, the line.

#### Scenario: Wildcard spelling

- **GIVEN** an allowlist file whose line 3 is `*.example.com`
- **WHEN** the user starts an `--egress-allowlist` session
- **THEN** `run.sh` exits 1, stderr names the file with `:3`, and suggests `.example.com`

#### Scenario: Invalid hosts

- **WHEN** the user passes, one at a time, `--egress-allow=` with `-a.example.com`, `a..example.com`, `example.com.`, a 254-character name and `.com`
- **THEN** `run.sh` exits 1 each time

#### Scenario: Bad ports

- **WHEN** the user passes, one at a time, `--egress-allow=` with `example.com:0` and `example.com:65536`
- **THEN** `run.sh` exits 1 each time

#### Scenario: IP address

- **WHEN** the user passes `--egress-allow=10.0.0.5`
- **THEN** `run.sh` exits 1, saying IP entries are not supported

#### Scenario: Provider hosts in any spelling

- **WHEN** the user passes, one at a time, `--egress-allow=` with `api.anthropic.com`, `API.Anthropic.com`, `.ANTHROPIC.com`, `.anthropic.com:8443` and `claude.ai`
- **THEN** `run.sh` exits 1 each time, saying the lock refuses model providers' hosts

#### Scenario: Entry covered by a suffix

- **WHEN** the user passes `--egress-allow=api.example.com --egress-allow=.example.com`
- **THEN** `run.sh` exits 1 naming both entries

#### Scenario: Entry smuggles squid config

- **GIVEN** an allowlist file line `example.com all`
- **WHEN** the user starts an `--egress-allowlist` session
- **THEN** `run.sh` exits 1 naming the file and line

### Requirement: Unlisted names are not resolved

Under `--egress-allowlist`, the proxy SHALL deny a request whose host is not a listed name before any rule that matches on a resolved address. The proxy SHALL send no DNS query for such a host.

#### Scenario: Rule order

- **WHEN** the bats suite inspects the generated config under `--egress-allowlist`
- **THEN** the `deny !egress_allowed_names` rule comes before every `http_access` line that uses a `dst` ACL

#### Scenario: Secret in an unlisted name

- **GIVEN** the generated config run by squid from the proxy image, with `dns_nameservers` pointed at a stub resolver that logs every query
- **WHEN** requests for `https://secret.attacker.example/`, `http://SECRET.attacker.example/` and `https://1.2.3.4.secret.attacker.example/` are sent through it
- **THEN** each is answered with 403
- **AND** the stub's log has no query containing `secret`

### Requirement: DNS is closed under --egress-allowlist

Under `--egress-allowlist`, after the internal network exists and before the agent container starts, `run.sh` SHALL check from a probe container on that network that an external name does not resolve. When it resolves, or the probe can't run, `run.sh` SHALL abort and say that `--egress-allowlist` needs an engine that doesn't forward external DNS from internal networks.

#### Scenario: Engine forwards DNS

- **GIVEN** an engine whose internal networks resolve external names
- **WHEN** the user starts an `--egress-allowlist` session
- **THEN** `run.sh` aborts before the agent container starts, naming DNS as the reason

#### Scenario: Engine doesn't forward DNS

- **GIVEN** Docker 26 or later
- **WHEN** the user starts an `--egress-allowlist` session
- **THEN** the probe passes and the session starts

## MODIFIED Requirements

### Requirement: The session's egress log is saved on the host

When an `--egress-lock` session whose proxy was started exits, the EXIT trap SHALL, before removing the proxy, write the proxy's access log to `<state>/claude-docker/egress/<start>-<id>.log` and a `<start>-<id>.meta` file of `key=value` lines (`start`, `end`, `user`, `host`, `workspace`, `image`, `image_id`, `proxy_image`, `proxy_image_id`, `endpoint`), where `<state>` is `$XDG_STATE_HOME` or `~/.local/state`. Under `--egress-allowlist` only, the `.meta` file SHALL also have `allowlist_file` (resolved path, or empty) and `allowlist_sha256` (of the bytes parsed, or empty), and the generated squid config SHALL be saved as `<start>-<id>.conf`. That directory SHALL NOT be mounted into any container. `run.sh` SHALL NOT rotate or delete saved logs. `run.sh` SHALL check the copy: when the engine's log command fails, or the proxy had been accepting connections but the engine's output lacks squid's `Accepting HTTP Socket connections` startup line, it SHALL print that the log in that file is missing or incomplete, with the reason, and SHALL NOT report it saved. The check SHALL NOT fail the teardown.

#### Scenario: Log saved

- **WHEN** an `--egress-lock` session that connected to `example.org` exits
- **THEN** a `.log` file under the state directory contains the `CONNECT example.org:443` line
- **AND** its `.meta` file contains `endpoint=` followed by the `ANTHROPIC_BASE_URL` host
- **AND** `run.sh` reports the log saved

#### Scenario: Allowlist evidence

- **GIVEN** an `--egress-allowlist=$HOME/team.allow --egress-allow=example.com` session
- **WHEN** the session exits
- **THEN** the `.meta` file has `allowlist_file=` with the resolved path of `$HOME/team.allow` and `allowlist_sha256=` with its SHA-256
- **AND** a `.conf` file next to it is the squid config the proxy ran with, including `example.com`

#### Scenario: Allowlist evidence without a file

- **GIVEN** an `--egress-allowlist --egress-allow=example.com` session with no allowlist file
- **WHEN** the session exits
- **THEN** the `.meta` file has empty `allowlist_file=` and `allowlist_sha256=`, and the `.conf` file is saved

#### Scenario: The copy fails

- **GIVEN** an `--egress-lock` session whose engine can't return the proxy's log (for example, a log driver that doesn't support reading)
- **WHEN** the session exits
- **THEN** `run.sh` prints that the log in the `.log` file is missing or incomplete, with the engine's error
- **AND** it doesn't print "egress log saved"

#### Scenario: The log driver rotated the log

- **GIVEN** an `--egress-lock` session whose engine kept only the end of the proxy's log (`max-size` in `daemon.json`)
- **WHEN** the session exits
- **THEN** `run.sh` prints that the log in the `.log` file is incomplete because squid's startup line is gone
- **AND** it doesn't print "egress log saved"

### Requirement: Model traffic reaches only the configured endpoint

Under `--egress-lock`, `run.sh` SHALL exit 1 before creating any container resource when `ANTHROPIC_BASE_URL` is unset or empty, when its host is not a hostname or IPv4 address of at most 253 characters made of letters, digits, `-` and `.`, when its port is not a number from 1 to 65535, or when its host, compared case-insensitively, is `anthropic.com`, `claude.ai` or `claude.com` or a subdomain of one. The endpoint's port is the one in `ANTHROPIC_BASE_URL`, or the scheme's default (80 for `http`, 443 otherwise). The proxy SHALL allow the `ANTHROPIC_BASE_URL` host on the endpoint's port, and on 80/443, and deny `.anthropic.com`, `.claude.ai` and `.claude.com`. Without `--egress-allowlist` it SHALL then allow every other destination that the metadata, loopback and port rules permit; under it, only what the allowlist requirements state. HTTPS SHALL be proxied by `CONNECT` without TLS interception.

#### Scenario: No endpoint

- **WHEN** the user runs `claude-docker --api --egress-lock ~/repo` with a token but no `ANTHROPIC_BASE_URL`
- **THEN** `run.sh` exits 1 with `--egress-lock needs ANTHROPIC_BASE_URL`, and no container starts

#### Scenario: Endpoint is a provider host

- **GIVEN** `ANTHROPIC_BASE_URL=https://api.anthropic.com`
- **WHEN** the user runs `claude-docker --api --egress-lock ~/repo`
- **THEN** `run.sh` exits 1, and no container starts

#### Scenario: Endpoint is a provider host in another case

- **GIVEN** `ANTHROPIC_BASE_URL=https://Api.Anthropic.com`
- **WHEN** the user runs `claude-docker --api --egress-lock ~/repo`
- **THEN** `run.sh` exits 1, and no container starts

#### Scenario: Endpoint on a non-default port

- **GIVEN** an `--egress-lock` session with `ANTHROPIC_BASE_URL=https://gw.example.eu:8443`
- **WHEN** the agent requests `https://gw.example.eu:8443/` and `https://example.org:8443/` through the proxy
- **THEN** the proxy allows the `CONNECT gw.example.eu:8443` and answers `CONNECT example.org:8443` with 403

#### Scenario: Invalid endpoint port

- **GIVEN** `ANTHROPIC_BASE_URL=https://gw.example.eu:65536`
- **WHEN** the user runs `claude-docker --api --egress-lock ~/repo`
- **THEN** `run.sh` exits 1 with `is not a port number`

#### Scenario: Endpoint smuggles squid config

- **GIVEN** an `ANTHROPIC_BASE_URL` whose host contains a newline or a space
- **WHEN** the user runs `claude-docker --api --egress-lock ~/repo`
- **THEN** `run.sh` exits 1 with `is not a valid hostname`

#### Scenario: Provider host refused, other hosts open

- **GIVEN** an `--egress-lock` session without `--egress-allowlist`, with `ANTHROPIC_BASE_URL=https://example.com`
- **WHEN** the agent requests `https://example.com/`, `https://example.org/` and `https://api.anthropic.com/` through the proxy
- **THEN** the first two succeed and the proxy answers `CONNECT api.anthropic.com` with 403

### Requirement: Metadata and loopback destinations are denied by resolved address

The proxy SHALL deny destinations in `169.254.0.0/16`, `fe80::/10`, `127.0.0.0/8`, `0.0.0.0/8` and `::1`, the metadata addresses `100.100.100.200` (Alibaba Cloud) and `fd00:ec2::254` (AWS IPv6), and the hostnames `metadata.google.internal` and `metadata.azure.internal`, above every allow rule. These checks SHALL apply to the address the proxy resolves for a hostname, not just to IP-literal requests. Without `--egress-allowlist`, private ranges SHALL be reachable. The proxy SHALL permit only ports 80 and 443, and `CONNECT` only to 443, except that the `ANTHROPIC_BASE_URL` host is also reachable on the endpoint's port, and under `--egress-allowlist` an entry listed with a port on that port; those exceptions SHALL come after the metadata, link-local and loopback denies.

#### Scenario: Cloud metadata

- **WHEN** `curl http://169.254.169.254/` runs in the agent container
- **THEN** the proxy answers 403

#### Scenario: Other clouds' metadata addresses

- **WHEN** a request for `http://100.100.100.200/` or `http://[fd00:ec2::254]/` is sent through the proxy
- **THEN** the proxy answers 403

#### Scenario: Name resolving to loopback

- **WHEN** a request for `http://localhost/` is sent through the proxy
- **THEN** the proxy answers 403

#### Scenario: Non-443 CONNECT

- **GIVEN** `ANTHROPIC_BASE_URL=https://example.com`
- **WHEN** a request for `https://example.com:8443/` is sent through the proxy
- **THEN** the proxy answers 403
