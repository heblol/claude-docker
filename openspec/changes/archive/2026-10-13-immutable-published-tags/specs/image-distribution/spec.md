## MODIFIED Requirements

### Requirement: The image is published on a version tag

The pipeline SHALL publish the built image to a container registry when, and only when, it
runs on a version tag ref: either because the version tag was pushed, or because the pipeline
was dispatched on that tag. A push to a branch, a dispatch on a branch, or any other event,
SHALL build and scan the image without publishing it.

A version tag created by the automated release on a merge to the default branch SHALL lead to
a published image without manual steps. Such a tag is created with a token whose events start
no workflow, so the release SHALL dispatch the pipeline on the tag it created.

A run on a version tag whose reference already exists in the registry SHALL NOT publish, and
SHALL fail, whatever started it: a push of the tag, a re-run, or a dispatch. The pipeline SHALL
check the registry for the reference before publishing, and SHALL treat any outcome of that
check other than a confirmed absence as the reference being present: a registry error fails
the run rather than letting it publish. A dispatch on a tag therefore only ever fills in a
missing image, and the project's documentation SHALL say so.

The published reference SHALL be immutable: the version tag SHALL be published verbatim, and
the pipeline SHALL NOT publish a moving tag such as `latest` alongside it. A consumer
therefore names a full version tag to pull, and a given reference always denotes the same
artifact. This is guaranteed by the pipeline's refusal to publish over an existing reference,
not by convention: the build is not reproducible, since its distribution packages are not
pinned, so a rebuild of the same tag would yield a different artifact.

A version tag carrying a pre-release suffix SHALL publish on the same terms as any other:
under its own verbatim tag, through the same lint and scan gate, and without creating or
moving a stable tag. Pre-releases exist so that a candidate can be pulled and verified
against the registry before a release is committed to, and removing that path would cost
more than the ambiguity it buys back.

A published pre-release SHALL NOT carry a support promise, and the project's documentation
SHALL say so rather than leave it to be inferred from the tag's spelling. A consumer who
encounters a defect on a pre-release is directed to the latest stable tag; the fix for such
a defect is the next release, not a change published against the pre-release itself.

#### Scenario: a version tag is pushed

- **GIVEN** the build, lint and scan stages all pass
- **AND** the registry has no image under that tag
- **WHEN** a version tag is pushed
- **THEN** the image is published to the registry under that tag

#### Scenario: a merge to the default branch cuts a release

- **GIVEN** a merge to the default branch whose commits call for a new version
- **WHEN** the automated release creates the version tag
- **THEN** it dispatches the pipeline on that tag
- **AND** given the build, lint and scan stages all pass, the image is published to the
  registry under that tag

#### Scenario: a pre-release tag is pushed

- **GIVEN** the build, lint and scan stages all pass
- **AND** the registry has no image under that tag
- **WHEN** a version tag carrying a pre-release suffix is pushed
- **THEN** the image is published to the registry under that tag verbatim
- **AND** no stable tag is created or moved by that publication

#### Scenario: a branch push does not publish

- **GIVEN** a push to the default branch
- **WHEN** the pipeline runs
- **THEN** the image is built, linted and scanned
- **AND** nothing is published to the registry

#### Scenario: a dispatch on a branch does not publish

- **GIVEN** the pipeline is dispatched on a branch
- **WHEN** it runs
- **THEN** the image is built, linted and scanned
- **AND** nothing is published to the registry

#### Scenario: a run on an already-published tag is refused

- **GIVEN** an image is already published to the registry under a version tag
- **WHEN** the pipeline runs on that tag again, whether by a re-run or a dispatch
- **THEN** nothing is published to the registry
- **AND** the run fails
- **AND** the tag still denotes the artifact it was first published with

#### Scenario: a registry error during the check is refused

- **GIVEN** the pipeline runs on a version tag
- **AND** the registry check returns an error other than the reference not being found
- **WHEN** the pipeline reaches publication
- **THEN** nothing is published to the registry
- **AND** the run fails

#### Scenario: a dispatch on a tag with no image publishes it

- **GIVEN** a version tag with no image in the registry
- **AND** the build, lint and scan stages all pass
- **WHEN** the pipeline is dispatched on that tag
- **THEN** the image is published to the registry under that tag

#### Scenario: no moving tag is published

- **WHEN** a version tag is published
- **THEN** no `latest` tag and no truncated variant of the version is published beside it
- **AND** a consumer pinning the full version tag receives the same artifact on every pull

#### Scenario: the pre-release policy is documented

- **GIVEN** a pre-release published to the registry and visible to anyone browsing it
- **WHEN** a consumer reads the project's documentation for the distribution channel
- **THEN** it states that pre-release tags are published there
- **AND** it states that they are unsupported, and names the latest stable tag as what to
  use instead

### Requirement: Publication is gated on the lint and scan stages

The image SHALL be linted and scanned for known vulnerabilities before it is published, and a
failure at any of those stages SHALL prevent publication. A version tag whose lint or scan stage
fails therefore results in no published image.

Because a failed run is indistinguishable from an unstarted one at the registry, this
outcome — a tag that exists in the repository with no corresponding published image — SHALL be
documented rather than left to be discovered. The documentation SHALL cover both causes: a
pipeline run on the tag that failed, and an automated release whose dispatch failed, so that
no pipeline run exists for the tag; and SHALL name how to start the pipeline on the tag in the
second case.

#### Scenario: a scan failure blocks the release

- **GIVEN** a version tag is pushed
- **AND** the vulnerability scan reports a finding above the threshold that is not recorded as
  accepted
- **WHEN** the pipeline runs
- **THEN** the image is not published
- **AND** the run fails

#### Scenario: the failure mode is documented

- **WHEN** a maintainer reads the project's documentation for the distribution channel
- **THEN** it states that a failing scan on a version tag publishes nothing

#### Scenario: a release whose dispatch failed is recoverable

- **GIVEN** the automated release created a version tag
- **AND** its dispatch of the pipeline failed, so no pipeline run exists for the tag
- **WHEN** a maintainer reads the project's documentation for the distribution channel
- **THEN** it states that a tag with no pipeline run means the dispatch failed
- **AND** it names the command that dispatches the pipeline on that tag
