# Personal iPhone development using remote builds

## Scope

- Target device: iPhone 15; iOS 27.0 as reported by the owner (exact OS build number not recorded). App launch and disappearance of the JIT warning are user-confirmed; guest execution is not yet tested.
- Development machine: Apple Silicon Mac without full Xcode; edit locally, build on GitHub-hosted macOS.
- Repository: https://github.com/nithilgadde/vita-ios
- Base: https://github.com/125hz/Tsubomi, branch `ios-upstream-core`.
- Initial source revision: `7e53bac87f442e770a39b1d9b032133418337f6d`.

This is an experimental fork, not an official Vita3K iOS release. Preserve GPLv2 and third-party license notices. Never commit firmware, game dumps, keys, pairing files, Apple credentials, signing certificates, or provisioning profiles.

## Build evidence and limitations

The source project's GitHub Actions run for the initial revision succeeded:
https://github.com/125hz/Tsubomi/actions/runs/31775338014

Our baseline build also succeeded:
https://github.com/nithilgadde/vita-ios/actions/runs/38083627473

- Source: `35f7a621df2b719b17235cf2402491326f455180` (documentation-only changes from the initial revision).
- Artifact: `Vita3K-upstream-core-iOS-unsigned.ipa`.
- SHA-256 of downloaded unsigned IPA: `8a46669990af03d1e543d12f1b9aee5f461ff8e3f578848992c3a6561c541470`.
- Archive integrity check passed. The packaged Info.plist reports `DTXcode=2630`, `DTSDKName=iphoneos26.2`, and minimum iOS `26.0`.
- App display name: **Tsubomi**. Signing with Sideloadly changes the IPA, so the hash above does not describe a subsequently signed copy.

Compilation, installation, JIT detection, and guest execution are separate milestones. The source project has older bootstrap documentation describing a different, limited interpreter path; do not confuse that with the upstream-core target.

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

## Device setup: user-confirmed progress

The owner reports the following on iPhone 15 / iOS 27.0:

- StikDebug 3.1.13 and our baseline Tsubomi IPA installed using Sideloadly on the Mac.
- Developer Mode enabled on the phone; the user's development signing identity trusted under Settings → General → VPN & Device Management.
- Tsubomi launches and its library opens.
- Pre-install, font, and main Vita firmware packages imported successfully from Sony's update manifest. These files remain on the device, not in Git or CI.
- iloader placed a pairing file directly into StikDebug.
- LocalDevVPN, installed from the App Store **on the iPhone**, reports Connected. Do not change Mac startup security to run its iPhone app on the Mac.
- Tsubomi's **JIT not active** warning disappeared after explicitly assigning `universal.js` in StikDebug.

### Important: assign the JIT script

StikDebug 3.1.13's automatic script assignment list does not include Tsubomi. This fork's Dynarmic patch uses the iOS 26+ universal JIT breakpoint protocol. A normal launch or brief attach/detach is not sufficient.

1. Keep LocalDevVPN connected on the iPhone.
2. In StikDebug, use the **JIT** app list, not **Other** (launch-only).
3. Long-press **Tsubomi → Assign Script → universal.js** (bundled with StikDebug).
4. Tap Tsubomi to begin the JIT session. Do not stop the script or force-close StikDebug while Tsubomi prepares JIT.
5. Verify the warning disappears. This is only JIT detection, not proof that a Vita program executes correctly.

Recheck the script assignment if a later signing operation changes the installed bundle ID.

### Location permission is optional for this setup

StikDebug uses background location as one background keep-alive mechanism, not because Vita emulation needs GPS. Its settings offer **Background Keep-Alive → Silent Audio ON / Background Location OFF** as an alternative. Its 3.1.13 denial handler selects the audio fallback while still displaying a location-access alert. Do not treat that alert alone as a JIT failure or require location access to troubleshoot.

Source references for these findings:

- https://github.com/StikDebug/StikDebug/blob/3.1.13/StikDebug/Support/AutoScriptAssignments.swift
- https://github.com/StikDebug/StikDebug/blob/3.1.13/StikDebug/Features/InstalledApps/InstalledAppRows.swift
- https://github.com/StikDebug/StikDebug/blob/3.1.13/StikDebug/Services/BackgroundLocationManager.swift
- https://github.com/StikDebug/StikDebug/blob/3.1.13/StikDebug/Views/SettingsView.swift
- `ios/patches/0001-oaknut-ios-rwx-jit.patch`

## Remaining device gates

1. Record the exact OS build if troubleshooting requires it.
2. Execute a small legally distributable Vita homebrew program. Verify visible guest output and input response, and collect relevant diagnostics.
3. Separately test audio, lifecycle, memory use, temperature, and selected game compatibility.

Keep signing and device pairing local. Never upload pairing records or Apple credentials to CI. No claim of working guest emulation or playable games should be made until it is actually tested.
