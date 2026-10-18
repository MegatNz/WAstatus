# StatusDrop Android app

StatusDrop prepares WhatsApp Status videos on the phone instead of paying for server-side FFmpeg work. It has no account, login, analytics SDK, advertising SDK, or broad storage permission.

It's available on [Google Play](https://play.google.com/store/apps/details?id=com.wastatusvideo.statusdrop). Each published version also has a [GitHub release](https://github.com/Shamanthnp1/WAstatus/releases) with its matching source.

## Implemented flow

1. The Android system picker selects up to 3 videos / 300 MiB total.
2. Selected documents are copied to private app cache.
3. Pinned FFmpegKit/x264 compresses and splits them locally.
4. After local compression, the app automatically uploads only the finished clips through the StatusDrop mobile API.
5. The server checks exact bytes, SHA-256, MP4/H.264/AAC streams, dimensions, frame rate, and duration before sealing the clips in R2.
6. Finalization creates the existing five-minute activation code.
7. The user opens WhatsApp and sends the prefilled message; the existing Baileys flow returns clips in order.
8. An accessible in-app 9-step guide shows how to download, resend, and forward the returned clip to Status in full HD.
9. A three-dot About & support menu provides the published Instagram, email, Privacy Policy, and GPL source/license links without exposing a phone number.

The app uses the backend's `/api/mobile/*` routes. The old website compressor was retired in October 2026, so the app is now the only way to use StatusDrop.

## Limits and profiles

| Limit/profile | Value |
|---|---:|
| Android version | Android 10 / API 29 or newer |
| CPU architecture | ARM64 (`arm64-v8a`) only |
| Source selection | 3 videos, 300 MiB total |
| Generated delivery | 20 clips maximum |
| Accepted clip size | 15,800,000 bytes maximum |
| 1080p profile | 1080×1920, up to 29 seconds, H.264 CRF 23 |
| 720p profile | 720×1280, up to 59 seconds, H.264 CRF 24 |
| Encode retries | 3 bounded attempts per clip |
| Activation deadline | 5 minutes after successful finalize |

Videos are contained inside a portrait canvas with aspect ratio preserved; landscape sources receive padding rather than destructive cropping. Audio is optional and, when present, is encoded as AAC.

## Project layout

```text
lib/
  models/                    Typed video, clip, profile, and event models
  screens/home_screen.dart   Accessible picker → process → upload → activate UI
  services/
    native_video_service.dart  Flutter/Kotlin channel
    mobile_api.dart            Streaming mobile API client
android/app/src/main/kotlin/.../
  MainActivity.kt            Picker, cache, URL, and channel boundary
  LocalVideoProcessor.kt     FFmpeg planning, retries, progress, hash, cancellation
../../../src/server/mobileUpload.js
                             Secure mobile create/upload/finalize routes
../../../server.js            Shared activation and Baileys delivery handoff
```

## Toolchain

Validated in this workspace with:

- Flutter 3.38.1 / Dart 3.10
- Android SDK 36, build-tools 36.1.0
- JDK 21
- `dev.ffmpegkit-maintained:ffmpeg-kit-full-gpl:8.1.7` (exactly pinned)
- `com.arthenica:smart-exception-java:0.2.1` (explicit FFmpegKit runtime dependency)

The GPL FFmpegKit package includes x264. Distribution must retain compatible source and license notices. This note is not legal advice.

## Build

From `mobile/statusdrop_app` in PowerShell:

```powershell
flutter pub get
flutter analyze
flutter test --no-pub
flutter build apk --debug --target-platform android-arm64 --split-per-abi
```

Output:

```text
build\app\outputs\flutter-apk\app-arm64-v8a-debug.apk
```

This local build is signed with Flutter/Android's **debug certificate**. Android may show an unknown-source or Play Protect warning, and it can't install over the Google Play version (the signatures differ); uninstall the Play version first. Release builds for Google Play follow [RELEASE.md](RELEASE.md).

To point a build at another HTTPS backend:

```powershell
flutter build apk --debug --target-platform android-arm64 --split-per-abi `
  --dart-define=STATUSDROP_API_URL=https://your-api.example.com
```

Cleartext HTTP is disabled. A physical phone cannot use the development computer's `localhost` as its backend.

## Backend

Backend settings, WhatsApp linking and deployment are in [SELF_HOSTING.md](../../SELF_HOSTING.md). Check `GET https://api.wastatusvideo.com/api/health` and confirm `baileys: "connected"` before an end-to-end test.

Mobile endpoints:

```text
POST /api/mobile/sessions
PUT  /api/mobile/sessions/:sessionId/clips/:order
POST /api/mobile/sessions/:sessionId/finalize
```

Upload capabilities are random bearer credentials. Do not log tokens, upload URLs, local paths, hashes, or media bytes. Upload sessions use a sliding 10-minute idle lease with a one-hour absolute cap. Sealed delivery objects remain governed by the five-minute activation lifecycle and existing post-delivery cleanup.

## Privacy and cleanup

- The system picker avoids broad photo/storage permission.
- Source copies and generated clips live only in private app cache.
- Cancelling or failing processing removes incomplete outputs but keeps selected sources for retry.
- Changing a source or profile invalidates old generated clips.
- Successful upload clears generated local clips; **Start over** clears all job cache.
- The server receives only finished clips, validates them, and removes delivery objects after send or expiry.
- `android:allowBackup="false"` prevents app-cache backup.

## Testing and releases

Version 1.0.0 (version code 7) went through a 14-day Google Play closed test on testers' own phones before its production release in October 2026. Use [TESTING.md](TESTING.md) for device checks before each new version.

To publish a new version:

1. Bump `version:` in `pubspec.yaml` (for example `1.0.1+8`) and update `play/RELEASE_NOTES.txt`.
2. Build, upload and roll out on Google Play using [RELEASE.md](RELEASE.md).
3. Tag the commit with the version name and push the tag (`git tag v1.0.1 && git push origin v1.0.1`). The release workflow checks that the tag matches `pubspec.yaml` and publishes the GitHub release.

Do not commit release keystores, passwords, Baileys credentials, user media, generated clips, or APK binaries.
