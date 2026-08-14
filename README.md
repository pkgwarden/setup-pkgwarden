# setup-pkgwarden

Install the [pkgwarden](https://pkgwarden.com) CLI on a GitHub Actions runner and point
`npm`, `pnpm`, `yarn` (classic), `uv`, and `pip` at the pkgwarden gate. Packages your
policy blocks never get downloaded, so a malicious version cannot run install scripts on
your runner in the first place.

## Usage

```yaml
- uses: pkgwarden/setup-pkgwarden@v1
  with:
    token: ${{ secrets.PKGWARDEN_CI_TOKEN }}

- run: npm ci
- run: uv sync
```

No flags on the install commands: the action writes the registry credentials and exports
the environment the package managers already read.

Run it **after** `actions/setup-node` / `actions/setup-python` and **before** the first
install step. It reads the lockfiles in the repository root by default; in a monorepo, set
the `working-directory` input (GitHub Actions does not allow the `working-directory`
keyword on a `uses:` step, which is why it is an input here):

```yaml
- uses: pkgwarden/setup-pkgwarden@v1
  with:
    token: ${{ secrets.PKGWARDEN_CI_TOKEN }}
    working-directory: services/api
```

The action **fails the job** if it finds no supported lockfile in that directory, or if the
only thing it finds is an ecosystem it cannot configure (a poetry-only or Yarn Berry-only
project). That is deliberate: silently continuing would install straight from the public
registry with no gate in front of it.

### Pin to a full commit SHA

```yaml
- uses: pkgwarden/setup-pkgwarden@0123456789abcdef0123456789abcdef01234567 # v1.0.0
```

A tag is a movable pointer. An attacker who gets write access to this repository can
re-point `v1` (or even a released `v1.0.0`) at new code, and every workflow that
references the tag runs it on the next build with your secrets in scope. A full 40-character
SHA is immutable, so the code you reviewed is the code that runs. This is the same reasoning
that makes the gate worth having; we would rather you applied it to us too.

### Organization rollout

Store the token once as an **organization secret** (Settings → Secrets and variables →
Actions → New organization secret), scoped to the repositories that should install through
the gate. Individual workflows then reference `${{ secrets.PKGWARDEN_CI_TOKEN }}` with no
per-repository setup. Use a token of type **ci**: it is scoped to package resolution and
cannot approve exceptions or change policy.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `token` | yes | (none) | pkgwarden gate API token. Use a `ci` token from a secret; never inline it. |
| `api-url` | no | `https://index.pkgwarden.com/api/v1` | Base URL of the gate API. Change only for a self-hosted deployment. |
| `version` | no | `0.2.2` | pkgwarden CLI version; resolves to release tag `pw-v<version>`. |
| `working-directory` | no | `.` | Directory holding the lockfiles to configure for. |

The action verifies the downloaded binary against the release's `SHA256SUMS` and fails the
job if it does not match.

## What gets configured

| Package manager | How |
|---|---|
| `npm`, `pnpm` | An `.npmrc` in `$RUNNER_TEMP` (0600) plus `NPM_CONFIG_USERCONFIG` |
| `yarn` (classic, v1) | The same `.npmrc`, plus `YARN_NPM_REGISTRY_SERVER` |
| `uv` | A `.netrc` in `$RUNNER_TEMP` (0600) plus `NETRC` and `UV_DEFAULT_INDEX` |
| `pip` | The same `.netrc` plus `PIP_INDEX_URL` |

`UV_DEFAULT_INDEX` (not `UV_INDEX`) is used deliberately: `UV_INDEX` only *adds* an index
and leaves PyPI as a fallback, which would let a blocked package resolve anyway.

## Not supported

- **Windows runners.** There is no Windows build of the pkgwarden CLI. The action fails
  with a clear message on any OS other than Linux or macOS.
- **Yarn Berry (v2+).** When a `.yarnrc.yml` is present the CLI skips yarn and says so.
  Configure it yourself:

  ```yaml
  # .yarnrc.yml
  npmRegistryServer: "https://index.pkgwarden.com/resolution/npm/"
  npmAlwaysAuth: true
  npmAuthToken: "${PKGWARDEN_CI_TOKEN}"
  ```

- **Poetry.** Declare the gate as a source in `pyproject.toml` and authenticate with
  `poetry config http-basic.<source> "$PKGWARDEN_CI_TOKEN" x`.

A repository that is Python-only or npm-only is fine: the ecosystem with no lockfile is
simply skipped. The job only fails when *nothing* configurable is found (see above).

## Versioning

Semver tags (`v1.2.3`) are immutable and never re-pointed. The major tag `v1` moves to the
newest `v1.x.y` release, following the GitHub Actions convention. Pin a SHA if you want
neither to move under you.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
