---
id: "0001"
title: "Plan: TWA Shortcut Launcher Activity"
project: "android-browser-helper"
author: "Antigravity"
status: "completed"
date: "2026-08-12"
design_doc: "../designs/0001-twa-shortcut-launcher-activity.md"
bug: "b/532016408, b/543098328"
---

<!--
**Agent Preamble:**
> **CRITICAL:** Run tests and verify code quality before completing each milestone.
-->

## 1. Purpose / Big Picture

This plan implements a dedicated `ShortcutTrampolineActivity` to act as an entry point for Trusted Web Activity (TWA) shortcuts (e.g. from `shortcuts.xml`). By using `android:noDisplay="true"`, this trampoline safely routes the intent without presenting an intermediate translucent window, which prevents the WindowManager on ChromeOS Desktop from breaking the app into a dysfunctional fullscreen state.

## 2. Context and Orientation

Currently, shortcuts resolve to `LauncherActivity`, which is translucent and does asynchronous work on cold starts. On desktop environments like ChromeOS, this translucent transient activity breaks task windowing logic. The fix introduces `ShortcutTrampolineActivity` that solely extracts the shortcut's URI, immediately delegates launch processing to `TwaLauncher` running against the Application Context, and synchronously finishes itself in `onCreate()`. 

To support the Application Context without architectural changes to `TwaLauncher`, `ShortcutTrampolineActivity` will use an anonymous subclass of `TwaLauncher` to override `onPrepareIntent(Intent)` and attach `Intent.FLAG_ACTIVITY_NEW_TASK`. Additionally, a custom `FallbackStrategy` will be specified that avoids relying on Activity-based UIs and safely handles intent routing if the TWA fails.

Security-wise, because the Activity is exported to accept shortcut intents, the trampoline will rigorously validate the incoming URI against the trusted origins configured in `LauncherActivityMetadata` to prevent unauthorized Intent spoofing.

## 3. Progress

- [x] **Milestone 1: Create `ShortcutTrampolineActivity`, Manifest, and Unit Tests**
- [x] **Milestone 2: Apply trampoline to Demo App shortcuts and Update Documentation**

## 4. Surprises & Discoveries

*(To be filled during execution)*

## 5. Decision Log

- **Security Validation:** Added intent URI validation against `LauncherActivityMetadata` to prevent origin spoofing via the exported activity.
- **Application Context support:** Used inline overrides for `onPrepareIntent` and `FallbackStrategy` to add `FLAG_ACTIVITY_NEW_TASK` instead of refactoring `TwaLauncher`.
- **Milestone merging:** Merged the creation of the activity and its unit tests into a single PR (Milestone 1) per project rules.

## 6. Plan of Work (Milestones)

### Milestone 1: Create `ShortcutTrampolineActivity`, Manifest, and Unit Tests

*   **Concrete Steps:**
    - Create `androidbrowserhelper/src/main/java/com/google/androidbrowserhelper/trusted/ShortcutTrampolineActivity.java`.
    - Implement `onCreate` to:
        1. Extract the URI and `LauncherActivityMetadata`.
        2. Verify the URI matches the expected origin/TWA domain to prevent intent spoofing. Drop invalid intents and finish immediately.
        3. Instantiate an anonymous `TwaLauncher` using `getApplicationContext()`, overriding `onPrepareIntent` to add `Intent.FLAG_ACTIVITY_NEW_TASK`.
        4. Call `launch()` providing a custom `FallbackStrategy` that also uses `FLAG_ACTIVITY_NEW_TASK` for default intents, bypassing any Activity-requiring dialogs.
        5. Synchronously call `finish()` in `finally` or at the end of `onCreate()`.
    - Update `androidbrowserhelper/src/main/AndroidManifest.xml` to declare `ShortcutTrampolineActivity` with `android:theme="@android:style/Theme.NoDisplay"`, `android:noDisplay="true"`, and `android:exported="true"`.
    - Create `androidbrowserhelper/src/test/java/com/google/androidbrowserhelper/trusted/ShortcutTrampolineActivityTest.java`.
    - Add Robolectric unit tests to verify:
        - `finish()` is called synchronously.
        - Security validation correctly drops unknown/malicious domains.
        - `TwaLauncher` handles the Application Context via overridden intent prep.
    - Build command: `./gradlew :androidbrowserhelper:assembleDebug`
    - Test command: `./gradlew :androidbrowserhelper:test`
*   **Interfaces and Dependencies:**
    - `ShortcutTrampolineActivity` extends `Activity`.
    - Custom inline overrides for `TwaLauncher` and `FallbackStrategy`.
*   **Validation and Acceptance:**
    - Code compiles and unit tests pass.
    - Test code accurately simulates both valid and spoofed URIs.
*   **Idempotence and Recovery:**
    - Safe to re-run.

### Milestone 2: Apply trampoline to Demo App shortcuts and Update Documentation

*   **Concrete Steps:**
    - Create `demos/twa-basic/src/main/res/xml/shortcuts.xml` pointing to `com.google.androidbrowserhelper.trusted.ShortcutTrampolineActivity` and update its manifest.
    - Update other demo applications that utilize `shortcuts.xml` if applicable.
    - Update `README.md` and/or relevant files in `docs/` to explicitly instruct developers on how to configure `shortcuts.xml` safely using `ShortcutTrampolineActivity`.
    - Build command: `./gradlew assembleDebug`
*   **Interfaces and Dependencies:**
    - XML Shortcut configurations and markdown documentation.
*   **Validation and Acceptance:**
    - Demos build successfully.
    - Documentation visibly advises the new shortcut routing path.
*   **Idempotence and Recovery:**
    - Safe to re-run.
