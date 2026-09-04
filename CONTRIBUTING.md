# Contributing to Storelytics

This is a **class/portfolio** CV demo, not a store product.

## Prerequisites
- Python 3, Redis, two cameras (or webcams)
- Node + Expo for the dashboard
- Your own Firebase project (credentials were revoked; use `*.example.*` templates)

## Run
```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install opencv-python face-recognition deepface redis firebase-admin numpy
cp storeDB.example.json storeDB.json   # fill in, never commit
redis-server
python camera.py
```

```bash
cd frontend
npm install
cp firebaseConfig.example.js firebaseConfig.js
npx expo start
```

Do not deploy without signage, retention, opt-out, and a serious ethics review. DeepFace labels are biased.
