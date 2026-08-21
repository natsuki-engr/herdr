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
| `b570f579` | `feat(navigator)`: single-pane tabs collapse into one navigator row |
| `84ce1090` | `ci`: macOS artifact build uses Homebrew's patched zig |
| `f102c494` | `ci`: tag-triggered release workflow |
| `5d10ea1c` | `ci`: build macOS arm64 + Linux musl targets |
| `bf53976d` | `ci`: fix static link check for musl x86_64 |

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
just check                # or: cargo fmt -- --check && cargo test --bins
git push origin local
```

Merges only ever go `master` → `local`. Merging the other way destroys the
mirror.

## Cutting a release

Tag `v<upstream-version>-local.<n>` and push. Everything else is automated by
`.github/workflows/local-release.yml`.

```bash
git tag v0.8.3-local.1 && git push origin v0.8.3-local.1
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

## Updating the tap

The tap is [natsuki-engr/homebrew-local](https://github.com/natsuki-engr/homebrew-local).
After a release, update `Formula/herdr.rb`: bump `version` and all three
`url`/`sha256` pairs, then push.

```bash
cd "$(brew --repo natsuki-engr/local)"
$EDITOR Formula/herdr.rb
git commit -am "herdr: 0.8.3-local.1" && git push
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

**macOS requires Homebrew's zig, not a vanilla one.**

```bash
brew install zig@0.15
```

Vanilla zig 0.15.2 — from mise, asdf, the official tarball, or the
`mlugg/setup-zig` action — cannot link against the macOS 26 SDK. Every libc
symbol comes up undefined, and even a hello-world `zig cc` fails. Upstream's
`ci.yml` calls the Homebrew build "patched Zig" for this reason.

`SDKROOT`, `ZIG_LIBC`, `xcode-select`, and wrapping `ZIG` to inject `--sysroot`
all fail to fix it: zig compiles its build runner before any of those apply.
zig 0.16 links correctly but `vendor/libghostty-vt` pins 0.15.2 via
`requireZig`.

Homebrew symlinks it into the brew prefix, so plain `cargo build` works with no
PATH setup.

Linux builds are unaffected and use `mlugg/setup-zig` normally.

## Gotchas

- **`herdr update` no longer works.** Self-update follows the upstream release
  channel, which does not know about this fork. Use
  `brew upgrade natsuki-engr/local/herdr`.
- **Swapping the binary does not restart the server.** A running server keeps
  its old inode, so `herdr status server` may report `compatible: no` until you
  `herdr server stop` and start again. That loses open panes.
- **`just check` needs cargo-nextest** (`brew install cargo-nextest`). Plain
  `cargo test --bins` runs every test in one process, which trips two tests that
  depend on process-global state (`generated_workspace_ids_are_short_base32_handles`,
  `manifest_action_invoke_injects_plugin_paths`). Both pass in isolation.
- **Do not open PRs upstream.** Unsolicited implementation PRs from accounts
  outside `.github/APPROVED_CONTRIBUTORS` are closed automatically. See
  `CONTRIBUTING.md`.
