# Mixtape Project Submission

## AI Usage Disclosure
I utilized an AI collaborator to assist with codebase onboarding by generating high-level structural summaries and tracing execution flows. During the debugging phase, I used AI to cross-reference Python's date boundary logic and compare working notification dispatches with missing ones. Every fix and hypothesis was manually tested and verified within a local development environment before committing.

---

## 🗺️ Codebase Map

### Core Architecture
The Mixtape codebase follows a clean **Separation of Concerns (SoC)** design pattern split into three distinct layers:
1. **Routing Layer (`app/routes/` or `routes/`):** Handles incoming HTTP requests, parses user input arguments, and returns JSON payloads with matching HTTP status codes.
2. **Service Layer (`services/`):** The engine room where business logic, state changes, validations, and database commits actually happen.
3. **Data Layer (`models.py`):** Defines the database structure using SQLAlchemy ORM models (Users, Songs, Playlists, etc.).

### Main Files & Responsibilities
*   `models.py`: Tracks data relationships. It includes a custom junction table `PlaylistSong` that uses an explicit `order` column to keep tracks in sequence instead of relying on database creation order.
*   `services/user_service.py`: Responsible for profile management, authentication flags, and maintaining active user listening streaks.
*   `services/feed_service.py`: Generates activity content streams for friends and determines what shows up on a user's dashboard.
*   `services/playlist_service.py`: Manages creating playlists, adding tracks, and ordering items in lists.
*   `services/song_service.py`: Handles catalog searching, track indexing, and processing ratings.

### Sample Data Flow
When a user adds a song to a playlist via `POST /playlists/<playlist_id>/songs`:
1. The request hits the route handler, which extracts the `song_id`.
2. The route controller instantly delegates the operation by calling `playlist_service.add_song_to_playlist()`.
3. The service queries the current song count to increment the order property, commits the new record to the database, and then safely returns the response back to the route.
