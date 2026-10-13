## Why

The image-distribution spec contradicts itself. It says a dispatch on a tag that
already has an image "republishes that tag with a newly built artifact", and in the
same requirement promises that "a given reference always denotes the same artifact"
and that a pinned tag is the same "on every pull". The build is not reproducible: the
apt layers are not pinned, so a rebuilt `:vX.Y.Z` gets a different digest. Reserving
the dispatch for tags with no image is a convention, and nothing stops a re-run or a
mistaken dispatch from replacing a published tag.

## What Changes

- The pipeline checks the registry before publishing on a version tag ref, whatever
  started the run (push, re-run or dispatch). If the tag already has an image there,
  the run fails and publishes nothing.
- The check fails closed: an error other than "not found" counts as the tag being
  present, so the run fails rather than risk replacing it.
- Immutability of a published tag is enforced by the pipeline, not left to
  convention. A dispatch on a tag only ever fills in a missing image, so the
  recovery command is safe to run.
- The README stops warning that a dispatch replaces a published tag, and says the
  pipeline refuses one instead.

## Capabilities

- `image-distribution`: MODIFIED *The image is published on a version tag* (a run on
  an already-published tag fails without publishing; the registry check fails closed;
  immutability is enforced; scenarios for a re-run or dispatch on a published tag, a
  registry error, and a dispatch that recovers a tag with no image).
  MODIFIED *Publication is gated on the lint and scan stages*: a tag whose lint or
  scan stage fails publishes nothing. It used to say whose "run fails", which a run
  refused for an already-published tag would contradict.

## Impact

- `.github/workflows/docker.yml` (the registry check, implemented separately),
  `README.md`.
