# End-to-end test: AntennaPod retention plugin

This walks through installing AntennaPod (debug) and this plugin, enabling it, and verifying
that only the newest N downloaded episodes of a podcast survive the automatic cleanup.

Requires a machine with the Android SDK and a device/emulator (`adb`). AntennaPod's debug build has
applicationId `de.danoeh.antennapod.debug`.

## 1. Build & install AntennaPod (debug)

From the AntennaPod repo root:

```
./gradlew :app:installPlayDebug
```

## 2. Build & install the retention plugin

The plugin is a standalone project (its own Gradle build). Easiest path is Android Studio:
**Open** this repository as a project, let it sync, then Run the `app` config.

Or from the command line, in this repository (the bundled Gradle wrapper needs only a
JDK):

```
./gradlew :app:installDebug
```

CI also builds the debug APK on every push/PR via `.github/workflows/build.yml` and uploads it as a
build artifact, so you can download a prebuilt APK from the Actions run instead of building locally.

The plugin app installs as `com.example.antennapodretentionplugin` and, unlike the transcript and
chapter samples, has a launcher icon: that is where the retention settings live.

## 3. Enable the plugin in AntennaPod

Open AntennaPod → **Settings → Plugins**. "AntennaPod Retention Plugin" appears in the list. Toggle it
**on**. (Plugins are disabled by default; discovery also happens live, so the entry appears without
restarting the app.)

## 4. Configure how many episodes to keep

Open the "AntennaPod Retention Plugin" app. Set **All podcasts** to e.g. `2`. Podcasts show up under
**Per podcast** once AntennaPod has asked about them, so you can override individual ones after the
first cleanup run.

## 5. Trigger a cleanup

1. Subscribe to a podcast and download 4 or more of its episodes.
2. Make sure the episodes are neither in the queue nor marked as favorite — those are never deleted.
3. Trigger AntennaPod's automatic cleanup (it runs with the automatic download job; on a test device
   the quickest way is Settings → Automatic Download, enable it, and let the job run, or simply
   refresh and wait for the periodic job).

## 6. Verify

- The two newest episodes are still downloaded, the older ones are not.

Confirm the plugin ran via logcat:

```
adb logcat -d | grep -E "RetentionPlugin|RemoteRetentionPolicy|PluginRetentionCleanup|PluginManager"
```

Expected lines include `Registering retention plugin 'keep-newest-episodes'`, `'<podcast>' has N
downloaded episodes, keeping 2`, and `Retention policies deleted X episodes`.

## Toggling off

Disabling the plugin in Settings makes `RemoteEpisodeRetentionPolicy.shouldApply` return `false`, so
cleanups no longer consult it. Setting the keep count to `0` keeps all episodes. Uninstalling the
plugin app removes it from the list (live, via the package monitor).
