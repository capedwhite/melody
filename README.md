
# Melody

**An AI-powered music streaming and discovery platform.**

Melody is a mobile music streaming app where listeners play songs, build playlists and follow artists. Its standout feature is **natural-language discovery**: a listener describes what they want to hear in everyday words, a language model turns that request into structured preferences, and Melody's own recommendation engine decides which songs match.

> **The AI understands what the user wants to hear. The application's own engine decides which songs match.**

Because results follow what the listener asks for rather than popularity alone, lesser-known artists also gain a fairer chance of being heard.

---

## Table of contents

1. [Features](#features)
2. [User roles](#user-roles)
3. [System architecture](#system-architecture)
4. [Core flows](#core-flows)
5. [Tech stack](#tech-stack)
6. [Data model](#data-model)
7. [Recommendation engine](#recommendation-engine)
8. [AI discovery layer](#ai-discovery-layer)
9. [Event tracking](#event-tracking)
10. [Audio streaming](#audio-streaming)
11. [API overview](#api-overview)
12. [Folder structure](#folder-structure)
13. [Development roadmap](#development-roadmap)
14. [Getting started](#getting-started)
15. [Scope and non-goals](#scope-and-non-goals)
16. [Documentation](#documentation)

---

## Features

**Listening**
- Search and browse songs, artists, albums and genres
- Play, pause, seek, skip, previous and queue
- Like songs, create playlists, follow artists
- Listening history

**Discovery**
- Natural-language prompts, for example: *"Find relaxing indie songs with soft vocals from artists I haven't listened to before."*
- Filters for genre, mood, vocal style, instruments, language and tempo
- Option to exclude artists the listener already knows and to prefer emerging artists
- Recommendations that improve from plays, skips, likes and saves

**Artists**
- Upload songs and cover images without prior verification
- Manage profile, albums and releases; edit or remove their own songs
- View analytics: plays, listeners, likes, followers
- Become eligible for verification review after reaching a popularity threshold

**Admins**
- Manage, edit, suspend or delete user and artist accounts
- Review eligible artists and approve or reject verification
- Review reports and remove content that violates platform policy
- Manage genres and catalog metadata; monitor platform activity

---

## User roles

| Role | Can do |
|---|---|
| **User (listener)** | Register and log in, stream music, search, like songs, build playlists, follow artists, view history, use AI discovery |
| **Artist** | Everything a listener can, plus upload songs and covers, manage albums and catalog, view analytics, become eligible for verification |
| **Admin** | Manage and edit accounts, decide artist verification, handle reports, remove content, manage genres, monitor activity |

Artist verification and content moderation are **separate processes**. A verified artist's song can still be reviewed, and an unverified artist's music is subject to the same platform rules. The popularity threshold only makes an artist *eligible*; an admin always makes the final decision.

---

## System architecture

```
┌──────────────────┐        HTTPS / JSON        ┌──────────────────────────┐
│  Mobile app      │ ─────────────────────────▶ │  Backend API             │
│  (listener,      │ ◀───────────────────────── │  auth · roles · uploads  │
│  artist, admin)  │                            │  events · business logic │
└────────┬─────────┘                            └───┬───────────┬──────────┘
         │ plays audio directly                     │           │
         │ from a signed URL                        │           │
         ▼                                          ▼           ▼
┌──────────────────┐                     ┌─────────────┐  ┌──────────────────┐
│  Cloudflare R2   │ ◀── upload / sign ──│  MongoDB    │  │  AI service      │
│  audio + covers  │                     │  metadata,  │  │  (LLM API)       │
└──────────────────┘                     │  history,   │  │  prompt → JSON   │
                                         │  accounts   │  └──────────────────┘
                                         └──────┬──────┘
                                                │
                                   ┌────────────▼─────────────┐
                                   │  Recommendation engine   │
                                   │  filter · score · rank   │
                                   └──────────────────────────┘
```

**Storage rule:** audio files and cover images live only in Cloudflare R2. MongoDB stores metadata plus the R2 object key and URL for each file.

---

## Core flows

### 1. Natural-language discovery (the primary flow)

```
User's prompt
   │
   ▼
AI interpretation        LLM converts text into the predefined JSON schema
   │
   ▼
Backend validation       Check shape; check values against genres + metadata_options
   │
   ▼
Recommendation engine    Filter → exclude → score → diversify
   │
   ▼
MongoDB metadata         Candidate songs and user history
   │
   ▼
Ranked songs shown in the app (with a short reason for each)
```

The AI is called only when the user submits a natural-language request, not for every song played.

### 2. Listening and feedback loop

```
User plays a song → app streams from a signed R2 URL
                 → app sends play / skip / complete / like events
                 → backend stores them in listening_history and updates counters
                 → future recommendations use this history
```

### 3. Artist flow

```
Create artist account and profile
   → upload song, cover and metadata (genre, mood required)
   → backend stores files in R2 and metadata in MongoDB
   → song becomes available for discovery per publication rules
   → platform records plays, likes, followers
   → threshold reached: artist becomes eligible for verification
   → admin approves or rejects
```

### 4. Admin flow

```
Open admin interface
   → manage, edit, suspend or delete user and artist accounts
   → review eligible artists and decide verification
   → handle reported music and policy violations
   → remove prohibited content
   → monitor activity, manage genres and catalog metadata
```

---

## Tech stack

| Layer | Choice |
|---|---|
| Mobile app | Flutter (`just_audio` for playback, `audio_service` for background and lock-screen controls) |
| Backend API | Node.js with Express |
| Database | MongoDB (Mongoose) |
| Object storage | Cloudflare R2 (S3-compatible API) |
| AI service | LLM API with structured JSON output |
| Validation | Zod |
| Auth | JWT with role-based middleware |

> The folder structure below assumes Node.js and Flutter. If you use a different stack, keep the same layering (routes, controllers, services, models) and rename files to match.

---

## Data model

MongoDB collections:

| Collection | Purpose |
|---|---|
| `users` | All accounts (user, artist, admin), preferences, status |
| `artists` | Artist profile, stats, verification state (1:1 with a user) |
| `genres` | Genre catalog |
| `metadata_options` | Controlled lists for mood, vocal style, instrument and language; used to validate AI output |
| `albums` | Albums, EPs and singles |
| `songs` | Song metadata and R2 keys |
| `playlists` / `playlist_songs` | Playlists and their ordered songs |
| `song_likes` | User to song likes |
| `artist_follows` | User to artist follows |
| `listening_history` | Play, skip, complete and save events |
| `ai_prompts` | Natural-language requests, parsed filters and results |
| `verification_requests` | Artist verification reviews |
| `reports` | User reports on songs |
| `admin_actions` | Audit log of admin decisions |
| `artist_daily_stats` | Daily analytics rollups |

Key relationships:
- User → Artist: 1:1 (`artists.userId`, unique)
- Artist → Albums and Songs: 1:N
- Album → Songs: 1:N (a song may be standalone)
- Songs ↔ Genres: M:N through `songs.genreIds[]`
- Playlists ↔ Songs: M:N through `playlist_songs`
- Users ↔ Songs (likes), Users ↔ Artists (follows): M:N through junction collections

See the [ER diagram](docs/diagrams/music_platform_er_diagram.png) for the full picture.

### Song metadata

The recommendation engine is only as good as song tags. Fields are filled like this:

| Field | Source |
|---|---|
| `durationSec`, `audioFormat`, `fileSizeBytes`, `tempoBpm` | Extracted automatically at upload |
| `genreIds`, `moods` | **Required**, chosen by the artist from fixed dropdowns |
| `vocalStyle`, `instruments`, `language` | Optional, chosen by the artist from fixed dropdowns |

Fixed lists (not free text) keep tags consistent with what the AI layer and validator expect.

---

## Recommendation engine

Input: `userId` and a filter object. Output: ranked songs with reasons.

**Filter schema (the contract between the AI layer and the engine):**

```json
{
  "genres": ["indie"],
  "moods": ["relaxing"],
  "vocalStyle": ["soft"],
  "instruments": [],
  "language": null,
  "tempo": { "min": null, "max": null },
  "discovery": { "excludeHeardArtists": true, "preferEmerging": true }
}
```

**Pipeline:**

1. **Hard filter:** published songs matching the required fields (genre, mood). Optional fields are used only for scoring.
2. **Exclusions:** artists already heard (when requested) and songs already liked.
3. **Scoring:** weighted match on genre, mood, vocal style and instruments, plus user affinity from history and an emerging-artist boost.

   ```
   score = 0.35 * genreMatch
         + 0.30 * moodOverlap
         + 0.15 * vocalMatch
         + 0.10 * instrumentOverlap
         + 0.10 * userAffinity
   if preferEmerging: score += emergingBoost
   ```
4. **Diversity:** at most 2 to 3 songs per artist.
5. **Relaxation:** if fewer than about 10 songs match, drop the weakest filters (instruments, then vocal style, then tempo) and retry. Genre and mood are dropped last.

Songs with missing optional fields score neutrally instead of being excluded.

---

## AI discovery layer

The AI has one job: turn a sentence into the filter schema above.

1. **Build the prompt from live data.** Include the schema and the allowed values loaded from `genres` and `metadata_options`.
2. **Request structured output** (JSON schema or function calling) at low temperature.
3. **Validate in the backend.** Never trust the model's output:
   - Check the shape with Zod.
   - Check every value against the allowed lists; drop unknown values and record them.
   - Apply a small synonym map (for example `chill` → `relaxing`).
   - Mark the result `valid`, `partially_valid` or `invalid`.
4. **Fail gracefully.** If nothing valid remains, ask the user to rephrase or fall back to plain text search. Never send empty filters to the engine.
5. **Log** the prompt, parsed filters and results in `ai_prompts`.

---

## Event tracking

The app sends an event for each meaningful action:

```
POST /events
{ "songId": "...", "eventType": "play | complete | skip | save", "listenedSec": 42, "source": "ai_prompt" }
```

The server adds `userId` and `artistId`, inserts a `listening_history` record, and increments counters on the song and artist. Events are tied to **player events** (started, 30 seconds reached, completed, skipped), not button taps.

A user profile for scoring is computed on demand from the last 50 to 100 events: completes and likes add weight to a song's genres and moods, skips subtract.

---

## Audio streaming

1. The song's `audioKey` points to a file in Cloudflare R2.
2. `GET /songs/:id/stream` returns a signed URL (long enough expiry, refreshed when each song starts).
3. The mobile player loads that URL and plays while downloading. R2 supports range requests, so seeking works.
4. Queue, next and previous are handled in the app. Files are MP3 or AAC at 128 to 256 kbps.

---

## API overview

**Auth**

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register` | Create an account |
| POST | `/auth/login` | Log in, receive a JWT |
| GET | `/auth/me` | Current user |

**Listening**

| Method | Endpoint | Description |
|---|---|---|
| GET | `/songs` | List and search songs (`?q=`, `?genre=`) |
| GET | `/songs/:id` | Song metadata |
| GET | `/songs/:id/stream` | Playable URL |
| POST | `/events` | Record a listening event |
| POST / DELETE | `/songs/:id/like` | Like or unlike |
| GET | `/me/history` | Listening history |
| POST / DELETE | `/artists/:id/follow` | Follow or unfollow |

**Playlists**

| Method | Endpoint | Description |
|---|---|---|
| GET / POST | `/playlists` | List or create |
| GET | `/playlists/:id` | Playlist with songs |
| POST | `/playlists/:id/songs` | Add a song |
| DELETE | `/playlists/:id/songs/:songId` | Remove a song |

**Discovery**

| Method | Endpoint | Description |
|---|---|---|
| POST | `/recommendations` | Body: filter JSON. Returns ranked songs |
| POST | `/discover` | Body: `{ prompt }`. Returns parsed filters and ranked songs |

**Artist** (role: artist)

| Method | Endpoint | Description |
|---|---|---|
| GET / PUT | `/artists/me` | View or edit artist profile |
| POST | `/artist/songs` | Upload a song (multipart) |
| PUT / DELETE | `/artist/songs/:id` | Edit or remove own song |
| POST / PUT / DELETE | `/artist/albums` | Manage albums |
| GET | `/artists/me/analytics` | Plays, listeners, likes, followers |

**Reports**

| Method | Endpoint | Description |
|---|---|---|
| POST | `/reports` | Report a song |

**Admin** (role: admin)

| Method | Endpoint | Description |
|---|---|---|
| GET / PATCH / DELETE | `/admin/users` | List, edit, suspend or delete users |
| GET / PATCH / DELETE | `/admin/artists` | List, edit or delete artist accounts |
| GET | `/admin/verification-requests` | Eligible and pending artists |
| PATCH | `/admin/verification-requests/:id` | Approve or reject |
| GET / PATCH | `/admin/reports` | Review and resolve reports |
| PATCH | `/admin/songs/:id/status` | Hide or remove a song |
| CRUD | `/admin/genres` | Manage genres |
| GET | `/admin/activity` | Platform activity overview |

---

## Folder structure

```
melody/
├── README.md
│
├── docs/
│   ├── diagrams/
│   │   ├── music_platform_er_diagram.mmd
│   │   ├── music_platform_er_diagram_compact.mmd
│   │   ├── music_platform_er_diagram.png
│   │   ├── music_platform_class_diagram.mmd
│   │   ├── music_platform_class_diagram_compact.mmd
│   │   └── music_platform_class_diagram.png
│   └── project-overview.md
│
├── backend/
│   ├── package.json
│   ├── .env.example                  # MONGO_URI, JWT_SECRET, R2_*, LLM_API_KEY
│   ├── src/
│   │   ├── server.js                 # starts the HTTP server
│   │   ├── app.js                    # express app, middleware, route mounting
│   │   │
│   │   ├── config/
│   │   │   ├── env.js                # loads and validates environment variables
│   │   │   ├── db.js                 # MongoDB connection
│   │   │   ├── r2.js                 # Cloudflare R2 client
│   │   │   └── llm.js                # LLM client settings
│   │   │
│   │   ├── models/                   # Mongoose schemas, one per collection
│   │   │   ├── User.js
│   │   │   ├── Artist.js
│   │   │   ├── Genre.js
│   │   │   ├── MetadataOption.js
│   │   │   ├── Album.js
│   │   │   ├── Song.js
│   │   │   ├── Playlist.js
│   │   │   ├── PlaylistSong.js
│   │   │   ├── SongLike.js
│   │   │   ├── ArtistFollow.js
│   │   │   ├── ListeningHistory.js
│   │   │   ├── AIPrompt.js
│   │   │   ├── VerificationRequest.js
│   │   │   ├── Report.js
│   │   │   ├── AdminAction.js
│   │   │   └── ArtistDailyStats.js
│   │   │
│   │   ├── routes/                   # URL → controller mapping
│   │   │   ├── auth.routes.js
│   │   │   ├── songs.routes.js
│   │   │   ├── events.routes.js
│   │   │   ├── playlists.routes.js
│   │   │   ├── artists.routes.js
│   │   │   ├── artistStudio.routes.js
│   │   │   ├── discover.routes.js    # /recommendations and /discover
│   │   │   ├── reports.routes.js
│   │   │   └── admin.routes.js
│   │   │
│   │   ├── controllers/              # request/response handling only
│   │   │   ├── auth.controller.js
│   │   │   ├── songs.controller.js
│   │   │   ├── events.controller.js
│   │   │   ├── playlists.controller.js
│   │   │   ├── artists.controller.js
│   │   │   ├── artistStudio.controller.js
│   │   │   ├── discover.controller.js
│   │   │   ├── reports.controller.js
│   │   │   └── admin.controller.js
│   │   │
│   │   ├── services/                 # business logic
│   │   │   ├── auth.service.js
│   │   │   ├── storage.service.js    # R2 upload, signed URLs, delete
│   │   │   ├── upload.service.js     # validate metadata, extract duration/tempo
│   │   │   ├── listening.service.js  # record events, update counters
│   │   │   ├── analytics.service.js  # daily stats, eligibility trigger
│   │   │   ├── verification.service.js
│   │   │   ├── moderation.service.js # reports, removals, audit log
│   │   │   │
│   │   │   ├── recommendation/
│   │   │   │   ├── engine.js         # recommend(userId, filters)
│   │   │   │   ├── filters.js        # hard filter and exclusions
│   │   │   │   ├── scoring.js        # weighted scoring
│   │   │   │   ├── userProfile.js    # affinity from recent events
│   │   │   │   ├── diversity.js      # per-artist cap
│   │   │   │   └── relaxation.js     # drop weakest filters if too few results
│   │   │   │
│   │   │   └── ai/
│   │   │       ├── ai.service.js     # prompt → filters
│   │   │       ├── promptBuilder.js  # builds the prompt from live allowed values
│   │   │       ├── schema.js         # Zod schema for the filter JSON
│   │   │       ├── validator.js      # check values, drop unknowns, set status
│   │   │       └── synonyms.js       # chill → relaxing, etc.
│   │   │
│   │   ├── middleware/
│   │   │   ├── auth.js               # verify JWT
│   │   │   ├── roles.js              # require user / artist / admin
│   │   │   ├── validate.js           # request body validation
│   │   │   ├── upload.js             # multipart handling
│   │   │   └── errorHandler.js
│   │   │
│   │   ├── utils/
│   │   │   ├── asyncHandler.js
│   │   │   ├── audioMeta.js          # duration, format, tempo extraction
│   │   │   └── logger.js
│   │   │
│   │   └── seed/
│   │       ├── seed.js               # wipe and reload demo data
│   │       └── data/
│   │           ├── genres.json
│   │           ├── metadataOptions.json
│   │           ├── artists.json
│   │           └── songs.json        # ~50 hand-tagged songs
│   │
│   └── tests/
│       ├── recommendation/
│       │   ├── engine.test.js
│       │   └── scoring.test.js
│       ├── ai/
│       │   └── validator.test.js
│       └── fixtures/
│           ├── filters/              # 8 to 10 hand-written filter JSONs
│           └── prompts.txt           # 15 sample prompts, including awkward ones
│
├── mobile/                           # Flutter app
│   ├── pubspec.yaml
│   └── lib/
│       ├── main.dart
│       ├── core/
│       │   ├── api_client.dart       # HTTP client, token handling
│       │   ├── theme.dart
│       │   └── constants.dart
│       └── features/
│           ├── auth/                 # login, register
│           ├── player/               # playback, queue, mini player, event reporting
│           ├── library/              # search, playlists, likes, follows, history
│           ├── discover/             # prompt box, filter chips, results
│           ├── artist/               # upload, albums, analytics
│           └── admin/                # accounts, verification, reports
│
└── scripts/
    └── upload_seed_audio.js          # upload demo MP3s to R2 and print their keys
```

Layering rule: **routes** map URLs, **controllers** handle requests and responses, **services** hold the logic, **models** define the data. Keep the recommendation engine and AI layer free of Express code so they can be tested on their own.

---

## Development roadmap

Finish and test each phase before starting the next. This keeps bad results traceable to one component.

| Phase | Work | Done when |
|---|---|---|
| **0. Setup** | Project, MongoDB connection, R2 bucket, `.env`, one hardcoded demo user | Server starts and can read and write a test document |
| **1. Data foundation** | Seed genres and metadata options; upload ~50 MP3s to R2; insert hand-tagged songs with a seed script | ~50 well-tagged songs, all files present in R2 |
| **2. Play API** | `GET /songs`, `GET /songs/:id`, `GET /songs/:id/stream` | A stream URL plays and seeks in a browser |
| **3. Event tracking** | `POST /events`, likes, counter updates | Playing and skipping from Postman produces correct history |
| **4. Recommendation engine** | `recommend()` with filtering, scoring, diversity, relaxation; `POST /recommendations` | Different filters give clearly different, sensible lists, and history changes later results |
| **5. AI layer** | Prompt builder, structured output, validator; `POST /discover` | A sample prompt returns a sensible list and a nonsense prompt fails gracefully |
| **6. Demo polish** | JWT login, playlists, follows, mobile app wiring, demo script | End-to-end demo runs on a physical phone |
| **7. Artist and admin** | Uploads, albums, analytics, verification, reports, account management | Artist and admin flows work end to end |

Rough effort for phases 0 to 6, working alone: 2 to 3 weeks. Spend real time on song tagging in Phase 1; weak tags make recommendations look random however good the engine is.

---

## Getting started

### Prerequisites
- Node.js 20 or newer
- A MongoDB instance (local or Atlas)
- A Cloudflare R2 bucket and API credentials
- An LLM API key
- Flutter SDK (for the mobile app)
- A physical phone for testing audio (emulators are unreliable for background playback)

### Backend

```bash
cd backend
cp .env.example .env        # fill in the values
npm install
npm run seed                # loads genres, metadata options and demo songs
npm run dev                 # starts the API
```

### Environment variables

```
MONGO_URI=
JWT_SECRET=
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=
LLM_API_KEY=
```

### Mobile

```bash
cd mobile
flutter pub get
flutter run
```

---

## Scope and non-goals

Melody is built as a **demo**, not a production-grade or highly scalable service.

**In scope:** core streaming, event tracking, rule-based recommendations, natural-language discovery, artist uploads, admin verification and moderation.

**Out of scope for now:**
- Adaptive streaming (HLS) and CDN tuning
- Real-time updates, caching layers and queues
- Machine-learning recommenders and collaborative filtering
- Automatic audio analysis for mood and instruments (a good future improvement)
- Payments, royalties and distributor-style licensing

---

## Documentation

- ER diagrams: `docs/diagrams/music_platform_er_diagram.*`
- Class diagrams: `docs/diagrams/music_platform_class_diagram.*`
- Mermaid files can be pasted into mermaid.live, draw.io, GitHub or Notion.