# Even Steven v2: shared groups with login (implementation plan)

Audience: an engineering agent implementing this. Read fully before starting. Work in small PRs, one per phase.

## 0. Goal and constraints
Turn the single-file, browser-only app into a small multi-user web app for friends and family.
1. Login, a dashboard of groups, and a group screen that keeps all current features.
2. Invite people to a group; group data persists in Firebase.
3. Expenses and payments added by one member appear for the others in real time.

Constraints: simple, free to run (Firebase Spark plan, no Cloud Functions), works on desktop, tablet and phone (Mac, iPad, iPhone, Android, Windows), installable later as a PWA, keeps the Apple-style UI already in `index.html`, offline tolerant, no secrets in the repo (Firebase web config is public by design; security comes from rules).

## 1. Stack (decided)
| Layer | Choice | Why |
|---|---|---|
| Build | Vite + TypeScript | fast, static output, works on GitHub Pages |
| UI | React 18 + React Router (HashRouter) | well known, hash routing needs no server config on Pages |
| Styling | Plain CSS with CSS variables (port the existing stylesheet into `src/styles/`), no UI kit | keeps the current look, no new dependency |
| Auth | Firebase Auth: Google sign-in + email link | zero passwords to manage |
| DB | Cloud Firestore with offline persistence | real-time listeners, per-record writes, rules, free tier is ample |
| Hosting | Keep GitHub Pages (existing `pages.yml`, switch it to build with Vite and publish `dist/`) | no change for the owner. Add the Pages domain to Firebase Auth authorized domains |
| Tests | Vitest (logic), `@firebase/rules-unit-testing` with the Firestore emulator (rules), Playwright (smoke) | |
| Export | Existing logic: PDF writer, SheetJS (import dynamically so it loads only on export), CSV | keep behavior |

## 2. Target repo layout
```
src/
  main.tsx, App.tsx, routes.tsx
  firebase/        app.ts (init + persistentLocalCache), auth.ts, db.ts (typed converters)
  domain/          money.ts (parseCents, allocate, buildShares), balances.ts (netBalances, directDebts, simplify),
                   report.ts (buildReport, reportSheets, toCsv), pdf.ts (PdfDoc, buildReportPdf), groupFile.ts (validate + import/export JSON)
  features/
    auth/          LoginPage, AuthGate
    dashboard/     GroupList, CreateGroupDialog, JoinPage (/join/:code)
    group/         GroupPage, PeopleBar, ExpenseForm, PaymentForm, TransactionList (swipe-to-delete), SettleView, ReportView, InviteDialog, GroupMenu
  ui/              SegmentedControl, Dialog, Icon (SVG set), Toast, BottomTabs
  styles/          tokens.css (light/dark palettes), base.css, components.css
tests/             domain/*.test.ts, rules/*.test.ts, e2e/*.spec.ts
firestore.rules, firestore.indexes.json, firebase.json, .firebaserc
docs/PLAN.md
```
Phase 1 moves code with no behavior change. The existing pure functions in `index.html` (core logic) move into `src/domain/` as typed modules and keep their tests.

## 3. Data model (Firestore)
All money is integer minor units (cents). Never store floats. Balances are never stored; they are derived from expenses on the client (and in tests).

```
users/{uid}
  displayName, email, photoURL, createdAt
groups/{gid}
  name, currency ('USD'|'EUR'|'GBP'|'INR'|'CAD'|'AUD'), smartSplit: bool
  ownerUid, memberUids: string[]        // used by rules and by the dashboard query
  createdAt, updatedAt
groups/{gid}/people/{pid}               // a "person" in the split; may or may not be a login user
  name, uid?: string (linked login), createdAt
groups/{gid}/expenses/{eid}             // expenses AND payments
  kind: 'expense' | 'payment'
  desc, notes, date: 'YYYY-MM-DD', amount: number (cents)
  payerPid, split: { type:'equal'|'percent'|'exact', values?: {pid: string} }, shares: {pid: number}
  (payments: kind='payment', payerPid=from, shares={toPid: amount})
  createdBy (uid), createdAt, updatedBy, updatedAt
invites/{code}                          // code = 10+ random chars (crypto.getRandomValues)
  gid, createdBy, createdAt, expiresAt, maxUses?, uses
```
Why people are separate from users: today's groups contain named people who may never sign up. A member can claim a person ("this is me") which sets `people.uid`; the dashboard can then show "you owe / are owed" per group.

Dashboard query: `groups where memberUids array-contains uid` (listener). Group screen: listeners on `people` and `expenses` (ordered by `date`, then `createdAt`).

## 4. Security rules (Spark plan, no functions)
Principle: only members read or write a group and its subcollections.
- `groups/{gid}`: read if `request.auth.uid in resource.data.memberUids`. Create if `request.resource.data.ownerUid == request.auth.uid` and `memberUids == [uid]`. Update by members for `name`, `currency`, `smartSplit`. Only the owner may remove members or delete the group.
- Joining: a non-member may update only `memberUids`, adding exactly their own uid, and only if `exists(/databases/$(database)/documents/invites/$(code))` for this gid, not expired, and under `maxUses`. The invite code is passed as a field the rule can verify (for example write `joinCode` in the same update and let rules `get()` the invite; clear the field after). Document this carefully and cover it with rule tests.
- Subcollections `people` and `expenses`: read/write if the caller is in the parent's `memberUids` (use `get(/groups/$(gid)).data.memberUids`). Validate on write: `amount` is an int > 0, `date` matches `YYYY-MM-DD`, `desc` and `notes` length caps, `shares` sum equals `amount`, `createdBy == request.auth.uid` on create.
- `invites`: create only by a group member for their group; read by any signed-in user (code is the secret, unguessable); update only to increment `uses`.
- `users/{uid}`: read and write only by that uid.
Rules are the security boundary; client validation is for UX only. Ship rule tests with the emulator and run them in CI.

