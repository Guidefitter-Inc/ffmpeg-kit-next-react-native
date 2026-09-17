# Provenance

This repository is the React Native plugin from
[arthenica/ffmpeg-kit-next](https://github.com/arthenica/ffmpeg-kit-next) (the official FFmpegKit
successor) **with the native binaries committed**. Upstream publishes no packages and git-ignores the
binaries, expecting every consumer to build them; this repo builds them once so apps can depend on it
by commit.

- Upstream: `arthenica/ffmpeg-kit-next` @ `v9.0.0` (`a724ed9958`), `react-native/` folder
- FFmpeg 9.0.1
- **Licence: LGPL v3** — built without `--enable-gpl`; every library reports `LGPL version 3 or later`
- Built 2026-09-17 on macOS 26.3 (arm64), Xcode 26.0.1, Determinate Nix 3.22.4

## Android

```bash
./nix-android.sh -p android-r27d \
  --enable-lib-android-media-codec \
  --enable-lib-android-zlib \
  --disable-arch-arm-v7a-neon
```

`android/libs-maven/com/arthenica/ffmpeg-kit-next/9.0.0/` — AAR 31.0 MB, API level 24, NDK r27d.

| ABI | FFmpeg libraries, uncompressed | Page alignment |
|---|---:|---|
| arm64-v8a | 15.35 MB (7.59 MB compressed) | 16 KB |
| armeabi-v7a | 14.23 MB | 4 KB (32-bit; Play's 16 KB rule does not apply) |
| x86 | 16.99 MB | 4 KB (32-bit) |
| x86_64 | 18.47 MB | 16 KB |

MediaCodec is enabled (`hevc_mediacodec`, `h264_mediacodec`). Only the plain `arm-v7a` 32-bit build is
included — no duplicate NEON set.

## iOS

```bash
./nix-ios.sh -p xcode26 -x \
  --enable-lib-ios-videotoolbox \
  --enable-lib-ios-zlib \
  --enable-lib-ios-bzip2 \
  --enable-lib-ios-libiconv \
  --disable-arch-arm64e \
  --disable-arch-arm64-mac-catalyst \
  --disable-arch-x86-64-mac-catalyst
```

`ios/Frameworks/` — eight **dynamic** xcframeworks (`ffmpegkit`, `libavcodec`, `libavdevice`,
`libavfilter`, `libavformat`, `libavutil`, `libswresample`, `libswscale`), deployment target 12.1.

- Slices: `ios-arm64` (devices, 14.51 MB total) and `ios-arm64_x86_64-simulator` (Apple Silicon and
  Rosetta simulators)
- VideoToolbox is enabled (`hevc_videotoolbox`, `h264_videotoolbox`); the frameworks link VideoToolbox,
  CoreMedia, CoreVideo, libz, libbz2 and libiconv themselves, so the podspec needs no changes
- No dSYMs are included

## Changes from upstream

- `.gitignore`: removed `ios/Frameworks` and `android/libs-maven`, so the binaries are committed
- `LICENSE`: copied from upstream's repository root (the `react-native/` folder has none)
- `PROVENANCE.md`: this file

No source, podspec or Gradle file is modified.

## Verified in the binaries

Every component the Guidefitter app's commands need is present on every Android ABI and on the iOS
device slice: `hevc`, `h264`, `aac`, `mjpeg`, `image2`, `mov`/`mp4`/`ipod` muxers, the
`mov,mp4,m4a,3gp,3g2,mj2` demuxer, `concat`, `lavfi`, `anullsrc`, `aresample`, `apad`, `loudnorm`,
`aac_adtstoasc`, `hevc_mp4toannexb`, `file`, `pipe`, plus the platform's hardware HEVC encoder.

To check a build, run `strings -n 3` on each library **file** (not via stdin — the default minimum
length of 4 hides `aac`, `mov` and `mp4`) and search for the exact names. Components live in the
`libav*` libraries; `libffmpegkit` is only the wrapper.

## Rebuilding

1. Install Nix. Clone upstream and check out the tag you want.
2. **macOS host-compiler shim (required).** FFmpeg's configure uses `gcc` as the host compiler, and the
   Nix shells provide only `cc`, so the build fails with `Host compiler lacks C11 support`. Put a `gcc`
   that forwards to `cc` first on `PATH`:

   ```bash
   mkdir -p ~/src/ffmpeg-hostcc-shim
   printf '#!/bin/sh\nexec cc "$@"\n' > ~/src/ffmpeg-hostcc-shim/gcc
   chmod +x ~/src/ffmpeg-hostcc-shim/gcc
   export PATH="$HOME/src/ffmpeg-hostcc-shim:$PATH"
   ```

3. Run both build commands above, then `react-native/copy_local_binaries.sh android ios`.
4. Copy the plugin over this repo (keep this repo's `.gitignore`, `LICENSE` and `PROVENANCE.md`), update
   this file, commit and tag `v<version>-gf.<n>`.

**A new FFmpeg command in the app may need an extra `--enable-lib-*` flag.** A missing component fails
at runtime with a non-zero `ReturnCode`, never at build time.
