# Cloud Diaries — Production Upgrade Package

This package updates the original single-file Cloud Diaries site while preserving its dark sakura styling, scrapbook editor, themes/templates, public wall, private vault, search, likes/comments UI, invites, and admin console UI. It adds a personal Studio with a dashboard, planner, habit tracker, bookmarks/collections, customization controls, JSON backup/restore, PWA installation files, Firebase Authentication, separated public/private data paths, callable server-side like/comment operations, and a collaborative-room scaffold.

## Important status: not yet live or production-verified

The HTML has been syntax-checked locally, but it has **not** been tested against your live Firebase project. I do not have your Firebase Console access or trusted deployment credentials. The backend features are not live until you configure Authentication, deploy the included Realtime Database rules and Cloud Functions, and complete the data migration below.

The existing database uses legacy `/users` and `/posts` records, including plaintext passwords and recovery answers. The upgraded frontend uses Firebase Authentication and new paths (`/profiles`, `/public_posts`, `/private_posts`, `/collaborationRooms`). The included rules intentionally deny access to legacy `/users` and `/posts`. **Do not deploy the new rules before backing up and migrating your legacy data.** Old username/password accounts cannot be safely imported as-is because they lack Firebase Auth identities and verified email addresses; users need to register again.

## Feature status

- **Personal dashboard:** counts and summary based on the signed-in profile, visible posts and local planner data.
- **Planner and habit tracker:** usable in this browser; stored in localStorage, not synced between devices yet.
- **Bookmarks and collections:** usable in this browser; saved per browser profile, not synced between devices yet.
- **Security upgrade:** Firebase email/password Authentication, Firebase-managed reset emails, no browser-stored account password, no client-side admin master key/OTP/security-answer authorization, separated public/private database paths, and deny-by-default rules. Publishing, liking/commenting and collaboration require a verified email. Requires deployment and migration.
- **Collaborative scrapbook:** real-time shared room content is wired to Realtime Database. Requires Firebase Auth and deployed rules. A room ID is a bearer invite; share it only with intended collaborators.
- **Advanced customization:** accent colour, compact/comfortable density preference, reduced-motion preference. Stored in this browser.
- **Installable PWA:** manifest, icons, and service worker included; host over HTTPS. The service worker caches the local app shell only. Since the app still loads some libraries from CDNs, this is **not a fully offline-capable app**.
- **Export and backup:** exports user-visible entries plus local planner/bookmark/customization data as JSON. Restore imports entries as private copies into the currently signed-in account.

## Deploy sequence

1. **Back up first.** Export your current Realtime Database from Firebase Console or use a trusted Admin SDK backup. Keep the backup private because the old `/users` node contains plaintext passwords and recovery answers.
2. Install Firebase CLI and authenticate with a trusted project owner account:
   ```bash
   npm install -g firebase-tools
   firebase login
   ```
3. Review `.firebaserc.example`; copy it to `.firebaserc` if the project ID is correct. The HTML and migration helper currently point at `sakura-no-hana-9d9f9`. Verify that this is your intended Firebase project before running commands.
4. Enable **Authentication → Sign-in method → Email/Password** in Firebase Console. Add your Hosting domain to Authorized domains.
5. Install backend dependencies:
   ```bash
   cd functions
   npm install
   cd ..
   ```
6. **Run a first migration/export before deploying the new rules.** With trusted Application Default Credentials (for example, after `gcloud auth application-default login` using a project-owner account), run:
   ```bash
   node scripts/migrate-legacy-posts.js
   ```
   It copies public entries into `/public_posts`, creates private-entry backup JSON files under `migration/private-backups/`, and writes a **sanitized** user index without passwords or recovery answers. Initially, public entries will have `ownerUID: "legacy-unclaimed"`; that is intentional until the new accounts exist. Inspect the output.
7. Deploy the new rules, functions, and website:
   ```bash
   firebase deploy --only database,functions,hosting
   ```
   The rules deny `/users` and `/posts`, and allow each newly registered account to create its own `/profiles/{uid}` record. Users will need to register again with email/password and verify their email; old credentials are not migrated.
