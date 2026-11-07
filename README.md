# npm transitive OSV remediation — repro

Minimal reproduction for [renovatebot/renovate#47832](https://github.com/renovatebot/renovate/pull/47832) — surface `package-lock.json` transitive deps for security updates (closes [#46379](https://github.com/renovatebot/renovate/issues/46379)).

## Scenario

- `package.json` declares four direct dependencies, none of which has a known advisory: `chokidar`, `make-dir`, `node-abi` and `normalize-package-data`.
- `package-lock.json` (`lockfileVersion: 3`) carries vulnerable versions of two packages which are **not** declared — they exist only in the lock file:
  - `braces` 3.0.2 (via `chokidar`, which allows `~3.0.2`) — [GHSA-grv7-fg5c-xmjg](https://github.com/advisories/GHSA-grv7-fg5c-xmjg), fixed in 3.0.3.
  - `semver`, locked three times: 7.5.1 (via `node-abi`), 6.3.0 (via `make-dir`) and 5.7.1 (via `normalize-package-data`) — [GHSA-c2qf-rxjj-qqgw](https://github.com/advisories/GHSA-c2qf-rxjj-qqgw), fixed in 7.5.2, 6.3.1 and 5.7.2.
- Every fix is within the range its parent declares, so each can be remediated by changing `package-lock.json` alone.

Without the PR, Renovate's npm manager only extracts declared dependencies, so neither advisory is acted on.

`renovate.json`:

```json
{
  "extends": ["security:only-security-updates"],
  "enabledManagers": ["npm"]
}
```

## Reproduce

```bash
# from a checkout of the PR branch
LOG_LEVEL=debug node lib/renovate.ts --platform=github --token="$GITHUB_TOKEN" khvn26/renovate-npm-transitive-osv-repro
```

## Result

Running the PR branch (`9f31341`) against this repo produced two security PRs, each changing `package-lock.json` only:

- **[#1 — Update dependency braces to v3.0.3 [SECURITY]](https://github.com/khvn26/renovate-npm-transitive-osv-repro/pull/1)** — `braces` 3.0.2 → 3.0.3.
- **[#2 — Update dependency semver to v5.7.2 [SECURITY]](https://github.com/khvn26/renovate-npm-transitive-osv-repro/pull/2)** — all three copies fixed: 5.7.1 → 5.7.3, 6.3.0 → 6.3.2, 7.5.1 → 7.8.6.

```
DEBUG: Vulnerability GHSA-grv7-fg5c-xmjg affects braces 3.0.2
DEBUG: Vulnerability GHSA-c2qf-rxjj-qqgw affects semver 5.7.1
DEBUG: Dependency: braces, is only in a lock file, but a vulnerability alert matches it - looking it up
DEBUG: Dependency: semver, is only in a lock file, but a vulnerability alert matches it - looking it up
DEBUG: 2 flattened updates found: braces, semver
DEBUG: Executing command npm update --package-lock-only --no-audit --ignore-scripts braces
DEBUG: Executing command npm update --package-lock-only --no-audit --ignore-scripts semver
```

On each branch, `npm ci` succeeds and `npm audit` no longer reports the remediated package.

Known gaps, visible here:

- `npm update` cannot take a version, so each copy moves to the newest release its parent allows, not the lowest fixed one, and the PR title of #2 names only the lowest locked copy (5.7.1 → 5.7.2).
- The Dependency Dashboard lists the transitive deps as `Updates: undefined`.
