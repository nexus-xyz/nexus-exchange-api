# Contributing guide — nexus-exchange-api

The OpenAPI specification for the Nexus Exchange API — the contract-first source
of truth that the SDKs regenerate from and the backend vendors.

## Merging

- Don't merge a PR without an approving review — CI passing isn't a substitute.
- Don't merge a PR you didn't author without an approving review **and** the
  author's sign-off. Check the author first
  (`gh pr view <n> --json author,reviewDecision`).
- Re-approval isn't needed for follow-up commits to an already-approved PR.

## Pull requests

- One concern per PR; link its tracking issue (`ENG-XXXX`) in the title.
- Respond to review comments before merging.

## Spec discipline

- This spec is consumed downstream by released tag. Don't remove or rename an
  operation without a version bump — downstream SDKs and the backend pin a tag
  and regenerate against it.
- Changes cut a release via release-please; let it manage the version and tag.
