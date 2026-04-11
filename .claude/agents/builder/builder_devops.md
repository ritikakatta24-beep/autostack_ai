# Sub-Agent: Builder DevOps

## Role
Generate all deployment configuration, CI/CD pipelines, and local dev tooling. Called by `builder.md` as Phase 3.

## Inputs
Read these fields from `system_plan.json`:
- `deployment_target` → android | ios | web | all
- `backend.firebase_services` → determines which Firebase deploy steps to include
- `app_name` → used in workflow name

---

## .flutter-version (always generate)

```
3.22.0
```

Pin Flutter to an exact version — prevents breaking changes from minor updates across all jobs and local machines.

---

## firebase.json (always generate)

```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "storage": {
    "rules": "storage.rules"
  },
  "hosting": {
    "public": "build/web",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [{ "source": "**", "destination": "/index.html" }]
  },
  "functions": {
    "source": "functions",
    "runtime": "nodejs20"
  },
  "emulators": {
    "auth": { "port": 9099 },
    "firestore": { "port": 8080 },
    "storage": { "port": 9199 },
    "functions": { "port": 5001 },
    "ui": { "enabled": true, "port": 4000 }
  }
}
```

Omit `hosting` block if `web` not in `deployment_target`.
Omit `functions` block if `functions` not in `backend.firebase_services`.
Omit individual `emulators` ports for services not in `backend.firebase_services`.

---

## .github/workflows/deploy.yml (always generate)

Always include the `test` job. Only include deployment jobs matching `deployment_target`.

```yaml
name: [app_name] Deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'      # pinned — matches .flutter-version
          channel: stable
      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs
      - run: flutter test                 # CI_002: never skip or comment this out

  deploy-web:                             # include if deployment_target is "web" or "all"
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
          channel: stable
      - run: flutter pub get
      - run: flutter build web --release
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId: live

  deploy-android:                         # include if deployment_target is "android" or "all"
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
          channel: stable
      - run: flutter pub get

      # Decode and write keystore from secret
      - name: Setup keystore
        run: |
          echo "${{ secrets.KEYSTORE_FILE }}" | base64 --decode > android/app/keystore.jks

      # Build signed release APK
      - name: Build signed APK
        env:
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}
        run: |
          flutter build apk --release \
            --dart-define=KEY_ALIAS=$KEY_ALIAS \
            --dart-define=KEY_PASSWORD=$KEY_PASSWORD \
            --dart-define=STORE_PASSWORD=$STORE_PASSWORD

      # Upload APK as artifact so it can be downloaded from GitHub Actions
      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: build/app/outputs/flutter-apk/app-release.apk

      # Upload to Play Store (internal track)
      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT }}
          packageName: com.[app_name].app
          releaseFiles: build/app/outputs/flutter-apk/app-release.apk
          track: internal

  deploy-ios:                             # include if deployment_target is "ios" or "all"
    needs: test
    runs-on: macos-latest                 # iOS requires macOS runner
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
          channel: stable
      - run: flutter pub get

      # Install Apple certificate and provisioning profile
      - name: Install Apple certificate
        env:
          APPLE_CERTIFICATE: ${{ secrets.APPLE_CERTIFICATE }}
          APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
        run: |
          echo "$APPLE_CERTIFICATE" | base64 --decode > certificate.p12
          security create-keychain -p "" build.keychain
          security import certificate.p12 -k build.keychain \
            -P "$APPLE_CERTIFICATE_PASSWORD" -T /usr/bin/codesign
          security list-keychains -s build.keychain
          security default-keychain -s build.keychain
          security unlock-keychain -p "" build.keychain
          security set-key-partition-list -S apple-tool:,apple: \
            -s -k "" build.keychain

      - name: Install provisioning profile
        env:
          APPLE_PROVISIONING_PROFILE: ${{ secrets.APPLE_PROVISIONING_PROFILE }}
        run: |
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "$APPLE_PROVISIONING_PROFILE" | base64 --decode \
            > ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision

      - name: Build IPA
        run: flutter build ipa --release

      # Upload IPA as artifact
      - name: Upload IPA artifact
        uses: actions/upload-artifact@v4
        with:
          name: release-ipa
          path: build/ios/ipa/*.ipa
```

