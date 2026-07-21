# macOS Apple Silicon CI build research

Checked on 2026-07-21 against first-party xSTUDIO, ASWF, Qt tooling, and GitHub documentation.

## Decision

xSTUDIO must be built on a native macOS runner to produce `xSTUDIO.app` and a DMG. The workflow uses the standard public-repository `macos-15` runner, which GitHub currently documents as an M1/arm64 machine with 3 CPU cores, 7 GB RAM, and 14 GB SSD. Standard runners are free and unlimited for public repositories.

The ASWF `ci-xstudio:2026` entry is a Linux CI image. ASWF explicitly documents that its Docker workflows on macOS still drive Linux containers and do not provide a native macOS toolchain. A Linux container cannot provide the Apple SDK, Xcode, `macdeployqt`, `install_name_tool`, `codesign`, or `hdiutil`, so it cannot replace the macOS runner for this deliverable.

## Upstream build inputs

The official macOS guide supports Intel and ARM and specifies:

- Xcode;
- CMake, Ninja, pkg-config, NASM, Autoconf, and Autoconf Archive;
- Qt 6.5.3 and the `qtimageformats` module;
- vcpkg commit `c2aeddd80357b17592e59ad965d2adf65a19b22f`;
- `MacOSNinjaRelease` for ARM;
- `cmake --build build --target install`.

The tracked CMake preset selects `VCPKG_TARGET_TRIPLET=arm-osx`. The triplet sets both `VCPKG_TARGET_ARCHITECTURE` and `VCPKG_OSX_ARCHITECTURES` to `arm64`, which produces one binary family for M1, M2, M3, M4, and M5 Macs.

The install target is important: xSTUDIO runs `macdeployqt` and its own rpath fixups during installation so that the application bundle is portable. The upstream output is `build/xSTUDIO.app`; DMG creation is an additional CI packaging step using `hdiutil`.

## Risks and mitigations

- The official guide warns that vcpkg configuration can take several hours, especially while building FFmpeg. The workflow uses a vcpkg binary cache and a six-hour timeout.
- The standard runner has limited RAM and disk. The workflow builds with three parallel jobs and cleans Homebrew downloads before dependency compilation.
- Upstream issue #151 recorded an ARM build failure involving x265/Python tooling. The workflow installs Automake, part of the confirmed workaround; the exact-tools vcpkg option remains a fallback if the failure recurs.
- xSTUDIO 1.3.0 includes the upstream fixes for Qt attempting to link the removed AGL framework and for non-portable dylib paths.
- No Apple Developer credentials are available. The bundle is ad-hoc signed but not notarized, so first-launch Gatekeeper verification may take time.

## Publication policy

The DMG and its SHA-256 file are uploaded directly from the build workspace to GitHub Releases. The workflow does not use `actions/upload-artifact`, avoiding duplicate artifact storage. A final `always()` cleanup deletes only artifacts belonging to the current workflow run and never touches Release assets.

## Primary sources

- [xSTUDIO macOS build guide](https://github.com/AcademySoftwareFoundation/xstudio/blob/main/docs/reference/build_guides/macos.md)
- [xSTUDIO Qt download guide](https://github.com/AcademySoftwareFoundation/xstudio/blob/main/docs/reference/build_guides/downloading_qt.md)
- [xSTUDIO CMake presets](https://github.com/AcademySoftwareFoundation/xstudio/blob/main/CMakePresets.json)
- [xSTUDIO ARM vcpkg triplet](https://github.com/AcademySoftwareFoundation/xstudio/blob/main/cmake/vcpkg_triplets/arm-osx.cmake)
- [xSTUDIO vcpkg manifest](https://github.com/AcademySoftwareFoundation/xstudio/blob/main/vcpkg.json)
- [xSTUDIO macOS ARM issue #151](https://github.com/AcademySoftwareFoundation/xstudio/issues/151)
- [ASWF Docker README](https://github.com/AcademySoftwareFoundation/aswf-docker#building-on-macos)
- [GitHub-hosted runner specifications](https://docs.github.com/en/actions/reference/runners/github-hosted-runners#standard-github-hosted-runners-for-public-repositories)
- [install-qt-action options](https://github.com/jurplel/install-qt-action#options)
