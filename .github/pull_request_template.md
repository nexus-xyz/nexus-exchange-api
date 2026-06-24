<!--
Thanks for contributing to the Nexus Exchange API spec!
Keep PRs focused; open separate PRs for unrelated changes.
See CONTRIBUTING.md for the full workflow.
-->

## Summary

<!-- What does this PR change in openapi.json, and why? Link the motivating issue. -->

Closes #

## Changes

<!-- Bullet the notable spec changes. Name new/removed/renamed operations and fields. -->

-

## Version impact

<!-- release-please derives the bump from your conventional-commit title. Check one. -->

- [ ] **Additive / fix** — new optional field, new endpoint, or compatible
      fix. Use a `feat:` or `fix:` title (patch bump).
- [ ] **Breaking** — removed endpoint, renamed field, new required parameter,
      or type change. Use a `feat!:` title (or a `BREAKING CHANGE:` footer) for
      a minor bump, **and** add the `breaking-change` label so the API diff
      check unblocks.

## Checklist

- [ ] PR title is a conventional commit (`feat:` / `fix:` / `feat!:` / `docs:`).
- [ ] `npx -y @redocly/cli@2 lint openapi.json` passes locally (the Spec Lint check).
- [ ] Reviewed the `oasdiff` classification (the API diff check) and the version
      impact above is correct.
- [ ] `CHANGELOG.md` and `$.info.version` left untouched — release-please owns them.
- [ ] Considered the downstream SDK ripple (`-rs` / `-py` / `-mcp` pin to spec
      releases); breaking changes were minimized and batched.

## Notes for reviewers

<!-- Anything reviewers should focus on, risks, or coordination needed. -->
