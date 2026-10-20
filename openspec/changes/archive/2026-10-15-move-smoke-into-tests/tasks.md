## 1. Spec

- [x] 1.1 MODIFIED the go-toolchain requirement whose scenario names `smoke/smoke.sh`
- [x] 1.2 `openspec validate move-smoke-into-tests --strict` passes

## 2. Move

- [x] 2.1 `git mv smoke tests/smoke`; `smoke.sh` resolves `run.sh` at `../..`
- [x] 2.2 `ci.yml`, `CONTRIBUTING.md`, `docs/maintenance.md`, comments
- [x] 2.3 `git mv tests/gh-proxy-integration.sh tests/integration/`; it resolves `run.sh` at `../..`

## 3. Verification

- [x] 3.1 `tests/smoke/smoke.sh` gets past its `run.sh` / `assert-in-container.sh` path checks (stops at "docker not found" locally)
- [x] 3.2 bats, unittest, shellcheck pass
- [x] 3.3 CI's smoke matrix passes from the new path (run 39268738629)