8. After the new accounts exist, create `migration/user-map.json` mapping each old username to the new Firebase Auth UID; see `migration/user-map.example.json`. Rerun `node scripts/migrate-legacy-posts.js` to assign public-entry ownership where a mapping exists. Each user can restore their private backup by signing into the new account and using **Studio → Restore JSON**. Keep private backups until each person confirms restoration.
9. Bootstrap the first admin role **from a trusted machine only**. Find the intended admin's Firebase Auth UID, set Application Default Credentials, then run:
   ```bash
   cd functions
   ADMIN_BOOTSTRAP_EMAIL=your-verified-email@example.com node set-admin-claim.js FIREBASE_AUTH_UID
   ```
   The email must match the target account. Sign out and sign in again to refresh its ID token. Never expose a service-account key or run this bootstrap in browser code. After the first admin is set, admins can grant/revoke roles using the `setAdminRole` callable function.
10. Open the hosted HTTPS URL and test the checklist below before inviting users.

> Do not use `--purge-legacy` until every required backup has been reviewed and the new accounts/private entries have been verified. It permanently removes the legacy `/users` and `/posts` nodes.
9. Bootstrap the first admin role **from a trusted machine only**. Find the intended admin's Firebase Auth UID, set Application Default Credentials, then run:
   ```bash
   cd functions
   ADMIN_BOOTSTRAP_EMAIL=your-verified-email@example.com node set-admin-claim.js FIREBASE_AUTH_UID
   ```
   The email must match the target account. Sign out and sign in again to refresh its ID token. Never expose a service-account key or run this bootstrap in browser code. After the first admin is set, admins can grant/revoke roles using the `setAdminRole` callable function.
10. Open the hosted HTTPS URL and test the checklist below before inviting users.

> Cloud Functions deployment may require the Firebase/Google Cloud billing plan and APIs appropriate to your project. Check the current Firebase plan and enable App Check after testing if you want additional abuse protection.

## Verification checklist

- [ ] New account registration creates Firebase Auth user and `/profiles/{uid}` only.
- [ ] Password reset sends Firebase-managed email; no security answers or passwords appear in Realtime Database.
- [ ] User A cannot read `/private_posts/{uid-of-user-B}` or `/profiles/{uid-of-user-B}`.
- [ ] Signed-out users can read only `/public_posts`; they cannot read legacy `/users` or `/posts`.
- [ ] Non-owner cannot edit/delete another person's public entry.
- [ ] Like/comment callable functions validate authentication and update public entries.
- [ ] A private post is stored under `/private_posts/{currentUid}` and is invisible to other accounts.
- [ ] Collaborative room members can live-sync content; an unlisted user cannot read a room without its ID and successful membership join.
- [ ] Admin operations fail for accounts without the server-side `admin` custom claim.
- [ ] Backup JSON exports correctly and restores into a test account.
- [ ] PWA install prompt/“Install app” is available on a supported browser over HTTPS.

## Files

- `public/index.html` — updated site frontend.
- `public/manifest.webmanifest`, `public/sw.js`, `public/icons/` — PWA assets.
- `database.rules.json` — deny-by-default Realtime Database rules for the new schema.
- `functions/index.js` — trusted callable functions for likes/comments and admin-role management.
- `functions/set-admin-claim.js` — one-time trusted admin bootstrap.
- `scripts/migrate-legacy-posts.js` — legacy data migration/export helper.
- `migration/` — migration mapping example and generated-backup destination.

## Security caveats to review

- Firebase web config/API key is a client identifier, not an authorization secret. Restrict its API usage in Google Cloud Console and rely on Authentication + Database Rules for authorization.
- This first migration version does not automatically transfer old accounts or passwords into Firebase Auth. Re-register users and manually restore private entries.
- The planner, habits, bookmarks, collections and visual preferences are browser-local. Do not treat localStorage as encrypted or as a secure vault.
- The original rich-text editor renders HTML. Before accepting untrusted public content at scale, add a vetted HTML sanitizer and consider Firebase App Check, rate limiting, reporting/moderation, and storage-backed image uploads.
- The collaborative room ID acts like an invitation secret. Use high-entropy IDs, distribute privately, and add moderation/member-management UX before a large public launch.

## Rebranding note

The visible product name is now **Cloud Diaries**. The existing Firebase project ID and internal local-storage/backup format identifiers were intentionally retained to avoid breaking the configured backend or existing local backup compatibility. The PWA icon files retain the original sakura artwork, renamed for Cloud Diaries.
