# Codebase map

## app.py

`app.py` is the Flask application factory and entry point for Mixtape. It initializes the Flask app, configures the database, registers API routes, and creates database tables on startup.

**Data flow - App startup:**
`create_app()` builds a Flask instance, loads configuration from environment variables (or defaults like SQLite and a dev secret key), initializes SQLAlchemy with `db.init_app(app)`, registers all route blueprints (`songs`, `playlists`, `users`, `feed`), and finally runs `db.create_all()` inside the app context to ensure tables exist before the app starts handling requests.

**Pattern I noticed:**
The app uses the application factory pattern, meaning the Flask app is created inside a function rather than at import time. All routes are modularized into blueprints, and database setup is centralized in one place, keeping configuration, routing, and initialization cleanly separated.

---

## models.py

`models.py` defines all database entities for Mixtape using SQLAlchemy, including users, songs, playlists, notifications, ratings, listening events, and the association tables that connect them.

At the core are five main models: `User`, `Song`, `ListeningEvent`, `Rating`, and `Playlist`, plus a `Tag` model for song metadata. There are also three association tables: `friendships`, `song_tags`, and `playlist_entries`.

**Data flow - Listening a song:**
When a user listens, a `ListeningEvent` is created linking `user_id` and `song_id` with a timestamp. This is later used by feed services to build both “Friends Listening Now” and activity feeds.

**Data flow - Sharing and interacting with songs:**
A `Song` is created with a `shared_by` user. That same ID is used in services to notify the original sharer when others interact (like adding to playlists). Ratings are stored in `Rating`, with a uniqueness constraint ensuring one rating per user per song.

**Data flow - Playlists:**
A `Playlist` is created by a user and optionally collaborative. Songs are attached through `playlist_entries`, which also stores position, who added the song, and when. The `songs` relationship exposes playlist contents, but ordering is handled via the join table.

**Data flow - Notifications:**
A `Notification` is created for a specific user and stores type, message body, timestamp, and read status. It is used by services to inform users about interactions with their shared songs.

**Pattern I noticed:**
The models are designed around clear relational boundaries with heavy use of association tables for many-to-many relationships. Most entities include a `to_dict()` method to act as a serialization boundary for API responses. Timestamps default to UTC consistently, and relationships are defined so services can stay lightweight and focus on orchestration rather than joins or formatting logic.

---

## feed_service.py

`feed_service.py` handles the app’s social listening feeds. It builds the “Friends Listening Now” feed and a general activity feed from friends’ listening events.

**Data flow - Friends Listening Now:**
`get_friends_listening_now()` loads the user, collects friend IDs, computes a 24-hour cutoff, and queries ListeningEvent records for those friends within that window. It sorts events newest-first, then deduplicates so only the most recent event per friend is kept. For each remaining event, it loads the User and Song, converts them with `to_dict()`, and returns a list of {friend, song listened_at} entries.

**Data flow - Activity feed:**
`get_activity_feed()` loads the user, gets friend IDs, queries all their ListeningEvent records (no time filter), orders them by newest first, applies a limit, and returns a chronological list of events with the same {friend, song, listened_at} structure.

**Pattern I noticed:**
Both functions follow the same pipeline-validate user → collect friend IDs → query ListeningEvent → load related User and Song → serialize results. The main difference is filtering: one enforces a 24-hour window with deduplication, while the other returns a raw recent activity stream with a limit.

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
`get_playlist_songs(playlist_id)` loads the Playlist, joins the Song table with the playlist_entries join table, orders songs by their position, converts each song with `to_dict()`, and returns the ordered list.

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
The service is read-only, using SQLAlchemy to query data without modifying the database. Both functions return JSON-ready dictionaries, with one focused on searching multiple songs and the other on retrieving a single song.

---

## streak_service.py

`streak_service.py` handles users' listening streaks. It records listening events, updates streaks based on consecutive listening days, and retrieves a user's current streak.

**Data flow - Recording a listening event:**
`record_listening_event()` loads the User, creates a new ListeningEvent, calls `update_listening_streak()` to update the user's streak, commits both changes to the database, and returns the new listening event.

**Data flow - Updating a streak:**
`update_listening_streak()` compares today's date with the user's last_listened_at. If it's the user's first listen, the streak starts at 1. If they already listened today, nothing changes. If they listened yesterday, the streak increases by 1. Otherwise, the streak resets to 1, and last_listened_at is updated.

**Pattern I noticed:** The service keeps streak logic in a separate helper function (`update_listening_streak()`), while `record_listening_event()` handles database operations. It validates users before making changes and separates updating streaks from simply retrieving them with `get_streak()`.
