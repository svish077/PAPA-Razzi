# Papa-Razzi Affair

A private daily scoreboard for **Vish** and **Papz**, who play **Wordle**, **Tradle** and **Krillion** every day and want to know who won each game and each day.

- **Live site:** https://svish077.github.io/PAPA-Razzi/ (GitHub Pages; the path is case-sensitive)
- **Repo:** https://github.com/svish077/PAPA-Razzi
- **Data:** Firebase Firestore, project `papa-razzi-7e5c7`
- **Status:** working in production as of 2026-10-06

This README doubles as a handoff note: it records what was built, how it works, the decisions made and why, so work can continue in VS Code / Claude Code without the original chat.

---

## Files

| File | What it is |
|---|---|
| `index.html` | The entire app: HTML, CSS and JS in one file. No build step. |
| `firestore.rules` | Firestore security rules (a copy; the live rules are pasted into the Firebase console). |
| `README.md` | This file. |

---

## How it works

**Stack:** static `index.html` on GitHub Pages → Firebase JS SDK v10.12.2 loaded as ES modules from `https://www.gstatic.com/firebasejs/10.12.2/…` → Firestore for storage, Firebase Auth (email/password) for access.

**Why Firebase and not GitHub for data:** GitHub Pages only serves static files. Saving scores into the repo would need a GitHub token embedded in a public page (anyone could copy it) or a GitHub account for each player (Papz doesn't have one). Firestore gives free, live-syncing storage behind a login.

**Auth model:** a single shared login (`vishva92@gmail.com` + an app-only password set in Firebase Authentication; it is *not* the Gmail password). Both players sign in with it. Either player can enter scores for both sides. Signing in persists per device.

**Firebase config** is inlined at the top of the `<script type="module">` in `index.html` (`apiKey`, `authDomain`, `projectId`, `appId`). These values are not secret; the Firestore rules are what protect the data.

**DB adapter:** the app was first built as a claude.ai Artifact using that platform's Firestore-like `db` API. When it moved to Firebase, a tiny adapter `makeDb(fs)` was added so the rest of the code kept calling `db.doc(path).set/onSnapshot` and `db.collection(path).limit(n).onSnapshot`. New code can use the Firebase modular API directly.

---

## Data model (Firestore)

```
settings/players            { p1: { name: "Vish" }, p2: { name: "Papz" } }
scores/{YYYY-MM-DD}_{p1|p2} { date: "2026-10-06", player: "p1",
                              wordle: 1-6 | "X" | null,
                              tradle: 1-6 | "X" | null,
                              krillion: <integer points> | null }
```

- **`p1` = Vish, `p2` = Papz.** Keys are fixed; display names come from `settings/players` and can be edited in the app ("Edit names").
- One document **per player per day**, so the two players never overwrite each other's entries.
- The app subscribes to the whole `scores` collection (`limit(1000)`), which is about 1.4 years of data at 2 docs/day. Raise or paginate before then.
- Dates are the device's **local** date (Geneva time).

---

## Scoring rules

Defined in the `GAMES` array near the top of the script:

```js
const GAMES=[
  {k:'wordle',  name:'Wordle',  type:'guess',  rule:'fewest guesses'},
  {k:'tradle',  name:'Tradle',  type:'guess',  rule:'fewest guesses'},
  {k:'krillion',name:'Krillion',type:'points', rule:'most points'}
];
```

- **`guess` games:** 1–6 or `X` (missed). Fewest guesses wins; `X` counts as 7, so any solve beats a miss and X vs X is a tie.
- **`points` games:** any integer, highest wins. (Krillion: 7 prompts/day, up to 100 points each.)
- A game is only decided when **both** players have a score.
- **Day winner:** whoever won more of the decided games that day; equal = tie.
- A day is **final** when all games are decided, or when it's in the past and at least one game was decided. Only final days count toward "days won", streaks and the lead chart.

**Adding a game** with an existing scoring type = add one line to `GAMES` (`k` must be a stable lowercase id). Everything else (entry row, history chips, tallies, averages chart, day winner) updates automatically. Other scoring styles (time-based, `/7` games, mistakes where 0 is best) need a new `type` handled in `val()`, `cmp()`, `clean()`, `fmt()`, the input builder in `buildRows()` and the averages chart.

---

## Features (current)

1. **Scoreboard:** days won per player, per-game win counts, current streak sentence, last-10-days form dots.
2. **Day entry card:** date navigation (prev/next/date picker, no future dates). Dropdowns (1–6, X) for guess games, a number input for Krillion. Saves on change. Winner highlighted per game, plus a day summary line.
3. **Paste share text:** parses Wordle (`Wordle 1,570 4/6`) and Tradle (`#Tradle #1300 X/6`) share strings for a chosen player. **Krillion's share format is unknown**; the regex only catches `NNN pts/points` next to the word "Krillion". Needs a real share sample to fix.
4. **"Who's ahead, day by day" chart:** SVG line of the running lead in days won. **Papz above the line, Vish below** (requested). Area shaded in the leader's colour, hover tooltip per day, endpoint label. Appears after 2 finished days.
5. **"Who's better at what" chart:** paired horizontal bars of each player's average per game (X = 7 for averages), with an "Edge" tag.
6. **History list:** every day newest first, with per-game chips and the day winner. Tap a day to edit it.
7. Sign-in card, sign-out link, light/dark themes, mobile-first layout, home-screen meta tags.

**Design:** player colours are orange (`--p1`, Vish) and purple (`--p2`, Papz), validated for colour-blind separation in light and dark. Game accents: Wordle green, Tradle blue, Krillion teal. Fonts: Bricolage Grotesque (display), Figtree (body), JetBrains Mono (numbers), all from Google Fonts.

---

## Firebase setup (already done; for reference)

1. Project `papa-razzi-7e5c7` created with a personal Gmail account (no organisation / parent resource).
2. **Firestore** (not Realtime Database) created in **locked/production mode**, European location. A Realtime Database may also exist in the project; it is unused and can be ignored or deleted.
3. **Authentication → Email/Password** enabled. One user: `vishva92@gmail.com`.
4. **Firestore Rules** (live copy in the console):
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       function isPlayer() {
         return request.auth != null
           && request.auth.token.email in ['vishva92@gmail.com'];
       }
       match /{document=**} {
         allow read, write: if isPlayer();
       }
     }
   }
   ```
   To give Papz her own login later: add her user in Authentication, add her email to the list, and publish.
5. Web app registered; config values copied into `index.html`.
6. **Authentication → Settings → Authorized domains** includes `svish077.github.io`.

**Gotchas hit during setup:**
- "Line 1: Parse error" when saving rules = rules pasted into **Realtime Database** instead of **Firestore**.
- Curly quotes from TextEdit/Word break the rules; paste straight into the console editor.

---

## Deploying changes

Edit `index.html` → commit → push to `main`. GitHub Pages redeploys in about a minute. On phones, close and reopen the home-screen app (or pull to refresh) to pick up the new version. Scores in Firestore are unaffected by code changes.

To test locally, serve the folder (e.g. `npx serve .` or VS Code Live Server) and add `localhost` to Firebase **Authorized domains** if sign-in complains. Local testing writes to the **real** database.

---

## History of decisions (from the original chat, 2026-10-06)

- Started as a claude.ai Artifact with shared storage. Moved to GitHub Pages + Firebase so it has its own link and Papz doesn't need a claude.ai account.
- Name went through "Daily Puzzle Duel" → "Ranked Couples Therapy" → "Papa-razzi & Vishy Business" → "Papa-razzi" → **"Papa-Razzi Affair"**.
- Charts added on request; lead chart flipped so Papz is above the line.
- Single shared login chosen for simplicity.
- The original claude.ai Artifact version still exists separately but is no longer the main app. Its one day of data (2026-10-06) was re-entered manually.

---

## Ideas backlog (not built)

- Fix Krillion paste parsing once a real share string is captured.
- "Paste the whole morning": one box that fills all games at once (Wordle/Tradle parsing already exists).
- Weekly/monthly verdicts and trophies; streak badges.
- Groups beyond two players (family/office league): would need player docs instead of fixed `p1`/`p2` keys and per-user logins.
- Join-by-invite instead of creating logins in the Firebase console.
- Split `index.html` into separate CSS/JS files if it keeps growing.
- Pagination or yearly archive docs before the 1,000-document subscription limit (~2027-03).
