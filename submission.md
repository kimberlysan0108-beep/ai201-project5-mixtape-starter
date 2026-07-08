# Mixtape Project Submission

## AI Usage Disclosure
I utilized an AI collaborator to assist with codebase onboarding by generating high-level structural summaries and tracing execution flows. During the debugging phase, I used AI to cross-reference Python's date boundary logic and compare working notification dispatches with missing ones. Every fix and hypothesis was manually tested and verified within a local development environment before committing.

---

## Codebase Map

### Core Architecture
The Mixtape codebase follows a clean **Separation of Concerns (SoC)** design pattern split into three distinct layers:
1. **Routing Layer (`app/routes/` or `routes/`):** Handles incoming HTTP requests, parses user input arguments, and returns JSON payloads with matching HTTP status codes.
2. **Service Layer (`services/`):** The engine room where business logic, state changes, validations, and database commits actually happen.
3. **Data Layer (`models.py`):** Defines the database structure using SQLAlchemy ORM models (Users, Songs, Playlists, etc.).

### Main Files & Responsibilities
*   `models.py`: Tracks data relationships. It includes a custom junction table `PlaylistSong` that uses an explicit `order` column to keep tracks in sequence instead of relying on database creation order.
*   `services/streak_service.py`: Responsible for profile management, authentication flags, and maintaining active user listening streaks.
*   `services/feed_service.py`: Generates activity content streams for friends and determines what shows up on a user's dashboard.
*   `services/playlist_service.py`: Manages creating playlists, adding tracks, and ordering items in lists.
*   `services/song_service.py`: Handles catalog searching, track indexing, and processing ratings.

### Sample Data Flow
When a user adds a song to a playlist via `POST /playlists/<playlist_id>/songs`:
1. The request hits the route handler, which extracts the `song_id`.
2. The route controller instantly delegates the operation by calling `playlist_service.add_song_to_playlist()`.
3. The service queries the current song count to increment the order property, commits the new record to the database, and then safely returns the response back to the route.

---

## Root Cause Analysis & Fixes

### Issue #1 — My listening streak keeps resetting
*   **How you reproduced it:** Looked at Kenji's user report detailing that streaks dropped back to 1 exclusively on Sunday mornings despite consecutive daily listening. Traced the calculation logic under `services/streak_service.py`.
*   **How you found the root cause:** Navigated to `services/streak_service.py` and inspected the `update_listening_streak` function. Found a conditional check appending `and today.weekday() != 6` onto a valid consecutive day validation.
*   **The root cause:** In Python, `datetime.weekday()` returns `6` for Sunday. Because of the hardcoded `and today.weekday() != 6` check, when a user listened on Sunday after listening on Saturday (`days_since_last == 1`), the condition evaluated to `False`. This bypassed the streak increment and dropped straight into the `else` block, which reset the user's streak to `1`.
*   **Your fix and side-effect check:** Removed the restrictive `and today.weekday() != 6` clause entirely, changing the condition to `elif days_since_last == 1:`. This ensures streaks increment on every consecutive calendar day. Verified that weekday activities (Monday-Saturday) are completely untouched and continue to calculate sequentially.

### Issue #2 — Friends Listening Now shows people from yesterday
*   **How you reproduced it:** Reviewed Nova's user report noting that songs played late yesterday evening (e.g., 11:00 PM) lingered in the "Listening Now" section at 9:00 AM the following morning.
*   **How you found the root cause:** Inspected `services/feed_service.py` and examined the `cutoff` calculation in `get_friends_listening_now`.
*   **The root cause:** The feed used a static rolling window defined by `RECENT_THRESHOLD = timedelta(hours=24)`. Subtracting 24 hours from the current time meant that any tracks played yesterday remained eligible for the feed until a full 24 hours elapsed, bleeding yesterday's late-night activity into the next morning's feed.
*   **Your fix and side-effect check:** Modified the `cutoff` timestamp to reset dynamically to the start of the current day using `.replace(hour=0, minute=0, second=0, microsecond=0)`. This guarantees yesterday's data drops off completely at midnight UTC. Checked that `get_activity_feed` is unaffected since it bypasses the recency cutoff entirely.
