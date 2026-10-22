# PnW Serving Scheduler (GitHub Pages)

Static web app. `index.html` is the whole app. Reads availability and roles from your Google Sheet (read-only) and keeps edits and rosters in your browser.

## 1. Publish on GitHub Pages

1. Create a GitHub repo (for example `pnw-scheduler`).
2. Add `index.html` to the repo root and commit.
3. Repo → Settings → Pages → Build and deployment → Source: **Deploy from a branch**, Branch: **main**, folder **/(root)** → Save.
4. Your site is `https://<your-username>.github.io/pnw-scheduler/`.

## 2. Allow Google Sheet access (one-time)

The Sheet stays private. Each person signs in with their own Google account, and only people who can already open the Sheet can sync.

1. Go to https://console.cloud.google.com and create a project.
2. APIs & Services → Library → enable **Google Sheets API**.
3. APIs & Services → OAuth consent screen → User type **External** → fill in the app name and your email → add the scope `.../auth/spreadsheets.readonly`.
4. While the app is in **Testing**, add every team member's Google email under **Test users**.
5. Credentials → Create credentials → **OAuth client ID** → Application type **Web application**.
6. Authorized JavaScript origins: `https://<your-username>.github.io` (origin only, no path).
7. Copy the Client ID and paste it into `index.html`:

   ```js
   const CLIENT_ID='1234567890-abc.apps.googleusercontent.com';
   ```

8. Commit. Open the site, click **Sign in with Google**, then it syncs. It refreshes every minute while you stay signed in (sign-in lasts about an hour).

The Client ID is not a secret. The app only reads the first sheet plus the tabs `vocal_main`, `vocal_sub`, `piano`, `drum`, `bass`, `pa`, `ppt`. The Google scope itself can't be limited to those tabs, so the limit is in the code.

## 3. Saved data

- Edits (availability changes, roles, partners, remarks) and rosters are saved in **this browser** (`localStorage`).
- They are **not shared** between people or devices. Use **Backup** to download a JSON file and **Restore** to load it on another browser.
- The last Sheet data (names, roles, availability) is also saved, so the roster still shows names when you are signed out or offline. Backups include it too.
- Clearing site data erases them, so back up before big changes.
- For one shared roster across the team you need a small backend (for example Firebase, Supabase or a Google Apps Script). Ask if you want that added.

## 4. PDF

**Download PDF** saves a landscape page that looks like your sample table. It is an image of the table so Chinese names render everywhere.
