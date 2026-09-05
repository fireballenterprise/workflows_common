# Fireball Enterprise — Common Workflows

Shared GitHub Actions primitives — one composite action plus a set of reusable workflows — for
the family's CI. Consumed by [`workflows_shopify`](https://github.com/fireballenterprise/workflows_shopify),
[`workflows_web`](https://github.com/fireballenterprise/workflows_web), and the AI/Python repos
(`fireball_sidecar_toolkit`, `fireball_orchestrator`, `fireball_ai_vault`, `ai_vault`).

## Versioning
Tags use a `v` prefix: `vMAJOR.MINOR.PATCH` (e.g. `v1.0.0`), dual-tagged with a floating major
(`v1`) force-moved to the newest `v1.x.x`. Callers reference `@v1`; pin an exact tag only when
reproducibility matters more than picking up non-breaking updates. Breaking changes bump the
major. Cutting a release: bump `VERSION` in a PR — merging it to `main` fires `publish_release.yml`,
which tags and publishes the GitHub Release (re-run safe).

## Auth model
Every reusable workflow that pushes a branch or tag resolves its token the same way:
`${{ steps.bot.outputs.token || github.token }}`, where the bot step runs only
`if: ${{ vars.BOT_APP_ID != '' }}`.

- Repos with branch protection (`workflows_shopify`, `fireball_sidecar_toolkit`) set the repo
  variable `BOT_APP_ID` + secret `BOT_PRIVATE_KEY` → pushes go through the org-wide
  `fireball-actions-bot` App (a ruleset bypass actor).
- Repos with no protection (the landing sites) set neither → the default `GITHUB_TOKEN` is used.

Callers pass `secrets: inherit` so `BOT_PRIVATE_KEY` is visible where it exists. Automated commits
are authored `Levon Becker <LevonBecker@users.noreply.github.com>` — never `github-actions[bot]`.

## Composite action
`uses: fireballenterprise/workflows_common/actions/bump_version@v1`

| input | | does | output |
|---|---|---|---|
| `part` | `patch` \| `minor` \| `major` | bump the `VERSION` file (plain `X.Y.Z`); no commit | `version` |

## Reusable workflows
`uses: fireballenterprise/workflows_common/.github/workflows/<name>.yml@v1`

| workflow | inputs | does |
|---|---|---|
| `resolve_version.yml` | `bump` (none\|patch\|minor\|major), `branch` (development), `avoid_tag_collision` (true), `extra_git_paths` | milestone-bump + commit/push the working branch, or read as-is; outputs `version` |
| `promote.yml` | `source` (development), `target` (main), `message` | `git merge source --no-ff -X theirs` onto `target`, push |
| `github_release.yml` | `version` (req), `ref` (main), `tag` (false), `v_prefix` (false) | optionally tag (exact + floating major), then `gh release create --generate-notes` |
| `python_tests.yml` | `python_version` (3.14), `checks` (req, JSON array) | matrix CI: `uv sync` then one `invoke` task per `checks` entry `{name, invoke, node?, pre?}` |

`actionlint.yml` and `publish_release.yml` are **not** reusable — they self-test / self-release
this repo.
