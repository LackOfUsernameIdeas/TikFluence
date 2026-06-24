# TikFluence

> **Data analytics platform** that proves a real phenomenon: when a song gets used repeatedly across **TikTok** videos, the platform's exposure gradually pushes up that song's popularity on **Spotify and YouTube** over time - TikTok acting as a launchpad that influences a track's trajectory across other music platforms. An **Influence Algorithm** detects songs whose TikTok peak predates their Spotify peak. The platform surfaces this through daily-updated **leaderboards** (Top 200 Global/Bulgaria TikTok songs, Top 200 TikTokers, Top 200 videos), **per-song/per-creator stats pages**, and a **"My Statistics"** page showing a logged-in user's live TikTok profile data in real time. This was my **first serious project**, built primarily as a **learning exercise** to get comfortable working with the technologies involved

> **Note:** the live demo is visitable but **some features may not function** as the project is **no longer actively maintained** due to changes in Spotify API policies and access restrictions, which **removed access to the track popularity score (0–100)** used in core functionality and made long-term maintenance **impractical**

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Gitignored Configuration Files](#gitignored-configuration-files)
- [Local Development](#local-development)
- [Hosting / Deployment](#hosting--deployment)
- [TikTok API Access](#tiktok-api-access)

---

## Architecture

```
TikFluence/
├── *.php                # Page controllers (server-rendered views)
├── includes/
│   ├── config.php
│   ├── common.php
│   └── databaseManager.php
├── scraping/
│   ├── curlFunctions.php
│   └── tiktok.php
└── css/ scss/ js/ plugins/

```

| Part                               | Role                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------- |
| Root `*.php` files                 | Server-rendered pages for dashboards, leaderboards, creator and content analytics, and user stats |
| `includes/`                        | Backend utilities for database access, configuration, and shared helper functions                 |
| `scraping/`                        | Scheduled data collection layer for external platforms (Spotify, YouTube, TikTok, Chartex)        |
| `css/`, `scss/`, `js/`, `plugins/` | Frontend UI stack using Bootstrap, jQuery, and visualization libraries                            |
| _(external service)_               | Separate Node.js + Express + Socket.IO API for live TikTok profile stats streaming                |

---

## Tech Stack

**Frontend**  
PHP (server-rendered pages), Bootstrap, jQuery, jQuery DataTables, Chart.js, Socket.IO Client

**Backend**  
PHP, PDO, MySQL, JavaScript, Apache, Cronjob

**External APIs**  
Chartex API, Spotify API, YouTube API, TikTok API

**Real-time proxy (separate service, not included in this repository)**  
Node.js, Express.js, Socket.IO - bypasses TikTok's CORS restrictions to stream user's live profile stats to the frontend

---

## Gitignored Configuration Files

These files/values are excluded from version control and must exist locally before running the project. Templates below show the expected structure - fill in real values yourself.

### `includes/config.php`

```php
<?php
$dbopts = [
    'db_host' => 'localhost',
    'db_name' => 'tikfluence',
    'db_user' => 'root',
    'db_pass' => '',
    'db_port' => 3306
];
```

### API credentials in `scraping/curlFunctions.php`

Add your own values for the following, in the corresponding functions:

| Credential                            | Function(s)                                                                                                                                           |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Chartex API token                     | `fetchTiktokDatapoints`, `fetchTiktokDatapointsBG`, `fetchTiktokTopUsers`, `fetchTiktokTopVideos`                                                     |
| Spotify `client_id` / `client_secret` | `generateSpotifyToken`                                                                                                                                |
| YouTube Data API key                  | `fetchYoutubeDatapoints`                                                                                                                              |
| TikTok `client_key` / `client_secret` | `generateTikTokAccessToken`, `refreshTikTokAccessToken`                                                                                               |
| TikTok Ads session cookie             | `fetchTopHashtagsForTheLast7Days`, `fetchTopHashtagsForTheLast120Days` (expires periodically and needs re-exporting from a logged-in browser session) |

---

## Local Development

### 1. Add your local configuration

Create `includes/config.php` and add your API credentials in `scraping/curlFunctions.php` as shown in [Gitignored Configuration Files](#gitignored-configuration-files).

### 2. Start XAMPP

Open the XAMPP Control Panel and start:

- **Apache**
- **MySQL**

### 3. Set up the database

Create a database matching `db_name` in `config.php` (default: `tikfluence`) and import/create the required tables.

### 4. Run the project

Place the project folder in your Apache web root (e.g. `C:/xampp/htdocs/TikFluence`), then open:

```
http://localhost/TikFluence/index.php
```

### 5. Populate data

The leaderboard and stats pages read from the database, so run the scraper at least once before browsing them:

```bash
php scraping/tiktok.php
```

> Note: `scraping/tiktok.php` currently `include`s `databaseManager.php` via an absolute server path (`/home/noit1/public_html/fluence/includes/databaseManager.php`) left over from the production server - update this to a relative path (or your local absolute path) before running it locally.

That's it - no further setup is required for local development.

The **"My Statistics"** page (`individualStats.php`) additionally requires a working TikTok OAuth setup (see [TikTok API Access](#tiktok-api-access)) and the separate Socket.IO proxy service to show live data; all other pages are publicly accessible without login.

---

## Hosting / Deployment

The production instance runs on a standard Apache/PHP/MySQL hosting environment (cPanel-style), reachable at `https://fluence.noit.eu/`.

### Cron job

`scraping/tiktok.php` is scheduled to run once a day to fetch and store TikTok/Spotify/YouTube data. Example cPanel cron entry:

```bash
php /home/<your_user>/public_html/<your_app_dir>/scraping/tiktok.php
```

Make sure the absolute include path at the top of `tiktok.php` matches your server's actual path.

### TikTok OAuth redirect

The "Log in with TikTok" button on `individualStats.php` sends users to TikTok's authorize endpoint with a `redirect_uri` pointing back at `individualStats.php` on your domain. This URI must be registered exactly in the TikTok Developer Portal for your app, and updated in the code if you deploy to a different domain.

### Real-time stats proxy

The live profile stats on `individualStats.php` connect via Socket.IO to a separate Node.js + Express + Socket.IO service (not part of this repository) deployed at its own domain (e.g. `fluence-api.<your-domain>`). If you deploy this project, you'll need to stand up that proxy separately and update the Socket.IO client URL in `individualStats.php` to match.

---

## TikTok API Access

- The TikTok integration uses the **TikTok for Developers** Open API: OAuth login with `client_key` / `client_secret`, scopes `user.info.basic` and `video.list`, and the `/oauth/access_token/` and `/oauth/refresh_token/` endpoints.
- A successful login stores `tiktok_access_token` (1 hour) and `tiktok_refresh_token` (24 hours) as cookies.
- Getting API approval was the hardest part of this project - it took **14 submission attempts** to the TikTok Developer team before access was granted. The project documentation includes the email correspondence from that process.
- The hashtag-trend endpoints (`creative_radar_api`) are not part of the official API and instead rely on a logged-in TikTok Ads web session cookie, which expires periodically and needs to be re-exported manually (see the credentials note above).
