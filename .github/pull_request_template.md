<!--
This template is deliberately one paragraph. If you are looking for the old
checklist, it moved to the monorepo along with the spec — see EDR-010 and
CONTRIBUTING.md.
-->

## Please close this PR

**This repository does not accept pull requests.** `openapi.json` here is a
published copy of `eng/apps/exchange/api/openapi.json` in `nexus-xyz/nexus`,
republished automatically after every production deploy — so an edit made here is
overwritten by the next publish, and would merge, pass CI, cut a release the SDKs
pin, and then silently disappear. If you are outside the Nexus team, please
[open an issue](https://github.com/nexus-xyz/nexus-exchange-api/issues/new/choose)
instead; the fix lands at the source and reaches you in the next release. If you
are on the Nexus team, make the change in the monorepo, in the same PR as the
implementation that serves it. A human-authored PR opened here is closed
automatically with a comment saying the same thing. Two exceptions, both by
label: `repo-maintenance` for a genuinely repo-local change (CI, docs, these
templates) and `spec-reconciliation` for correcting this repo ahead of the
monorepo during an incident — add the label when you open the PR
(`gh pr create --label repo-maintenance`) and it stays open.
