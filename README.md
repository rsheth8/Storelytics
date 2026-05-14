# Storelytics

**Real-time, anonymized in-store analytics powered by computer vision.**

Storelytics turns a single camera at a storefront into a continuous stream of
business intelligence: how many people walked in, how long they stayed, and
inferred demographic + emotional breakdowns of the foot traffic — all visualized
in a React Native mobile dashboard for the owner.

> Built as an applied senior-year project. The pipeline ties together OpenCV,
> a face-recognition encoder for re-identification, DeepFace for demographic
> + emotion analysis, Redis for low-latency session state, and Firebase for
> persistent customer records.

---

## What it does

1. **A camera** at the entrance/exit captures a live video feed.
2. **`face_recognition` (dlib)** detects faces and produces a 128-D encoding
   per face on every relevant frame.
3. **Re-identification** — encodings are compared against an in-memory list
   (mirrored to Redis with a distributed lock so a second worker thread on
   the back camera can read consistent state) to decide if this is a new
   visitor or someone leaving.
4. On entry, a customer record is opened in Firebase Realtime Database with
   `time_in`. On exit, the matching record is closed with `time_out`.
5. **DeepFace** analyzes the exit frame for inferred `race`, `gender`, and
   `emotion`; these labels are attached to the finalized record in
   Firestore.
6. The **React Native app** (Expo) subscribes to Firestore via `onSnapshot`
   and renders live charts: gender / race / emotion breakdowns plus dwell
   time across selectable windows (day / week / month).

```
       Camera (entrance)             Camera (exit)
            │                              │
            ▼                              ▼
  ┌──────────────────────────────────────────────┐
  │  camera.py   (OpenCV + face_recognition)     │
  │                                              │
  │  • detect faces                              │
  │  • compute 128-D encoding                    │
  │  • compare to Redis-backed session set       │
  │      ─ new face   → enter_user()             │
  │      ─ match      → leave_user()             │
  │                       └─► DeepFace.analyze   │
  │                              (race/gender/   │
  │                               emotion)       │
  └──────────────────────────────────────────────┘
                  │                  │
        Redis (encodings)     Firebase
        + lock keys           (Realtime DB + Firestore)
                                     │
                                     ▼
                          React Native dashboard
                          (Expo + react-native-paper)
```

---

## Repo layout

```
Storelytics/
├── backend/
│   ├── camera.py            # Two-camera capture loop, threading, Redis-backed re-ID
│   ├── faceData.py          # Encoding extraction + persistence helpers
│   ├── Person.py            # In-memory person record (key + encoding)
│   ├── DbPerson.py          # Typed record persisted to Firebase
│   ├── db.py                # Firebase Realtime DB writes (time_in / time_out)
│   ├── firestore.py         # Firestore admin client (final customer docs)
│   ├── deepface.ipynb       # Demo notebook for DeepFace.analyze() on stills
│   ├── data/                # Sample input frames (not committed in full)
│   └── storeDB.example.json # Firebase service-account template — fill yours in
└── frontend/
    ├── App.js               # Expo entry: react-navigation stack + theme provider
    ├── realData.js          # Firestore subscription, push into shared state
    ├── firebaseConfig.example.js  # Web Firebase config template
    └── src/
        ├── screens/         # Start, Login, Register, Reset, Dashboard, Home, MoreData, DropDown
        ├── components/      # Card, Button, Background, Logo, TextInput, …
        ├── graphs/          # raceGraph, genderGraph, emotionGraph, graph
        ├── core/            # theme, typography
        ├── helpers/         # validators, formatters
        └── misc/
```

---

## Screenshots

<p align="center">
  <img src="Screen%20Shot%202023-04-19%20at%2011.24.42%20AM.png" width="200" />
  <img src="Screen%20Shot%202023-04-19%20at%2011.24.55%20AM.png" width="200" />
  <img src="Screen%20Shot%202023-04-19%20at%2011.25.06%20AM.png" width="200" />
  <img src="Screen%20Shot%202023-04-19%20at%2011.25.27%20AM.png" width="200" />
  <img src="Screen%20Shot%202023-04-19%20at%2011.25.51%20AM.png" width="200" />
</p>

---

## Setup

> **You will need a Firebase project of your own.** The credentials previously
> committed to this repo have been removed — see *Credentials* below.

### 1. Firebase project

Create a project at <https://console.firebase.google.com> and enable:
- **Authentication** (Email/Password for the Login/Register screens)
- **Realtime Database** (for live session records)
- **Firestore** (for finalized customer records)
- Generate a **service account key** (Project Settings → Service Accounts).

### 2. Backend

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install opencv-python face-recognition deepface redis firebase-admin numpy linq

# Drop your downloaded service account JSON in as backend/storeDB.json
# (the .example.json shows the expected shape — fill yours in)
cp storeDB.example.json storeDB.json
# …then paste in real values.

# Start Redis (or run `brew services start redis`)
redis-server

python camera.py
```

### 3. Frontend (Expo)

```bash
cd frontend
npm install

# Add Firebase web config
cp firebaseConfig.example.js firebaseConfig.js
# …fill in your values from Firebase Console → Project Settings → SDK setup

# Open in Expo Go (iOS / Android) or web
npx expo start
```

---

## Credentials

This repo's commit history was rewritten on **2026-05-13** to remove a
Firebase service-account private key that had been committed by accident.
The key was revoked from Google Cloud at the same time. The current state of
the repo contains only placeholder `*.example.*` files; copy them, fill in
your own values, and rely on `.gitignore` to keep them out of source control:

```
backend/storeDB.json         # ← real key, never commit
frontend/firebaseConfig.js   # ← real web config, never commit
```

If you fork this repo and use it with a real Firebase project, generate
**your own** service-account key.

---

## Stack

| Layer | Tech |
|---|---|
| Detection / encoding | OpenCV, `face_recognition` (dlib) |
| Demographic + affect | DeepFace (`race`, `gender`, `emotion`) |
| Session state / re-ID | Redis (with `redis.lock` for two-camera coherence) |
| Persistence | Firebase Realtime Database + Firestore |
| Mobile UI | React Native, Expo SDK 46, react-navigation, react-native-paper |
| Fonts / theming | `@expo-google-fonts/quicksand`, custom theme module |

---

## Privacy & ethics

The system stores **encodings, not images** — 128-dimensional floats from
`face_recognition` are not invertible into recognizable faces with any
practical reliability. Demographic and emotion labels come from DeepFace,
which has well-documented biases on race and affect classification; treat the
breakdowns as directional traffic insights, not as identity claims. For
production deployments, you would also want:

- Visible signage at the entrance (in many jurisdictions this is legally
  required for face-based analytics)
- A TTL on encodings in Redis (delete after a session closes) and on labels
  in Firestore (aggregate beyond a short retention window)
- Opt-out flow for individual customers

These are intentionally out of scope for a proof-of-concept project but are
the right next steps before any real deployment.
