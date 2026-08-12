---
id: "0001"
title: "Design: TWA Shortcut Launcher Activity (Dedicated Trampoline & Async Launch)"
project: "android-browser-helper"
author: "jetski@google.com"
status: "draft"
date: "2026-08-12"
bug: "b/532016408, b/543098328"
---

<!--
**Agent Preamble:**
> **CRITICAL:** Before reading this design or writing any code, you MUST read the project's AGENTS.md.
-->

## 1. Context and Goals

### Problem Formulation
On Android Desktop, launching a Trusted Web Activity (TWA) via shortcuts (defined in `shortcuts.xml`) causes the app window to become unresponsive and its taskbar icon to vanish (User Report: **b/532016408**).

This occurs because shortcuts target `LauncherActivity`, which is translucent and does not have `android:noDisplay="true"`. As detailed in the technical diagnosis (**b/543098328**), Android WindowManager detects this translucent activity entering the stack of the running TWA task and exempts the task from desktop mode, forcing it to fullscreen. When `LauncherActivity` finishes immediately (or routes the intent to an existing instance), the window manager tries to revert the transition. This leads to a desynchronized state where the task is fullscreen (but invisible/inactive on the taskbar) and the inner activity has freeform bounds, breaking input.

### Background
WindowManager policy requires that transient trampoline activities must be marked `android:noDisplay="true"` to avoid disrupting desktop windowing state. 
`LauncherActivity` cannot set `noDisplay="true"` because:
1.  **Lifecycle Constraints:** `noDisplay="true"` requires `finish()` to be called synchronously before `onResume()`. `LauncherActivity` performs asynchronous work (binding to Custom Tabs service) during a cold start, which would lead to a `SuperNotCalledException` crash.
2.  **Fallback UIs:** `LauncherActivity` occasionally shows a fallback `AlertDialog`, which requires a valid window token and would crash (`BadTokenException`) with `noDisplay`.

Previous design iterations attempted either to route to `LauncherActivity` (which failed because WindowManager still observes the translucent window) or use a background `Service` (which fails due to Android 10+ Background Activity Launch restrictions).

### Goals
*   Fix TWA shortcut unresponsiveness on ChromeOS Desktop.
*   Introduce a dedicated foreground trampoline activity (`ShortcutTrampolineActivity`) specifically for shortcuts that safely uses `android:noDisplay="true"`.
*   Completely skip `LauncherActivity` UI rendering and translucent styling during a shortcut launch.
*   Delegate the Custom Tabs binding and TWA intent dispatching to the background asynchronously using the Application Context, without violating BAL (Background Activity Launch) rules.

### Non-Goals
*   Migrating `LauncherActivity` to the Jetpack Splash Screen API (this is an independent modernization effort).
*   Fixing the issue centrally on the platform/WindowManager side.

## 2. Proposed Architecture

We will introduce a new activity, `ShortcutTrampolineActivity`, designed strictly for `shortcuts.xml`. It will mark itself as `android:noDisplay="true"`.

When launched, it will instantiate a `TwaLauncher` using the `ApplicationContext` instead of an Activity context. It will invoke the asynchronous `TwaLauncher.launch()` method and then independently call `finish()` synchronously at the end of its `onCreate()`.

By decoupling the async launch process from the Activity lifecycle, `ShortcutTrampolineActivity` completely disappears from WindowManager's view immediately, perfectly satisfying `noDisplay` constraints and avoiding desktop mode corruption, while the TWA connection silently resolves in the background and brings the browser task to the foreground. 

### Subsystems Affected
*   [x] Core Library (`androidbrowserhelper`)
*   [ ] Location Delegation (`locationdelegation`)
*   [ ] Play Billing (`playbilling`)
*   [x] Demos (to verify shortcut configuration)

### Thread Model
*   The `ShortcutTrampolineActivity` executes entirely in a single continuous UI thread execution inside `onCreate()` and calls `finish()`.
*   `TwaLauncher` handles its Custom Tabs Service IPC asynchronously in the background. Once the callback executes, the Android OS completes the window transition directly to the browser.

### Data Models & Schemas
*   Consuming applications will need to update `shortcuts.xml` to point their `<intent>` targets to `ShortcutTrampolineActivity` instead of `LauncherActivity`.

### API Surface & Public Interfaces
*   A new public Java class `ShortcutTrampolineActivity`.
*   Exported Android manifest component for the new activity.

## 3. Alternatives Considered

**Alternative 1: Service-based Trampoline**
*   *Design:* Use a background Service to intercept shortcuts and launch the TWA.
*   *Rejection Reason:* Triggers Background Activity Launch (BAL) blocks on Android 10+, preventing the TWA from launching from the background context if the app is entirely dormant.

**Alternative 2: Modify `LauncherActivity` with `noDisplay="true"`**
*   *Design:* Apply `noDisplay="true"` directly to `LauncherActivity`.
*   *Rejection Reason:* `LauncherActivity`'s asynchronous binding process violates the synchronous `finish()` requirement of `noDisplay`, leading to immediate crashes. Fallback UIs would also crash.

**Alternative 3: Routing to `LauncherActivity`**
*   *Design:* Have the `ShortcutTrampolineActivity` securely construct an intent and launch `LauncherActivity`.
*   *Rejection Reason:* Because `LauncherActivity` lacks `noDisplay` and is translucent, its brief appearance in the visual stack triggers the ChromeOS rendering bug regardless of how securely or quickly it is launched.

