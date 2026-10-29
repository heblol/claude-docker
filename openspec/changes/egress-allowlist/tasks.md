## 0. Verify squid and engine behaviour first (blocking)

Each result either confirms the design or changes it before any code is written. Keep each check as a permanent case in the 2.3 harness.

- [ ] 0.1 squid 7.6 warns about and drops one of two overlapping `dstdomain` values (D5 dedup).
- [ ] 0.2 squid normalises uppercase and trailing-dot request hosts before `dstdomain` matching (rule order scenarios).
- [ ] 0.3 An IPv4 `ANTHROPIC_BASE_URL` host matches `egress_allowed_names` as a literal under `dstdomain -n` (D6).
- [ ] 0.4 squid treats `167772165`, `10.5` and `0x7f000001` request hosts as addresses (D5 refusal).
- [ ] 0.5 `getent hosts` exit codes in the agent image: found 0, not found 2, and what a resolver timeout returns; `timeout` gives 124 (D7).

## 1. run.sh

- [ ] 1.1 Parse `--egress-allowlist[=FILE]`: exit 1 without `--egress-lock` and when neither the flag value nor `CLAUDE_DOCKER_EGRESS_ALLOWLIST` names a file; the space-form hint (D2).
- [ ] 1.2 Help text for `--egress-allowlist` and `CLAUDE_DOCKER_EGRESS_ALLOWLIST`; fix the `--egress-lock` line's spec wording (`--api` or `--api-direct`).
- [ ] 1.3 In `validate_opts`, read the file per D4: `~/` expansion, bounded `readlink` loop plus `cd -P`/`pwd -P`, workspace containment on a `/` boundary, regular-file and control-byte checks, one copy into the stage dir, hash with `sha256sum` or `shasum -a 256`.
- [ ] 1.4 Parse and validate entries per D5; report every invalid line with file and line.
- [ ] 1.5 The `--gh` / `--glab` host check and the upstream-override refusal (D3).
- [ ] 1.6 `gen_egress_squid_conf`: allowlist rules per D6, with `egress_allow_<n>` port pairs and dedup of `egress_allowed_names`; output without the flag unchanged.
- [ ] 1.7 DNS probes per D7, between `create_egress_networks` and `start_egress_sidecar`; add the probe to the EXIT trap.
- [ ] 1.8 Evidence per D8: write the `.conf` to the evidence dir at generation time; add the `.meta` keys in `egress_save_log` under the flag only.
- [ ] 1.9 Update the in-code design notes that this change reverses: the `gen_egress_squid_conf` header (new inputs and their validator, the allowlist order) and the `--egress-lock` comment in `validate_opts` ("everything else stays open").

## 2. Tests

Traceability: every scenario in the spec delta maps to one row below; tier in brackets.

- [ ] 2.1 `tests/test_egress_policy.py` [exit code and stderr only], with a helper that takes explicit workspaces and a temp `HOME`: flag prerequisites, no file named, space-form hint, missing file, directory, control bytes, every containment case (inside, symlinked workspace, symlink into workspace, `--ro` workspace, sibling), every invalid-entry case with its message and `:line`, several invalid lines, addresses, provider spellings and non-provider labels, the `--gh` / `--glab` host check with only missing hosts named, the upstream overrides, env var ignored without the flag. Each WHEN uses a full valid command line, with a control case that reaches the `CLAUDE_DOCKER_RUNTIME` sentinel.
- [ ] 2.2 `tests/bats/run.bats` [generated config and saved files]: source selection and precedence, `~` expansion and no fallback file (asserting `egress_allowed_names`); grammar, normalisation, dedup and boundary values; rule order parsed into an ordered list of `http_access` lines with relative-position assertions, and every `dstdomain` using `-n`; byte-identical config and `.meta` keys without the flag, against golden fixtures generated from the pre-change commit; `--api-direct`; evidence keys, `.conf` mode 600 and survival of a failed log copy; `allowlist_sha256` against an external `sha256sum` of CRLF and no-trailing-newline fixtures; DNS probe order and every abort path via a fake-engine shim; help output lists the new flag and env var and says `--egress-allowlist` needs `--egress-lock`.
- [ ] 2.3 `tests/egress-proxy/` harness [real squid from the proxy image, dnsmasq stub resolver with fixed answers and `--log-queries`]: table-driven cases for every allow/refuse scenario (listed and unlisted, plain HTTP, sidecar port, IP literals, on-prem name to `10.20.1.2`, name to `169.254.169.254`, `localhost:8080`, provider spellings, entry and suffix ports, endpoint port), plus the no-query cases. A positive control (a listed name must produce a logged query) and a barrier (a final listed lookup that must appear in the log before the log is read) keep the no-query assertions from passing vacuously; A, AAAA and PTR queries are all checked. Includes the 0.x cases.
- [ ] 2.4 `tests/smoke/egress.sh` allowlist cell [full session]: listed host allowed, unlisted refused, `git clone` under `--gh` with the GitHub hosts listed, `--tfe` and `--gh-direct` refused without entries. Unit cases for the new cell's argument parsing and assertions in `tests/bats/egress.bats`.
- [ ] 2.5 `.github/workflows/ci.yml`: steps for the allowlist smoke cell and the 2.3 harness.

## 3. Docs

- [ ] 3.1 `docs/auth.md`: new "Egress allowlist" section: the flag and env var, the file format, nothing implied (with the hosts each credential flag needs and the `--gh` / `--glab` check), `.suffix` only for zones the organisation controls, keeping the file and env files outside every workspace and not running on `$HOME`, building a first list from an `--egress-lock` log after reviewing every host; limitations from design Risks.
- [ ] 3.2 `docs/security.md`: threat model and "if a session is compromised" for allowlist sessions; not a DLP control.
- [ ] 3.3 `docs/usage.md` flag reference.
- [ ] 3.4 `docs/vision-and-scope.md`: an Options entry for restricted egress (no child-image route exists: the boundary is the internal network and sidecar, outside the agent's `CapBnd 0xc5`).

## 4. Validation

- [ ] 4.1 `shellcheck`, `bats tests/bats`, `python3 -m unittest discover -s tests -p 'test_*.py'` pass.
- [ ] 4.2 `openspec validate egress-allowlist --strict` passes.
- [ ] 4.3 CI green, including the allowlist cell and the 2.3 harness.

## 5. Archive

- [ ] 5.1 Archive the change.
- [ ] 5.2 Update the `api-egress-policy` Purpose, which still says other hosts stay open. A Purpose edit can't ride in a delta; this follows the precedent in `archive/2026-08-20-add-go-toolchain/tasks.md` 3.5, and the PR's OpenSpec section records it. Decide there whether to scope the DNS sentence of "The network is the boundary" (design, Open Questions).
