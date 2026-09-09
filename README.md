# Syncthing Live Update

Android 16 introduced **Live Updates**: promoted ongoing notifications that surface in the status bar chip, on the lock screen and on Always-on Display. Syncthing already knows how far along a sync is — it simply has no way to say so in that format. This module supplies the missing part, without adding a notification of its own.

## What it does

- Promotes Syncthing's sync progress to a Live Update, visible in the status bar chip, on the lock screen and on Always-on Display.
- Rewrites the notification Syncthing already posts instead of adding a second one. Its title, icon, Exit action and tap target are kept, and it stays the foreground-service notification the app depends on.
- Draws a `Notification.ProgressStyle` bar for overall sync completion, and a percentage in the status bar chip through `setShortCriticalText`.
- Names the folders currently transferring on a second line, using the labels configured in Syncthing rather than folder ids. Past two, the remainder is shown as a count.
- Badges the status icon with the transfer direction: down while this device pulls from a remote, up while a remote pulls from this device, and both when each is happening at once.
- Reads every figure from the host's own callbacks. No REST polling, no API key, no timers.
- Passes the notification through untouched when nothing is transferring, so idle behaviour is exactly what Syncthing ships with.
- Carries no native library, no DEX-search dependency and no background service.

## Scope

The module hooks two processes, and **both are required**:

```text
system
android
com.github.catfriend1.syncthingfork
```

Without the system server scope the notification is rewritten but never promoted, and no chip appears. The system server scope generally has to be confirmed by hand in the manager. Do not add other applications.

## Requirements

- **Android 16 (API 36) or newer.** Live Updates do not exist before that. Developed and validated on Android 17.
- **Root.** [Vector](https://github.com/JingMatrix/Vector) with Magisk or KernelSU and Zygisk enabled, implementing the modern Xposed API version 101 or newer.
  LSPatch cannot run this module: a rootless patch injects into the target app only, and the promotion flag has to be restored inside `system_server`.
- [Syncthing-Fork](https://github.com/researchxxl/syncthing-android) (`com.github.catfriend1.syncthingfork`).

## Installation

1. Install the APK from the release attached to this repository.
2. Enable **Syncthing Live Update** in the framework manager.
3. Grant both scope entries: the system server **and** Syncthing-Fork.
4. **Reboot.** The `system_server` hook is installed while the system server starts, so it cannot take effect until the next boot.
5. Framework log and logcat entries are tagged:

```text
SyncthingLiveUpdate
```

With the Vector CLI, steps 2 and 3 are:

```sh
su -c '/data/adb/lspd/cli modules enable io.github.mrxsin.syncthingliveupdate'
su -c '/data/adb/lspd/cli scope set io.github.mrxsin.syncthingliveupdate system/0 com.github.catfriend1.syncthingfork/0'
```

## How it works

Two problems have to be solved, and they live in different processes.

### In the Syncthing process

`NotificationHandler.updatePersistentNotification(SyncthingService, Boolean, int, int)` is where the host hands over the figures it has just recalculated in `RestApi.onTotalSyncCompletionChange()`. Observing that one method yields the overall completion percentage without a single REST call. The values are read *before* the call proceeds, because the host posts its notification inside it.

Direction comes from the two figures that percentage is derived from: `LocalCompletion.getTotalFolderCompletion()` falls below 100 while this device pulls, `RemoteCompletion.getTotalDeviceCompletion()` falls below 100 while remotes pull from it, and both below 100 means both directions at once.

Folder names come from `LocalCompletion.setFolderStatus(String, Boolean, FolderStatus)`, paired with the labels the host caches in `updateFromConfig(List<Folder>)`. A folder counts as transferring in the `syncing` and `sync-preparing` states — not while it is merely `scanning`, `scan-waiting` or `starting`, which is why a plain rescan never puts a folder name on the notification.

The rewrite itself happens on `Service.startForeground(...)`, filtered to the host's persistent channel. `Notification.Builder.recoverBuilder(...)` reopens the notification the host just built, so nothing is reconstructed from scratch and the module only adds what promotion requires.

### In `system_server`

Android sets `FLAG_PROMOTED_ONGOING` only for a package holding `android.permission.POST_PROMOTED_NOTIFICATIONS`. Syncthing-Fork does not declare it, and an install-time permission cannot be granted after the fact — `pm grant` will not do it. The module therefore restores that single flag inside `NotificationManagerService.fixNotificationWithChannel(...)`, and only when the package is Syncthing-Fork **and** the channel is the module's own. Every other notification on the device, including Syncthing's own, keeps the platform's verdict. Nothing else about the platform's promotion policy is touched.

### The channel

The host posts its persistent notification on an `IMPORTANCE_MIN` channel, and the platform refuses to promote anything posted there. During a transfer the notification is moved to `05_syncthing_live_update` at `IMPORTANCE_LOW`; when the sync completes it returns to `01_syncthing_persistent` and behaves exactly as before.

### A note on Android versions

The rule for what may be promoted changed between releases, and the two are mutually exclusive:

| Release | `hasPromotableCharacteristics()` requires |
| --- | --- |
| Android 16 | `setColorized(true)` |
| Android 17 | `setRequestPromotedOngoing(true)`, and **not** colorized |

The module never requests colorized, and sets the promotion request reflectively so the call is simply skipped on Android 16.

Every hook fails open: one that cannot install is logged and skipped, and Syncthing is left exactly as it was.

## Validation

Validated on a Pixel 8 Pro running Android 17 (SDK 37) with KernelSU, Vector 2.2 and Syncthing-Fork 2.1.3.0. During a live transfer the notification was confirmed as a single record carrying:

```text
channel=05_syncthing_live_update
flags=ONGOING_EVENT|ONLY_ALERT_ONCE|NO_CLEAR|FOREGROUND_SERVICE|PROMOTED_ONGOING
```

and reverting to `01_syncthing_persistent` without the promotion flag once the sync completed.

## Troubleshooting

| Problem | Suggested action |
| --- | --- |
| Progress bar appears but no chip | The system server scope is missing, or the device has not been rebooted since it was granted. That hook is only installed at boot. |
| Nothing changes at all during a sync | Check logcat for `SyncthingLiveUpdate`. `Host notification method not found` means a Syncthing update renamed the method the module observes. |
| Chip shows the icon but no percentage | Expected while Syncthing itself is the visible app, or while the notification is pinned as a heads-up. SystemUI collapses the chip to an icon in both cases; the percentage returns once another app is in front. |
| Chip and the status bar clock overlap or flicker | Another Xposed module is redrawing the status bar and colliding with the chip animation. The notification itself is unaffected. |
| Progress shows but no folder name | Either only a remote is pulling from this device, so no local folder enters a transferring state, or the log reports `Folder names unavailable`, meaning a Syncthing update moved the folder model. |
| Direction badge never appears | Look for `Transfer direction unavailable` in the log. Progress still works without it. |

## Credits

| Project | Contribution |
| --- | --- |
| [Syncthing-Fork](https://github.com/researchxxl/syncthing-android) | The host application whose sync figures and notification this module builds on. MPL-2.0. |
| [Vector](https://github.com/JingMatrix/Vector) by JingMatrix | The ART hooking framework used in both the app process and the system server. GPL-3.0. |
| [libxposed API](https://github.com/libxposed/api) | The modern Xposed module API this module compiles against. Apache-2.0. |

## Disclaimer

Not affiliated with, endorsed by, or sponsored by the Syncthing project or the maintainers of Syncthing-Fork. Provided for educational and personal use. Host app updates or Android platform changes may break the hooks without notice.
