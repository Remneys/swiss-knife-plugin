# Prepare the local iPhone executor

Run this preparation yourself when handling a local iPhone build. The pinned
release runs on Apple Silicon Macs, including a terminal running under Rosetta.
Intel Macs and other operating systems are refused before downloading anything.
Full Xcode tools are still required for building; Bun, npm, a GitHub login and
access to the product's source repository are not required to install the executor.

Run the following block with **bash**, as one command. It installs into the
current user's directory, verifies both a cached file and a fresh download, and
prints the absolute executable path. It does not change shell profiles, `PATH`,
system binaries or build workers. Keep the resulting path for the rest of this
task; a later shell does not inherit variables from this block.
`SWISS_KNIFE_EXECUTOR_ROOT` can select another private cache directory.

```bash
set -euo pipefail
umask 077
if [ "$(uname -s)" != Darwin ]; then
  echo "Local iPhone development requires macOS." >&2
  exit 2
fi
if [ "$(uname -m)" != arm64 ] && [ "$(sysctl -n sysctl.proc_translated 2>/dev/null || true)" != 1 ]; then
  echo "This executor release requires an Apple Silicon Mac; no files were installed." >&2
  exit 2
fi
swiss_executor_sha=4b243fcec5fb9e1feb72e57e8450a754bf73ee3a2bfb1834ec7d53141a20a8a8
swiss_executor_dir="${SWISS_KNIFE_EXECUTOR_ROOT:-${HOME:?}/.local/share/swiss-knife/executors}/25413ba57698"
swiss_executor_bin="$swiss_executor_dir/swiss-build"
mkdir -p "$swiss_executor_dir"
swiss_executor_matches() {
  [ -f "$1" ] && [ ! -L "$1" ] &&
    [ "$(shasum -a 256 "$1" | cut -d ' ' -f 1)" = "$swiss_executor_sha" ]
}
if ! swiss_executor_matches "$swiss_executor_bin"; then
  swiss_executor_tmp="$(mktemp "$swiss_executor_dir/.download.XXXXXX")"
  trap 'rm -f "$swiss_executor_tmp"' EXIT
  curl --fail --silent --show-error --location \
    --proto '=https' --proto-redir '=https' --tlsv1.2 \
    --connect-timeout 10 --max-time 180 \
    'https://github.com/remneys/swiss-knife-plugin/releases/download/swiss-build-25413ba57698/swiss-build-darwin-arm64' \
    --output "$swiss_executor_tmp"
  if ! swiss_executor_matches "$swiss_executor_tmp"; then
    echo "Executor SHA-256 mismatch. Nothing was installed or executed." >&2
    exit 1
  fi
  chmod 700 "$swiss_executor_tmp"
  mv -f "$swiss_executor_tmp" "$swiss_executor_bin"
  trap - EXIT
fi
chmod 700 "$swiss_executor_bin"
printf '%s\n' "$swiss_executor_bin"
```

Use the returned absolute path to run `--cli-version` and `dev ios --help`.
The version must be `0.1.0+25413ba57698` and the help must include `--device`,
`--identity`, `--no-install`, `--prepare-signing` and `--signing-bundle`. Then continue the local development workflow.
This release reads existing macOS Keychain keys in both PEM and hexadecimal form.
An executor update preserves the existing signing key; do not delete or replace
it to recover from an older executor's `Invalid PEM formatted message` error.
For example, invoke `"/absolute/path/from/preparation" --json dev ios ...`;
do not fall back to a different executable found on `PATH`.

A failed download, checksum mismatch or unsupported machine is a concrete
installation blocker. Report its output; do not substitute an unverified URL
or request credentials. This release is public and needs no GitHub token.
Installation does not grant server permissions. Continue with the connected
plugin's `ios_development_prepare` workflow in SKILL.md; a member with write
access can prepare their own device when the app's signing setup is enabled.
An Apple login or per-device browser approval is not part of that flow.
