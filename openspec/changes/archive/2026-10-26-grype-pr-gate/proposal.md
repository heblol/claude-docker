## Why

The publishing pipeline (`docker.yml`) scans the image with grype and its
accepted-finding rules in `.grype.yaml`. The pull-request and scheduled scans
use Trivy with `.trivyignore`. The two accept findings differently: Trivy by
identifier across the whole image, grype by package, version and location. So
a change can pass every pull-request check and still fail the publishing scan.

#206 did exactly that. The pin refresh moved npm to 12.2.1, whose bundled
undici no longer matched a version-scoped `.grype.yaml` rule. The pull request
was green, because `.trivyignore` accepts the same CVE by ID. The first red
was the run on the default branch, and the v0.3.11-v0.3.13 tag runs that
should have published those releases failed in the same way.

Running the publishing pipeline on pull requests would catch this, but at the
cost of a second, two-architecture build per pull request. The pull-request
pipeline already builds and loads the image, so it can run grype on that
image at no extra build cost.

## What Changes

- `ci.yml`'s image scan also runs grype, with `.grype.yaml`, on the image it
  already built, in two passes like Trivy's. The first reports every HIGH and
  CRITICAL finding. The second fails the job on those with a fix available.
- The weekly scan (`image-scan.yml`) runs the same two grype passes, so it
  keeps reporting what the next pull request will be blocked by.
- grype is pinned to the same scan-action revision as the publishing pipeline,
  so the pull-request and publishing scans run the same scanner.
- The scan documentation names both scanners, says a finding must be accepted
  in `.trivyignore` and, where grype reports it, `.grype.yaml`, and explains
  why the expiry contract still holds.

## Capabilities

- `image-vulnerability-scan`: ADDED *The image is also scanned with the
  publishing pipeline's scanner and rules*.

## Impact

- `.github/workflows/ci.yml`, `.github/workflows/image-scan.yml`,
  `docs/security.md`.
- A pull request whose image has a fixable HIGH or CRITICAL finding that
  `.grype.yaml` does not accept now fails `Docker build (validate, no push)`,
  a required check. That is the same policy as Trivy's gate, applied with the
  rules the publishing pipeline uses.
- No extra image build: about a minute of scanning on top of the existing job.
