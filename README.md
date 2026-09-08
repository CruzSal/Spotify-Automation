# Spotify-Automation
The purpose of this project is to create an app that manages LARGE Spotify playlists.

### Tech Stack
* **Workflow Automation:** n8n
* **Database:** PostgreSQL
* **Scripting / Transformations:** JavaScript (ES6+)
* **External APIs:** Spotify Web API (OAuth 2.0)
* **Infrastructure:** Docker, Docker Compose


### Problem & Solution
An avid user of Spotify would have probably faced the challenge of maintaining a large playlist (+800 songs), while the playlist itself remains completely functional, the mere act of scrolling through it exposes the user to an incredible amount of "lag".
For users that enjoy having an ordered collection (i.e., comprehensive library) of all their liked songs, once this threshold is passed, it becomes a challenge.

This project intends to address this issue by having a completely automated manager that is able to periodically organize said library from smaller playlists that can be easily managed by the user.

I implemented in n8n a low-code proof of concept that takes one huge playlist, splits it, maintains a Postgres database, and can then maintain the original according to future inputs in the smaller ones.
The workflow was optimized to minimize the number of API calls 

### Pipeline Architecture (Phase 1: n8n Prototype)

The current implementation is an automated ETL pipeline built in n8n:

* **Token Lifecycle:** Automatically requests fresh access tokens from the Spotify Accounts service via `grant_type: refresh_token` and environment variables (`SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, `SPOTIFY_REFRESH_TOKEN`).
* **Dynamic Pagination:** Handles Spotify API pagination recursively by evaluating the `next` URL dynamically until all playlist items are retrieved.
* **Data Transformation:** Custom JavaScript code nodes extract nested payloads, format track IDs into strict Spotify URIs (`spotify:track:<id>`), and calculate local positional indices.
* **Batch Processing:** Uses batching mechanisms to process track operations within Spotify's 100-item request limits.
* **Idempotent Storage:** Employs PostgreSQL `UPSERT` operations across relational tables to ensure repeat runs update existing records without generating duplicate rows.

<img width="1237" height="447" alt="image" src="https://github.com/user-attachments/assets/b40248fd-69ca-4a37-b39e-7cdadad7f3e4" />

--- 

### Database Schema

```sql
-- Core playlist metadata
CREATE TABLE playlists (
    playlist_id VARCHAR PRIMARY KEY,
    name VARCHAR NOT NULL,
    merge_order INT DEFAULT 0
);

-- Normalized track details
CREATE TABLE tracks (
    spotify_id VARCHAR PRIMARY KEY,
    track_name VARCHAR NOT NULL,
    artist_name VARCHAR,
    album_name VARCHAR
);

-- Junction table with positional and chronological data
CREATE TABLE playlist_tracks (
    playlist_id VARCHAR REFERENCES playlists(playlist_id),
    spotify_id VARCHAR REFERENCES tracks(spotify_id),
    local_position INT,
    added_at TIMESTAMP,
    PRIMARY KEY (playlist_id, spotify_id)
);
```

---

### Roadmap

* **Phase 1: n8n Prototype (Current)**
  * [x] Spotify OAuth refresh pipeline
  * [x] Recursive playlist fetching and JSON parsing
  * [x] PostgreSQL relational upserting
  * [x] Partitioned playlist creation & sync
* **Phase 2: Standalone Service**
  * Port workflow logic to a dedicated backend application (e.g., Python or Go/Node.js).
  * Add webhook listeners for event-driven updates.
* **Phase 3: Smart Organization**
  * Automated deduplication across libraries.
  * Audio-feature clustering (tempo, valence, genre) using Spotify audio analysis endpoints.

---
### Setup & Installation

1. **Clone repository:**
   ```bash
   git clone https://github.com/<your-username>/spotify-automation.git
   cd spotify-automation
   ```

2. **Configure Environment Variables:**
   Set the following variables in your environment or Docker `.env` file:
   ```env
   SPOTIFY_CLIENT_ID=your_client_id
   SPOTIFY_CLIENT_SECRET=your_client_secret
   SPOTIFY_REFRESH_TOKEN=your_refresh_token
   ```

3. **Import Workflow:**
   * Open your n8n web dashboard.
   *    * Edit this placeholders in the JSON file before importing:
     ```
     $MY_USER = your_username
     $POSTGRES_ID = your_postgresID
     $SPOTIFY_ID = your_spotifyID
     $N8N_INSTANCE = your_n8nInstance
   ```
   * Go to **Workflows** > **Import from File**.
   * Select `Spotify Manager.json` and link your database credentials.
