# PnW Serving Scheduler (GitHub Pages + Firebase)

Static web app. `index.html` is the whole app. Everyone who opens the link and signs in sees the same roster, and changes show up live for everyone.

## How it works

- **Shared data:** people edits and rosters are stored in Firebase Firestore (project `cgc-pnw-roster`).
- **Google Sheet:** an editor presses **Sync Sheet**. The app reads the first sheet plus the role tabs (`vocal_main`, `vocal_sub`, `piano`, `drum`, `bass`, `pa`, `ppt`), read-only, and saves a copy to Firestore so viewers see names and availability without Sheet access.
- **Name order:** people are listed in the same order as the first sheet (People, Availability, dropdowns and the counts table). People added by hand come after them. Press **Sync Sheet** once after updating the app so the order is saved.
- **Who can do what:** decided by the access list in Firestore (`config/access`). Editors can change things. Viewers can only look. Everyone else sees "No access".
- **Roster tab:** opens on the current month each time you click it. Use the month box to move to another month.
- **People tab:** the search box filters the list as you type.
- **Preview tab:** pick one or more months. Each month has a **version dropdown** next to its tick box: **Latest** (the current roster, always the default) and **v1, v2, v3…** (saved exports). The table is real text, so you can select and copy it. **Export As** (PDF or JPG) saves exactly the versions you picked. The bottom right shows `Updated at:`: the last roster change for Latest, or the save time of that version.
- **Versions:** exporting **Latest** saves a new version of that month, but only if something changed since the last version. Exporting an older version does not create a new one.
- **Show Changes:** highlights names that differ from the version before the one you are viewing. **Latest** is compared with the last export. **v3** is compared with **v2**. **v1** has nothing to compare, so it shows no highlights. If nothing was edited since the last export, nothing is highlighted. `Highlighted = Recent update`.
- **Assistant (💬, editors only):** a chat button at the bottom right. It explains its reasoning first and only proposes changes when you ask for them.
  - **Reasoning mode (default):** ask things like “check role rotation for this month”, “who can cover Drum on 10/18?”, “is Auto-fill good enough?”. It does not change anything. The **Check rotation** button asks this in one click. To keep each request small, only these facts are sent: the people who matter this month (they have a role and are assigned, available or unknown on some Sunday), their roles, limits, availability string and roles served this year; the current roster (written once); current rule problems; services this month; who is free per role per Sunday; and a **read-only Auto-fill simulation** (what it would add and why slots would stay empty). The rules, the legend and the app's features are in a short system prompt. A locked month is flagged so it does not suggest Auto-fill.
  - **Action mode:** only when you clearly ask (“swap …”, “change 10/4 Lead to …”, “apply”, “帮我把…换成…”) or press **Prepare these changes** under an answer. The proposed changes are listed and only happen after you press **Confirm**. It can set a slot, change availability, refresh a Sunday, auto-fill or clear a month, and lock or unlock a month. Locked months cannot be changed. Everything is logged under `Chat` in the Log tab.
  - **Names:** before anything is sent, every person is replaced by a random code (P01, P02…). The code list lives only in your browser and changes when you press the bin icon. Replies are turned back into real names on your screen. Nicknames that are not in the People list are sent as typed, so use the names from the list.
  - **Remarks:** only sent when your question mentions remarks, notes or partners, and the names inside them are replaced by codes.
  - **Month:** it uses the month shown in Roster or Availability.
  - **Models (free, no billing):** reasoning tries `gemini-3.8-flash` with medium thinking first, then `gemini-3.5-flash`, then `gemini-3.5-flash-lite`. The next model is used when one is out of quota, missing, overloaded (503) or takes longer than 60 seconds, and the chat then says which model answered and why the others were skipped. No temperature is set (Google advises against it for the 3.x models) and the output limit is large because thinking tokens count against it. Turning a request into changes uses `gemini-3.5-flash-lite`. If the free quota runs out the chat pauses and nothing is charged. There is no message limit in the app, only a 2 second gap between messages.
  - **App Check (required for the AI):** Firebase AI Logic rejects requests without an App Check token (`401 Firebase App Check token is invalid`). One-time setup:
    1. Google Cloud console (same project) → Security → **Fraud Defense** (reCAPTCHA) → enable the reCAPTCHA Enterprise API if asked → create a **Web** key. Domains: `<your-username>.github.io` (domain only). Leave "Use checkbox challenge" unselected. Copy the key.
    2. Firebase console → Security → **App Check** → **Apps** tab → your web app → choose **reCAPTCHA Enterprise**, paste the key, keep the default token lifetime and risk threshold (0.5), **Save**.
    3. Paste the key into `APPCHECK_KEY` near the top of the Assistant code in `index.html`, then commit.
    Only the AI chat uses App Check. Do not turn on App Check enforcement for Firestore or Authentication unless you also add App Check to the main app.
  - **Errors:** when the AI fails, the chat shows a plain message and a `(details: …)` line with the real error from Google. If one model is out of quota or missing, the next model is tried automatically.
  - **Formatting:** the model is told to use plain text. If it still writes `**bold**`, `*italic*` or `*` bullets, the chat shows them as bold, italic and dots instead of raw asterisks.
  - Quick buttons (**Check rotation**, **Empty slots**, **Counts**, **Problems**) and the commands `refresh 10/25`, `lock`, `unlock`: only **Check rotation** calls the AI.
