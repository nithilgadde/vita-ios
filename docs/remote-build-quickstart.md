# Personal iPhone development using remote builds

## Scope

- Target device: iPhone 15; iOS 27 as reported by the owner (exact version/build and JIT support still need verification).
- Development machine: Apple Silicon Mac without full Xcode; edit locally, build on GitHub-hosted macOS.
- Repository: https://github.com/nithilgadde/vita-ios
- Base: https://github.com/125hz/Tsubomi, branch `ios-upstream-core`.
- Initial source revision: `7e53bac87f442e770a39b1d9b032133418337f6d`.

This is an experimental fork, not an official Vita3K iOS release. Preserve GPLv2 and third-party license notices. Never commit firmware, game dumps, keys, pairing files, Apple credentials, signing certificates, or provisioning profiles.

## Build evidence and limitations

The source project's GitHub Actions run for the initial revision succeeded:
https://github.com/125hz/Tsubomi/actions/runs/31775338014

That is evidence of a previous successful build, not of runtime compatibility or safety. Our own build and physical-device testing remain separate milestones. The source project has older bootstrap documentation describing a different, limited interpreter path; do not confuse that with the upstream-core target.

Use **`.github/workflows/ios-upstream.yml`**, named **Build upstream-core iOS IPA**. Do not substitute the older `ios.yml` diagnostic build.

The selected workflow:

- runs on the standard `macos-15` hosted runner;
- selects an available Xcode 26 installation;
- builds a physical-device arm64 app with an iOS 26 deployment target;
- fetches submodules and builds dependencies remotely;
- pins vcpkg to a commit and checks the downloaded MoltenVK archive's SHA-256;
- packages an unsigned IPA with no Apple secrets;
- retains the IPA artifact for 14 days and permits up to 180 minutes per job.

The runner image and action version tags are not immutable, and the Xcode selection is not pinned to an exact patch version. Record the actual toolchain from every run before comparing results. A newer device OS does not by itself require changing the deployment target or SDK; device behavior and the signing/JIT tools still require testing.

## Local setup

The checkout at `~/Desktop/vita-ios` is shallow and intentionally has **no initialized submodules**. Do not download the full dependency graph or install Xcode locally merely to edit code.

Remotes:

- `origin`: `https://github.com/nithilgadde/vita-ios.git`
- `upstream`: `https://github.com/125hz/Tsubomi.git`

The GitHub CLI is authenticated separately. Never put its token in this repository.

## Start a build

From this directory:

```sh
gh workflow run ios-upstream.yml \
  --repo nithilgadde/vita-ios --ref ios-upstream-core

gh run list --repo nithilgadde/vita-ios \
  --workflow ios-upstream.yml --limit 5
```

Record the run ID, URL, source SHA, selected Xcode version, and SDK version. To inspect a particular run:

```sh
gh run view RUN_ID --repo nithilgadde/vita-ios
```

After it finishes successfully, inspect the artifact name and download only that run's IPA:

```sh
gh api repos/nithilgadde/vita-ios/actions/runs/RUN_ID/artifacts \
  --jq '.artifacts[] | {name, size_in_bytes, expired}'

gh run download RUN_ID --repo nithilgadde/vita-ios \
  --name 'Vita3K-upstream-core-iOS-SOURCE_SHA-unsigned' \
  --dir artifacts/RUN_ID

shasum -a 256 artifacts/RUN_ID/*.ipa
unzip -tq artifacts/RUN_ID/*.ipa
```

Replace `RUN_ID` and `SOURCE_SHA` with the recorded values. An Actions artifact ZIP is a container; extract it rather than renaming it to `.ipa`.

If the build fails, inspect its failure before changing dependencies:

```sh
gh run view RUN_ID --repo nithilgadde/vita-ios --log-failed
```

## Device gates (not completed)

1. Confirm the phone's exact iOS version and build in Settings → General → About.
2. Verify a compatible, trusted signing/sideloading route. An unsigned IPA cannot install directly on a stock phone.
3. Launch the app and retrieve logs; do not treat a visible library screen as emulation success.
4. Verify JIT on that exact device/OS. This fork checks for debugger/JIT availability and patches Dynarmic's code allocation; sideloading alone does not enable JIT. Its references to StikDebug are not evidence of iOS 27 compatibility.
5. Run a legally distributable homebrew program, then check graphics, input, audio, lifecycle, memory use, and temperature.

Keep signing and device pairing local. Never upload pairing records or Apple credentials to CI. No working-on-device claim should be made until these checks have actually passed.
