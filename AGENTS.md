# Contributing guide — nexus-exchange-api

The OpenAPI specification for the Nexus Exchange API — the contract the SDKs
regenerate from.

## This repository does not accept pull requests

`openapi.json` is **generated**. Its source of truth is the Nexus monorepo, at
`eng/apps/exchange/api/openapi.json`; this repo *receives* it as a bot PR after
every production deploy, and release-please cuts the tag the SDKs pin. That
direction reversed in ENG-5886 (EDR-010).

**To change the API:** edit `eng/apps/exchange/api/openapi.json` in
`nexus-xyz/nexus`, in the same PR as the implementation that serves it. It
publishes here automatically. Do not open a PR here to change the spec, and do
not offer one — a human-authored PR is closed automatically by the
`Publish-only mirror` workflow (ENG-10966), because an edit here is overwritten by
the next publish and reaches users without passing the implementation, the tests
or review in the monorepo.

Two labels are exempt, and either must be applied deliberately by a maintainer:

- `repo-maintenance` — the PR is repo-local (CI, docs, templates). Those have no
  other home. Apply it when you open the PR: `gh pr create --label repo-maintenance`.
- `spec-reconciliation` — the generated spec has to be corrected here ahead of the
  monorepo (an incident, or undoing a bad publish). Nothing detects the divergence
  for you, so land the matching monorepo change too or the next publish silently
  overwrites it.

## Merging

- Don't merge a PR without an approving review — CI passing isn't a substitute.
- Don't merge a PR you didn't author without an approving review **and** the
  author's sign-off. Check the author first
  (`gh pr view <n> --json author,reviewDecision`).
- Re-approval isn't needed for follow-up commits to an already-approved PR.

## Pull requests

- One concern per PR; link its tracking issue (`ENG-XXXX`) in the title.
- Respond to review comments before merging.
- Before opening a PR here at all, check that it belongs here. A spec change does
  not. A CI, docs or template change does, and needs the `repo-maintenance` label.

## Spec discipline

- This spec is consumed downstream by released tag. Don't remove or rename an
  operation without a version bump — downstream SDKs and the backend pin a tag
  and regenerate against it. Removals are breaking; make them deliberately, in
  the monorepo.
- Changes cut a release via release-please; let it manage the version and tag.
