# Jellyfin Web Enhancement Scripts

A client-side JavaScript enhancement for Jellyfin that injects additional metadata into media detail pages. The script is designed to be loaded via the **Jellyfin Web JavaScript Injector** plugin.

---

## Script

### `play-stats.js`
Adds a **play count badge** to any media detail page (movies, episodes, etc.). Hover over the badge to see a **per-user pie chart** showing who watched it and how many times, sourced live from the Playback Reporting Plugin's SQLite database.

---

## Requirements

| Requirement | Notes |
|---|---|
| [Jellyfin JavaScript Injector](https://github.com/n00bcodr/Jellyfin-JavaScript-Injector) | Loads custom scripts into the Jellyfin web client — paste JS directly in the plugin settings. **Requires adding the repo manually** (not in official catalog). |
| [File Transformation Plugin](https://github.com/IAmParadox27/jellyfin-plugin-file-transformation) | Required by the JavaScript Injector to function. Install it but no configuration needed. **Requires adding the repo manually** (not in official catalog). |
| [Playback Reporting Plugin](https://github.com/jellyfin/jellyfin-plugin-playbackreporting) | Required — provides the play history database |

---

## Installation

### 1. Install the plugins

In your Jellyfin dashboard:

1. Go to **Dashboard → Plugins → Repositories** and add the custom repository URLs from each repo's README:
   - [Jellyfin JavaScript Injector](https://github.com/n00bcodr/Jellyfin-JavaScript-Injector)
   - [File Transformation Plugin](https://github.com/IAmParadox27/jellyfin-plugin-file-transformation)
2. Go to **Dashboard → Plugins → Catalog** and install:
   - **Jellyfin JavaScript Injector** (from the custom repo above)
   - **File Transformation** (from the custom repo above)
   - **Playback Reporting** (available in the default catalog)
3. Restart Jellyfin after installing

---

### 2. Inject the script via JS Injector

1. In the Jellyfin dashboard, go to **Dashboard → Plugins → JavaScript Injector**
2. Paste the full contents of `play-stats.js` directly into the script field
3. Save and hard-refresh the Jellyfin web client (`Ctrl+Shift+R`)
4. Open the browser console (`F12`) to confirm it loaded — you should see:
   ```
   [PlayStats] Script loaded
   ```

---

## How it works

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

The script logs to the browser console with a `[PlayStats]` prefix. Open **F12 → Console** to observe it.

---

## Troubleshooting

**Badge never appears**
- Confirm the script file loads at all: check the Network tab for the script URL and the Console for the `Script loaded` line.
- Check that `.itemMiscInfo-primary` exists on the page — some Jellyfin themes rename or remove this element.
- Confirm the Playback Reporting Plugin is installed and has data: navigate to **Dashboard → Plugins → Playback Reporting** and check that play history is being recorded.

**`play-stats.js` shows `0 plays` for everything**
- The Playback Reporting Plugin only records plays going forward after installation. Historical plays before the plugin was installed will not appear.
- Confirm the plugin's custom query endpoint is accessible: open `https://your-server/user_usage_stats/submit_custom_query` — it should return a 405 (Method Not Allowed) rather than a 404, confirming it exists.

**Wrong item ID detected**
- The scripts extract the item ID from the URL hash (`#/details?id=...`). If the hash format differs in your Jellyfin version, open the console and look for the `[PlayStats] No valid item ID in URL` warning.

**Popup appears in the wrong position**
- The popup is positioned relative to the badge using `getBoundingClientRect()` and clamped to the viewport. If your Jellyfin theme uses unusual scroll containers, the position may be slightly off but should still be visible.

**Script keeps re-running on the same page**
- This is normal until the header DOM is ready. The MutationObserver fires until `insertOrUpdateBadge` succeeds, then stops re-triggering for that page.
