# Shadewright Labs Homebrew tap

Homebrew formulae for the command line tools of [Shadewright Labs](https://github.com/shadewright-labs), for macOS and Linux (amd64 and arm64). The tap holds formulae only; the code of each product stays in its own repository.

## Products

| Product | Formula | Status |
| --- | --- | --- |
| Shadewright | `kubectl-shadewright` (kubectl plugin) | Available. Installs from the signed release archives of [shadewright-labs/shadewright](https://github.com/shadewright-labs/shadewright). |
| Sablecourt | none | Not available yet. Sablecourt is a private preview with no release, so there is no formula and `brew install shadewright-labs/tap/sablecourt` fails with "No available formula". This table changes when the first release is published. |

## Install

```bash
brew tap shadewright-labs/tap
brew install kubectl-shadewright
kubectl shadewright version
```

Or in one step: `brew install shadewright-labs/tap/kubectl-shadewright`. kubectl finds the plugin because the binary is named `kubectl-shadewright` and sits on your `PATH`.

If Homebrew refuses a formula from this tap as untrusted, run `brew trust shadewright-labs/tap` once and retry.

### Moving from the old tap

The formula used to live in `pkalin/tap`. If you installed it from there, switch once:

```bash
brew uninstall kubectl-shadewright
brew untap pkalin/tap
brew tap shadewright-labs/tap
brew install kubectl-shadewright
```

## How releases are verified

The formula downloads the archive for your OS and architecture and Homebrew checks its **sha256**, which the formula pins. The digests come from the `checksums.txt` of the release. That catches a corrupted or swapped download. It does not catch a compromised release, because the same workflow publishes the checksum file and the archive.

For more, verify the release yourself. Each release carries `checksums.txt` and its Sigstore bundle `checksums.txt.sigstore.json`, signed by the `release-plugin` workflow of the Shadewright repository:

```bash
VERSION=0.6.0
gh release download "v${VERSION}" --repo shadewright-labs/shadewright \
  --pattern checksums.txt --pattern checksums.txt.sigstore.json
cosign verify-blob checksums.txt \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity-regexp '^https://github\.com/shadewright-labs/shadewright/\.github/workflows/release-plugin\.yml@refs/tags/v[0-9].*$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Releases up to v0.6.0 were signed before the project moved to the `shadewright-labs` organisation: write `pkalin` instead of `shadewright-labs` in the identity. The full procedure, including `gh attestation verify` for the archive, is on the [kubectl plugin page](https://github.com/shadewright-labs/shadewright/blob/main/docs/how-to/check-status.md) of the Shadewright docs. The formula itself verifies only the sha256.

## How the formulae are kept current

`tools.json` lists every tool: name, GitHub repository, tag pattern (only matching, non-pre-release, non-draft releases count), archive name template, binary, test arguments, description, homepage and license. `scripts/bump.py` (Python 3, standard library only) reads the latest release through the public GitHub API, downloads `checksums.txt` as text (nothing downloaded is executed), validates versions and digests strictly and renders `Formula/<name>.rb` from `templates/formula.rb.tmpl`. If the latest release has no matching assets, it writes nothing. Never edit a formula by hand: change `tools.json` or the template.

The `Bump formulas` workflow runs daily and on demand with `GITHUB_TOKEN` only (no secret, no access to other repositories). It needs the permissions `actions: write`, `contents: write` and `pull-requests: write`, and the repository setting "Allow GitHub Actions to create and approve pull requests" turned on. It opens or updates one pull request per formula on the branch `bump/<name>`.

Pull requests created with `GITHUB_TOKEN` do not trigger other workflows, so the bump job starts `Check formulas` and `PR title` on its branch itself (workflow_dispatch) and assigns the pull request to the repository owner. Merge when the checks pass: on macOS and Linux runners `Check formulas` taps the repository, runs `brew audit --strict --online`, installs from the formula and runs `brew test`.

## Adding a product

1. The product's repository publishes release archives named by a stable template and a checksums file.
2. Add an entry to `tools.json` (copy `kubectl-shadewright`).
3. Run `python3 scripts/bump.py --tool <name>` or wait for the daily workflow.
4. Open a pull request (`feat(formula): add <name>`) and update the product table above.

## Development

```bash
python3 -I -m unittest discover -s tests   # offline tests with fixtures
actionlint && zizmor .github/workflows     # workflow lint
```

See [AGENTS.md](AGENTS.md) for the rules. Licensed under Apache-2.0 (see [LICENSE](LICENSE)).
