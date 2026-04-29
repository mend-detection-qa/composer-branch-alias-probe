# branch-alias

## Feature exercised

This probe exercises Composer's **branch-alias** mechanism: a `dev-<branch>` requirement
where the depended-on package's own `composer.json` maps the branch to a numbered alias
(`dev-main` -> `3.x-dev`) via its `extra.branch-alias` field.

## Branch-alias semantics

When a package declares:

```json
"extra": {
    "branch-alias": {
        "dev-main": "3.x-dev"
    }
}
```

it tells Composer that the `main` branch behaves like a `3.x` development line. This
lets other packages write version constraints like `^3.0` and have them satisfied by
`dev-main` when `minimum-stability` allows dev versions. The alias is purely advisory
metadata — it does not rename the version in the lockfile.

## What Composer records in the lockfile

Composer records `monolog/monolog` at version **`dev-main`** — the raw branch version,
not the alias `3.x-dev`. The alias mapping (`"dev-main": "3.x-dev"`) is stored inside
the package entry under `packages[].extra.branch-alias`, NOT in the top-level `aliases[]`
array. The top-level `aliases[]` array is reserved for *inline* root-level aliases such as
`"monolog/monolog": "dev-main as 3.x-dev"` in the root `require`.

Key values from the generated lockfile:
- `packages[0].version`: `"dev-main"`
- `packages[0].extra.branch-alias`: `{"dev-main": "3.x-dev"}`
- `aliases`: `[]` (empty — no inline root aliases used)

## Asserted version

The probe asserts **`dev-main`** as the canonical version because that is what the
lockfile `version` field contains. If Mend instead reports `3.x-dev`, that represents
alias-resolution behavior (reading from `extra.branch-alias`) — document whether this
occurs, but it is a behavioral difference rather than a clear defect.

## Expected dependency tree

- `monolog/monolog`
  - version: `dev-main`
  - scope: `require` (production)
  - source: `git` (resolved from VCS repository `https://github.com/Seldaek/monolog.git`)
  - commit: `68b974809baff3f071893de61447212e9e688ee7`
  - branch-alias metadata: `dev-main` -> `3.x-dev`
  - direct: `true`
  - children:
    - `psr/log` @ `3.0.2` — transitive, source: `registry` (Packagist)

Total packages: 2 (1 direct, 1 transitive)

## Failure modes exercised

1. **Alias ignored** — Mend reports the raw branch name `dev-main` and does not surface
   the `3.x-dev` alias at all (no `extra.branch-alias` parsed).
2. **Package not found** — Mend fails to detect `monolog/monolog` because it does not
   process `dev-` prefixed versions from VCS sources.
3. **Duplicate entries** — Mend creates two entries: one for `dev-main` and one for
   `3.x-dev`, treating the alias as a separate package version.
4. **Source misclassification** — Mend reports `monolog/monolog` as `registry` instead of
   `git`, ignoring the VCS repository declaration.
5. **Version swapped** — Mend uses the alias `3.x-dev` as the reported version when the
   lockfile `version` field says `dev-main`.

## Probe metadata

| Field             | Value                                                   |
|-------------------|---------------------------------------------------------|
| pattern           | branch-alias                                            |
| pm                | composer                                                |
| schema_version    | 1.0                                                     |
| minimum-stability | dev                                                     |
| php_min           | >=8.1                                                   |
| composer_min      | 2.8                                                     |
| monolog version   | dev-main (commit 68b974809baff3f071893de61447212e9e688ee7) |
| alias             | dev-main -> 3.x-dev                                     |
| psr/log version   | 3.0.2                                                   |
| target repo       | mend-detection-qa/composer-branch-alias-probe           |