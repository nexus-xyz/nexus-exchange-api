# Contributing guide — nexus-exchange-api

The OpenAPI specification for the Nexus Exchange API — the contract the SDKs
regenerate from.

## `openapi.json` is generated — do not edit it here

Its source of truth is the Nexus monorepo, at
`eng/apps/exchange/api/openapi.json`. This repository *receives* it: the monorepo
publishes the contract a production deploy is actually serving, then
release-please cuts the tag the SDKs pin.

That direction reversed in ENG-5886 (EDR-010). It used to be the other way round
— this repo was canonical and the monorepo vendored a released tag — which meant
the spec was published and externally visible while the implementation was still
in review, and shipping one route took four steps across two repos. It also meant
the published contract could document operations nothing served: five
`/v1/bridge` operations sat here for four weeks with no implementation behind
them.

**To change the API:** edit `eng/apps/exchange/api/openapi.json` in
`nexus-xyz/nexus`, in the same PR as the implementation, and bump `info.version`
there by the rule in that repo's `eng/apps/exchange/api/README.md`. It publishes
here automatically on the next production deploy.

## This repository does not accept pull requests

Do not open a PR here to change the spec, and do not offer one — a human-authored
PR is closed automatically by the `Publish-only mirror` workflow (ENG-10966),
because an edit here is overwritten by the next publish and reaches users without
ever passing the implementation, the tests or review in the monorepo.

A PR that edits `openapi.json` also turns the `Spec Source of Truth` check red
unless it is the publish bot's or release-please's — the check requires a bot
author, not just the branch name, so naming a branch after the bot does not get
you past it. **That check is visible, not a gate:** `main` has no required status
checks, so a red guard is a signal to the CODEOWNER reviewing the PR rather than
something that stops the merge. The auto-close is what actually closes the write
path, and it is the *only* thing that closes the external one — forking cannot be
disabled on a public repository, so there is no setting that stops a fork PR from
being opened.

Two labels are exempt, and either must be applied deliberately by a maintainer,
so an outside contributor cannot self-exempt:

- `repo-maintenance` — the PR is repo-local (CI, docs, templates). Those have no
  other home. Apply it when you open the PR: `gh pr create --label repo-maintenance`.
- `spec-reconciliation` — the generated spec has to be corrected here ahead of the
  monorepo (an incident, or undoing a bad publish). Land the matching change in
  the monorepo too, because nothing detects the divergence for you: the monorepo's
  `not-behind-public` check was removed in ENG-10517 and ENG-10531's canary does
  not exist yet, so an unreconciled correction is silently overwritten by the next
  publish.

## Merging

- Don't merge a PR without an approving review — CI passing isn't a substitute.
- Don't merge a PR you didn't author without an approving review **and** the
  author's sign-off. Check the author first
  (`gh pr view <n> --json author,reviewDecision`).
- Re-approval isn't needed for follow-up commits to an already-approved PR.
- The publish PR from the monorepo bot still needs human review: it is the
  external contract, and nothing auto-merges it.

## Pull requests

- One concern per PR; link its tracking issue (`ENG-XXXX`) in the title.
- Respond to review comments before merging.
- Before opening a PR here at all, check that it belongs here. A spec change does
  not. A CI, docs or template change does, and needs the `repo-maintenance` label.

## Spec discipline

- This spec is consumed downstream by released tag. Don't remove or rename an
  operation without a version bump — downstream SDKs pin a tag and regenerate
  against it. Removals are breaking; make them deliberately, in the monorepo.
- `info.version` is owned by the **monorepo**, not by this repo (ENG-11154). It
  moves in the monorepo PR that changes the contract, by the bump rule in
  `eng/apps/exchange/api/README.md`, together with every layer that announces it
  — work from `.github/scripts/api-version-pins.json` there, never from a list in
  prose, this one included.
- In *this* repo, release-please is the only actor that should write
  `info.version`, and it writes the number the publish commit's `Release-As:`
  footer names rather than deriving one. Don't hand-edit the version or the tag.
