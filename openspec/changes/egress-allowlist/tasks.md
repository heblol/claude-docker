## 1. run.sh

- [ ] 1.1 Parse `--egress-allowlist[=FILE]` and repeatable `--egress-allow=ENTRY`; exit 1 for either without its prerequisite and for an empty `--egress-allow=`; when `--egress-allowlist` has no value and a workspace argument is a regular file, suggest `--egress-allowlist=FILE`. Help text for both and `CLAUDE_DOCKER_EGRESS_ALLOWLIST`.
- [ ] 1.2 `validate_opts`: pick the file (flag, else env var; `~/` expanded; nothing else read), refuse one that isn't a readable regular file, read it once into memory and hash those bytes, refuse it inside any workspace using `cd -P`/`pwd -P` on both sides with a `/` boundary.
- [ ] 1.3 Parse and validate every entry (D5): host rules, two-label suffixes, `*.` hint, ports, IPv4 refused, provider hosts from `EGRESS_MODEL_PROVIDERS`, trailing comments and CR, duplicates dropped, covered entries refused; errors name the source and line.
- [ ] 1.4 Under `--gh` / `--glab`, refuse to start unless the sidecar's hosts are covered by an entry (D3).
- [ ] 1.5 `gen_egress_squid_conf`: under the allowlist, the name gate (rule 3), provider deny moved up, per-entry port ACL pairs, `egress_allowed_names`, `deny all` (D6). Without it, output byte-identical to today.
- [ ] 1.6 DNS probe (D7) after the egress network exists and before the agent starts; fail closed.
- [ ] 1.7 `egress_save_log`: `allowlist_file` / `allowlist_sha256` in `.meta` and `<start>-<id>.conf`, under the allowlist only (D8).

## 2. Tests

- [ ] 2.1 `tests/test_egress_policy.py`: flag dependencies, file selection, `~` expansion, no fallback file, the env var ignored without the flag, every containment case, every entry rule and error message with line numbers, provider spellings, covered entries, the `--gh` / `--glab` host check.
- [ ] 2.2 `tests/bats/run.bats`: generated config under the allowlist (rule order, name gate above every `dst` line, provider deny above the port pairs, per-entry ports), byte-identical config and unchanged `.meta` keys without it, `.meta` and `.conf` contents, DNS-probe abort paths. Run `squid -k parse` from the proxy image on the generated configs.
- [ ] 2.3 DNS-leak harness: run the proxy image with the generated config plus `dns_nameservers` pointed at a logging stub resolver (dnsmasq `--log-queries`); assert 403 and no logged query for unlisted names, including `1.2.3.4.<label>.example` and uppercase forms.
- [ ] 2.4 `smoke/egress.sh` allowlist cell: listed host allowed, unlisted refused (CONNECT and plain HTTP), IP literal refused, listed host on its own port, `git clone` under `--gh` with the GitHub hosts listed.

## 3. Verification

- [ ] 3.1 Confirm squid 7.6 drops overlapping `dstdomain` entries with a warning, and normalises uppercase and trailing-dot hosts before ACL matching.
- [ ] 3.2 Confirm an IPv4 `ANTHROPIC_BASE_URL` host matches `egress_allowed_names` as a literal under `dstdomain -n`.

## 4. Docs

- [ ] 4.1 `docs/auth.md`: new "Egress allowlist" section: sources, format, nothing implied (list the hosts each credential flag needs, and the `--gh` / `--glab` check), keeping the file and env files outside every workspace and not running on `$HOME`, building a first list from an `--egress-lock` log after reviewing every host; limitations (Risks in design.md).
- [ ] 4.2 `docs/security.md`: threat model and "if a session is compromised" updated for allowlist sessions; not a DLP control.
- [ ] 4.3 `docs/usage.md`, `README.md` flag table.

## 5. Validation and archive

- [ ] 5.1 `shellcheck`, `bats tests/bats`, `python3 -m unittest discover -s tests -p 'test_*.py'` pass.
- [ ] 5.2 `openspec validate egress-allowlist --strict` passes.
- [ ] 5.3 CI egress cells green, including the allowlist cell and the DNS-leak harness.
- [ ] 5.4 After archive, update the `api-egress-policy` Purpose, which still says other hosts stay open.
