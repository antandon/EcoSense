# EcoSense

EcoSense is a smart waste-bin monitoring dashboard. Each bin has a load cell and fill-level sensor connected to an ESP32, which streams live readings to Firebase. The web app shows every bin's weight and fill level in real time, raises on-screen and audible warnings when a bin is nearly full or too heavy, and sends WhatsApp alerts to the people responsible for emptying it.

## How it works

1. **Sensors** — an ESP32 (`Weightwise_arduino/`) reads the load cell and fill level, then pushes `weight` and `level` for its bin to **Firebase Realtime Database**.
2. **Log in** with a 10-digit PIN. Accounts have a role: `admin` or `user`.
3. **Monitor bins** — the dashboard streams each bin's data live, showing weight (g/kg), an animated fill-level gauge, a recent-readings chart, and the last-seen timestamp. Status is colour-coded: green below 75%, yellow 75–90%, red above 90%.
4. **Get warned** — when a bin crosses the configured level or weight threshold, a warning appears in the app with a beep sound. Bins that stop reporting for 30 minutes are flagged as offline.
5. **WhatsApp alerts** — a server-side flow sends `level_alert` / `weight_alert` template messages to the configured recipients through the **WhatsApp Cloud API**, with retry/backoff, rate limiting and a de-duplication lock so the same alert isn't sent twice. Delivery status and errors are logged to Firestore.
6. **Admins** manage users and roles, edit global settings (thresholds, alert interval, recipient numbers), and can send test WhatsApp messages from the admin panel.
7. The sidebar has a **search bar** to find bins by name, location or device ID.

## Tech stack

- **Frontend**: Next.js 15 (App Router), React 18, TypeScript, Tailwind CSS, shadcn/ui (Radix), Recharts
- **Auth**: Firebase Authentication (PIN-based login with role stored in Firestore)
- **Data**: Firebase Realtime Database (live sensor data) + Cloud Firestore (bins, users, settings, alerts, WhatsApp logs)
- **Server**: Next.js API routes with `firebase-admin`; Genkit flow for alert handling
- **Messaging**: WhatsApp Cloud API (Meta) with a webhook for delivery status and incoming messages
- **Hardware**: ESP32 + load cell, Arduino/PlatformIO sketch using `Firebase_ESP_Client`
- **Hosting**: Firebase App Hosting (`apphosting.yaml`)

## Setup

### 1. Install dependencies

```bash
npm install
```

### 2. Firebase project (not included in this repo)

Create your own project in the [Firebase Console](https://console.firebase.google.com/) with Authentication (Email/Password), Realtime Database and Firestore enabled, then:

- Update the project id in `.firebaserc`.
- Generate a service account key (Project Settings → Service Accounts) for the server-side credentials below.
- Deploy the Firestore rules in `firestore.rules` (see `DEPLOY-FIRESTORE-RULES.md`).

### 3. Environment variables

Create a `.env.local` file in the project root:

```
# Firebase (client)
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_DATABASE_URL=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=

# Firebase Admin (server)
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=

# WhatsApp Cloud API
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_BUSINESS_ACCOUNT_ID=
WHATSAPP_WEBHOOK_VERIFY_TOKEN=
```

### 4. Run

```bash
npm run dev
```

The app runs at http://localhost:9002.

### 5. Flash the ESP32

Open `Weightwise_arduino/Weightwise_arduino.ino`, fill in your own Wi-Fi and Firebase details, and upload it to the board. It accepts `weight,level` lines over serial and writes them to Firebase.

## Documentation

| File | Covers |
|------|--------|
| `WHATSAPP-SETUP-GUIDE.md` | Full WhatsApp Cloud API setup (Meta app, token, templates, webhook) |
| `WHATSAPP-INTEGRATION-SUMMARY.md` | Technical overview of the messaging system |
| `QUICK-REFERENCE.md` | API and development quick reference |
| `FIRESTORE-MIGRATION-GUIDE.md`, `docs/database-migration-plan.md` | Moving data from Realtime Database to Firestore |
| `SEARCH-FEATURE.md` | Bin search feature |
| `esp32-firebase-example.md` | ESP32 → Firebase example |

## Notes

- WhatsApp alerts use pre-approved message templates named `level_alert` and `weight_alert`; create these in Meta Business Manager first.
- Default thresholds are 90% fill level and 35 kg, with a 1-hour alert interval; all are editable from the admin settings.
- The app falls back to generated demo data when no live sensor data is available.

## Security

- `.env*` files are git-ignored — never commit credentials.
- Login currently derives a Firebase account from the PIN using a shared secret defined in `src/context/auth-context.tsx` and `src/app/api/admin/users/route.ts`. Change this before deploying anywhere real.
- `Weightwise_arduino/Weightwise_arduino.ino` contains hard-coded Wi-Fi and Firebase values from development. Replace them with your own and keep real ones out of version control.
- If you're forking or reusing this project, make sure all Firebase and WhatsApp credentials are your own, kept out of version control, and rotated if ever exposed.
