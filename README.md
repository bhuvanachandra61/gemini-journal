# Personal Gemini Journal

My submission for the [Gen AI Academy APAC — Cohort 3](https://hack2skill.com) Ideathon Challenge:
**Build a Secure "Personal Gemini Journal"**.

A private, per-user journaling web app built on Gemini + Firebase + Cloud Run, with production-grade
isolation, secret handling, and defence-in-depth baked in from the first commit.

- 🔴 Live app: https://gemini-journal-92079923394.asia-south1.run.app
- 📂 Code: this repo

## What it does

Sign in with Google → chat with Gemini as a private journaling companion → every conversation is
automatically saved to your own isolated slice of Firestore, given a short auto-generated title, and
becomes part of your reflection history. Once you've been at it a week, one click gives you a
Gemini-generated reflective summary of your themes, mood, and one question to sit with.

## The 3 phases of the challenge

### Phase 1 — Security-first Custom Instructions in Google AI Studio

Before writing a line of code I set up AI Studio with a "constitution" that constrains every
generation: mandatory threat modeling, secure coding standards, multi-tenant isolation as a non-negotiable,
secrets-in-vault (never in code), LLM prompt-injection defence, and an explicit refuse-list for anything
that would skip auth, embed hardcoded credentials, or bypass tenant boundaries.

The full text lives in [PHASE1_AI_STUDIO_INSTRUCTIONS.md](./PHASE1_AI_STUDIO_INSTRUCTIONS.md).
Screenshot of the AI Studio panel is included with the submission.

### Phase 2 — The four required capabilities

| Requirement | How it's met |
|---|---|
| **User authentication (Firebase)** | Google sign-in via Firebase Auth. Every backend call verifies the ID token with `firebase_admin.auth.verify_id_token` and uses the resulting UID — never a client-supplied ID. |
| **Multi-turn Gemini conversations** | `google-generativeai` `start_chat(history=…)`, with a system prompt that mirrors the AI Studio constitution. Full history is replayed on each turn so Gemini has context. |
| **Isolated Firestore storage** | All reads/writes scoped to `users/{uid}/conversations/{id}` and `users/{uid}/summaries/{id}`. Firestore security rules enforce it (`request.auth.uid == userId`); the backend re-checks the stored `uid` on every read as defence-in-depth. See [firestore.rules](./firestore.rules). |
| **Secret management** | The Gemini API key lives in Google Cloud Secret Manager. Cloud Run mounts it into the container as an env var via `--set-secrets`. The compute service account has only `roles/secretmanager.secretAccessor` on that one secret. No keys in the repo, no keys in `.env` in production, no service account JSON. |

### Phase 3 — Original feature enhancement

**Weekly mood & theme summary** (`POST /api/summary/weekly`). One click aggregates the user's last
7 days of entries and asks Gemini to produce a short reflective summary — dominant themes, overall
mood, and one open-ended question to journal on next. The summary itself is persisted (per-user)
so it can be reviewed later without regenerating.

I also added a per-conversation auto-summary so the sidebar shows a human-readable label for each
entry ("Anxiety about a deadline", "Ideas for a hackathon submission") instead of a raw first message —
which matches the challenge line *"conversations automatically summarised and saved"*.

## Architecture

```
Browser ── Firebase Auth (Google sign-in) ── ID token ──► FastAPI on Cloud Run
                                                            │
                                                            ├─► Firestore (rules + backend UID re-check)
                                                            ├─► Secret Manager (Gemini API key)
                                                            └─► Gemini 3.6 Flash (system prompt = security constitution)
```

Everything runs in `asia-south1` (Mumbai) for latency.

## Security model, in short

| Layer | Control |
|---|---|
| Auth | Firebase ID token verified server-side on every call |
| Data isolation | Firestore rules deny cross-user access + backend re-checks `uid` on the doc |
| API keys | Secret Manager only; injected into Cloud Run via `--set-secrets`, never in code or committed env files |
| Model safety | System prompt refuses instruction-override attempts, credential requests, and cross-user references |
| Repo hygiene | `.gitignore` + `.dockerignore` exclude `.env`, `serviceAccountKey.json`, and other sensitive paths |

## Project layout

```
.
├── main.py                              # FastAPI backend
├── static/index.html                    # Firebase Auth + chat UI (single file, no build step)
├── firestore.rules                      # Per-user isolation rules
├── Dockerfile                           # Cloud Run container
├── requirements.txt
├── .env.example                         # Local dev template (no real values)
├── DEPLOY.md                            # Exact gcloud commands used to deploy
├── PHASE1_AI_STUDIO_INSTRUCTIONS.md     # The security directives pasted into AI Studio
└── SOCIAL_POST.md                       # #AccelerateAIwithCloudRun post drafts
```

## Local development

```powershell
py -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
# fill .env with a Gemini key (from aistudio.google.com) and your Firebase web config
gcloud auth application-default login   # for Firestore/Firebase Admin locally
uvicorn main:app --reload --port 8080
```

Open http://localhost:8080.

## Deploy to Cloud Run

Full sequence in [DEPLOY.md](./DEPLOY.md). Short version:

```bash
gcloud run deploy gemini-journal \
  --source . \
  --region asia-south1 \
  --allow-unauthenticated \
  --labels dev-tutorial=cloud-run-ai-challenge \
  --set-env-vars GOOGLE_CLOUD_PROJECT=$PROJECT_ID,FIREBASE_API_KEY=…,FIREBASE_AUTH_DOMAIN=…,FIREBASE_PROJECT_ID=$PROJECT_ID \
  --set-secrets GEMINI_API_KEY=gemini-api-key:latest
```

The `dev-tutorial=cloud-run-ai-challenge` label is the challenge submission requirement.

## What I'd add next

- A soft-delete + export flow for entries (right-to-be-forgotten friendly)
- Rate-limit `/api/chat` per-uid to prevent runaway Gemini spend if a user's account is compromised
- Firestore composite index on `(uid, updated_at DESC)` once entry counts grow past ~1000
- End-to-end test that provisions a scratch Firebase user and verifies cross-user reads are denied
