# Random notes that should be moved to a proper documentation place in future

## NOTE (always be aware of these when doing admin for Jellyfin)

* If transcoding is still an issue, go to Server > Users > Edit User on each user and turn off "Allow video playback that requires transcoding". Remuxing and audio transcodes should be fine to leave on.
* Disable real-time monitoring for all libraries! It will not make the slow NFS happy.
* When adding music to Jellyfin, remember that your files will have metadata already embedded into it. Ensure Jellyfin isn't using external sources to collect metadata, and ensure that it isn't overriding any metadata. It might be wise to mount the PVC as read-only in Kubernetes...
* Remember that other media applications share the same PVC as Jellyfin. We may want to figure out how to make Jellyfin mount the PVC as read-only if possible, to prevent any overwrites of metadata and any other possible issues that could arise.

## Changes Applied

Changes made to default configuration:

* Login banner was added for fun. Can be changed from what I have currently to something else.
* In Server > Playback > Transcoding, Throttle Transcodes has been enabled so that a full movie doesn't transcode all at once. It transcodes up to 180 seconds ahead of the current play position, and then pauses. Saves system resources.
* In Advanced > Networking > Server Address Settings, the pod and service CIDRs were added to the "Known Proxies" field: `10.244.0.0/16, 10.96.0.0/12`. That way Jellyfin hopefully will know about the Gateway when deciding if an IP address connecting is a remote connection or not.
* In Advanced > Networking > Network Discovery, Enable Auto Discovery was disabled because the Jellyfin helm chart doesn't have a service that exposes that port, and I decided not to make the service and UDPRoute and the additional Gateway listener.

I probably don't need to document these changes, but I don't see much harm in doing so.

## Plugins

Installed plugins and sources:

* AudioDB (default installed)
* ListenBrainz Similarity Provider (default installed)
* MusicBrainz (default installed)
* OMDb (default installed)
* Studio Images (default installed)
* TMDb (default installed)
* [AnimeMultiSource](https://github.com/webbster64/jellyfin-plugin-AnimeMultiSource) [manifest "AnimeMultiSource"](https://raw.githubusercontent.com/webbster64/jellyfin-plugin-AnimeMultiSource/main/manifest.json) (Anime metadata)
* Chapter Segments Provider (default repo)
* LrcLib Lyrics (default repo) (for song lyrics)
* Playback Reporting (default repo)
* Reports (default repo)
* Subtitle Extract (default repo)
* TMDb Box Sets (default repo) (I believe this is the thing making the nice movie collections)
* Webhook (default repo) (To get NTFY notifications for certain things)
* [Intro Skipper](https://github.com/intro-skipper/intro-skipper) ([manifest "intro-skipper"](https://intro-skipper.org/manifest.json)) (Skip button for intros!)
* [InPlayerEpisodePreview](https://github.com/Namo2/InPlayerEpisodePreview) ([manifest "InPlayerEpisodePreview"](https://raw.githubusercontent.com/Namo2/InPlayerEpisodePreview/master/manifest.json)) (Requires File Transformation)
* [File Transformation](https://github.com/IAmParadox27/jellyfin-plugin-file-transformation) ([manifest "File Transformation"](https://www.iamparadox.dev/jellyfin/plugins/manifest.json))
* [Media Bar](https://github.com/IAmParadox27/jellyfin-plugin-media-bar) (Requires File Transformation)

## Webhooks & ntfy

The Webhook plugin is capable of sending ntfy notifications for certain events.

The ntfy template from [jellyfin/jellyfin-plugin-webhook/Jellyfin.Plugin.Webhook/Templates](https://github.com/jellyfin/jellyfin-plugin-webhook/tree/master/Jellyfin.Plugin.Webhook/Templates) was taken and modified so that it could be used with my ntfy instance.

### Admin ntfy notifications

Webhook URL: `http://ntfy-svc.monitoring.svc.cluster.local`

Template:

```json
{
    "topic": "jellyfin-admin",

    {{#if_equals NotificationType 'PlaybackStart'}}
        "priority": 1,
        "tags": ["arrow_forward"],
        "attach": "{{{ServerUrl}}}Items/{{{ItemId}}}/Images/Primary",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/#/details?id={{ItemId}}" }],
            {{#if_equals ItemType 'Audio'}}
                "title": "{{{NotificationUsername}}} | Playback started: {{{Artist}}} - {{{Name}}} ({{Year}})",
                "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Action:** Playback started\n**- Play Method:** {{{PlayMethod}}}\n**- Playback Position:** {{{PlaybackPosition}}}\n\n**- Artist:** {{{Artist}}}\n**- Track:** {{{Name}}}\n**- Album:** {{{Album}}} ({{Year}})\n**- Runtime:** {{RunTime}}"
            {{else}}
            {{#if_equals ItemType 'Episode'}}
                "title": "{{{NotificationUsername}}} | Playback started: {{{SeriesName}}} ({{Year}}) - S{{SeasonNumber00}}E{{EpisodeNumber00}}",
                "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Action:** Playback started\n**- Play Method:** {{{PlayMethod}}}\n**- Playback Position:** {{{PlaybackPosition}}}\n\n**- Series:** {{{SeriesName}}} ({{Year}})\n**- Episode:** S{{SeasonNumber00}}E{{EpisodeNumber00}} - {{{Name}}}\n**- Runtime:** {{RunTime}}\n\n**- Description:**\n{{{Overview}}}"
            {{else}}
                "title": "{{{NotificationUsername}}} | Playback started: {{{Name}}} ({{Year}})",
                "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Action:** Playback started\n**- Play Method:** {{{PlayMethod}}}\n**- Playback Position:** {{{PlaybackPosition}}}\n\n**- Movie:** {{{Name}}} ({{Year}})\n**- Runtime:** {{RunTime}}\n\n**- Description:**\n{{{Overview}}}"
            {{/if_equals}}
            {{/if_equals}}
    {{/if_equals}}

    {{#if_equals NotificationType 'PlaybackStop'}}
        "priority": 1,
        "tags": ["stop_button"],
        "attach": "{{{ServerUrl}}}Items/{{{ItemId}}}/Images/Primary",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/#/details?id={{ItemId}}" }],
            {{#if_equals ItemType 'Audio'}}
                "title": "{{{NotificationUsername}}} | Playback stopped: {{{Artist}}} - {{{Name}}} ({{Year}})",
                "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Action:** Playback stopped\n**- Played To Completion:** {{{PlayedToCompletion}}}\n**- Playback Position:** {{{PlaybackPosition}}}\n\n**- Artist:** {{{Artist}}}\n**- Track:** {{{Name}}}\n**- Album:** {{{Album}}} ({{Year}})\n**- Runtime:** {{RunTime}}"
            {{else}}
            {{#if_equals ItemType 'Episode'}}
                "title": "{{{NotificationUsername}}} | Playback stopped: {{{SeriesName}}} ({{Year}}) - S{{SeasonNumber00}}E{{EpisodeNumber00}}",
                "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Action:** Playback stopped\n**- Played To Completion:** {{{PlayedToCompletion}}}\n**- Playback Position:** {{{PlaybackPosition}}}\n\n**- Series:** {{{SeriesName}}} ({{Year}})\n**- Episode:** S{{SeasonNumber00}}E{{EpisodeNumber00}} - {{{Name}}}\n**- Runtime:** {{RunTime}}\n\n**- Description:**\n{{{Overview}}}"
            {{else}}
                "title": "{{{NotificationUsername}}} | Playback stopped: {{{Name}}} ({{Year}})",
                "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Action:** Playback stopped\n**- Played To Completion:** {{{PlayedToCompletion}}}\n**- Playback Position:** {{{PlaybackPosition}}}\n\n**- Movie:** {{{Name}}} ({{Year}})\n**- Runtime:** {{RunTime}}\n\n**- Description:**\n{{{Overview}}}"
            {{/if_equals}}
            {{/if_equals}}
    {{/if_equals}}

    {{#if_equals NotificationType 'ItemAdded'}}
        "priority": 1,
        "tags": ["heavy_plus_sign"],
        "attach": "{{{ServerUrl}}}Items/{{{ItemId}}}/Images/Primary",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/#/details?id={{ItemId}}" }],
        {{#if_equals ItemType 'Audio'}}
            "title": "Audio Track Added: {{{Artist}}} - {{{Name}}} | {{{Album}}} ({{Year}})",
            "message": "---\n**- Artist:** {{{Artist}}}\n**- Track:** {{{Name}}}\n**- Album:** {{{Album}}} ({{Year}})\n**- Runtime:** {{RunTime}}\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
        {{else}}
        {{#if_equals ItemType 'MusicAlbum'}}
            "title": "Album Added: {{{Artist}}} - {{{Name}}} ({{Year}})",
            "message": "---\n**- Artist:** {{{Artist}}}\n**- Album:** {{{Name}}} ({{Year}})\n**- Runtime:** {{RunTime}}\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
        {{else}}
        {{#if_equals ItemType 'Movie'}}
            "title": "Movie Added: {{{Name}}} ({{Year}})",
            "message": "---\n**- Movie:** {{{Name}}} ({{Year}})\n**- Runtime:** {{RunTime}}\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
        {{else}}
        {{#if_equals ItemType 'Season'}}
            "title": "Season Added: {{{SeriesName}}} ({{Year}}) - S{{SeasonNumber00}}",
            "message": "---\n**- Series:** {{{SeriesName}}} ({{Year}})\n**- Season:** {{{Name}}}\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
        {{else}}
        {{#if_equals ItemType 'Series'}}
            "title": "Series Added: {{{Name}}} ({{Year}})",
            "message": "---\n**- Series:** {{{Name}}} ({{Year}})\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
        {{else}}
            "title": "Episode Added: {{{SeriesName}}} ({{Year}}) - S{{SeasonNumber00}}E{{EpisodeNumber00}}",
            "message": "---\n**- Series:** {{{SeriesName}}} ({{Year}})\n**- Episode:** S{{SeasonNumber00}}E{{EpisodeNumber00}} - {{{Name}}}\n**- Runtime:** {{RunTime}}\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
        {{/if_equals}}
        {{/if_equals}}
        {{/if_equals}}
        {{/if_equals}}
        {{/if_equals}}
    {{/if_equals}}

    {{#if_equals NotificationType 'PendingRestart'}}
        "title": "Jellyfin Restart Required",
        "priority": 3,
        "tags": ["warning", "arrows_counterclockwise"],
        "message": "---\n**- Server:** {{{ServerName}}}\n**- Version:** {{{ServerVersion}}}\n\nJellyfin needs to be restarted.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'AuthenticationFailure'}}
        "title": "Alert: {{{Username}}} | Authentication Failure",
        "priority": 4,
        "tags": ["warning", "key"],
        "message": "---\n**- User:** {{{Username}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n\nLogin attempt failed. Check whether it was expected.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'AuthenticationSuccess'}}
        "title": "{{{NotificationUsername}}} | Authentication Success",
        "priority": 1,
        "tags": ["white_check_mark", "key"],
        "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Last Login:** {{{LastLoginDate}}}\n**- Last Activity:** {{{LastActivityDate}}}\n\nUser logged in successfully.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'SessionStart'}}
        "title": "{{{NotificationUsername}}} | Session Started",
        "priority": 1,
        "tags": ["arrow_forward"],
        "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n\nA new Jellyfin session started.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'PluginInstalling'}}
        "title": "Jellyfin | Installing Plugin: {{{PluginName}}}",
        "priority": 1,
        "tags": ["package"],
        "message": "---\n**- Plugin:** {{{PluginName}}}\n**- Version:** {{{PluginVersion}}}\n**- Source:** {{{PluginSourceUrl}}}\n\nPlugin installation started."
    {{/if_equals}}

    {{#if_equals NotificationType 'PluginInstallationCancelled'}}
        "title": "Jellyfin | Plugin Installation Cancelled",
        "priority": 1,
        "tags": ["no_entry_sign", "package"],
        "message": "---\n**- Plugin:** {{{PluginName}}}\n**- Version:** {{{PluginVersion}}}\n\nInstallation of this plugin was cancelled."
    {{/if_equals}}

    {{#if_equals NotificationType 'PluginInstallationFailed'}}
        "title": "Jellyfin | Plugin Installation Failed",
        "priority": 1,
        "tags": ["warning", "package"],
        "message": "---\n**- Plugin:** {{{PluginName}}}\n**- Version:** {{{PluginVersion}}}\n**- Error:** {{{ExceptionMessage}}}\n\nPlugin installation failed.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'PluginInstalled'}}
        "title": "Jellyfin | Plugin Installed: {{{PluginName}}}",
        "priority": 1,
        "tags": ["white_check_mark", "package"],
        "message": "---\n**- Plugin:** {{{PluginName}}}\n**- Version:** {{{PluginVersion}}}\n\nPlugin installed. A restart may be required.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'PluginUpdated'}}
        "title": "Jellyfin | Plugin Updated: {{{PluginName}}}",
        "priority": 1,
        "tags": ["arrows_counterclockwise", "package"],
        "message": "---\n**- Plugin:** {{{PluginName}}}\n**- Version:** {{{PluginVersion}}}\n\nPlugin updated. A restart may be required.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'PluginUninstalled'}}
        "title": "Jellyfin | Plugin Uninstalled: {{{PluginName}}}",
        "priority": 1,
        "tags": ["wastebasket", "package"],
        "message": "---\n**- Plugin:** {{{PluginName}}}\n**- Version:** {{{PluginVersion}}}\n\nPlugin uninstalled. A restart may be required.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'UserCreated'}}
        "title": "Jellyfin User Created: {{{NotificationUsername}}}",
        "priority": 3,
        "tags": ["heavy_plus_sign", "bust_in_silhouette"],
        "message": "---\n**- Username:** {{{NotificationUsername}}}\n**- User ID:** {{{UserId}}}\n\nA new Jellyfin user was created.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'UserDeleted'}}
        "title": "Jellyfin User Deleted: {{{NotificationUsername}}}",
        "priority": 3,
        "tags": ["x", "bust_in_silhouette"],
        "message": "---\n**- Username:** {{{NotificationUsername}}}\n**- User ID:** {{{UserId}}}\n\nA Jellyfin user was deleted.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'UserLockedOut'}}
        "title": "CRITICAL: User Locked Out | {{{NotificationUsername}}}",
        "priority": 5,
        "tags": ["rotating_light", "lock"],
        "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n\nUser locked out after too many failed login attempts. Investigate immediately.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}

    {{#if_equals NotificationType 'UserPasswordChanged'}}
        "title": "Jellyfin Password Changed: {{{NotificationUsername}}}",
        "priority": 3,
        "tags": ["warning", "key"],
        "message": "---\n**- Username:** {{{NotificationUsername}}}\n**- User ID:** {{{UserId}}}\n\nThe password for this user was changed.",
        "actions": [{ "action": "view", "label": "Visit Jellyfin", "url": "{{{ServerUrl}}}web/" }]
    {{/if_equals}}
}
```

Request Header:

* Key: `Authorization`
* Value: `Bearer <token>`
* Key: `X-Markdown`
* Value: `true`

## TODO

* https://github.com/CyferShepard/Jellystat
* Create a requests page or similar. Then point Custom Tabs to it.
