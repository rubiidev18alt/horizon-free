# Fork releases

`.github/workflows/fork-release.yml` builds Horizon from **this fork** and publishes the archives as a
GitHub release on `rubiidev18alt/horizon-free`. It exists because upstream's
[release flow](release-flow.md) cannot run here:

| Upstream needs | This fork has |
| --- | --- |
| Self-hosted runners (`ludovics-mac-mini`, `playhorizon`) holding the maintainer's own Minecraft, which is where the macOS builds, the exporter classes and the smoke test are made | Nothing self-hosted: every job runs on GitHub's `ubuntu-*`, `windows-*` and `macos-*` runners |
| `RELEASE_SIGNING_KEY` in the `release` environment, which every launcher checks a download against | No signing key, so the builds are checksummed but **not signed** |
| The batch train (`release.yml`, `release_train.py`, `release-blocker` issues) that decides what ships | One manual run: you pick the tag and the version |

Upstream's workflows are untouched: they still run (or wait for a runner) exactly as before.

## Running it

A tag is the trigger that needs no API token, and the one to prefer:

```sh
git fetch origin main && git tag fork-v1 origin/main && git push origin fork-v1
```

That builds `main`'s commit and publishes it as the release `fork-v1`. A tag of any commit that is
not on `main` is refused: a release has to name a commit on the default branch.

From the command line or the browser (**Actions → fork release → Run workflow**) a run can be
tuned instead:

```sh
gh workflow run "fork release"                              # build everything, publish nothing
gh workflow run "fork release" -f publish=true              # build and publish
gh workflow run "fork release" -f platforms=linux           # one platform, quicker
gh workflow run "fork release" -f publish=true -f tag=v26.3-free -f version=0.2.1
```

The inputs are:

| Input | Default | What |
| --- | --- | --- |
| `tag` | `fork-v<run number>` | The release's tag |
| `version` | `0.2.<run number>` | `HORIZON_VERSION` built into the programs (`--version`, user agents, crash reports); there are no version-bump commits |
| `publish` | off | Publish a release. A run off `main` builds and publishes nothing, because a release has to name a commit on the default branch |
| `platforms` | all | `linux`, `windows` or `macOS` alone, for a quicker run |
| `exporters` | empty | URL of an `original-exporters` artifact zip (see below) |

## What a run makes

Game folders, the same shape `xtask package` gives upstream's releases: `horizon-client`,
`horizon-ctl`, `horizon-prepare`, `horizon-server`, `PLAY.txt`, `research-lock.json`, the `tools/`
and `crates/horizon-prepare` sources the launcher compares, and the exporter bundle when there is one.

- `horizon-linux-x86_64.tar.gz` and `horizon-server-linux-x86_64.tar.gz` (the engine without audio or
  controllers, which `packaging/server/install.sh` installs)
- `horizon-windows-x64.zip` (the C runtime linked in)
- `horizon-macos-arm64.zip`
- `horizon-launcher-{linux-x86_64,windows-x64.exe,macos-arm64}`, the bare programs as `launcher.yml`
  publishes them and not archives: the launcher replaces the file it runs from when it updates itself
- `SHA256SUMS` for all of the above

A run takes about 25 minutes: the release profile is thin LTO with one codegen unit, so the Windows
build alone is roughly 18 minutes on a hosted runner.

Every build is checked the way upstream checks it: `tools/check_imports.py` (no library players may
not have), `--help` on each program, `--version` matching the release's version, the archive unpacked
again and inspected, and a server folder made from the engine. Once published, the release's own
files are downloaded back, `SHA256SUMS` is verified against them, each archive is unpacked and each
Linux launcher is run with `--version`.

A failing step keeps its log (`build.log`, `package.log`, `imports.log`, `unpack.log`, `server.log`)
and a `report` job posts the first errors of every platform that failed as a comment on the commit:
the job logs themselves need a GitHub sign-in, and this repository has issues disabled.

## What it does not do

- **Sign or notarize.** The macOS programs are ad hoc signed by the toolchain, so the first launch is
  right-click → Open. `SHA256SUMS` has no `SHA256SUMS.sig`: `crates/horizon-release-sign/release-keys.txt`
  holds upstream's key only, and the official launcher refuses a release it cannot verify. These
  builds are for this fork's users, not an official Horizon release.
- **Refuse a Linux launcher's extra libraries.** `tools/check_imports.py` warns there instead, as
  `launcher.yml` does: a program built on `ubuntu-22.04` can ask for `GLIBC_2.35` and the newer
  libstdc++ that distributions current today have.
- **Ship compiled exporter classes by default.** They are compiled from
  `mods/horizon/src/test/java` with `javac` against the pinned Minecraft jars, which only exist on a
  machine that has the game. Without them `horizon-prepare compile-exporters` builds them at
  preparation time, which needs a **JDK 25** rather than a Java runtime; with them, preparing needs
  only a runtime. To publish a bundle, run `horizon-prepare compile-exporters` where the game files
  are, zip `exporters/`, upload it as an artifact, and pass its artifact URL as `exporters`. (A URL
  of one of upstream's `original-exporters` artifacts works too.)
- **Smoke-test the game.** The offscreen run of a 1500-frame world needs the prepared resources,
  which need Minecraft. The checks stop at "every program starts, with the right version".
- **Publish a Nightly, or touch upstream's channel.** One manual release per run.

## Building locally

Exactly what the workflow runs, from a clean checkout:

```sh
sudo apt-get install -y libasound2-dev libudev-dev            # Linux only
export HORIZON_VERSION=0.2.1
cargo build --release --locked -p horizon-client --bin horizon-client --bin horizon-ctl \
  -p horizon-prepare --bin horizon-prepare -p horizon-server --bin horizon-server
cargo run --locked -p xtask -- package target/release/horizon-client dist/horizon-linux-x86_64
tar czf dist/horizon-linux-x86_64.tar.gz -C dist horizon-linux-x86_64
```

The profile is the release one from `Cargo.toml`: thin LTO, one codegen unit, so it compiles and
links slowly (tens of minutes on a hosted runner) and runs fast.

## The preparer

`crates/horizon-prepare` in this fork did not build: a `download_fallback` module called
`download_fallback(&lock, log)` with three arguments and wrote placeholder bytes (including
non-jar "jars") when a download failed, which would have failed every checksum anyway. It was
removed, and `prepare_impl` is upstream's again: the game files come from the player's own install
(`--prepare`, `--minecraft-dir`, `--server-jar`). Nothing downloads Minecraft for you.
