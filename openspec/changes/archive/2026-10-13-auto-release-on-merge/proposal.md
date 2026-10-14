## Why

Releasing means tagging by hand. `mcvs-general-action` v0.11.1 can derive the next
version from the Conventional Commits on `main` and tag it, but it tags with
`GITHUB_TOKEN`, and a tag created with that token starts no workflow. `docker.yml`'s
`tags: [v*]` trigger never fires for it, so the tag would get no image. The action's
docs recommend dispatching the image workflow on the new tag instead, and
`mcvs-docker-action` v0.14.0 pushes on a `workflow_dispatch` of a tag ref.

The image-distribution spec says the image is published "when, and only when, a
version tag is pushed", and that any other event publishes nothing. A dispatch on a
tag is another event, so the spec has to change with the code.

## What Changes

- `release.yml`: on every push to `main`, auto-release tags the next version and
  creates a GitHub Release, then dispatches `docker.yml` on the new tag.
- `docker.yml`: `mcvs-docker-action` v0.13.2 → v0.14.0, which also publishes on a
  dispatch of a tag ref.
- The spec publishes when the pipeline runs on a version tag ref, whether the tag was
  pushed or the pipeline was dispatched on it. A dispatch on a tag that already has
  an image republishes it, so it is reserved for a tag that has none.
- The README covers the second way a tag can have no image: the release created the
  tag but the dispatch failed, so there is no Docker run at all, and names the
  command that recovers it.

## Capabilities

- `image-distribution`: MODIFIED *The image is published on a version tag* (publish on
  a tag ref, pushed or dispatched; auto-release scenario; a branch dispatch does not
  publish) and *Publication is gated on the lint and scan stages* (the
  dispatch-failed case is documented with its recovery).

## Impact

- `.github/workflows/release.yml` (new), `.github/workflows/docker.yml`, `README.md`.
