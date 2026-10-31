# Go implementation notes

Use these notes for native Go commands, adapting module paths and build tooling to the repository.

## Entrypoint and version

Keep a shared updater package called from each real `main`, before argument processing or business work. Avoid putting network checks in `init` or a testable `run` function: importing packages or running unit tests should not start update discovery.

Export a string such as `update.Version = "dev"`. Set the release tag with linker flags:

```bash
go build -trimpath \
  -ldflags="-s -w -X ${module_path}/internal/update.Version=${release_tag}" \
  -o "$output_binary" "./cmd/$command"
```

The variables above come from the repository's release script; do not hardcode a past project's module path or version. Ensure official Release builds inject the version; a default `dev` binary deliberately skips updates.

Use `context.WithTimeout` for the discovery budget and `http.NewRequestWithContext` for every request. Share that context with `exec.CommandContext` when reusing `gh auth token`. Account for subprocess pipe-wait behavior: `exec.Cmd.WaitDelay` can bound pipe cleanup, and its time must fit the total budget too. Cancel the discovery context when it finishes; use a separate context after consent for downloads.

The HTTP body must be read under the same deadline. Read at most `limit+1` bytes and reject oversized metadata, archives, and decompressed executables. Propagate read errors instead of accepting partial binaries.

## GitHub assets and archive contract

For a GitHub deployment, API paths normally take this form:

```text
GET https://api.github.com/repos/{owner}/{repo}/releases/latest
Accept: application/vnd.github+json

GET https://api.github.com/repos/{owner}/{repo}/releases/assets/{asset_id}
Accept: application/octet-stream
```

Read the asset API URL from release metadata and restrict credential-bearing initial requests to the configured GitHub API origin. GitHub Enterprise needs its configured API host. Use the standard client's credential-stripping redirect behavior or an explicit equivalent; reject HTTP downgrades. Check the current GitHub REST documentation when changing authentication or asset handling.

A practical archive convention is:

```text
{tool}_{tag}_{GOOS}_{GOARCH}.tar.gz
{tool}_{tag}_windows_{GOARCH}.zip
checksums.txt
```

For Windows/macOS/Linux on x64 and ARM64, map Go's `windows/darwin/linux` and `amd64/arm64` to Node's `win32/darwin/linux` and `x64/arm64` in the bootstrap. Add `.exe` only for Windows. Restrict the declared support matrix to platforms that are actually built and tested. Prefer `CGO_ENABLED=0` only when dependencies support it.

Use `crypto/sha256`, `archive/tar`, `compress/gzip`, and `archive/zip` for verification and selective extraction. Match one expected regular executable, rather than unpacking everything by archive path. `os.CreateTemp` in the destination directory avoids cross-filesystem rename; `os.Executable` followed by `filepath.EvalSymlinks` locates the real install target.

POSIX rename can replace a running executable. Windows replacement has different overwrite, sharing, and cleanup rules: test the selected strategy on Windows with a running executable, not only a dummy file. A failed rollback is an installation failure with a recovery path, not a successful fallback.

## Packaging and CI evidence

Keep the platform archives, checksum manifest, and optional npm bootstrap archive under one release tag. Compute checksums from the final packaged bytes. An npm tarball should include only intended package files; verify its contents before publication. Use an actual packed tarball to test npm/npx, rather than testing only a source-tree wrapper.

An effective release job depends on OS test jobs, validates tag/package-version consistency, builds the complete supported matrix, validates the packed bootstrap, and creates the GitHub Release only after those steps pass. Keep credentials scoped to the job that needs them.

Test global installation under a temporary prefix containing spaces, delete the isolated npx cache, then run every installed command. Verify custom prefixes, help, invalid flags, and native exit codes. Use a temporary binary compiled with an older version string to test against a real, already published newer Release when authorized; keep the user's actual installation untouched.
