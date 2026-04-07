# Jellyfin Web Enhancement Scripts

Two client-side JavaScript enhancements for Jellyfin that inject additional metadata into detail pages. Both scripts are designed to be loaded via the **Jellyfin Web JavaScript Injector** plugin and served through the **Jellyfin FileTransformation** plugin.

---

## Scripts

### `collection-runtime.js`
Adds **total runtime** and an **"Ends at"** clock to collection detail pages (e.g. a Marvel movie franchise box). The info is inserted inline in the header bar next to the content rating, styled to match native Jellyfin elements.

### `play-stats.js`
Adds a **play count badge** to any media detail page (movies, episodes, etc.). Hover over the badge to see a **per-user pie chart** showing who watched it and how many times, sourced live from the Playback Reporting Plugin's SQLite database.

---

## Requirements

| Requirement | Notes |
|---|---|
| [JS Injector Plugin](https://github.com/danieladov/jellyfin-plugin-js-injection) | Loads custom scripts into the Jellyfin web client |
| [FileTransformation Plugin](https://github.com/jellyfin/jellyfin-plugin-filetransformation) | Serves the script files from your server to the client |
| [Playback Reporting Plugin](https://github.com/jellyfin/jellyfin-plugin-playbackreporting) | Required for `play-stats.js` — provides the play history database |

---

## Installation

### 1. Install the plugins

In your Jellyfin dashboard:

1. Go to **Dashboard → Plugins → Catalog**
2. Search for and install:
   - **JavaScript Injector**
   - **FileTransformation**
   - **Playback Reporting** *(only needed for `play-stats.js`)*
3. Restart Jellyfin after installing

---

### 2. Serve the script files via FileTransformation

The FileTransformation plugin lets you expose arbitrary files from your server's filesystem to the Jellyfin web client at a predictable URL.

1. Copy `collection-runtime.js` and/or `play-stats.js` to a directory on your server, e.g.:
   ```
   /opt/jellyfin/scripts/collection-runtime.js
   /opt/jellyfin/scripts/play-stats.js
   ```
   On Windows:
   ```
   C:\ProgramData\Jellyfin\Server\scripts\collection-runtime.js
   C:\ProgramData\Jellyfin\Server\scripts\play-stats.js
   ```

2. In the Jellyfin dashboard, go to **Dashboard → Plugins → FileTransformation** and add a mapping for each file, e.g.:
   - Source path: `/opt/jellyfin/scripts/play-stats.js`
   - Virtual path / URL: `/web/scripts/play-stats.js`

3. Verify by opening `https://your-jellyfin-server/web/scripts/play-stats.js` in a browser — you should see the raw JavaScript.

---

### 3. Inject the scripts via JS Injector

1. In the Jellyfin dashboard, go to **Dashboard → Plugins → JS Injection**
2. Add an entry for each script you want to load, pointing to the virtual URL you set up in step 2:
   ```
   /web/scripts/collection-runtime.js
   /web/scripts/play-stats.js
   ```
3. Save. The plugin will inject a `<script src="...">` tag into the Jellyfin web client on every page load.
4. Hard-refresh the Jellyfin web client (`Ctrl+Shift+R`) and open the browser console (`F12`) to confirm the scripts loaded — you should see:
   ```
   [CollectionRuntime] Script loaded
   [PlayStats] Script loaded
   ```

---

## How each script works

### `collection-runtime.js`

- Watches for navigation to `#/details` pages that contain a `.collectionItems` section
- Finds all movie cards in the collection via `.collectionItemsContainer .card[data-type="Movie"]`
- Fetches each movie's `RunTimeTicks` from `/Users/{userId}/Items/{itemId}`
- Sums the ticks, converts to hours/minutes, and inserts two `<div class="mediaInfoItem">` elements into `.itemMiscInfo-primary`:
  - Total runtime (e.g. `9h 32m`)
  - Ends-at time based on current clock (e.g. `Ends at 11:42 PM`)
- Caches by item ID set, skips re-fetch if nothing changed
- Scoped to the active SPA page to avoid stale DOM from Jellyfin's hidden page cache

### `play-stats.js`

- Watches for navigation to any `#/details` page
- Sends a custom SQL query to the Playback Reporting Plugin endpoint:
  ```
  POST /user_usage_stats/submit_custom_query
  { "CustomQueryString": "SELECT UserId, COUNT(*) as PlayCount FROM PlaybackActivity WHERE ItemId = '<id>' GROUP BY UserId ORDER BY PlayCount DESC" }
  ```
- Resolves user IDs to display names via `GET /Users`
- Inserts a `▶ N plays` badge into `.itemMiscInfo-primary`
- **Hovering** over the badge shows a floating popup with:
  - An SVG pie chart sliced per user
  - A legend with each user's name, play count, and percentage
  - A total plays footer
- Each user's color is deterministically derived from their username (djb2 hash → HSL hue), so colors are consistent across sessions
- All DOM lookups are scoped to the active SPA page (`.page:not(.hide)`) to handle Jellyfin's page caching correctly

---

## Console logging

Both scripts log to the browser console with prefixed tags. Open **F12 → Console** to observe them.

| Prefix | Script |
|---|---|
| `[CollectionRuntime]` | `collection-runtime.js` |
| `[PlayStats]` | `play-stats.js` |

---

## Troubleshooting

**Badge / runtime never appears**
- Confirm the script file loads at all: check the Network tab for the script URL and the Console for the `Script loaded` line.
- Check that `.itemMiscInfo-primary` exists on the page — some Jellyfin themes rename or remove this element.
- For `play-stats.js`, confirm the Playback Reporting Plugin is installed and has data: navigate to **Dashboard → Plugins → Playback Reporting** and check that play history is being recorded.

**`play-stats.js` shows `0 plays` for everything**
- The Playback Reporting Plugin only records plays going forward after installation. Historical plays before the plugin was installed will not appear.
- Confirm the plugin's custom query endpoint is accessible: open `https://your-server/user_usage_stats/submit_custom_query` — it should return a 405 (Method Not Allowed) rather than a 404, confirming it exists.

**Wrong item ID detected**
- The scripts extract the item ID from the URL hash (`#/details?id=...`). If the hash format differs in your Jellyfin version, open the console and look for the `[PlayStats] No valid item ID in URL` warning.

**Popup appears in the wrong position**
- The popup is positioned relative to the badge using `getBoundingClientRect()` and clamped to the viewport. If your Jellyfin theme uses unusual scroll containers, the position may be slightly off but should still be visible.

**Script keeps re-running on the same page**
- This is normal until the header DOM is ready. The MutationObserver fires until `insertOrUpdateBadge` succeeds, then stops re-triggering for that page.
