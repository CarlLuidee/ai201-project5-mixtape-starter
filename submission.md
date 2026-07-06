# AI Usage

Claude was used to help in the debugging process by helping me understand each component of the code. I provided Claude with the code for a line or function I was unfamiliar with and asked it to explain what each component means and executes. In particular, Claude helped me understand how `get_playlist_songs()` functions. However, it is not perfect. It had trouble giving me a clear explanation of `search_songs()`, so I had to ask it to explain each part of what makes up the results list and piece it together myself. I also asked Claude to assist me in improving the codebase map and root cause analysis sections to convey my findings and actions more effectively.

# Codebase map

---

## app.py

`app.py` is the Flask application factory and entry point for Mixtape. It initializes the Flask app, configures the database, registers API routes, and creates database tables on startup.

**Data flow - App startup:**
`create_app()` builds a Flask instance, loads configuration from environment variables, initializes SQLAlchemy with `db.init_app(app)`, registers all route blueprints, and finally runs `db.create_all()` inside the app context to ensure tables exist before the app starts handling requests.

**Pattern I noticed:**
The app uses the application factory pattern, where the Flask app is created inside a function rather than at import time. All routes are modularized into blueprints, and database setup is centralized in one place, keeping configuration, routing, and initialization cleanly separated.

---

## models.py

`models.py` defines all database entities for Mixtape using SQLAlchemy, including users, songs, playlists, notifications, ratings, listening events, and the association tables that connect them.

At the core are five main models: User, Song, ListeningEvent, Rating, and Playlist, plus a Tag model for song metadata. There are also three association tables: friendships, song_tags, and playlist_entries.

**Data flow - Listening to a song:**
When a user listens, a ListeningEvent is created linking user_id and song_id with a timestamp. This is later used by feed services to build both "Friends Listening Now" and activity feeds.

**Data flow - Sharing and interacting with songs:**
A Song is created with a shared_by user. That same ID is used in services to notify the original sharer when others interact (like adding to playlists). Ratings are stored in Rating, with a uniqueness constraint ensuring one rating per user per song.

**Data flow - Playlists:**
A Playlist is created by a user, optionally collaboratively. Songs are attached via playlist_entries, which also store the position, the user who added the song, and when it was added. The songs relationship exposes playlist contents, but ordering is handled via the join table.

**Data flow - Notifications:**
A Notification is created for a specific user and stores type, message body, timestamp, and read status. It is used by services to inform users about interactions with their shared songs.

**Pattern I noticed:**
The models are designed around clear relational boundaries with heavy use of association tables for many-to-many relationships. Most entities include a `to_dict()` method to act as a serialization boundary for API responses. Timestamps default to UTC consistently, and relationships are defined so services can stay lightweight and focus on orchestration rather than joins or formatting logic.

---

## feed_service.py

`feed_service.py` handles the app's social listening feeds. It builds the "Friends Listening Now" feed and a general activity feed from friends' listening events.

**Data flow - Friends Listening Now:**
`get_friends_listening_now()` loads the user, collects friend IDs, computes a 24-hour cutoff, and queries ListeningEvent records for those friends within that window. It sorts events newest-first, then deduplicates, keeping only the most recent event per friend. For each remaining event, it loads the User and Song, converts them with `to_dict()`, and returns a list of {friend, song listened_at} entries.

**Data flow - Activity feed:**
`get_activity_feed()` loads the user, gets friend IDs, queries all their ListeningEvent records (no time filter), orders them by newest first, applies a limit, and returns a chronological list of events with the same {friend, song, listened_at} structure.

**Pattern I noticed:**
Both functions follow the same pipeline: validate user → collect friend IDs → query ListeningEvent → load related User and Song → serialize results. The main difference is filtering: one enforces a 24-hour window with deduplication, while the other returns a raw recent activity stream with a limit.

---

## notification_service.py

`notification_service.py` handles notification and song interaction logic. It creates notifications, adds songs to playlists, saves song ratings, retrieves notifications, and marks notifications as read.

**Data flow - Adding a song to a playlist:**
`add_to_playlist()` loads the Song, User, and Playlist, adds the song if it isn't already in the playlist, then calls `create_notification()` to create a notification for the user who originally shared the song, unless they added it themselves.

**Data flow - Rating a song:**
`rate_song()` validates the rating, loads the Song and User, checks if a Rating already exists, updates it if it does or creates a new one if it doesn't, commits the changes, and returns the rating. Unlike `add_to_playlist()`, this function does not create a notification.

**Data flow - Retrieving notifications:**
`get_notifications()` queries the Notification table, optionally filters for unread notifications, orders them by newest first, converts them with `to_dict()`, and returns the list.

**Pattern I noticed:**
The service validates records before making changes, keeps notification creation in a reusable `create_notification()` helper, and separates notification creation, song interactions, notification retrieval, and read-status updates into individual functions.

---

## playlist_service.py

`playlist_service.py` handles playlist management. It creates playlists, retrieves playlist metadata, returns a playlist's songs in order, and lists all playlists created by a user.

**Data flow - Creating a playlist:**
`create_playlist()` first checks that the user exists, creates a new Playlist object, saves it to the database, and returns the created playlist.

