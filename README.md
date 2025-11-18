# ergox

## FFmpeg Export Workflow

The repository ships `.github/workflows/ffmpeg-release.yml`, which packages FFmpeg via vcpkg for every runtime we need. Trigger it manually with **Run workflow** (optionally passing a `reason`) or let it run automatically whenever the workflow itself or anything under `vcpkg/` changes. Every run checks out this repo and, if a `vcpkg/` folder is missing (for example on fresh forks), clones the upstream `microsoft/vcpkg` repository before bootstrapping so the build never fails with “No such file or directory”.

Each job exports a raw vcpkg payload into `third_party/ffmpeg/<platform>` and uploads the folder as an artifact named `<platform>-ffmpeg`. The current target matrix is:

- `linux-x64` (`x64-linux` triplet)
- `linux-arm64` (`arm64-linux` triplet) – installs `g++-aarch64-linux-gnu` and `pkg-config-aarch64-linux-gnu` before building
- `android-arm64` (`arm64-android` triplet) – installs `zip`, `unzip`, and `default-jdk`
- `macos-arm64` (`arm64-osx` triplet)
- `macos-x64` (`x64-osx` triplet, runs on the macOS-13 Intel runner)
- `ios-arm64` (`arm64-ios` triplet)
- `windows-x64` (`x64-windows` triplet)
- `windows-arm64` (`arm64-windows` triplet)

Download the artifact for the platform you need, then drop its contents into `third_party/ffmpeg/<platform>` inside any consumer repo (for example GGUFx) to keep the prebuilt binaries consistent across projects.

## Building for Android

Cross-compilation is mandatory for Android because device binaries cannot be produced with a desktop compiler alone. The workflow above runs these steps on Ubuntu, but you can follow the same approach locally if needed:

1. **Install Android NDK** – download the latest NDK from Google and configure `ANDROID_NDK_HOME` or run from Android Studio. The NDK bundles the cross-compilers for ARM and x86 targets.
2. **Configure vcpkg** – use the Android triplets (`arm64-android`, `x64-android`, etc.). For whisper.cpp compatibility, always include `swresample` in the feature list:

   ```bash
   ./vcpkg install ffmpeg[avcodec,avformat,avdevice,avfilter,swresample,swscale] --triplet arm64-android
   ```

3. **Integrate outputs** – copy the generated `.so` files into your Android app module and consume them via JNI or native C++ (NDK). The exported artifacts land in `third_party/ffmpeg/android-arm64` when built via GitHub Actions.

## Building for iOS

iOS binaries must be produced on macOS with Xcode, so our workflow pins these jobs to Apple runners. Local steps mirror the automation:

1. **Install Xcode + command-line tools** – provides the iOS SDK and compilers.
2. **Configure vcpkg** – use the `arm64-ios` (devices) or `x86_64-ios` (simulator) triplets. Include `swresample` for parity with desktop builds:

   ```bash
   ./vcpkg install ffmpeg[avcodec,avformat,avdevice,avfilter,swresample,swscale] --triplet arm64-ios
   ```

3. **Integrate outputs** – add the produced `.a` or `.dylib` files to your Xcode project, set `ARCHS` appropriately (`arm64` for devices, `x86_64` for simulator), and link them alongside whisper.cpp.

## Key Considerations

- You cannot build iOS payloads on Windows or Linux; use macOS runners (or machines) with Xcode.
- Android builds can run on Windows, macOS, or Linux as long as the Android NDK is installed.
- vcpkg triplets define the target platform and architecture; our workflow covers `x64-windows`, `arm64-windows`, `x64-linux`, `arm64-linux`, `macos-x64`, `macos-arm64`, `arm64-ios`, and `arm64-android`.
- Always include `swresample` (and typically `swscale`) when building FFmpeg for whisper.cpp to guarantee resampling support.
- Use the GitHub Actions matrix to keep artifacts reproducible; every run publishes zipped payloads that can be copied into `third_party/ffmpeg/<platform>`.

## Summary

- **Android** – build with the NDK, target `arm64-android`, ship `.so` libraries.
- **iOS** – build with Xcode on macOS, target `arm64-ios`, link `.a` or `.dylib` outputs.
- **Desktop** – x64/arm64 for Windows, Linux, and macOS are prebuilt automatically.
- Download artifacts from the workflow and drop them into the matching `third_party/ffmpeg` folder to keep GGUFx and related projects in sync.