- **Lock (Roster tab):** the lock icon at the far right of the Clear month row locks the roster for the selected month, for everyone. **Auto-fill empty slots**, **Clear month** and every slot dropdown are disabled, so nothing in that month's roster can change. Click it again to unlock. Availability and People can still be edited.
- **Log tab (editors only):** who did what, grouped by date. Click a date to open it.

## Publish

1. Put `index.html` in the root of a GitHub repo.
2. Settings → Pages → Deploy from a branch → `main` → `/(root)`.
3. Firebase console → Authentication → Settings → Authorized domains → add `<your-username>.github.io`.

## Access list

Firebase console → Firestore Database → Data → `config` → `access`:

- `editors` (array of strings): lowercase Google emails who may edit.
- `viewers` (array of strings): lowercase Google emails who may only view.

Change these any time. No code change needed.

## Firestore rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() {
      return request.auth != null && request.auth.token.email_verified == true;
    }
    function access() {
      return get(/databases/$(database)/documents/config/access).data;
    }
    function isEditor() {
      return signedIn() && request.auth.token.email in access().editors;
    }
    function isViewer() {
      return isEditor() || (signedIn() && request.auth.token.email in access().viewers);
    }

    match /config/{doc} {
      allow read: if isEditor();
      allow write: if false;
    }
    match /people/{id} {
      allow read: if isViewer();
      allow write: if isEditor();
    }
    match /roster/{id} {
      allow read: if isViewer();
      allow write: if isEditor();
    }
    match /sheet/{id} {
      allow read: if isViewer();
      allow write: if isEditor();
    }
    match /baseline/{id} {
      allow read: if isViewer();
      allow write: if isEditor();
    }
    match /logs/{id} {
      allow read: if isEditor();
      allow create: if isEditor();
      allow update, delete: if false;
    }
  }
}
```

## Google Sheet access (editors)

The Sheet stays private. Each editor must be able to open the Sheet with their Google account. Sync Sheet uses the OAuth Client ID set as `CLIENT_ID` in `index.html`. In the Google Cloud project that owns that Client ID: enable the Google Sheets API, add the scope `https://www.googleapis.com/auth/spreadsheets.readonly` on the OAuth consent screen, add editors as Test users while the app is in Testing, and add `https://<your-username>.github.io` under the client's Authorized JavaScript origins.

## Notes

- Two people changing the same slot at once: the last save wins. Different slots do not overwrite each other.
- Data from the old browser-only version is not moved over. Re-create it, or use Auto-fill.
- The Firebase config in `index.html` is not secret. The rules above protect the data.
- Free Spark plan limits: 50,000 reads and 20,000 writes per day. A small team stays far below this.
