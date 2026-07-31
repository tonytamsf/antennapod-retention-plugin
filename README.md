# AntennaPod retention plugin

A standalone Android app that plugs into [AntennaPod](https://github.com/AntennaPod/AntennaPod) and
keeps only the **newest N downloaded episodes of each podcast**, configurable per podcast
([AntennaPod/AntennaPod#2077](https://github.com/AntennaPod/AntennaPod/issues/2077)).

It is its own APK, running in its own process — AntennaPod never loads its code. AntennaPod only offers
an `EPISODE_RETENTION` extension point and asks installed plugins, during its automatic cleanup, which
downloaded episodes of a subscription are no longer needed. The rule *and* its per-podcast
configuration live here.

Requires an AntennaPod build that has the plugin system including the episode retention plugin point
(`EpisodeRetentionPolicy` in `:plugin:api`, `ACTION_EPISODE_RETENTION` in `:plugin:host`).

## How integration works

1. **Contract.** The app compiles against the AntennaPod "plugin SDK": the AIDL interface
   (`IEpisodeRetentionPlugin`) and Parcelables (`PluginRetentionRequest`, `PluginRetentionResult`,
   `PluginEpisodeInfo`, `PluginContract`) from AntennaPod's `:plugin:host` module. Until an SDK artifact
   is published, those files are vendored under `app/src/main/aidl/` and
   `app/src/main/java/de/danoeh/antennapod/plugin/host/`.
2. **Declaration.** `AndroidManifest.xml` exposes a `Service` with:
   - an `<intent-filter>` for `de.danoeh.antennapod.plugin.action.EPISODE_RETENTION`,
   - `android:permission="de.danoeh.antennapod.permission.PLUGIN"`,
   - `<meta-data>` giving the plugin id (`keep-newest-episodes`) and capability bitmask (`4` = episode
     retention).
3. **Discovery.** AntennaPod finds this service via `PackageManager` and registers it as an
   `EpisodeRetentionPolicy` (live, via its package monitor, or on next launch). It stays disabled until
   the user enables it in **Settings → Plugins**.
4. **Invocation.** During automatic cleanup, AntennaPod calls `selectForDeletion(PluginRetentionRequest)`
   once per subscription. The request carries the feed and its downloaded episodes; the service returns
   the ids of the episodes beyond the newest N. AntennaPod deletes those media files and ignores ids it
   did not offer.

## Settings

The launcher activity (`RetentionSettingsActivity`) configures how many episodes to keep for all
podcasts and, per podcast, an override. Podcasts appear there as AntennaPod asks about them. `0` keeps
all episodes, which effectively disables the rule for that podcast.

Queued and favorite episodes are never deleted by AntennaPod; they are still part of the request
(flagged via `PluginEpisodeInfo#isDeletable()`) so the plugin counts them towards the newest N.

## Building

A Gradle wrapper is bundled, so `./gradlew :app:assembleDebug` works with just a JDK and the Android
SDK. CI (`.github/workflows/build.yml`) also builds the debug APK on every push/PR and uploads it as an
artifact, so a prebuilt APK can be downloaded from the Actions run instead of building locally.

```
./gradlew :app:installDebug
# then in AntennaPod: Settings → Plugins → enable "AntennaPod Retention Plugin"
```

`TESTING.md` has the full end-to-end runbook.

## Security

The service is protected by the `de.danoeh.antennapod.permission.PLUGIN` permission, and AntennaPod
runs it only after the user enables it in **Settings → Plugins**. It runs in its own OS-sandboxed
process and never sees any episode file: the request contains only metadata, and the plugin can only
ask for episodes of the feed AntennaPod asked about.
