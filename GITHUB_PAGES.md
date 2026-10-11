# Publishing the front end on GitHub Pages

1. Create a repo (for example `cloud-diaries`) and upload the contents of the `public/` folder to the repo root, so `index.html`, `manifest.webmanifest`, `sw.js`, `favicon.png`, `favicon.ico` and `icons/` sit at the top level.
2. In the repo go to Settings > Pages, choose the `main` branch and `/ (root)`.
3. In the Firebase Console go to Authentication > Settings > Authorized domains and add `kuro2607.github.io`.
4. GitHub Pages only serves the website. Sign-in, the database rules and the Cloud Functions still need to be deployed with the Firebase CLI as described in README.md. Do that, including the migration steps, before inviting users.
5. Keep `functions/`, `scripts/`, `migration/` and any service-account keys out of the public repo. The included `.gitignore` already skips the private backup files.
