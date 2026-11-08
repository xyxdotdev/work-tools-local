# Work Tools

[![CI](https://github.com/brxyxn/work-tools-local/actions/workflows/ci.yml/badge.svg)](https://github.com/brxyxn/work-tools-local/actions/workflows/ci.yml)

A local-first macOS desktop application for Text Diff, JSON visualization,
Base64-to-PDF conversion, and Mermaid diagram editing. This repository is the
Wails migration target for `work-tools-web`.

## Install a prebuilt release

Download the latest `work-tools-local-<version>-macos-arm64.dmg` from the
[Releases](https://github.com/brxyxn/work-tools-local/releases) page, open it,
and drag **work-tools-local** into **Applications**.

Release builds are ad-hoc signed but **not** notarized, so Gatekeeper blocks
the first launch:

1. Open the app once and dismiss the warning.
2. Open **System Settings › Privacy & Security**, scroll to **Security**, and
   click **Open Anyway** next to the work-tools-local message.
3. Confirm with **Open**. Later launches work normally.

Alternatively, remove the quarantine attribute from a terminal:

```bash
xattr -dr com.apple.quarantine /Applications/work-tools-local.app
```

Each release also ships a `SHA256SUMS.txt`. To verify a download, compare the
output of the following command with the matching line in that file:

```bash
shasum -a 256 work-tools-local-*-macos-arm64.dmg
```

## Build from source

### Requirements

- macOS 13 or newer on Apple Silicon, with the Xcode Command Line Tools
  (`xcode-select --install`)
- Go 1.26.3
- Wails CLI 3.0.0-beta.9
- Node.js 22.21.x
- pnpm 11.8.0

Xcode 26 or newer on macOS 26 is optional. With it, the build regenerates the
Icon Composer asset (`build/darwin/Assets.car`); without it, the committed
asset is used and `icons.icns` is generated from `build/appicon.png`.

Install the Wails CLI at the pinned version, and make sure `$(go env GOPATH)/bin`
is on your `PATH`:

```bash
go install github.com/wailsapp/wails/v3/cmd/wails3@v3.0.0-beta.9
```

Install the pinned pnpm through Corepack (it reads `packageManager` from
`frontend/package.json`):

```bash
corepack enable
```

Check that the toolchain is complete:

```bash
wails3 doctor
```

### Develop

Run the app with hot reload:

```bash
wails3 dev
```

Run the frontend checks from `frontend/`:

```bash
pnpm install --frozen-lockfile
```

```bash
pnpm lint && pnpm typecheck && pnpm test
```

Run the Go checks from the repository root. `main.go` embeds
`frontend/dist`, so build once first (`wails3 build`) to create it:

```bash
go vet ./... && go test -race ./...
```

### Build and package

All outputs go to `bin/`, which is git-ignored.

| Command | Output |
| --- | --- |
| `wails3 build` | Production arm64 binary at `bin/work-tools-local` |
| `wails3 package` | Ad-hoc signed bundle at `bin/work-tools-local.app` |
| `wails3 task darwin:package:dmg` | Styled disk image at `bin/work-tools-local.dmg` |
| `wails3 task release` | Everything a GitHub release ships, in `bin/release/` |

`wails3 task release` produces the same files as the release workflow:

```text
bin/release/
├── work-tools-local-v0.1.0-macos-arm64.app.zip
├── work-tools-local-v0.1.0-macos-arm64.dmg
└── SHA256SUMS.txt
```

The version defaults to the `CFBundleShortVersionString` in
`build/darwin/Info.plist`. Override it with `VERSION`:

```bash
wails3 task release VERSION=v0.1.0-local
```

Open the packaged app directly with:

```bash
open bin/work-tools-local.app
```

## Continuous integration

`.github/workflows/ci.yml` runs on every pull request and push to `master`:

- **Frontend checks** (Ubuntu): locked `pnpm install`, lint, typecheck, Vitest,
  and a production Vite build.
- **Go checks and arm64 build** (macOS 26, Apple Silicon): gofmt, `wails3 package`,
  a check that `go.mod`, `go.sum`, and `frontend/bindings` are committed
  up to date, `go vet`, and `go test -race`. The packaged `.app` is uploaded as
  a zipped workflow artifact for seven days.

## Releasing

`.github/workflows/release.yml` builds and publishes the unsigned arm64
release.

1. Bump the version in `build/config.yml` and in `CFBundleVersion` /
   `CFBundleShortVersionString` in `build/darwin/Info.plist` and
   `build/darwin/Info.dev.plist`. (`wails3 task common:update:build-assets`
   regenerates these from `build/config.yml`, but overwrites any manual
   changes to the build assets.) Commit and merge to `master`.
2. Tag the merged commit and push the tag:

   ```bash
   git tag v0.1.0
   ```

   ```bash
   git push origin v0.1.0
   ```

3. The workflow checks that the tag matches the `Info.plist` version, runs
   `wails3 task release`, runs the frontend and Go tests, and creates a GitHub
   release with the `.dmg`, the zipped `.app`, and `SHA256SUMS.txt`. Tags with
   a suffix such as `v0.2.0-rc.1` are published as pre-releases.

Running the workflow manually from the Actions tab (`workflow_dispatch`)
builds the same artifacts, named `v<version>-<short-sha>`, and uploads them
to the workflow run without creating a release.

Code signing with a Developer ID and notarization are not part of the
automated release yet. To do them locally, run `wails3 setup` once, then
`wails3 task darwin:sign:notarize`.

## Design documents

The complete architecture, pinned technology stack, feature parity rules, and
stacked-PR sequence are documented in
`docs/superpowers/specs/2026-08-16-work-tools-local-migration-design.md` and
`docs/superpowers/plans/2026-08-16-work-tools-local-migration.md`.
