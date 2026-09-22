# Fork operations

Personal fork of [herdrdev/herdr](https://github.com/herdrdev/herdr) carrying a
small patch set, built and distributed through a private Homebrew tap.

This file lives only on the `local` branch. It is not upstream content.

## Branch layout

| Branch | Role |
| --- | --- |
| `master` | Pure mirror of `upstream/master`. Never commit here. |
| `local` | Personal patches plus the release workflow. Build and tag from here. |

`master` stays a mirror so `git diff master local` always answers "what have I
changed vs upstream?" exactly. If `git merge --ff-only upstream/master` ever
fails, something was committed to `master` by mistake — reset it:

```bash
git switch master && git reset --hard upstream/master
```

Remotes:

```bash
git remote add upstream https://github.com/herdrdev/herdr.git   # once
```

## Current patches

| Commit | Change |
| --- | --- |
| `f102c494` | `ci`: tag-triggered release workflow |
| `5d10ea1c` | `ci`: build macOS arm64 + Linux musl targets |
| `bf53976d` | `ci`: fix static link check for musl x86_64 |

Two patches were dropped when tracking upstream at v0.9.1:

- `b570f579` `feat(navigator)`: single-pane tabs collapse into one navigator row.
  Upstream moved the navigator to `src/client/shell/aggregate_navigation.rs`,
  and its row builder emits one row per pane with the tab name folded into the
  label. The behaviour is upstream's now.
- `84ce1090` `ci`: macOS artifact build uses Homebrew's patched zig. Upstream
  moved to zig 0.16.0, which links fine from a vanilla tarball.

## Tracking upstream

Merge, do not rebase. Release tags stay reachable in `local`'s history that way,
and each conflict is resolved once instead of resurfacing on every update.

```bash
git fetch upstream

git switch master
git merge --ff-only upstream/master
git push origin master

git switch local
git merge master          # resolve conflicts, then git merge --continue
just ci                   # lint + nextest + maintenance + integration assets
git push origin local
```

Merges only ever go `master` → `local`. Merging the other way destroys the
mirror.

After a large merge, also check:

- the zig version in `vendor/libghostty-vt/build.zig.zon` against what is
  installed locally and what `local-release.yml` installs;
- whether upstream renamed or added a workflow, since `gh workflow disable` is
  keyed on the workflow, not the filename.

## Shipping an update to this machine

`brew upgrade` on its own never picks up a new build. The tap's formula pins an
exact release tag, so nothing moves until that formula is edited and pushed. The
full chain, from a merged `local` to a running new binary:

```bash
# 1. tag — the Local release workflow builds the three artifacts (~6 min)
git tag v0.9.1-local.1 && git push origin v0.9.1-local.1
gh run watch --repo natsuki-engr/herdr \
  "$(gh run list --repo natsuki-engr/herdr --workflow local-release.yml \
       --limit 1 --json databaseId --jq '.[0].databaseId')"

# 2. read the url/sha256 pairs the workflow printed into the release notes
gh release view v0.9.1-local.1 --repo natsuki-engr/herdr

# 3. bump the formula: version + all three url/sha256 pairs
cd "$(brew --repo natsuki-engr/local)"
$EDITOR Formula/herdr.rb
git commit -am "herdr: 0.9.1-local.1" && git push

# 4. now brew has something to upgrade to
brew update && brew upgrade natsuki-engr/local/herdr

# 5. restart the server — a swapped binary does not replace the running one
herdr server stop     # open panes are lost
```

Step 5 is not optional. The running server holds the old inode, so
`herdr status server` keeps reporting `compatible: no` until it is stopped and
started again.

Skipping step 3 is the usual mistake: `brew update` refreshes Homebrew and the
tap's git contents, but the formula still names the previous tag, so
`brew upgrade` correctly reports nothing to do.

Details for each step are in the sections below.

## Cutting a release

Tag `v<upstream-version>-local.<n>` and push. Everything else is automated by
`.github/workflows/local-release.yml`.

```bash
git tag v0.9.1-local.1 && git push origin v0.9.1-local.1
```

The workflow builds three targets and publishes a GitHub release:

- `herdr-macos-aarch64`
- `herdr-linux-x86_64` (this is the WSL target)
- `herdr-linux-aarch64`

Intel Mac is deliberately excluded. The release notes and `SHA256SUMS.txt` both
carry the `url`/`sha256` pairs the formula needs.

The workflow is tag-triggered rather than `workflow_dispatch` because dispatch
requires the workflow file to exist on the default branch, and the default
branch is the untouched upstream mirror.

To rebuild a bad release, bump to the next `-local.<n>` rather than moving a
tag.

### Inherited upstream workflows are disabled

The fork inherits upstream's workflows, and several of them misfire here. They
are disabled with `gh workflow disable` (state `disabled_manually`, which
survives upstream merges):

| Workflow | Why |
| --- | --- |
| `release.yml` | Triggers on `push: tags: ['v*']`, so it matches `v*-local.*` and runs upstream's full publish pipeline on every release tag. |
| `website-deploy.yml`, `preview.yml` | Publish upstream docs and the preview channel. `website-deploy.yml` replaced `website.yml` in v0.9.1; a rename resets the disabled state, so re-check after a rename. |
| `distribution.yml` | Validates upstream docs/distribution contracts on every master mirror push; nothing here consumes it. |
| `nix.yml`, `windows-arm64.yml` | Fire on master mirror updates; nothing here consumes them. |
| `label-next-release-issues.yml` | Closes "released" issues on every master push. |
| `pr-gate.yml` | Auto-closes unsolicited PRs — on a fork it would close your own. |

Still active: `ci.yml`, `local-release.yml`, `build-artifacts-manual.yml`. Note
that `ci.yml` only covers `master` and pull requests, so it never validates the
patches on `local` — run `just check` locally before tagging.

Re-enable any of them with `gh workflow enable <file> --repo natsuki-engr/herdr`.

## Updating the tap

The tap is [natsuki-engr/homebrew-local](https://github.com/natsuki-engr/homebrew-local).
After a release, update `Formula/herdr.rb`: bump `version` and all three
`url`/`sha256` pairs, then push.

```bash
cd "$(brew --repo natsuki-engr/local)"
$EDITOR Formula/herdr.rb
git commit -am "herdr: 0.9.1-local.1" && git push
brew update && brew upgrade natsuki-engr/local/herdr
```

The tap's CI runs `brew style` only. The scaffolding `brew tap-new` generates
(test-bot, pr-pull, autobump, dependabot) was removed: it targets taps that
build and publish bottles from source, `brew readall` fails on the uncovered
Intel macOS combination, and autobump would rewrite the formula to upstream
herdr versions. Do not restore it. A wrong `sha256` is not caught by CI, but
`brew install` reports a checksum mismatch clearly.

## Installing on a new machine

```bash
brew tap natsuki-engr/local
brew install natsuki-engr/local/herdr
```

Remove any existing herdr first. The formula's `conflicts_with "herdr"` catches
the homebrew-core package but **cannot** see a binary placed by the official
`install.sh`:

```bash
herdr server stop            # running panes are lost
brew uninstall herdr         # if installed via homebrew-core
rm -f ~/.local/bin/herdr     # if installed via install.sh
which -a herdr               # confirm exactly one, under the brew prefix
```

`~/.config/herdr` and `~/.local/state/herdr` are compatible — keep them to carry
settings and sessions over.

### WSL

Needs Homebrew on Linux:

```bash
sudo apt update && sudo apt install -y build-essential procps curl file git
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
```

`brew services` will not work unless systemd is enabled in `/etc/wsl.conf`. Run
`herdr server` directly instead.

## Build toolchain

Upstream pins **zig 0.16.0** through `vendor/libghostty-vt/build.zig.zon`
(`minimum_zig_version`). Any vanilla 0.16.0 works, including mise:

```bash
mise use -g zig@0.16.0
```

This was not always true. zig **0.15.2** — the pin before v0.9.1 — could not
link against the macOS 26 SDK from a vanilla tarball (mise, asdf, the official
release, or `mlugg/setup-zig`): every libc symbol came up undefined, and even a
hello-world `zig cc` failed. `SDKROOT`, `ZIG_LIBC`, `xcode-select`, and wrapping
`ZIG` to inject `--sysroot` all failed to fix it, because zig compiles its build
runner before any of those apply. The only working macOS 0.15.2 was Homebrew's
patched `zig@0.15`. Both this file and `local-release.yml` carried workarounds
for it; 0.16.0 removed the need, and upstream's own workflows now install a
vanilla tarball on macOS too.

If `zig@0.15` is still installed via Homebrew it only shadows the mise version
in shells without mise activation, and the build then fails `requireZig`. Remove
it with `brew uninstall zig@0.15`.

Linux builds were never affected.

## Gotchas

- **`herdr update` no longer works.** Self-update follows the upstream release
  channel, which does not know about this fork. Use
  `brew upgrade natsuki-engr/local/herdr`.
- **Swapping the binary does not restart the server.** A running server keeps
  its old inode, so `herdr status server` may report `compatible: no` until you
  `herdr server stop` and start again. That loses open panes.
- **`cargo test --bins` is not a usable fallback.** It runs every test in one
  process, where tests that re-invoke the test binary die on `SIGPIPE`
  (`io error when listing tests: BrokenPipe`), and two tests that depend on
  process-global state fail (`generated_workspace_ids_are_short_base32_handles`,
  `manifest_action_invoke_injects_plugin_paths`). Use the real toolchain:

  ```bash
  mise use -g just@latest
  mise use -g cargo:cargo-nextest@latest   # builds from source, ~3 min
  ```

  `cargo-nextest` is not in mise's registry and its release assets do not match
  the `ubi`/`github` backend's filter, so the `cargo:` backend is the way in.
- **`just check` also runs `windows-lint`**, which needs the xwin Windows SDK
  (`just setup-windows-cross`). The fork does not ship a Windows artifact, so
  `just ci` is the gate that matters before tagging.
- **Do not open PRs upstream.** Unsolicited implementation PRs from accounts
  outside `.github/APPROVED_CONTRIBUTORS` are closed automatically. See
  `CONTRIBUTING.md`.