**Alternative 4: Jetpack Splash Screen Migration (Deleting Legacy Code)**
*   *Design:* Delete the legacy custom Java-based splash screen system entirely (`PwaWrapperSplashScreenStrategy`, etc.), utilize the Jetpack `androidx.core:core-splashscreen` API, and apply `android:noDisplay="true"` directly to the main `LauncherActivity`.
*   *Rejection Reason:* Too risky for this bug fix. The primary risk of deleting the legacy custom code is that we lose the ability to keep the splash screen visible *while the Chrome web page is loading* (which the custom Splash Activity currently facilitates). Relying purely on the native OS-level Jetpack splash screen would mean the splash screen disappears as soon as Chrome's activity starts, briefly showing the user a blank Chrome loading page before the content actually renders. While simpler, the UX regression is too severe for an immediate shortcut bug patch.

## 4. Core Principle Considerations

### Speed & Efficiency
*   **Startup & Critical Paths:** The trampoline executes in a minimal amount of time. It delegates launch execution entirely to background IPC, preventing main thread stalls.
*   **APK Size Impact:** Negligible (one minimal Activity class and manifest entry).

### Security
*   **Intent Security:** `ShortcutTrampolineActivity` relies entirely on its implicit parsing of `getData()`. It does not forward random intents to internal activities; it only triggers `TwaLauncher` with the extracted URI, thereby providing a secure entry point.
*   **BAL Compliance:** The launch of `ShortcutTrampolineActivity` itself originates from a foreground user action (clicking a shortcut). Even though the TWA launch resolves asynchronously after `finish()` is termed, Android attributes the launch token's grace period originating from the initial explicit foreground touch event, ensuring the TWA launches reliably without BAL blockades.

### Stability & Simplicity
*   Guarantees that no translucent app-level window interacts with the foreground stack during shortcut resolution, circumventing the WindowManager bug entirely.
*   Separates non-UI routing (shortcuts) from UI-driven routing (Splash Screens / Fallbacks).

## 5. Privacy & Accessibility

### Privacy
*   **N/A:** No change to privacy or data handling.

### Accessibility (A11y)
*   **N/A:** The trampoline has no UI and is imperceptible to users and screen readers.

## 6. Testing Plan
*   **Unit Tests (Robolectric):** 
    *   Verify `ShortcutTrampolineActivity` calls `finish()` synchronously in `onCreate()`.
    *   Verify `ShortcutTrampolineActivity` instantiates `TwaLauncher` using the Application Context, not the Activity Context.
*   **Manual Testing:**
    *   Build a demo app with `shortcuts.xml` pointing to `ShortcutTrampolineActivity`.
    *   Run app on ChromeOS Desktop / windowed environment.
    *   Click shortcut while app is running in the background; verify the app is brought to foreground without corrupting window state or vanishing from the taskbar.

## 7. Detailed Implementation

1.  **Create `ShortcutTrampolineActivity.java`:**
    ```java
    package com.google.androidbrowserhelper.trusted;

    import android.app.Activity;
    import android.content.Intent;
    import android.net.Uri;
    import android.os.Bundle;

    public class ShortcutTrampolineActivity extends Activity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            try {
                Uri uri = getIntent().getData();
                if (uri != null) {
                    LauncherActivityMetadata metadata = LauncherActivityMetadata.parse(this);
                    
                    // Instantiate using Application Context. This prevents activity-leakage when finishing and 
                    // ensures the launch acts independently of this activity's swift shutdown.
                    TwaLauncher twaLauncher = new TwaLauncher(getApplicationContext(), metadata.launchingBrowser);
                    
                    TrustedWebActivityIntentBuilder builder = new TrustedWebActivityIntentBuilder(uri);
                    
                    // Launch asynchronously
                    twaLauncher.launch(
                            builder,
                            new QualityEnforcer(),
                            null /* splashScreenStrategy */,
                            () -> {
                                // Clean up the launcher once the async operation finishes
                                twaLauncher.destroy();
                            }
                    );
                }
            } finally {
                // Must finish synchronously to satisfy noDisplay="true"
                finish();
            }
        }
    }
    ```
    *Note: `TwaLauncher` must ensure that the `TrustedWebActivityIntentBuilder` or resulting Intent uses `Intent.FLAG_ACTIVITY_NEW_TASK` when initiated from a non-Activity context. Otherwise an `AndroidRuntimeException` will be thrown.

2.  **Update `AndroidManifest.xml` (Library):**
    ```xml
    <activity
        android:name="com.google.androidbrowserhelper.trusted.ShortcutTrampolineActivity"
        android:theme="@android:style/Theme.NoDisplay"
        android:noDisplay="true"
        android:exported="true" />
    ```

3.  **Update Documentation/Demos:**
    *   Instruct developers to target `ShortcutTrampolineActivity` in their `shortcuts.xml` `<intent>` tags instead of `LauncherActivity`.

## 8. Future Work & Technical Debt
*   In the future, the fallback checks and UI rendering of `LauncherActivity` could be further decoupled, allowing both primary launches and shortcuts to utilize a shared background-routing strategy where possible.
