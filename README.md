# E3A Asset Desk

A mobile-friendly equipment checkout/check-in tracker. Officers scan a barcode or QR tag to check equipment out or in; everyone sees the same live, shared list on their own phone. Built as a Progressive Web App (PWA) — no App Store needed, installs straight from a browser.

**Live app:** `https://inquisitive-lamington-91c05f.netlify.app`

---

## 1. What it does

- **Scan to Check Out / Check In** — pick a mode, scan a barcode or QR code (or search manually), done instantly
- **Shared across everyone** — every phone reads and writes to the same live database (Firebase), so one person's scan shows up on everyone else's phone
- **Excel import** — bring in an existing "equipment out for event" spreadsheet in one upload; items not yet marked "Returned" load as checked out
- **Excel export** — pull the current status back out as a spreadsheet at any time
- **No-password login** — pick your name from a staff list; anyone can add themselves (no password, by design, since only a few trusted people have the app link)
- **Printable QR tags** — for equipment that doesn't already have a barcode
- **Activity log** — full history of who checked what in/out and when
- **Works offline-ish** — the app shell loads even with no signal; live data needs a connection

---

## 2. Using the app (for staff)

### Logging in
1. Open the app
2. Tap your name from the list — no password needed
3. Not listed yet? Type your name under "Not on the list?" and tap **Add Me & Continue**
4. Your phone remembers you after that — you won't need to log in again unless you tap **Switch**

### Checking equipment out or in
1. Go to the **Scan** tab
2. Tap **Check Out** or **Check In** (button turns red/green when active)
3. Tap **Start Camera** and point at the barcode/QR tag — or use **Search by name or barcode** and tap the item manually
4. Each scan applies instantly — no confirmation popup, so you can scan a whole batch quickly
5. If an item's already in that state (e.g. already checked in), you'll get a warning toast instead of a duplicate action

### Other tabs
- **Inventory** — see everything tracked, its status, who has it or what event it's out for
- **Log** — full history of check-ins/check-outs
- **Staff** — see or remove staff members

---

## 3. Admin tasks

### Importing an Excel sheet
1. Inventory tab → **Import from Excel**
2. Upload your sheet — it reads: Asset ID, Product Name, Serial Number, Date of Item Drawn Out, Date of Item Returned, Status, Original Storage Location (plus Event Name / Event Location from the header rows)
3. Rows marked "Returned" import as available; everything else imports as checked out, tagged with the event name

### Exporting current status
Inventory tab → **Export to Excel** — downloads a fresh .xlsx with current status, event, and timestamps for every item.

### Adding equipment manually
Inventory tab → fill in name/category → either scan an existing barcode into the "Existing barcode" field, or leave it blank to auto-generate a QR tag (use **Print QR Tags** to print and stick it on the item).

### Clearing everything (start fresh for a new event)
Inventory tab → **Clear All Items & Log** (red button). Warns you if anything's still checked out. Staff list is kept — only equipment and the log are wiped. This cannot be undone.

### Adding/removing staff
Staff tab → add a name, or remove someone (blocked if they currently have equipment checked out).

---

## 4. How it's built (technical overview)

| Piece | What it does |
|---|---|
| Single `index.html` file | The entire app — HTML, CSS, and JavaScript in one file |
| [ZXing](https://github.com/zxing-js/library) | Reads QR codes and standard barcodes (UPC, EAN, Code128, etc.) via the phone camera |
| [QRCode.js](https://davidshimjs.github.io/qrcodejs/) | Generates printable QR tags for equipment without an existing barcode |
| [SheetJS (xlsx)](https://sheetjs.com/) | Reads and writes Excel files entirely in the browser |
| **Firebase Firestore** | The real shared database — every phone reads/writes here, live |
| `manifest.json` + `sw.js` | Makes the app installable to a home screen and gives it an offline app shell |
| Netlify | Hosts the static files and gives you a live HTTPS URL |
| GitHub | Where the source files live; Netlify auto-deploys from here on every upload |

**Data storage:** Equipment, staff, and the activity log are stored in a Firestore collection called `e3a_asset_desk`, as three documents: `equipment-list`, `staff-list`, `activity-log`. The "who's logged in on this phone" value is stored locally on-device (`localStorage`), not in the shared database.

**Security note:** There is no password anywhere in this app — login is just "pick your name," and Firestore rules are set to allow open read/write to the `e3a_asset_desk` collection. This is intentional for a small trusted team, but it means anyone with the app link could read or modify the data. Don't use this for anything higher-stakes without adding real authentication.

---

## 5. Redeploying updates

Whenever you (or I) update `index.html`, `manifest.json`, or `sw.js`:

1. Go to the `asset-desk` repo on GitHub
2. **Add file → Upload files** → select the changed file(s) → **Commit changes**
3. Netlify auto-redeploys within ~30 seconds — no action needed there
4. On your phone, clear the cached version so you see the update:
   - iPhone Settings → **Safari** → **Advanced** → **Website Data** → find `netlify.app` → **Delete**
   - Remove the old home screen icon, reopen the link fresh, **Add to Home Screen** again

*(This manual cache-clear step is a one-time nuisance per update — the app is set up to fetch the latest version automatically going forward, but iOS can still hold onto an old cached copy stubbornly on occasion.)*

---

## 6. Firebase setup (already done, for reference)

If this ever needs to be rebuilt from scratch:

1. [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → name it → skip Analytics
2. **Build → Firestore Database → Create database → Start in test mode** → pick a location (this app uses Singapore/`asia-southeast1`)
3. **Project settings → Your apps → Web (`</>`)** → register an app → copy the `firebaseConfig` block
4. Paste that config into `index.html` (already done — see the `firebaseConfig` object near the top of the script)
5. **Firestore → Rules** → replace the default with:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /e3a_asset_desk/{document=**} {
         allow read, write: if true;
       }
       match /{document=**} {
         allow read, write: if false;
       }
     }
   }
   ```
   → **Publish**. (This avoids the 30-day test-mode expiry that Firebase warns about — this version doesn't expire.)

---

## 7. Troubleshooting

| Problem | Fix |
|---|---|
| Camera says "unavailable" | Only works over HTTPS (the live Netlify link), never on a local `file://` file. Also check the browser was given camera permission. |
| App shows an old design after an update | Clear cached website data for `netlify.app` in Safari settings, remove and re-add the home screen icon. |
| Data disappears when the app is closed | Should no longer happen — this was a bug from before Firebase was connected. If it recurs, check Firestore rules haven't reverted to "deny all." |
| Multiple Netlify sites / GitHub repos | Only one of each should be kept — check **Project configuration → Build & deploy** on Netlify to see which repo it's actually linked to, and delete the other site/repo. |
| "No equipment matches that barcode" | The item needs to be added to Inventory (or imported via Excel) before it can be scanned. |
