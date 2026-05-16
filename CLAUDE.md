# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Ghar Kharcha is a two-person household expense tracker. It is a pure static web app — two HTML files with all CSS and JS inline, no build step, no npm, no framework. Open the files directly in a browser or serve them from any static host.

## Running locally

```bash
# Any static server works — e.g.:
python3 -m http.server 8080
# or
npx serve .
```

Then open `http://localhost:8080` (index.html is the entry point).

## File structure

- **`index.html`** — Auth and onboarding flow. Google OAuth sign-in, then create-or-join a group.
- **`app.html`** — The main expense tracker (budget, add/edit/delete expenses, reports, CSV export).

All CSS (`:root` variables, component styles) and all JavaScript (Firebase SDK, app logic) live inline in each file.

## Firebase backend

- **Project:** `ghar-kharcha-24293`
- **Auth:** Google OAuth via `signInWithPopup`
- **Database:** Firestore, real-time with `onSnapshot`

### Firestore data model

```
users/{uid}
  groupId / familyId   ← string (group code like "XKQR-7823")
  name, email, role

groups/{groupId}          ← written by index.html on group creation
  groupName, createdBy, members: { [uid]: { name, email, role, ... } }
  settings.budget

families/{familyId}       ← read by app.html
  settings.name1, settings.name2

families/{familyId}/expenses/{docId}
  who ("p1"|"p2"), amount, desc, year, month, ts (ms string), addedBy, addedByEmail

families/{familyId}/budgets/{YYYY-MM}
  amount, month, year, setAt
```

> **Known discrepancy:** `index.html` creates documents in the `groups` collection and writes `groupId` to the user doc, but `app.html` reads `familyId` from the user doc and queries the `families` collection. These are two separate data paths that have not been unified — if you touch onboarding or auth, check both collections.

## Design system

CSS custom properties defined in `:root` (both files share the same values):

| Variable | Value | Use |
|---|---|---|
| `--primary` | `#C9622F` | Terracotta — CTAs, active states |
| `--accent` | `#2F6E8C` | Teal — Person 1 (p1) highlight |
| `--bg` | `#FDF8F3` | Warm white page background |
| `--success` | `#2D7D4E` | Group code display, remaining budget |
| `--danger` | `#C0392B` | Expense amounts, over-budget |

Fonts: `Playfair Display` (headings, large numbers) + `DM Sans` (body). Both loaded from Google Fonts.

## Key app.html patterns

- **Month navigation:** `curY`/`curM` state variables; `chMon(±1)` reloads expenses and budget for the new month.
- **Real-time sync:** `unsubExpenses` holds the Firestore `onSnapshot` unsubscribe function — always call it before re-subscribing (e.g., on month change).
- **Voice input:** Uses `window.SpeechRecognition` / `webkitSpeechRecognition` with `lang: 'en-IN'`. `parseVoiceInput()` extracts a number (digit or word) and strips filler words to populate the amount/description fields.
- **Action sheet:** Bottom sheet (`sheet` + `sov` overlay) for per-expense edit/delete; `actId` tracks which expense is active.
- **Expense "who":** Two members only, hardcoded as `p1` and `p2`; display names come from `familyData.settings.name1/name2`.
- **Budget bar color:** green → `warn` (orange, ≥70%) → `dng` (red, ≥100%).