**Data flow - Getting playlist songs:**
`get_playlist_songs(playlist_id)` loads the playlist, joins the Song table with the playlist_entries join table, orders songs by their position, converts each song with `to_dict()`, and returns the ordered list.

**Pattern I noticed:**
Each function validates that required records exist before querying, uses SQLAlchemy for all database access, and returns JSON-ready dictionaries with `to_dict()`. The service separates playlist metadata (`get_playlist()`) from playlist contents (`get_playlist_songs()`).

---

## search_service.py

`search_service.py` handles song search and retrieval. It searches songs by title or artist and retrieves a single song by its ID.

**Data flow - Searching for songs:**
`search_songs()` queries the Song table, performs a case-insensitive search on the song title and artist using `ilike()`, joins the song_tags table so tag data is available, converts matching songs with `to_dict()`, and returns the results.

**Data flow - Getting a song:**
`get_song()` loads the Song by ID, raises an error if it doesn't exist, converts it with `to_dict()`, and returns the song data.

**Pattern I noticed:**
The service is read-only, using SQLAlchemy to query the database without modifying it. Both functions return JSON-ready dictionaries: one for searching multiple songs and the other for retrieving a single song.

---

## streak_service.py

`streak_service.py` handles users' listening streaks. It records listening events, updates streaks based on consecutive listening days, and retrieves a user's current streak.

**Data flow - Recording a listening event:**
`record_listening_event()` loads the user, creates a new ListeningEvent, calls `update_listening_streak()` to update the user's streak, commits both changes to the database, and returns the new listening event.

**Data flow - Updating a streak:**
`update_listening_streak()` compares today's date with the user's last_listened_at. If it's the user's first listen, the streak starts at 1. If they have already listened today, nothing changes. If they listened yesterday, the streak increases by 1. Otherwise, the streak resets to 1, and last_listened_at is updated.

**Pattern I noticed:** The service keeps streak logic in a separate helper function (`update_listening_streak()`), while `record_listening_event()` handles database operations. It validates users before making changes and separates updating streaks from simply retrieving them with `get_streak()`.

---

## Bug reproduction

**Bug 1:**
The test fails when the second consecutive day is a Sunday. 

The test was run using the command line: `pytest tests/test_streaks.py -v`

The bug can be reproduced when the listening streak is updated for the second time on a Sunday.

**Bug 2:**
The test fails because a song can be returned several times.

The test was run using the command line: `pytest tests/test_search.py -v`

The bug can be reproduced with a song with multiple tags.

**Bug 3:**
The test fails because the last song in the playlist is always omitted.

The test was run using the command line: `pytest tests/test_playlists.py -v`

The bug can be reproduced with a list of any length.

---

# Root cause analysis

## Bug 1

**Issue number and title**

Issue 1: My listening streak keeps resetting

**How you reproduced it**

The bug was reproduced by running `test_streaks.py`. `test_streak_increments_on_sunday()` failed, revealing the bug.

**How you found the root cause**

`test_streak_increments_on_sunday()` from `test_streaks.py` was first checked, then narrowed down to `update_listening_streak()` from `streak_service()`.

**The root cause**

The bug was caused by the special Sunday condition in the `update_listening_streak()` function, which reset the streak whenever the second consecutive day was a Sunday.

**Your fix and side-effect check**

The fix was to remove the `and today.weekday() != 6` part of the condition from the function. The streak tests were rerun to confirm the bug was fixed, along with the other tests, to ensure the change didn't negatively affect the rest of the code.

## Bug 2

**Issue number and title**
Issue 3: Search results show the same song multiple times

**How you reproduced it**
The bug was reproduced by running `test_search.py`. `test_search_no_duplicates_multi_tag_song()` failed, revealing the bug.

**How you found the root cause**
`test_search_no_duplicates_multi_tag_song()` pointed to `search_songs()` in `search_service.py`.

**The root cause**
The `outerjoin()` in `search_songs()` causes one result row per tag associated with a song. Without a `.distinct()` call after the join, a song with N tags is returned N times instead of once.

**Your fix and side-effect check**
The fix was to add `.distinct()` to the query chain in `search_songs()`, right before `.all()`. `test_search.py` was rerun afterward to confirm that all tests passed, along with the other tests, to ensure the change didn't negatively affect the rest of the code.

## Bug 3

**Issue number and title**
Issue 5: My playlist is missing the last song

**How you reproduced it**
The bug was reproduced by running `test_playlists.py`. Two tests failed: `test_playlist_returns_all_songs()` and `test_playlist_returns_songs_in_order()` which returned incomplete lists.

**How you found the root cause**
Both failed tests pointed to `get_playlist_songs()` in `playlist_service.py`.

**The root cause**
The final return statement sliced the songs list with `songs[:-1]`, which always omits the last song, regardless of the playlist's length.

**Your fix and side-effect check**
The fix was to remove `[:-1]` from the return, resulting in: `return [song.to_dict() for song in songs]`. `test_playlists.py` was rerun afterward to confirm that all tests passed, ensuring the fix didn't break the empty-playlist edge case.