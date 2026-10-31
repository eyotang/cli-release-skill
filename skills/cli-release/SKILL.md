---
name: cli-release
description: Build and release native command-line tools with npm/npx bootstrap installation, GitHub Release self-updates, and cross-platform delivery checks. Use when implementing CLI installation or self-updates, or preparing a native CLI release.
license: MIT
---

# CLI development and release

Deliver a native CLI whose installed executable owns its future updates. Use npm/npx as the initial installation bridge and GitHub Releases as the update source. Apply this architecture when it fits the requested tool; preserve explicitly chosen package-manager channels and repository conventions.

## Establish the contract

Inspect the command entrypoints, installer, version source, release workflow, supported platforms, and repository visibility. Trace every invocation path, including native binaries, wrappers, aliases, and custom installation prefixes.

Before implementation, identify:

- Which native executables are installed and whether they update independently or as a bundle.
- How a release version reaches the compiled binary; development builds must be distinguishable.
- Which assets each OS/architecture receives and how their names match updater selection.
- How public users or authorized private-repository users can obtain those assets.

Reuse the current implementation language and installed dependencies. Keep transfer/business logic separate from installation and update side effects. Help, argument errors, cancellation, and child-process exit codes must retain their existing meanings.

For Go CLIs, read [references/go.md](references/go.md) when implementing the updater or release builds.

## First installation

The bootstrap package selects the target platform, stages the native executable, and installs into a documented writable directory. A custom prefix must survive installation and updates. Installed files must remain runnable after clearing the npx cache; copy or pack files rather than globally linking into that cache.

Bundle the supported native binaries in the public npm package, or download authenticated/verified Release assets using the chosen installer design. State the installer's actual prerequisites. A public npm package must not promise anonymous access to private GitHub assets.

Prefer a native command on PATH. When wrappers remain, keep them thin: inherit the terminal, forward cancellation, propagate the native exit status, and let the native executable own update discovery and installation. Updating a CLI must not require rerunning npx or republishing npm.

When introducing self-updates to a CLI that previously lacked them, document the one-time manual/bootstrap upgrade. If the npm bootstrap still embeds an older binary, report that separately; publish a refreshed bootstrap only when authorized and authenticated.

## Update before execution

Put the check at the native process entrypoint, before normal command work, for every executable. Default behavior:

| Condition | Observable behavior |
| --- | --- |
| Development build | Skip update discovery |
| Check takes more than 1.5 seconds | Cancel it and run the original command |
| Authentication, network, HTTP, or metadata failure | Skip the check and run the original command |
| Same, older, draft, or prerelease version | Run the original command |
| New stable release, interactive terminal | Show current/new version and ask `[y/N]` |
| New stable release, noninteractive execution | Print a short notice to stderr and continue without reading stdin |
| Decline, empty input, or EOF | Run the original command |
| Explicit `y` / `yes` | Install the selected release |
| Successful installation | Exit before business work and ask the user to rerun |
| Installation failure | Report the failure; continue with the intact old version when recovery succeeded |

Treat the 1.5 seconds as one wall-clock budget covering credential lookup, DNS, connection, redirects, response body, and metadata parsing. A socket inactivity timeout or a fresh timeout per phase does not meet the contract. Cancel subprocesses as well as HTTP requests. Keep the longer download/install deadline separate from discovery.

Use semantic version comparison. A stable numeric `major.minor.patch` parser is sufficient when that is the project's release scheme; implement prerelease ordering only when that channel is required. Validate the tag before using it in paths or commands.

Write notices/prompts to stderr so machine-readable stdout remains usable. Read stdin only when both input and the prompt output are terminals. Approval applies to that offered version; do not install automatically on a later invocation because of an earlier answer.

## GitHub discovery and verified installation

Use the repository's GitHub Release API and select the published stable release. Match the current OS/architecture and the chosen tag exactly to an asset. Keep release asset naming and updater naming in agreement.

For private repositories, reuse configured `GH_TOKEN`, `GITHUB_TOKEN`, or an existing `gh auth login` within the discovery deadline. Never embed credentials in bootstrap packages, source, URLs, logs, or release artifacts. An unauthenticated private-repository 404 is a skipped check, not proof that the installed version is current. Strip credentials from cross-origin download redirects and require HTTPS.

After confirmation:

1. Download the selected artifact and its checksum manifest with bounded sizes and an installation deadline. Use authenticated API asset downloads for private releases.
2. Verify the archive's SHA-256 against the exact filename in the manifest before extracting or replacing anything. Checksums from the same Release detect corruption; they do not independently authenticate a compromised publisher. If signed manifests are part of the project's trust model, preserve signature verification and pin the public verification key.
3. Extract only the expected regular executable; never extract arbitrary archive paths or links into the installation directory. Bound decompressed size too.
4. Resolve the actual executable path, including symlinks. Stage a replacement on the same filesystem, preserve executable permissions, flush/close it, then replace it.
5. Use an OS-appropriate replacement strategy. On Windows, handle the running executable with rename/backup or a narrowly scoped post-exit helper; roll back a failed replacement. Clean up backups when possible. Validate running-process replacement on Windows before claiming it works there.

Finish all verification before touching the working executable. Preserve user configuration and credentials. If the install directory is unwritable, explain the failure without automatically escalating privileges. Report rollback failures and any recovery file path explicitly.

## Verify and release

Choose a new version after checking local tags, remote tags/releases, and the bootstrap registry when relevant. Keep the package version, release tag, archive names, and compiled version consistent. Build release artifacts from the exact commit being tagged.

Leave runnable checks for the meaningful behavior: version ordering, failed/slow discovery, noninteractive stdin handling, explicit confirmation, missing platform assets, checksum rejection, extraction limits, and replacement/rollback. Exercise installed commands and cancellation/exit codes through the real bootstrap. Clear its cache and verify the global installation still works.

Run native tests on each supported OS. Cross-compilation is useful build evidence, not evidence of Windows/macOS runtime behavior. Verify at least one actual old-to-new update end to end with a temporary executable, then start the updated executable. Do not publish artificial old releases merely to manufacture this test.

When publication is requested:

1. Complete relevant tests and review the diff, then commit the intended files.
2. Push the commit and a unique version tag using the repository's release workflow. Never move an already published tag to hide a failure.
3. Wait for CI to finish, inspect failures, and stop on an unchanged blocker rather than repeatedly publishing or retrying. Correct a failed release through the project's documented recovery process.
4. Verify the Release is published, its target commit is correct, and all promised platform archives, bootstrap package, and checksum files exist.
5. Publish npm only when it is part of the requested delivery and credentials are available. GitHub publication and npm publication are separate outcomes.

Report the commit, Release URL, actual checks completed, and anything still blocked. Distinguish compilation, native tests, installed-package tests, and live upgrade verification. A successful push or a running workflow is not a completed release.
