## 1. Spec

- [x] 1.1 ADDED *The image is also scanned with the publishing pipeline's scanner and rules*
- [x] 1.2 `openspec validate grype-pr-gate --strict` passes

## 2. Workflows

- [x] 2.1 `ci.yml`: grype advisory pass (HIGH+, unfixed included, never fails) and gate pass (HIGH+ with a fix available, fails the job), both with `.grype.yaml`, on `claude-docker:ci`
- [x] 2.2 `image-scan.yml`: the same two grype passes
- [x] 2.3 Pin `anchore/scan-action` to the revision `mcvs-docker-action` uses

## 3. Docs

- [x] 3.1 `docs/security.md`: both scanners, both accepted-finding files, and why the expiry contract still holds