### GitHub Secrets required (tell user after generation)
| Secret | Required for |
|---|---|
| `FIREBASE_SERVICE_ACCOUNT` | Web deploy — CI_001 if missing |
| `KEYSTORE_FILE` | Android signing — base64 encoded .jks file |
| `KEY_ALIAS` | Android signing |
| `KEY_PASSWORD` | Android signing |
| `STORE_PASSWORD` | Android signing |
| `PLAY_STORE_SERVICE_ACCOUNT` | Play Store upload |
| `APPLE_CERTIFICATE` | iOS signing — base64 encoded .p12 file |
| `APPLE_CERTIFICATE_PASSWORD` | iOS signing |
| `APPLE_PROVISIONING_PROFILE` | iOS signing — base64 encoded .mobileprovision |

---

## Makefile (always generate)

```makefile
.PHONY: setup run run-android run-ios run-web test analyze \
        emulate deploy deploy-hosting deploy-functions \
        build-android build-ios build-web clean

setup:
	flutter pub get
	dart run build_runner build --delete-conflicting-outputs

run-web:
	flutter run -d chrome

run-android:
	flutter run -d android

run-ios:
	flutter run -d ios

emulate:
	firebase emulators:start

emulate-import:
	firebase emulators:start --import=./emulator-data --export-on-exit

test:
	flutter test --coverage

analyze:
	flutter analyze

deploy:
	firebase deploy

deploy-hosting:
	flutter build web --release && firebase deploy --only hosting

deploy-functions:
	cd functions && npm run build && firebase deploy --only functions

build-android:
	flutter build apk --release

build-ios:
	flutter build ipa --release

build-web:
	flutter build web --release

clean:
	flutter clean
	flutter pub get
```

---

## .gitignore additions (append to Flutter default)

```
# Firebase config — never commit these
google-services.json
GoogleService-Info.plist
.firebase/
firebase-debug.log
firestore-debug.log
storage.rules.debug.log
ui-debug.log

# Secrets
*.keystore
*.jks
*.p12
*.mobileprovision
.env
.env.*

# Build outputs
build/
*.apk
*.ipa

# Emulator data (optional — remove if you want to commit seed data)
emulator-data/
```

---

## Post-Generation User Instructions

Always output this block after Phase 3 completes:

```
📋 Manual steps before first deploy:

1. Add Firebase config files:
   → android/app/google-services.json
   → ios/Runner/GoogleService-Info.plist
   (Download from Firebase Console → Project Settings)

2. Add GitHub secrets (repo Settings → Secrets → Actions):
   → FIREBASE_SERVICE_ACCOUNT          (all targets)
   → KEYSTORE_FILE                     (android)
   → KEY_ALIAS, KEY_PASSWORD           (android)
   → STORE_PASSWORD                    (android)
   → PLAY_STORE_SERVICE_ACCOUNT        (android Play Store upload)
   → APPLE_CERTIFICATE                 (ios)
   → APPLE_CERTIFICATE_PASSWORD        (ios)
   → APPLE_PROVISIONING_PROFILE        (ios)

3. Encode secrets to base64 before pasting into GitHub:
   → base64 -i keystore.jks | pbcopy
   → base64 -i certificate.p12 | pbcopy
   → base64 -i profile.mobileprovision | pbcopy

4. Local setup:
   → make setup
   → make emulate      ← start Firebase emulators first
   → make run-web      ← or run-android / run-ios
```

---

## Error Codes (report to error_handler.md)
| Code | Trigger |
|---|---|
| `CI_001` | `FIREBASE_SERVICE_ACCOUNT` secret missing — escalate to user |
| `CI_002` | Flutter test fails in CI — show failing test, never skip |
| `CI_003` | `flutter build web` fails — check locally with `make build-web` |
| `CI_004` | iOS cert/profile missing or expired — escalate, provide Apple setup steps |
| `CI_005` | Android keystore missing or wrong password — check `KEYSTORE_FILE` and `STORE_PASSWORD` secrets |
| `CI_006` | Play Store upload rejected — check package name matches Play Console and track is correct |
| `CI_007` | Emulator not starting — check ports 4000/5001/8080/9099/9199 are free |