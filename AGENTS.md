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
`nexus-xyz/nexus`, in the same PR as the implementation. It publishes here
automatically on the next production deploy.

A PR that edits `openapi.json` on any other branch fails the `Spec Source of
Truth` check. An edit that slipped through would be silently reverted by the next
publish, so the check is the thing that keeps it visible. If this repo genuinely
has to be corrected first — an incident, or undoing a bad publish — add the
`spec-reconciliation` label and land the matching change in the monorepo.

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

## Spec discipline

- This spec is consumed downstream by released tag. Don't remove or rename an
  operation without a version bump — downstream SDKs pin a tag and regenerate
  against it. Removals are breaking; make them deliberately, in the monorepo.
- Changes cut a release via release-please; let it manage the version and tag.
  Nothing else should write `info.version`, here or in the monorepo.
