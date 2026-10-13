## 1. Spec

- [x] 1.1 MODIFIED *The image is published on a version tag*: publish on a tag ref, pushed or dispatched; scenarios for the auto-release dispatch and a branch dispatch
- [x] 1.2 MODIFIED *Publication is gated on the lint and scan stages*: the tag-with-no-run case is documented with its recovery
- [x] 1.3 `openspec validate auto-release-on-merge --strict` passes

## 2. Workflows

- [x] 2.1 `release.yml`: auto-release on push to `main`, dispatch `docker.yml` on the released tag
- [x] 2.2 `docker.yml`: `mcvs-docker-action` v0.14.0, header explains the dispatched tag

## 3. Docs

- [x] 3.1 README GHCR section: a tag with no Docker run means the dispatch failed; recover with `gh workflow run docker.yml --ref <tag>`, only for a tag with no image
