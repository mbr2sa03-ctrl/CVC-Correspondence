# MBR2 Sponsorship Dashboard

Two connected apps sharing one live dataset:

- **`dashboard.html`** — full sponsorship management dashboard (RC info, correspondence, monitoring, SOI, settings, name translator, etc.)
- **`cvc-app.html`** — lightweight, phone-friendly single-screen build for CVC volunteers to action correspondence in the field

Both files run the exact same underlying logic that was built and tested inside Claude. The only thing that changed to make them work outside Claude is how they talk to storage.

## How they connect

Both files load `js/storage-firebase.js` before their own app script. That file recreates the `window.storage.get/set/delete/list` API the app already expects — but backed by your Firestore project (`mbr2-dashboard`) instead of Claude's built-in artifact storage. Because it's the same Firestore project and the same storage collection (`sd_storage`), **the dashboard and the CVC app read and write the same data** — a CVC marking a letter "Done" on their phone shows up in the dashboard's Correspondence tab, and RC records imported in the dashboard show up in the CVC app.

Nothing in the ~2,500 lines of existing app logic in either HTML file needed to change — they were already written entirely around the `window.storage` interface, and `storage-firebase.js` is a drop-in replacement for it.

## Setup

1. **Firestore rules** — in the [Firebase Console](https://console.firebase.google.com/) → Firestore → Rules, paste in `firestore.rules` from this repo and publish. This scopes read/write to only the collection the app uses.
2. **Firestore database** — make sure you've created a Firestore database (Native mode) in the `mbr2-dashboard` project if you haven't already (Firebase Console → Build → Firestore Database → Create database).
3. That's it — the Firebase config is already embedded in `js/storage-firebase.js`.

## Deploy (GitHub Pages)

1. Push this repo to GitHub.
2. Repo Settings → Pages → Source: deploy from branch `main`, folder `/ (root)`.
3. Your two apps will be live at:
   - `https://<username>.github.io/<repo>/dashboard.html`
   - `https://<username>.github.io/<repo>/cvc-app.html`
4. Share the `cvc-app.html` link with your CVC volunteers, and the `dashboard.html` link with your team.

## Known limitations (please read)

**No login.** The Firestore rules are open — anyone with the link (and the public Firebase config, which is unavoidable in client-side apps) can read and write data. This matches the app's current "select your name from a dropdown" model, but it means the link itself is the access control. Don't post it anywhere public. If you need real access control later, add Firebase Authentication (email link sign-in works well for teams without technical staff) — ask and I can wire that in.

**Concurrent edits can overwrite each other.** The app saves by rewriting the *entire* dataset for a given key (e.g. the whole RC roster, or the whole correspondence list) whenever anything in it changes, rather than writing just the one record that changed. If two people edit different records at almost the same moment, whichever save finishes last wins — the other person's change can be silently lost. This was fine when it was one person using Claude's artifact storage, but is a real risk now that multiple CVCs and staff can use it at once from different devices. It won't come up often (writes are fast and most edits are a few seconds apart), but if your team starts noticing changes "disappearing," this is why. I can rebuild the storage layer to save one record at a time instead, which would remove this risk — happy to do that as a follow-up if it becomes a problem.

**Large datasets auto-split across Firestore documents.** Firestore caps each document at ~1MB. Once your RC roster or correspondence log grows large enough to exceed that, `storage-firebase.js` automatically splits the value across multiple documents and reassembles it on read. You don't need to do anything for this to work, but it does mean a very large save touches several Firestore writes instead of one.

## File structure

```
dashboard.html          full dashboard (all tabs)
cvc-app.html             CVC-only phone build (single tab)
js/storage-firebase.js  Firestore-backed window.storage implementation (shared by both apps)
firestore.rules          Firestore security rules — paste into Firebase Console
```
