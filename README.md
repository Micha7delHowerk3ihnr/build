# SingTools build orchestrator

This repository contains only GitHub Actions workflows. It checks out an exact
revision of the private SingTools source repository and delegates all native and
application packaging to that revision's `tool/native_build.dart` and
`setup.dart` entrypoints.

Before packaging, `test.yml` resolves the requested branch, tag, or commit to an
immutable source SHA. All test jobs and subsequent package/native build jobs
check out that SHA, even if the requested branch moves while tests are running.
The release workflow records this resolved SHA in the release body.

The Go toolchain version is read from the checked-out source's
`libcore/go.mod`; there is no second dependency lock file in this repository.
The same revision supplies the exact Flutter version in
`.github/workflows/test.yml` and the Gradle version in its Android Gradle
wrapper. This keeps tests, app packaging, and native builds on the same Flutter
SDK for `dev-next`, `release/macos13`, and historical tags.

Android jobs explicitly install the minimum platform (at least API 24), plus the
compile SDK and NDK declared by the selected Flutter SDK. The target native API
comes from the checked-out source's `native/targets.json`; its Clang and Clang++
executables must exist before building. Jobs no longer select whichever NDK
happens to be newest on the runner. Raising the minimum runtime to Android 7.0
does not lower the compile SDK required by Flutter.

- `build.yaml` handles both `build-trigger` and `release-trigger` dispatches.
  A manual run with a release tag publishes a stable release. A run without
  a release tag replaces the public `pre-release` assets.
- `test.yml` is a reusable test gate and can also be started manually. It runs
  Flutter behavior tests and the CMake manifest contracts on Linux, plus native
  Go tests with race detection on Linux, macOS, and Windows. Coverage reports are
  uploaded separately from package artifacts. Go feature tags come from the
  selected source's `libcore/Makefile`.
  The Flutter gate also runs static analysis and installs RPM tools so package
  content tests execute. macOS additionally compiles and runs the Swift system
  proxy transaction suite against isolated preferences without changing the
  runner's network settings.
- `native.yml` is a manual manifest-only diagnostic build, gated by the same
  test workflow.
- `package.yml` is the shared private-source packaging matrix. It waits for every
  test job to pass before starting the build matrix; a failure prevents packaging
  and the dependent release publication. To build the macOS 13 compatibility
  branch without replacing any public release, manually run **Package private
  source** with `source_ref: release/macos13`. Download the installers from that
  run's Actions artifacts.

Release downloads contain application installers only. Native manifests remain
in the Actions artifacts for diagnostics and are not attached to releases.

Pre-release runs share one concurrency group and replace the entire existing
release, including its notes, assets, and tag. Cleanup errors stop publication;
only an absent release or tag is ignored. The fixed `pre-release` URL is kept.
Its tag points to a fresh commit with the public workflow revision's unchanged
tree, so its date reflects publication rather than an old workflow commit. No
branch is changed. This refreshes its position in GitHub's chronological release
list; GitHub does not provide permanent pinning or allow prereleases to be marked
as the latest stable release.

Each workflow run assigns an eight-digit build number shared by all of its
packages. Stable and pre-release runs draw from the same sequence, which starts
above previously published date-based Android version codes. Rerunning a job
uses the same build number. The version override applies in the build checkout.
The release body records the build number, date, and exact source SHA.

The selected source revision must include the test suites, including
`test/native/manifest_contract_test.py`. Older source revisions lacking that
contract are rejected instead of silently skipping the manifest checks. macOS
jobs also require `test/native/system_proxy_test.py` and its Swift implementation.
These jobs do not replace device tests or installed-package/FFI smoke tests. The native
race suite uses the source's default Go feature tags. Windows also builds the
pinned Subconverter static library and runs the native conversion tests from
the selected source revision's `test-subconverter` Make target.

Required repository secrets are `PRIVATE_REPO` and `ACCESS_TOKEN`. Android
packages in both release channels use `KEYSTORE`, `KEY_ALIAS`, `STORE_PASSWORD`,
and `KEY_PASSWORD` so users can install newer packages across channels.
