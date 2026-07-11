# THE ONE — Firebase (login + private data + live hosting + CI/CD)

This version gives you real **account protection**: you sign in with Google, and your data lives in Firestore behind security rules that make it readable **only by you** — not by anyone who has the URL. It also deploys to Firebase Hosting with **CI/CD**, so every push auto-updates the live site, which is perfect for building features later.

**What's in this folder**

```
the-one-firebase/
├── public/
│   ├── index.html            ← the app (edit the firebaseConfig near the top)
│   ├── favicon.svg
│   └── manifest.webmanifest
├── firestore.rules           ← security: each user sees only their own data
├── firestore.indexes.json
├── firebase.json             ← hosting + firestore config
├── .firebaserc               ← your project id
├── .github/workflows/
│   ├── firebase-deploy.yml   ← CI/CD: auto-deploy on push to main
│   └── firebase-preview.yml  ← preview URL for each pull request
└── SETUP-Firebase.md         ← this guide
```

> You'll do a one-time setup. I can't log into Firebase for you, but every step below is copy-paste.

---

## Prerequisites

- A Google account (you already have kalab1275@gmail.com).
- **Node.js** installed (https://nodejs.org) — needed for the Firebase CLI.
- Install the Firebase CLI once:

```bash
npm install -g firebase-tools
firebase login
```

---

## Step 1 — Create the Firebase project

1. Go to https://console.firebase.google.com → **Add project**. Name it e.g. `the-one`. You can disable Google Analytics.
2. When it's ready, open the project.

## Step 2 — Enable Google sign-in

1. Left sidebar → **Build ▸ Authentication ▸ Get started**.
2. **Sign-in method** tab → **Google** → enable → pick a support email → **Save**.

## Step 3 — Create the database

1. Left sidebar → **Build ▸ Firestore Database ▸ Create database**.
2. Choose **Production mode** (we ship our own rules) → pick a location → **Enable**.

## Step 4 — Register a Web App and copy the config

1. Project **Settings** (gear icon, top-left) → **General** → scroll to **Your apps** → click the **`</>`** (Web) icon.
2. Nickname it `the-one-web` → **Register app** (skip hosting for now) → you'll see a `firebaseConfig = { ... }` block.
3. Copy those values into **`public/index.html`** — find the `firebaseConfig` object near the top of the `<script type="module">` and replace the `PASTE_...` placeholders:

```js
const firebaseConfig = {
  apiKey: "AIza…",
  authDomain: "the-one-xxxx.firebaseapp.com",
  projectId: "the-one-xxxx",
  storageBucket: "the-one-xxxx.appspot.com",
  messagingSenderId: "1234567890",
  appId: "1:1234567890:web:abc123"
};
```

> These values are **not secret** — they're meant to live in client code. Your data is protected by Auth + the security rules, not by hiding the config.

4. Put your **project id** into **`.firebaserc`** (replace `PASTE_PROJECT_ID`) and into both workflow files in `.github/workflows/` (replace `PASTE_PROJECT_ID`).

## Step 5 — Point the CLI at your project & deploy

From inside the `the-one-firebase` folder:

```bash
firebase use --add            # pick your project, alias it "default"
firebase deploy --only firestore:rules,hosting
```

When it finishes, it prints your live URL:

```
Hosting URL: https://the-one-xxxx.web.app
```

Open it, click **Continue with Google**, and you're in. Your data now syncs live to Firestore.

## Step 6 — Authorize your domains (usually automatic)

If Google sign-in shows an `auth/unauthorized-domain` error, add your domains in **Authentication ▸ Settings ▸ Authorized domains**: `localhost`, `the-one-xxxx.web.app`, and `the-one-xxxx.firebaseapp.com` are added by default, so this is rarely needed.

---

## Step 7 — CI/CD: auto-deploy on every push

This is the part that makes future features one command to ship.

1. Put the folder in a **GitHub repo** (create one, then):

```bash
git init && git add . && git commit -m "THE ONE (Firebase)"
git branch -M main
git remote add origin https://github.com/<you>/the-one.git
git push -u origin main
```

2. Wire the deploy secret automatically:

```bash
firebase init hosting:github
```

Answer the prompts (point it at your repo). This command:
- creates a **service account** and stores it as the GitHub secret **`FIREBASE_SERVICE_ACCOUNT`**, and
- may add its own workflow file — you can keep mine (`firebase-deploy.yml`) or its version; just make sure only one deploys the `live` channel.

3. Make sure `PASTE_PROJECT_ID` is replaced with your real project id in both `.github/workflows/*.yml`, commit, and push.

From now on: **push to `main` → GitHub Actions deploys to your live URL automatically.** Open a **pull request** → you get a temporary **preview URL** to test a feature before merging.

> Note: the GitHub Action deploys **hosting**. When you change `firestore.rules`, deploy those with `firebase deploy --only firestore:rules` (or add it to the workflow with a full-CLI step).

---

## Put it on your phone

Open your live URL on your phone, sign in, then:
- **iPhone (Safari):** Share ▸ **Add to Home Screen**
- **Android (Chrome):** ⋮ ▸ **Install app**

It runs full-screen with the THE ONE icon, and shows the same private data as your laptop.

---

## How your privacy actually works

`firestore.rules` contains:

```
match /users/{userId} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}
```

This means a request can only read or write the document whose id equals the **signed-in user's uid**. Even though the app's code and URL are public, **the data is not** — Firestore rejects any read of another user's document. That's the difference between real auth and the old "secret URL" approach.

---

## Building features later

- Edit `public/index.html`, commit, push → it's live in ~1 minute.
- New data fields just go inside the same per-user document; no rule changes needed.
- If you add new collections, add matching rules and run `firebase deploy --only firestore:rules`.
- Use pull requests to get preview URLs before shipping.

## Troubleshooting

- **Login screen says "connect to Firebase"** → you haven't replaced the `PASTE_...` values in `public/index.html`.
- **`auth/popup-blocked`** → allow popups for your site, or click the button again.
- **`Missing or insufficient permissions`** → rules didn't deploy; run `firebase deploy --only firestore:rules`.
- **CI fails on deploy** → check `FIREBASE_SERVICE_ACCOUNT` secret exists and `projectId` in the workflow matches your project.
