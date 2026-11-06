# MBF EDITOR

A React, Vite, and Tailwind CSS editor for personalizing and exporting loan approval letters. The project also includes an Electron wrapper for Windows.

## Requirements

- Node.js 22 or later
- npm

## Local development

```sh
npm ci
npm run dev
```

Create a production build with:

```sh
npm run build
```

The generated static site is written to `dist/`.

## Google sign-in and private account data

The hosted web app requires Google sign-in. Each account's profile and letter are stored in that user's private Firestore and Storage paths; the browser also keeps a UID-specific local cache. Existing unscoped browser data is not automatically copied into a signed-in account. The Windows desktop app opens directly in local-only mode: letter and profile data stay on that device, with no Google sign-in or cloud sync. After exporting a letter, File Explorer automatically opens with the saved file selected; use the **Show folder** button in the success message to open it again. Browser downloads remain controlled by the browser and cannot trigger Explorer to reveal their local download path. Use the hosted HTTPS app for account access and cloud sync.

To configure Firebase:

1. Create a Firebase project and register a **Web app**.
2. In **Authentication → Sign-in method**, enable **Google**. Add the deployed web app's hostname (such as `your-name.github.io` or your custom domain) under **Authorized domains**.
3. Create a Cloud Firestore database and enable Firebase Storage.
4. Copy `.env.example` to `.env.local` for local development and fill in the five Firebase web app values. These are public client configuration values; do not put service-account credentials in the frontend.
5. In GitHub repository **Settings → Secrets and variables → Actions → Variables**, add `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_STORAGE_BUCKET`, and `VITE_FIREBASE_APP_ID`. The Pages workflow injects these values at build time.
6. Deploy the included owner-only Firestore and Storage rules to the same Firebase project:

   ```sh
   npx firebase-tools deploy --project YOUR_FIREBASE_PROJECT_ID --only firestore:rules,storage
   ```

The production build can still complete without Firebase values, but the sign-in screen will explain that account access is not configured. Do not deploy the site as an account-enabled app until the Firebase settings, Google provider, authorized domain, and rules are configured.

## Google Sheets usage tracking

The app can send `Opened App` once per app load and `Saved Loan Letter` after a successful Ctrl/Cmd+S save or letter export. Each event contains the account email (or `Guest User`), a persistent browser-local `MBF-MAC-XXXXXXXX` ID, and the action name. This is usage data: disclose it to users and obtain any consent required for your deployment before enabling it.

The app already sends the Machine ID in its request. The Apps Script endpoint must read `machineId` from the JSON body and write it into column C; a header alone does not populate that column. The ready-to-deploy handler is in [`google-apps-script/Code.gs`](./google-apps-script/Code.gs).

To connect it to the usage-tracking spreadsheet:

1. Open the spreadsheet and select **Extensions → Apps Script**.
2. Replace the editor's `Code.gs` contents with `google-apps-script/Code.gs` and save. The handler writes `Timestamp`, `User Email`, `Machine ID`, and `Action` to the first sheet (or `Sheet1` if present); it creates those headers only if the first row is empty.
3. Select **Deploy → Manage deployments**, edit the existing Web App deployment, select a new version, and deploy. If there is no existing deployment, create a **Web app** deployment that executes as you and allows access to anyone who uses the app. Copy its `/exec` URL.
4. Ensure the deployed URL is the one configured as `VITE_GOOGLE_APPS_SCRIPT_URL` in `.env.local` and as the GitHub Actions variable for hosted builds. Rebuild and redeploy the app only if the URL changed; updating the Apps Script deployment at the same URL does not require an app rebuild.
5. Open the app or save a letter, then check for a new row. The Machine ID should appear in column C. Existing rows are not backfilled.

The URL must use the `https://script.google.com/macros/s/.../exec` format. The app sends JSON as `text/plain` with `no-cors` mode to avoid a browser preflight; this is a one-way request, so the browser cannot verify whether Apps Script accepted or appended the row. A missing URL in the build is reported in the browser console. For a deployed GitHub Pages app, `.env.local` is not uploaded; set the Actions variable and run the Pages deployment again. A client-side email or machine ID is not verified identity and can be altered; do not use these rows for authentication or security decisions.

The public Web App endpoint accepts only the app's two tracking actions and the expected Machine ID format, but it is still publicly callable. Do not store sensitive information in these tracking rows.

## GitHub Pages

The `Build` workflow checks pull requests. The `Deploy to GitHub Pages` workflow builds and publishes `dist/` after a push to `main`.

In the repository's **Settings → Pages**, set the build and deployment source to **GitHub Actions**.

This project uses relative Vite asset paths so the built site can be served from a GitHub Pages project subpath as well as a root domain.

## Windows desktop build

```sh
npm run dist:win
```

Windows packages are written to `release/` and are intentionally excluded from Git.

## Update checks

The installed Windows desktop app checks GitHub Releases on startup and from **Check Update**. Available updates download automatically; use **Restart to update** after the download completes. The hosted web app and development builds do not run the Electron updater.

MBF EDITOR uses its own public update repository, `iismenunz168-cpu/mbf-editor`, so releases for other apps cannot replace its `latest.yml`. The Windows NSIS build generates `latest.yml`, the versioned installer, and its `.blockmap`; all three must be attached to the same GitHub release. The release workflow publishes these assets from **Actions → Publish MBF EDITOR desktop release → Run workflow**.

If that workflow runs in a repository other than `iismenunz168-cpu/mbf-editor`, add an Actions secret named `MBF_RELEASE_TOKEN` there. Use a fine-grained token with **Contents: read and write** permission for the `mbf-editor` repository. The built-in `GITHUB_TOKEN` works when the workflow runs inside the target repository itself.

The update feed is active only after a release is published with a `v`-prefixed tag matching the app version (for example, `v1.0.15`) and includes the generated `latest.yml`. Since existing installed builds do not contain the updater, install this first updater-enabled release manually; later releases can update automatically.

## Linting

No lint script is currently configured. CI verifies dependency installation and the production Vite build.
