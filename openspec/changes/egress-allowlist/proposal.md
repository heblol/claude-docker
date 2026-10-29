## Why

`--egress-lock` (#75, #101) keeps Claude Code's model traffic on the `ANTHROPIC_BASE_URL` endpoint and logs every connection, but every other host on 80/443 stays reachable: the squid config ends in `http_access allow all`. That was the agreed scope for #75, and the egress-lock change lists "a host or repo allowlist, in any form" as a non-goal.

The teams that use `--egress-lock` mostly have a stricter requirement: all traffic goes to the LLM proxy and to known on-prem hosts, and nothing else (#167). Today `docs/security.md` says to "assume exfiltration already happened" for every session, with or without the lock.

The egress-lock design (D3) rejected a deny-all list because every colleague would hit blocked hosts and edit a policy file. That objection holds for a default, not for a second opt-in that only the teams who need it turn on. This change reverses the non-goal for that opt-in only, and keeps the first version to the core: deny everything except the endpoint and the listed hosts.

## What Changes

- New wrapper flag **`--egress-allowlist[=FILE]`**. It requires `--egress-lock`; given without it, it is a startup error. `--egress-lock` without it keeps its behaviour, its generated squid config and its `.meta` keys.
- **Deny by default.** Under the flag, the squid config ends in `http_access deny all`. Before that, it allows only:
  - the `ANTHROPIC_BASE_URL` endpoint, and under `--api` the auth-proxy sidecar's model-proxy name, as under `--egress-lock` today;
  - the entries of at most one host-side **allowlist file** (`FILE` from the flag, else `CLAUDE_DOCKER_EGRESS_ALLOWLIST`);
  - the entries of repeatable **`--egress-allow=ENTRY`** flags.
- **Nothing is implied.** Credential flags don't add hosts. `--gh` and `--glab`, whose auth-proxy sidecar the agent can reach without going through squid, refuse to start unless the hosts that sidecar serves are on the list.
- **Entries** are an exact host or a `.suffix`, each with an optional `:port`.
- **Strict validation.** Every entry is validated before any container resource exists. Invalid hosts, single-label suffixes (`.com`), `*.` wildcards, IP addresses, provider hosts in any spelling, entries already covered by a listed suffix, and unreadable files are fatal. So is a file inside a mounted workspace, checked with symlinks resolved.
- **Unlisted names are never resolved.** A name-only deny comes before every rule that looks at a resolved address, so squid sends no DNS query for a host that isn't on the list.
- **DNS must be closed.** A probe container on the internal network checks that external names don't resolve before the agent starts; otherwise the session aborts.
- **Evidence.** Under the flag, `.meta` records the file's resolved path and the SHA-256 of the bytes parsed, and the generated `squid.conf` is saved next to the log.

## Capabilities

### New Capabilities

None. The allowlist is a mode of the existing egress proxy.

### Modified Capabilities

- `api-egress-policy`: adds the `--egress-allowlist` mode, its sources, validation, rule order, DNS check and evidence. The modified requirements change nothing for sessions without the flag.
- `cli-help`: documents `--egress-allowlist`, `--egress-allow` and `CLAUDE_DOCKER_EGRESS_ALLOWLIST`.

## Non-goals

- Changing what `--egress-lock`, `--api` or any session without `--egress-allowlist` allows.
- Hosts implied by credential flags, presets, IP and CIDR entries, and a fixed allowlist path that applies without being named. Follow-ups are listed in design.md.
- A per-repository allowlist file (`.claude-docker/allowed-hosts`, #100), which would need hash approval on the host.
- Subscription (OAuth) sessions, which need hosts the lock refuses by design.
- A data-loss control. Data can still leave through any allowed host (design, Risks).
- TLS interception, per-path rules, SSH and other non-HTTP protocols.

## Impact

- `run.sh`: flag parsing, entry parsing and validation, the `--gh` / `--glab` host check, squid config generation, DNS probe, `.meta` keys and `.conf`, help text.
- Tests: `tests/test_egress_policy.py`, `tests/bats/run.bats`, a DNS-leak harness, `smoke/egress.sh` (an allowlist cell).
- Docs: `docs/auth.md` (new "Egress allowlist" section, limitations), `docs/security.md` (threat model), `docs/usage.md`, `README.md` flag table.
