---
title: "How building, syncing and running the app works in Capacitor?"
date: 2025-09-10T13:00:00+02:00
authors: ["Mikołaj Wilczek"]
---

Using advanced features of Capacitor requires you to understand exactly how the app is built and synced. In this article, we’ll briefly go through how a Capacitor app is built, synced, and run — to demystify the process.

![Cover](/generic/building-syncing-in-capacitor.png)

# Building

The app has two main parts:

- native part (it is platform specific code + configuration)
- web part

## Native part

Capacitor creates iOS and Android projects automatically.
In basic cases, this is only the shell containing all the needed native logic called from JS. It provides the app icon, title, and required permissions that the app can use.
In advanced cases, this is the place where you can make your webapp a real mobile app. Adding system capabilities, connecting with other applications, using Bluetooth, health sensors and even utilising the onboard AI in the future. Some of these capabilities are possible using libraries. Others are "inaccessible" from JS, meaning you need to declare it in native languages (like rich notifications).

### Where are native files

- in node modules (packages have iOS and Android folders which are linked to native build)
- in ios / android folder of your app

### Gradle and Pods

Native dependencies are managed by platform-specific package managers:

- For Android, it is the Gradle: the build tool and the dependency manager
- For iOS - it is Cocoapods, build on ruby (that's why ruby is needed for iOS projects). This third party package manager.

There are plans to replace Cocoapods with SwiftPM for iOS, but this is not production ready (both for lib makers and end products).

## Web part

This part is similar to bundled site served through a server, but in that case your app is like a server. All your web dependencies, js code and logic will eventually land there. You will build it like any other webapp. Bundle (using build tools Vite/Webpack) is the input for the Capacitor.

## Where are web files

- on iOS, `<project root>/ios/App/App/public`
- on Android `<project root>/android/app/src/main/assets/public`

There are a few additional, _special_ configuration files located next to the `public` folder

- `capacitor.config.json` - this file is used as a configuration file for the whole app
- `capacitor.plugins.json`- this file is used as to load plugins on Android
- `config.xml` - for iOS, this is a compatibility layer for Cordova plugins, mostly empty for new projects

### Internal url

Your app is served on:

- `capacitor://localhost` for iOS
- `https://localhost` for Android. `http` was used as default beforehand. Capacitor 6 defaults to HTTPS because some modern web APIs require a secure context.

### Dev server mode

If you use dev server, you can define the URL which will serve the webapp (just like any other site).
It boosts the development as you don't need to rebuild the whole app. It works exactly like in the web development.
Moreover, you can test native capabilities (you can't do it in the browser). The only downside is you need to ensure your dev server is resolved and reachable by device (unless you are working with iOS simulator). There could be certificate trust issues if your development server uses https.

#### Why not use the dev server in production

It sounds promising, but there are some factors that should be taken into account:

- no offline support
- security issues
- there are some cases (like service workers) that make some of the development problematic
- load time

Fortunately, you can utilize "LiveUpdate" feature, which takes the best from two solutions. You can have offline support with the possibility to deploy changes quickly. You can either use a ready solution or create your own using (capacitor-updater)[https://github.com/Cap-go/capacitor-updater]

LiveUpdate mechanism works by checking the server for the new version. If present - it is downloaded. Then the web bundle path is changed to the downloaded one and the web part is restarted - the native app remains intact (user experiences a short UI reload).

## Syncing

The most mysterious command is in fact one of the simplest. Basically it:

- Moves built web files (path defined in capacitor config) to iOS/Android web locations
- Generates `capacitor.config.json` from `capacitor.config.ts` (you can use envs to adjust for current built)
- Links capacitor and cordova plugins:
  - Links iOS dependencies to XCode's workspace using Cocoapods
  - Creates necessary gradle files to link Android dependencies
  - Creates cordova configs

**Note:** Capacitor `sync` does not produce the web build, it only copies it. You need to build the webapp by yourself.

# Takeaways

- there is no magic in the whole process
- native files are minimized, but still essential
- web bundle is built by your preferred tool
- syncing links dependencies and copies web bundles to native
- Capacitor projects require understanding both web and native layers
- mastering syncing is the key to smooth development