## 5. Screens and flows
1. **Login** (`/login`): "Continue with Google" and "Email me a link". After login go to `/` (or the pending `/join/:code`).
2. **Dashboard** (`/`): cards for each group (name, member avatars, your balance, last activity), "New group" (name + currency), "Join with code", profile and sign-out menu, theme setting.
3. **Join** (`/join/:code`): if signed out, sign in and return here. Show the group name, "Join", then add or claim a person.
4. **Group** (`/g/:gid`): current functionality: People bar, Transactions (expense and payment forms, list with swipe-to-delete, edit), Settle (smart split toggle, "Paid in full", "Part payment"), Report (summary, insights, audit, PDF/Excel/CSV), group menu (rename, currency, invite, save group file, delete if owner, leave group). Phone: bottom tab bar as today.
5. **Invite dialog**: shows the share link `https://<site>/#/join/<code>`, copy button, `navigator.share` on phones, optional expiry, regenerate/revoke.
6. Real-time: every list reflects changes from other members within about a second. Show who added/edited an item ("Added by Ben") and a small "synced / offline" indicator.

## 6. Behavior details
- **Offline**: use `initializeFirestore` with `persistentLocalCache({ tabManager: persistentMultipleTabManager() })`. Writes queue offline and sync later. Show the indicator from `onSnapshot` metadata (`hasPendingWrites`, `fromCache`).
- **Concurrent edits**: one document per expense, last write wins per expense, fine for this scale. Deletes are hard deletes; confirm with the existing two-step or swipe pattern.
- **Group file**: keep "Save group file" and "Load group file" as backup/import. Import creates a new group owned by the user (batched writes, chunked at 500 ops). Keep the existing `validateGroup` rules (limits: 100 people, 5000 expenses, name lengths).
- **Migration of local data**: on first login, if `localStorage['even-steven:v1']` exists, offer "Import your existing groups" and then keep a backup file download.
- **Limits** (enforced in UI and rules): 100 people and 5000 expenses per group, 50 groups per user.
- **Accessibility and UI**: keep the Apple-style layer (system fonts, 44px targets, segmented controls, reduced motion/transparency/contrast support). Icons are inline SVG only (no emoji or text glyph icons). Serve UTF-8 normally (no more ASCII escaping once on Vite).
- **Privacy**: collect only name, email and photo from the auth provider. Add a short privacy note on the login page. Provide "Delete my account" (remove the user doc, remove uid from groups, keep expenses with a "former member" label) in a later phase.

## 7. Phases (each ends with passing tests and a deployable build)
**Phase 1: Restructure (no behavior change).** Scaffold Vite + React + TS. Move pure logic to `src/domain/` with typed signatures and port the existing tests to Vitest. Port the UI to React components with local state and the current `localStorage` store behind a `GroupRepository` interface. Update `pages.yml` to build and publish `dist/`. Acceptance: app looks and behaves as it does today; all current tests pass.
**Phase 2: Firebase foundation.** Create the Firebase project, enable Auth (Google, email link) and Firestore. Add `src/firebase/*`, env config via `VITE_FIREBASE_*` (public values). Add `firestore.rules`, emulator config and rule tests. Acceptance: rule tests pass for members vs non-members.
**Phase 3: Auth and dashboard.** Login page, `AuthGate`, dashboard listing groups, create group, sign out. Acceptance: a new user can sign in, create a group and see it after refresh on another device.
**Phase 4: Group data on Firestore.** Implement `FirestoreGroupRepository` (people, expenses, payments) with real-time listeners; port forms, list, settle and report to it. Acceptance: two browsers on different accounts see each other's changes live.
**Phase 5: Invites.** Invite dialog, `/join/:code`, rules for joining, person claiming. Acceptance: an invited user joins via link and sees the group; an expired or revoked link fails; a non-member cannot read anything (rule tests).
**Phase 6: Polish.** Offline indicator and tests, import/export, local-data migration, empty and error states, loading skeletons, "added by" labels, leave/delete group, account deletion.
**Phase 7 (optional).** PWA (manifest, service worker, icons), then new tools (budget, bills) as modules under `features/`.

## 8. Definition of done (overall)
- Fresh user signs up, creates a group, invites a friend by link; both add expenses and payments and see the same balances in real time.
- Smart split, percent/exact splits, payments (partial or full), dates, notes, PDF/Excel/CSV and group file still work.
- Works at 390px (phone), 768px (tablet) and desktop widths, in light and dark mode.
- Rules tests, domain tests and one end-to-end smoke test run in CI on every PR.
- No secrets committed; Firebase usage stays within the Spark free tier for normal use.

## 9. Risks and notes
- Join-by-invite via rules is the trickiest part; if it proves too fragile, fall back to upgrading to the Blaze plan and using a small callable Cloud Function (still within free quotas for this scale).
- Keep Firestore reads low: one listener per open group, not per component.
- Email-link sign-in on iOS Safari needs the same browser to complete the flow; keep Google sign-in as the primary option.
- Do not store derived balances; always compute from expenses to avoid drift.
