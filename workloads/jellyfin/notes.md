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
* MusicBrainz (default installed)
* OMDb (default installed)
* Studio Images (default installed)
* TMDb (default installed)
* [Intro Skipper](https://github.com/intro-skipper/intro-skipper) ([manifest](https://intro-skipper.org/manifest.json)) (Skip button for intros!)
* [File Transformation]() ([manifest](https://www.iamparadox.dev/jellyfin/plugins/manifest.json)) (Optional requirement for Intro Skipper)
* AniList (default repo)
* Custom Tabs
* [EditorsChoice](https://github.com/lachlandcp/jellyfin-editors-choice-plugin/) ([manifest](https://github.com/lachlandcp/jellyfin-editors-choice-plugin/raw/main/manifest.json)) (Fancy featured content slider)
* Media Bar (Might remove it? Need to see what it does.)
* Playback Reporting (default repo)
* Reports (default repo)
* Subtitle Extract (default repo)
* TMDb Box Sets (default repo) (I believe this is the thing making the nice movie collections)
* Webhook (default repo) (I will try to get NTFY notifications with this!!)
* Chapter Segments Provider (default repo)
* [InPlayerEpisodePreview]() ([manifest](Namo2/InPlayerEpisodePreview))

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
        "priority": 2,
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
        "priority": 2,
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
    "priority": 3,
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
            "title": "Series Added: {{Name}} ({{Year}})",
            "message": "---\n**- Series:** {{Name}} ({{Year}})\n**- Status:** Available\n\n**- Description:**\n{{{Overview}}}"
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
        "priority": 4,
        "message": "---\n**- Message:** Jellyfin needs to be restarted, please restart jellyfin as soon as possible!"
    {{/if_equals}}

    {{#if_equals NotificationType 'AuthenticationFailure'}}
        "title": "Alert: {{{Username}}}: Authentication Failure",
        "priority": 5,
        "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Issue:** Login request was denied: Wrong password!"
    {{/if_equals}}

    {{#if_equals NotificationType 'AuthenticationSuccess'}}
        "title": "{{{NotificationUsername}}}: Authentication Success",
        "priority": 3,
        "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Message:** Successfully logged in!"
    {{/if_equals}}

    {{#if_equals NotificationType 'UserLockedOut'}}
        "title": "Alert: {{{NotificationUsername}}}: User Locked Out",
        "priority": 5,
        "message": "---\n**- User:** {{{NotificationUsername}}}\n**- Device/Client:** {{{DeviceName}}} - {{{ClientName}}}\n**- IP Address:** {{{RemoteEndPoint}}}\n**- Issue:** User has been locked out!"
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
