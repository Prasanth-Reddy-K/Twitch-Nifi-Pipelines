# Twitch-Nifi-Pipelines
## ⚙️ Overview

The pipeline automates the **end-to-end Twitch data collection and enrichment** process:

1. **Extracts** live stream data from the **Twitch Helix API**
2. **Transforms** and **enriches** it with detailed game and channel metadata
3. **Stores** the enriched JSON data into a **database or data lake** for analysis  

All data movement and transformation are orchestrated via **NiFi processors**, providing visual flow management, error handling, and scalability.

---

# 🎮 Twitch Data Enrichment Pipeline (Apache NiFi)

This repository implements a **fully automated Twitch Data Enrichment Pipeline** using **Apache NiFi** to fetch, enrich, and store real-time Twitch stream and channel data.  
It leverages the **Twitch Helix API**, processes and enriches data through NiFi flows, and stores the final structured results into a database for analytics or downstream consumption.

---


---

## 🧰 Prerequisites

- **Apache NiFi 2.x+**
- **Redshift/ PostgreSQL Database**
- **Twitch Developer Account**
- Twitch API credentials (`client_id`, `client_secret`)

---
## 📦 Data Flow Logic

1. **Get Access Token**  
   - Makes a POST request to Twitch OAuth2 API to generate a token.  
   - Token is cached in NiFi variable registry and refreshed automatically.

2. **Fetch Streams (Helix API)**  
   - Uses the token to call the Twitch Helix API for live stream data.  
   - Supports pagination using `after` cursors.

3. **Validate Stream Records**  
   - Ensures each stream has valid numeric IDs, strings, and at least 10 viewers.  
   - Invalid records are routed to a “failed” relationship for inspection.

4. **Transform Data Schema**  
   - Normalizes Twitch’s raw JSON structure to match your internal format (`PullData` model schema).

5. **Store to Database**  
   - Validated and transformed records are written in chunks of 100 into `pull_data` table.  
   - A summary row is added to the `pull` table with statistics.

6. **Trigger Next Job**  
   - Once data is saved, NiFi triggers the downstream process (`twitch:fill` or similar enrichment flow).


---

## 🚀 Pipeline Architecture

### 1. Streams Extraction
- NiFi fetches real-time stream data from Twitch Helix API (`/streams` endpoint).  
- Configured to poll periodically using `InvokeHTTP` processors with authentication headers.
- The data is stored temporarily as JSON flow files in NiFi.

### 2. Data Storage (Pull Data)
- Raw Twitch streams data is persisted in the database as **PullData** records.  
- Each pull is identified by a `pull_id` and stored with a status (`pending`, `filling-games`, `filling-channels`, `success`).
- NiFi manages this persistence using database processors like:
  - `PutDatabaseRecord`
  - `QueryDatabaseTable`
  - `UpdateRecord`

### 3. Game Metadata Enrichment
- For each stream, the pipeline extracts `game_id`s and checks if the corresponding **TwitchGame** entries exist in the database.
- Missing games trigger a **Helix API `/games`** call to fetch metadata (name, box art URL, etc.).
- Newly fetched games are inserted into the database and linked to the corresponding streams.

### 4. Streams & Channels Enrichment (Equivalent to Laravel `FillTwitchStreams` Command)
This stage replicates the logic from the Laravel command, now implemented through NiFi.

#### Step 1: Collect Game & Channel IDs
- NiFi extracts all unique `game_id` and `channel_id` values from stored PullData.

#### Step 2: Fill Missing Games
- NiFi calls the Helix `/games` endpoint (in chunks of 100 IDs).
- Enriched `TwitchGame` data (name, box art) is merged back into each stream record.

#### Step 3: Enrich Channels (Users)
- NiFi calls Helix `/users` endpoint for each chunk of channel IDs.
- Channel profiles are enriched with:
  - `login`, `display_name`, `profile_image_url`
  - `view_count`, `broadcaster_type`, `offline_image_url`
- Missing or unmatched channels are logged for review.

#### Step 4: Final Merge
- Enriched data (streams + games + channels) is restructured and stored as updated PullData JSON.

### 5. Finalization
- Once all enrichment steps complete successfully, NiFi updates the pull status to **`success`** and logs the completion timestamp.

---


## 🔐 Environment Variables

| Variable | Description |
|-----------|-------------|
| `TWITCH_CLIENT_ID` | Twitch app client ID |
| `TWITCH_CLIENT_SECRET` | Twitch app client secret |
| `TWITCH_API_URL` | `https://api.twitch.tv/helix/` |
| `TWITCH_TOKEN_URL` | `https://id.twitch.tv/oauth2/token` |
These can be stored securely in the or **Parameter Context**.

---

## 🖥️ Deployment

1. Import the NiFi pipelines from github.
2. Configure your Parameter Context with Twitch credentials and DB connection.
3. Start the NiFi Process Group.
4. Monitor progress using NiFi’s built-in provenance and bulletins.

---

## 📊 Monitoring & Logging

- **NiFi Provenance** — to track each flowfile’s lifecycle.
- **PutSQL Logs** — records pull status and counts in database.
- **Error Ports** — route failed validation or API calls to log file or dead-letter queue.
