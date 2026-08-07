# tfenv

tfenv is a version manager for [Terraform](https://www.terraform.io/), inspired by tools
like rbenv and pyenv. It lets you install and switch between multiple versions of Terraform.

## Installation

```sh
git clone https://github.com/<org>/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bash_profile
```

Make sure `tfenv` and shims are available on your `PATH`.

## Usage

```sh
tfenv install [<version>]     # Install a specific version, or the latest/required version
tfenv use [<version>]         # Switch the active version
tfenv uninstall <version>     # Uninstall a version
tfenv list                    # List installed versions
tfenv list-remote             # List installable versions
tfenv pin                     # Write the current active version to .terraform-version
```

`<version>` can be a specific version (e.g. `1.7.5`), `latest`, `latest:<regex>`,
`min-required` (reads `required_version` from your `*.tf` files), or `latest-allowed`.

## Configuration

Key environment variables:

- `TFENV_ROOT` — install location for tfenv itself (default: parent of `bin/`).
- `TFENV_CONFIG_DIR` — where versions and the `version` file are stored (default: `TFENV_ROOT`).
- `TFENV_REMOTE` — base URL to download Terraform releases from (default: `https://releases.hashicorp.com`).
- `TFENV_ARCH` — override architecture detection (e.g. `amd64`, `arm64`).
- `TFENV_CURL_OUTPUT` — `0` (silent), `1` (default curl output), `2` (progress bar, default).
- `TFENV_NETRC_PATH` — path to a `.netrc` file used for authenticated downloads.
- `TFENV_DEBUG` — set to `1`+ for debug logging, `3`+ for full shell tracing.
- `TFENV_BASHLOG` — `1` to always load the full logging library, `0` to always use the lightweight logger.

## JFrog Artifactory support

tfenv can download Terraform releases through a JFrog Artifactory mirror instead of
`releases.hashicorp.com`.

- `JFROG_URL` — base URL of your Artifactory instance. When set (and `TFENV_REMOTE`
  is not already set), `TFENV_REMOTE` is automatically derived as
  `${JFROG_URL}/artifactory/${JFROG_REPO:-terraform}`.
- `JFROG_REPO` — Artifactory repository name to use with `JFROG_URL` (default: `terraform`).
- `JFROG_TOKEN` — if set, sent as a `Bearer` token on every download request.
- `JFROG_USER` / `JFROG_PASS` — if `JFROG_TOKEN` is not set and both are set, used for
  HTTP basic auth on every download request.

None of these are required — with none set, tfenv downloads from HashiCorp's public
release server exactly as before, with no authentication added.

When downloading via `JFROG_URL`, OpenPGP signature verification of the SHASUMS file is
skipped by default (Artifactory Xray scanning is assumed to run at the source), unless a
`use-gnupg` or `use-gpgv` file is present in `TFENV_CONFIG_DIR`, in which case local
verification takes precedence as usual.

## License

See [LICENSE](LICENSE).
