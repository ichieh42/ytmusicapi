---
name: ytmusicapi
description: Write code against the ytmusicapi Python library for YouTube Music, or modify the ytmusicapi source at c:\Users\John\Trae\1786935631\ytmusicapi. Invoke when the user asks to build scripts, search/browse music, manage playlists or library, get lyrics, upload songs, or work with this project.
---

# ytmusicapi

`ytmusicapi` is an unofficial Python 3 library that talks to the YouTube Music API by emulating the YouTube Music web client's requests. It supports both authenticated and unauthenticated requests. This skill covers the public API surface, authentication, and the source layout of the repo.

## Requirements

- Python 3.10+
- Only runtime dependency: `requests >= 2.22`

## Install

```bash
pip install ytmusicapi
```

From the repo source: `pip install -e .` (project root: `c:\Users\John\Trae\1786935631\ytmusicapi`).

## Authentication

`YTMusic` accepts an `auth` argument which may be a string, a path to a file, or an OAuth token dict. Without it, requests are unauthenticated (works for search, browsing, charts, explore; **fails** for library/playlist write operations).

There are three auth modes, resolved automatically from the provided credentials (`ytmusicapi.auth.auth_parse.determine_auth_type`):

1. **Browser headers (currently the only working auth)** — raw request headers copied from a logged-in browser (`setup(headers_raw=...)` or CLI `ytmusicapi browser`). Requires the `__Secure-3PAPISID` cookie. Marked deprecated upstream, but OAuth is broken so this is the practical choice.
2. **OAuth (broken, see issue #813)** — via `setup_oauth(client_id, client_secret, filepath=...)` from `ytmusicapi` or the CLI `ytmusicapi oauth` (`ytmusicapi.setup:main`). Saves a token JSON that refreshes itself (`RefreshingToken`). Since 2025-08-29 the YTM inner API rejects these tokens with HTTP 400 "Request contains an invalid argument" on **all** requests, including `search` (see https://github.com/sigma67/ytmusicapi/issues/813). Do not rely on it. If you still load an OAuth token file, the current code requires the client credentials:
   ```python
   from ytmusicapi import OAuthCredentials, YTMusic
   yt = YTMusic("oauth.json", oauth_credentials=OAuthCredentials(client_id, client_secret))
   ```
   (passing only `YTMusic("oauth.json")` raises `YTMusicUserError`).
3. **Full custom OAuth headers** — used by downstream clients that provide their own complete header set.

Other `YTMusic` init options: `user` (brand account ID), `requests_session` (custom session, e.g. to change the 30s default timeout), `proxies`, `language` (16 supported, default `en`), `location` (per `gl`), `oauth_credentials`.

## API Catalog

### Search (`SearchMixin`)
- `search(query, filter=None, scope=None, limit=20, ignore_spelling=False)` — `filter` ∈ `songs, videos, albums, artists, playlists, community_playlists, featured_playlists, profiles, podcasts, episodes`; `scope` ∈ `library, uploads`. With `scope=library`, `community_playlists`/`featured_playlists` filters are not allowed; `scope=uploads` allows no filter. Results are dicts with `resultType`, and `videoId`/`browseId` depending on type.
- `get_search_suggestions(query, detailed_runs=False)` → `list[str]`
- `remove_search_suggestions(suggestions, indices=None)` → `bool` (auth required)

### Browsing (`BrowsingMixin`)
- `get_home(limit=3)` — home page content sections
- `get_artist(channelId)` — artist overview incl. sections, `params` tokens and `albums` for `get_artist_albums`
- `get_artist_albums(channelId, params, limit=100, order=None)` — `order` ∈ `Recency, Popularity, Alphabetical`; `params` comes from `get_artist`
- `get_user(channelId)` — user/channel info; `get_user_playlists(channelId, params)`, `get_user_videos(channelId, params)` take `params` from `get_user`
- `get_album_browse_id(audioPlaylistId)` → browseId or None
- `get_album(browseId)` — album incl. `tracks`, `artists`, `year`, `duration_seconds`, etc.
- `get_song_credits(browseId)`, `get_song_related(browseId)`, `get_song(videoId, signatureTimestamp=None)` (playback/streaming info)
- `get_lyrics(browseId, timestamps=False)` — `browseId` starts with `MPLYt...` (from `get_watch_playlist`); `timestamps=True` returns `TimedLyrics` on mobile client when available; returns `None` if no lyrics
- `get_tasteprofile()` / `set_tasteprofile(artists, taste_profile=None)` — tune your YT Music profile

### Watch (`WatchMixin`)
- `get_watch_playlist(videoId=None, playlistId=None, limit=25, radio=False, shuffle=False)` — the queue shown when pressing play; each track carries `videoId`, `title`, `length`, `inLibrary`, `feedbackTokens`, and optional `counterpart` and `lyrics` browseId. `shuffle` needs `playlistId`, `radio` ignores `shuffle`. Also gives `listened`/`playingRadio` metadata.

### Charts & Explore (`ChartsMixin`, `ExploreMixin`)
- `get_charts(country="ZZ")` — global charts, incl. `countries`, `trending`, `videos`, `artists` (top artists/songs per country)
- `get_mood_categories()` → categories dict; `get_mood_playlists(params)` uses `params` from the category
- `get_explore()` — browse/moods/genres homepage sections

### Library (`LibraryMixin`) — all auth required
- Getters with optional `limit` and `order` (`order` ∈ `a_to_z, z_to_a, recently_added`): `get_library_songs(limit=25, validate_responses=False, order=None)`, `get_library_albums`, `get_library_artists`, `get_library_subscriptions`, `get_library_podcasts`, `get_library_channels` (all `limit=25, order=None`). Note: `get_library_playlists(limit=25)` takes only `limit` (no `order`).
- `get_history()`; `add_history_item(song)` (takes a search-result song dict); `remove_history_items(feedbackTokens)`
- `rate_song(videoId, rating)` and `rate_playlist(playlistId, rating)` — `rating` is a `LikeStatus` (`LIKE`, `DISLIKE`, `INDIFFERENT`)
- `edit_song_library_status(feedbackTokens)` — add/remove songs from library
- `subscribe_artist(channelId)`, `subscribe_artists(channelIds)`, `unsubscribe_artists(channelIds)`
- `get_account_info()` — channels and profile info

### Playlists (`PlaylistsMixin`) — write ops require auth
- `get_playlist(playlistId, limit=100, related=True, suggestions_limit=0, rich_playlist=False)` — full track list; items contain `videoId`, `setVideoId`, `feedbackTokens`. `playlistId` may start with `VL`.
- `get_liked_songs(limit=100)`, `get_saved_episodes(limit=100)`
- `create_playlist(title, description, privacy_status="PRIVATE", video_ids=None, source_playlist=None)` — `privacy_status` ∈ `PUBLIC, PRIVATE, UNLISTED`; raises if title contains `<` or `>`
- `edit_playlist(playlistId, title=None, description=None, privacyStatus=None, collaboration=None, moveItem=None, addPlaylistId=None, sortOrder=None, addToTop=None, voteOption=None)` — `moveItem` uses `setVideoId`; enabling collaboration returns a `joinCollaborationToken`
- `add_playlist_items(playlistId, videoIds=None, source_playlist=None, duplicates=False)` — needs `videoIds` and/or `source_playlist`; returns new `setVideoId`s
- `remove_playlist_items(playlistId, videos)` — each item needs both `videoId` and `setVideoId`
- `delete_playlist(playlistId)`, `join_collaborative_playlist(playlistId, joinCollaborationToken)`

### Podcasts (`PodcastsMixin`)
- `get_podcast(playlistId, limit=100)` — episodes; `get_episode(videoId)` — full episode data incl. transcript URL
- `get_channel(channelId)`; `get_channel_episodes(channelId, params)` (params from `get_channel`); `get_episodes_playlist(playlist_id="RDPN")`

### Uploads (`UploadsMixin`) — auth required
- `get_library_upload_songs(limit=25, order=None)`, `get_library_upload_albums`, `get_library_upload_artists` (all `limit=25, order=None`)
- `get_library_upload_artist(browseId, limit=25)`, `get_library_upload_album(browseId)`
- `upload_song(filepath)` — uploads a local audio file (mp3/m4a/flac/wav); `delete_upload_entity(entityId)` — delete uploads

### Helpers & utilities
- `ytmusicapi.setup.setup(filepath=None, headers_raw=None)` / `setup_oauth(client_id, client_secret, filepath=None, ...)`; CLI entry point `ytmusicapi` (subcommands `oauth`, `browser`)
- `ytmusicapi.models.content.enums`: `LikeStatus`, `VideoType`, `PlaylistSortOrder`, `VoteStatus`, `PlaylistVoteEditOptions`, `PrivacyStatus`
- Exceptions in `ytmusicapi.exceptions`: `YTMusicUserError` (bad input/auth) and `YTMusicServerError` (HTTP >= 400)
- `YTMusic.as_mobile()` context manager temporarily switches the client to Android to alter responses (e.g. timed lyrics)

## ID conventions

- `videoId` — a song/video
- `browseId` — album/artist/user/lyrics pages; `VL` + playlistId also works as a playlist browseId
- `playlistId` — a playlist (may be prefixed with `VL`)
- `channelId` — artist or user channel
- `setVideoId` — unique id of an item *inside* a playlist (needed for moving/removing items)

## Common Pitfalls

- Library and playlist write methods call `_check_auth()`; without credentials they raise `YTMusicUserError`.
- OAuth auth is broken against the YTM inner API: all requests (even `search`) return HTTP 400 "Request contains an invalid argument". Use browser headers auth instead. See https://github.com/sigma67/ytmusicapi/issues/813
- When pasting Chromium "Copy request headers" into `ytmusicapi browser`, the request line (e.g. `/youtubei/v1/log_event?alt=json` / `music.youtube.com`) can be parsed as junk header keys; the Google front-end then returns an HTML "Error 400 (Bad Request)" page instead of JSON. Remove those extra keys from the generated `browser.json`.
- `remove_playlist_items` silently drops items lacking `videoId`/`setVideoId` — fetch the playlist first and pass the original item dicts.
- `get_lyrics` needs a `browseId` starting with `MPLYt...`; obtain it from `get_watch_playlist` tracks.
- Default session timeout is 30s; pass a custom `requests_session` to change it.
- `as_mobile()` is not thread-safe; it mutates `context` in place and restores it on exit.
- Empty playlists and missing lyrics can return `None`; always check return values.
- Search `scope` restricts allowed `filter` values; invalid combinations throw.
- The watch-playlist `INDIFFERENT` likeStatus may actually mean `DISLIKE` due to ambiguous YTM data.

## Source Layout (for modifying the project)

- `ytmusicapi/ytmusic.py` — `YTMusic`/`YTMusicBase`: session, context, headers, `_send_request`, auth setup
- `ytmusicapi/mixins/` — one file per feature area (search, browsing, watch, charts, explore, library, playlists, podcasts, uploads); `_protocol.py` defines the internal request interface
- `ytmusicapi/parsers/` — response parsing; `navigation.py`/`continuations.py` — navigating JSON keys and paging
- `ytmusicapi/auth/` — `browser.py` (header setup), `oauth/` (device-flow token + refresh)
- `ytmusicapi/models/` — typed models (`lyrics.py`, `content/enums.py`)
- Tests in `tests/` use fixtures from `tests/data/`; run with `pytest` (see `pyproject.toml`, which also configures `ruff` and `mypy` with `strict = true`)

## Documentation

- Docs: https://ytmusicapi.readthedocs.io — see `usage.rst` (setup), `reference/` (per-feature method docs), `faq.rst` (supported languages/locations)
- Repo: https://github.com/sigma67/ytmusicapi
- The `tests/` directory is a good source of realistic usage examples.
