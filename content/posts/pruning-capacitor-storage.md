---
title: "How to clear stuck secure storage data using Capacitor on iOS?"
author: "Mikołaj Wilczek"
date: 2025-08-28T21:00:00+02:00
draft: true
---

When building mobile apps with Capacitor, storing tokens and secrets securely is critical. `localStorage` is unsafe, and even secure HTTPS-only cookies are not always reliable, as they are not persisted across app reloads or terminations (if you use CapacitorCookies).

That’s why most developers reach for secure storage solutions which utilises the system's secure storage. One of the most common options is [Capacitor Secure Storage](https://github.com/martinkasa/capacitor-secure-storage-plugin).

If you’re debugging or releasing an app that suddenly breaks because of stuck secure storage data, this guide shows you how to fix and prevent it.

![Cover](/generic/pruning-capacitor-storage.png)

# The Problem: Why iOS is Different

During development, some values can be mistakenly added or removed in the app. On Android, the solution is fairly easy — we can clear app data and open it again. All stored data will be cleared ✅

## How iOS Secure Storage Really Works

On iOS, the situation is much more complex. Since the secure storage plugin uses Apple Keychain underneath, [you cannot delete](https://apple.stackexchange.com/questions/332574/deleting-an-app-s-data-from-the-ios-keychain-or-resetting-the-keychain-entirely) the app’s Keychain data outside the app unless you have a jailbroken device (which is strongly not recommended).
Even worse, if you have iCloud Keychain enabled, the data can persist even after a full device erasure. Fortunately, this is not the case for the simulator, where you can always select `Device -> Erase All Content and Settings...`.

But what about **real devices**?  
If a user ends up with corrupted secure storage, your app might be completely broken — and simply reinstalling won’t help.

# Fixing It: Options for Developers

## Update the app

We can ship an app update that clears all data on load. This is the most reliable solution, but also the slowest 🐌. However, it allows us to fix the problem permanently.

## Quick Fix with Safari DevTools

If the app is inspectable with Safari (and you have access to the device from macOS), you can launch Safari DevTools’s console and access Secure Storage, as every plugin is accessible from `window.Capacitor.Plugins`.

In that case, you can trigger a manual wipe of secure storage and reload the app like this:

```js
window.Capacitor.Plugins.SecureStoragePlugin.clear().then(() =>
  window.location.reload()
);
```

## How to prevent such events?

You can use `@capacitor/preferences`, which uses `UserDefaults` underneath, to create a flag `hasLaunchedBefore`. This lets you check if the app is launched for the first time since installation.

The best prevention is to detect the **first launch** after install, so the app knows if it’s freshly installed. On first launch, you simply clear the secure storage.

**Note:** If used in an already deployed app, this approach will:

- Delete existing tokens (treating the app as fresh install)
- That’s useful here, but in production you might want to mitigate it by:
  - Shipping two updates (first add Preferences, then clear), or
  - Validating stored tokens with your backend before clearing

```js
import { Preferences } from "@capacitor/preferences";
import { SecureStoragePlugin } from "capacitor-secure-storage-plugin";

/** Ensures corrupted tokens are cleared after fresh install. Can be launched only for iOS */
const onAppLaunch = async () => {
  const hasLaunchedBefore = await Preferences.get({ key: "hasLaunchedBefore" });
  if (hasLaunchedBefore.value !== "true") {
    // First launch after install
    await SecureStoragePlugin.clear();
    Preferences.set({ key: "hasLaunchedBefore", value: "true" });
  }
};
```

## Why use `@capacitor/preferences` instead of `localStorage`?

`localStorage` is usually more heavily used and more volatile than `@capacitor/preferences`. This data should not be altered or cleared until the app is uninstalled.

# Takeaway

- Use secure storage for tokens and secrets
- Remember: on iOS, secure data survives app deletion
- For development, Safari DevTools can save the day
- In production, add a **first-launch detector** to automatically reset storage on a fresh install.

That way, you avoid broken states for your users — and headaches for yourself. 🚀
