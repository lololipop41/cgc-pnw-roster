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
