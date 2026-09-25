# 0001 — React Native + Expo for the patient and doctor mobile apps

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [02-patient-app](../specs/02-patient-app.md), [03-doctor-app](../specs/03-doctor-app.md), [06-identity-auth](../specs/06-identity-auth.md), [08-qr-subsystem](../specs/08-qr-subsystem.md)

## Context

Atlas ships two mobile apps on iOS and Android: the patient app and the doctor app. The specs require identical screen content across platforms with platform-native chrome, a shared visual language (Meridian, spec 01), camera-based QR scanning, biometric and passkey authentication, and OTP sign-in. The prototype and design assets are already React-flavored (`design/ios-frame.jsx`, `design/android-frame.jsx`, `Atlas App.dc.html`).

Scale framing (~1M patients) does not stress the client tier — mobile framework choice is not scale-sensitive. The deciders are team leverage across four client surfaces, design-system reuse, and coverage of the hard native features.

## Decision

Both mobile apps are built with **React Native + Expo** in TypeScript, in the shared monorepo:

- Two app targets (patient, doctor) sharing the `@atlas/ui` Meridian package and generated API clients (ADR 0011).
- **EAS** for builds, signing, and store submission; Expo OTA updates for non-native fixes.
- **react-native-vision-camera** for the QR scan path (spec 08) — native camera pipeline, not the JS-thread scanner.
- Expo modules for biometrics and passkeys (spec 06); platform-specific chrome follows each OS.

## Consequences

- One mobile codebase serves both platforms; Meridian is implemented once and shared with the web surfaces (ADR 0002).
- React skills transfer across all four client surfaces; one hiring profile for the client tier.
- Native modules (camera, biometrics, secure storage) are the risk surface — pinned via Expo dev clients, not bare ejects.
- The doctor scan → 360° flow is the product's signature moment; its latency budget is dominated by the backend resolve path, but scan-screen camera performance must be validated on low-end Android early.

## Alternatives rejected

- **Native Swift + Kotlin:** double the mobile effort across two apps and four codebases, duplicated design system, no payoff at this scale. Revisit only if measured scan-screen performance proves inadequate.
- **Flutter:** Dart is an island — splits the stack from the three React web/mobile surfaces, duplicates the Meridian implementation, and discards the existing React prototype assets.
