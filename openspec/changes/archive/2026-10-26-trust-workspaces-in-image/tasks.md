## 1. Spec

- [x] 1.1 MODIFIED *Windows hosts get a working in-container identity and git*: drop the `safe.directory` bullet
- [x] 1.2 ADDED *In-container git trusts only the mounted workspaces*
- [x] 1.3 `openspec validate trust-workspaces-in-image --strict` passes

## 2. Image and wrapper

- [x] 2.1 Dockerfile: `git config --system safe.directory '/workspaces/*'` next to `git lfs install --system`
- [x] 2.2 `run.sh` `detect_msys`: stop passing `GIT_CONFIG_*`

## 3. Docs

- [x] 3.1 `docs/windows.md`: the `safe.directory` note points at the image setting
- [x] 3.2 `docs/usage.md` § *File ownership*: git trusts `/workspaces/*` on every host

## 4. Verification

- [x] 4.1 BATS and unittest: Windows and Linux both pass no `GIT_CONFIG_*`
- [x] 4.2 Smoke: system scope lists only `/workspaces/*`; a forced owner mismatch is accepted under `/workspaces` (also after a `GIT_CONFIG_*` export) and refused under `/tmp` (written; runs in CI smoke — no container engine locally)
- [x] 4.3 shellcheck, `bats tests/bats`, unittest suite pass locally; hadolint runs in CI (not installed locally)
