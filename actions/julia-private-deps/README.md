# julia-private-deps

Lets GitHub Actions in a DerangedIons repository clone **private sibling repositories** during CI, so Julia `Pkg` can resolve `[sources]` entries (or `Pkg.add(url=...)`) that point at other private repos in the org.

## How it works

1. `actions/create-github-app-token` mints a one-hour installation token for the org-owned **DerangedIons CI Reader** GitHub App (repository permission *Contents: read* only, installed on **all repositories**, so new repos are covered automatically).
2. `git config url.<token-url>.insteadOf` rewrites every `https://github.com/DerangedIons/…` and `git@github.com:DerangedIons/…` clone to carry that token.
3. `JULIA_PKG_USE_CLI_GIT=true` is exported so Pkg shells out to `git` (its default LibGit2 backend ignores URL rewrites).

The token is masked in logs and revoked when the job ends. Nothing token-bearing is written into `~/.julia`, so `julia-actions/cache` stays clean.

## Usage

The org is on the GitHub Free plan, where organization secrets are not visible to private repositories, so the app's private key lives as a **repository secret** named `DERANGEDIONS_CI_APP_KEY` in each repo (`JuliaPackageTemplate.generate()` installs it for new DerangedIons packages). Because the `secrets` context is unavailable in a step-level `if`, surface its presence through the job `env` and gate the step on that:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      HAS_APP_KEY: ${{ secrets.DERANGEDIONS_CI_APP_KEY != '' }}
    steps:
      - uses: actions/checkout@v7
      - uses: julia-actions/setup-julia@v3
      - uses: julia-actions/cache@v3
      - name: Authenticate to private DerangedIons deps
        if: env.HAS_APP_KEY == 'true'
        uses: DerangedIons/.github/actions/julia-private-deps@main
        with:
          app-private-key: ${{ secrets.DERANGEDIONS_CI_APP_KEY }}
      - uses: julia-actions/julia-buildpkg@v1
      - uses: julia-actions/julia-runtest@v1
```

Put the step **before** anything that runs `Pkg` (buildpkg, runtest, docs instantiate). Repos without the secret, and pull requests from forks, skip the step.

## Adding the secret to a repo

```bash
gh secret set DERANGEDIONS_CI_APP_KEY -R DerangedIons/<Repo> < /path/to/derangedions-ci-reader.pem
```

The `.pem` lives in the team password manager; never commit it.

## Another organization

Install the same pattern elsewhere by creating that org's own read-only app and passing `owner:` and `app-id:` explicitly. Two orgs can be authenticated in one job by calling the action twice; the URL rewrites are prefix-scoped and coexist.

## Gotchas

- `[sources]` in `Project.toml` requires Julia ≥ 1.11. On 1.10 Pkg ignores it and fails with "expected package X to be registered" regardless of authentication.
- `[sources]` is not transitive: a package must list every unregistered dependency of its own dependencies too.
