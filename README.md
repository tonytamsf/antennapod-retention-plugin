# AntennaPod retention plugin

A standalone Android app that plugs into [AntennaPod](https://github.com/AntennaPod/AntennaPod) and
keeps only the **newest N episodes of each podcast**, configurable per podcast
([AntennaPod/AntennaPod#2077](https://github.com/AntennaPod/AntennaPod/issues/2077)).

It is its own APK, running in its own process — AntennaPod never loads its code. AntennaPod only offers
the extension points; the rule *and* its per-podcast configuration live here.

The app uses two of those extension points:

- **`FEED_CONTENT`** — the primary mechanism. AntennaPod hands over each downloaded feed document
  before parsing it, and `FeedContentPluginService` returns the same feed cut down to the newest N
  items. Everything older is gone before the app reads the feed, so those episodes are never listed,
  never counted and never downloaded — AntennaPod does not know they exist.
- **`EPISODE_RETENTION`** — the cleanup half. `RetentionPluginService` tells AntennaPod which
  *already downloaded* episodes are past N, so files fetched before the limit was lowered get removed
  during the next automatic cleanup.

Both services share one plugin id (`keep-newest-episodes`) and one N per podcast, so the plugin is
enabled once and takes one number.

Requires an AntennaPod build with both plugin points (`FeedContentProcessor` and
`EpisodeRetentionPolicy` in `:plugin:api`; `ACTION_FEED_CONTENT` and `ACTION_EPISODE_RETENTION` in
`:plugin:host`).

## How integration works

1. **Contract.** The app compiles against the AntennaPod "plugin SDK": the AIDL interfaces
   (`IFeedContentPlugin`, `IEpisodeRetentionPlugin`) and Parcelables (`PluginFeedContentRequest`,
   `PluginFeedContentResult`, `PluginRetentionRequest`, `PluginRetentionResult`, `PluginEpisodeInfo`,
   `PluginContract`) from AntennaPod's `:plugin:host` module. Until an SDK artifact is published, those
   files are vendored under `app/src/main/aidl/` and
   `app/src/main/java/de/danoeh/antennapod/plugin/host/`.
2. **Declaration.** `AndroidManifest.xml` exposes two `Service`s, each with
   `android:permission="de.danoeh.antennapod.permission.PLUGIN"` and the same `<meta-data>` plugin id
   (`keep-newest-episodes`):
   - `FeedContentPluginService` — `<intent-filter>` for
     `de.danoeh.antennapod.plugin.action.FEED_CONTENT`, capability bitmask `8`,
   - `RetentionPluginService` — `<intent-filter>` for
     `de.danoeh.antennapod.plugin.action.EPISODE_RETENTION`, capability bitmask `4`.
3. **Discovery.** AntennaPod finds both services via `PackageManager` and registers them as a
   `FeedContentProcessor` and an `EpisodeRetentionPolicy` (live, via its package monitor, or on next
   launch). They stay disabled until the user enables the plugin in **Settings → Plugins**.
4. **Invocation, feed content.** After downloading a feed and before parsing it, AntennaPod calls
   `processFeed(PluginFeedContentRequest)`. The request carries the feed id and url plus a read-only
   descriptor for the downloaded document and a descriptor to write to. The service parses the XML,
   drops every item past the newest N, and writes the result. AntennaPod parses the rewritten feed, so
   the dropped episodes never enter its database.
5. **Invocation, retention.** During automatic cleanup, AntennaPod calls
   `selectForDeletion(PluginRetentionRequest)` once per subscription. The request carries the feed and
   its downloaded episodes; the service returns the ids of the episodes beyond the newest N. AntennaPod
   deletes those media files and ignores ids it did not offer.

## Settings

The launcher activity (`RetentionSettingsActivity`) configures how many episodes to keep for all
podcasts and, per podcast, an override. Podcasts appear there as AntennaPod asks about them. `0` keeps
all episodes, which effectively disables the rule for that podcast.

The same N drives both services. A podcast the app has never fetched has feed id `0` at feed content
time, so a brand-new subscription uses the "all podcasts" value on its first fetch.

Queued and favorite episodes are never deleted by AntennaPod; they are still part of the retention
request (flagged via `PluginEpisodeInfo#isDeletable()`) so the plugin counts them towards the newest N.

Feed truncation keeps document order when items have no parseable dates, since feeds are conventionally
newest-first; when every item carries a `pubDate` / `published` / `updated`, the newest N are kept
regardless of the order the feed lists them in.

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
