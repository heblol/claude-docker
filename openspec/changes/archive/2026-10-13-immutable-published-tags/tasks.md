## 1. Spec

- [x] 1.1 MODIFIED *The image is published on a version tag*: a run on a tag already in the registry fails without publishing; the check fails closed; immutability is enforced; scenarios for a re-run or dispatch on a published tag, a registry error, and a dispatch on a tag with no image
- [x] 1.2 MODIFIED *Publication is gated on the lint and scan stages*: "a run that fails" → "a lint or scan stage that fails", since a run refused for an existing tag also fails but that tag has an image
- [x] 1.3 `openspec validate immutable-published-tags --strict` passes

## 2. Workflows

- [x] 2.1 `docker.yml`: on a version tag ref, check the registry for the tag before publishing; fail without publishing if it exists or if the check errors with anything other than not-found

## 3. Docs

- [x] 3.1 README GHCR section: the recovery dispatch is safe because the pipeline refuses a tag that is already published
