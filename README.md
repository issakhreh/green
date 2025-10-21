# .github/workflows/flutter-ci-cd.yml
name: Flutter CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:

jobs:
  test:
    name: 🧪 Test on Flutter ${{ matrix.channel }}
    runs-on: macos-latest
    strategy:
      matrix:
        channel: [stable, beta]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 'latest'
          channel: ${{ matrix.channel }}

      - name: Cache Pub packages
        uses: actions/cache@v3
        with:
          path: ~/.pub-cache
          key: ${{ runner.os }}-pub-${{ matrix.channel }}-${{ hashFiles('**/pubspec.yaml') }}

      - name: Install dependencies
        run: flutter pub get

      - name: Static analysis
        run: flutter analyze

      - name: Run tests & collect coverage
        run: |
          flutter test --coverage
          mkdir -p coverage
          cp coverage/lcov.info coverage/lcov-${{ matrix.channel }}.info

      - name: Upload coverage reports
        uses: actions/upload-artifact@v3
        with:
          name: coverage-${{ matrix.channel }}
          path: coverage/lcov-${{ matrix.channel }}.info

  build-android:
    name: 📱 Build Android APK
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup Flutter (stable)
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 'latest'
          channel: stable

      - name: Cache Pub packages
        uses: actions/cache@v3
        with:
          path: ~/.pub-cache
          key: ${{ runner.os }}-pub-stable-${{ hashFiles('**/pubspec.yaml') }}

      - name: Install dependencies
        run: flutter pub get

      - name: Build release APK
        run: flutter build apk --release

      - name: Upload APK artifact
        uses: actions/upload-artifact@v3
        with:
          name: app-release.apk
          path: build/app/outputs/flutter-apk/app-release.apk

  build-ios:
    name: 🍎 Build iOS IPA
    runs-on: macos-latest
    needs: test

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup Flutter (stable)
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 'latest'
          channel: stable

      - name: Cache Pub packages
        uses: actions/cache@v3
        with:
          path: ~/.pub-cache
          key: ${{ runner.os }}-pub-stable-${{ hashFiles('**/pubspec.yaml') }}

      - name: Install dependencies
        run: flutter pub get

      - name: Build release IPA (no codesign)
        run: flutter build ipa --release --no-codesign

      - name: Upload IPA artifact
        uses: actions/upload-artifact@v3
        with:
          name: app-release.ipa
          path: build/ios/ipa/*.ipa
