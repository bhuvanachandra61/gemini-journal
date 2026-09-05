# Deploy Guide — Personal Gemini Journal

Run these commands from **PowerShell** in the project directory. Replace `PROJECT_ID` with your actual GCP project ID once, at the top.

## 0. Set your project ID (run once at the top of every new terminal)

```powershell
$env:PROJECT_ID = "project-99c8f420-c814-4530-a25"   # <-- put your GCP project ID here
$env:REGION = "asia-south1"                           # Mumbai — closest region
gcloud config set project $env:PROJECT_ID
```

## 1. Enable the required APIs (one-time, ~2 min)

```powershell
gcloud services enable `
  run.googleapis.com `
  firestore.googleapis.com `
  secretmanager.googleapis.com `
  cloudbuild.googleapis.com `
  artifactregistry.googleapis.com `
  iam.googleapis.com `
  iamcredentials.googleapis.com `
  identitytoolkit.googleapis.com
```

## 2. Create the Firestore database (one-time)

```powershell
gcloud firestore databases create --location=$env:REGION
```

If it says already exists, you're good.

## 3. Deploy the Firestore security rules

Install Firebase CLI first if you don't have it:
```powershell
npm install -g firebase-tools
firebase login
```

Then from this directory:
```powershell
firebase deploy --only firestore:rules --project $env:PROJECT_ID
```

If you don't have Node.js and don't want to install it, paste the contents of `firestore.rules` directly in the Firebase Console: https://console.firebase.google.com → your project → Firestore Database → Rules → Publish.

## 4. Store the Gemini API key in Secret Manager

Get your API key from https://aistudio.google.com/app/apikey (make sure to select your GCP project when creating it).

```powershell
$KEY = Read-Host -AsSecureString "Paste your Gemini API key"
$PLAIN = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto(
  [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($KEY))

gcloud secrets create gemini-api-key --replication-policy="automatic"
echo -n $PLAIN | gcloud secrets versions add gemini-api-key --data-file=-
```

If the secret already exists, skip the `create` line and just run the `versions add`.

## 5. Grant Cloud Run's service account access to the secret

```powershell
$PROJECT_NUMBER = gcloud projects describe $env:PROJECT_ID --format="value(projectNumber)"
$SA = "$PROJECT_NUMBER-compute@developer.gserviceaccount.com"

gcloud secrets add-iam-policy-binding gemini-api-key `
  --member="serviceAccount:$SA" `
  --role="roles/secretmanager.secretAccessor"

# Also allow it to write to Firestore (usually already granted, but explicit is safer):
gcloud projects add-iam-policy-binding $env:PROJECT_ID `
  --member="serviceAccount:$SA" `
  --role="roles/datastore.user"
```

## 6. Get Firebase web config

Go to Firebase Console → your project → ⚙ Project Settings → General → "Your apps" → Web app (`</>`). If you haven't registered a web app yet, click "Add app → Web" first.

You'll see something like:
```
apiKey: "AIzaSy..."
authDomain: "your-project.firebaseapp.com"
projectId: "your-project-id"
```

Copy these three values for the next step.

## 7. Deploy to Cloud Run

```powershell
$FIREBASE_API_KEY = "AIzaSy...paste-from-firebase-console..."
$FIREBASE_AUTH_DOMAIN = "$env:PROJECT_ID.firebaseapp.com"

gcloud run deploy gemini-journal `
  --source . `
  --region $env:REGION `
  --allow-unauthenticated `
  --labels dev-tutorial=cloud-run-ai-challenge `
  --set-env-vars "GOOGLE_CLOUD_PROJECT=$env:PROJECT_ID,FIREBASE_API_KEY=$FIREBASE_API_KEY,FIREBASE_AUTH_DOMAIN=$FIREBASE_AUTH_DOMAIN,FIREBASE_PROJECT_ID=$env:PROJECT_ID" `
  --set-secrets "GEMINI_API_KEY=gemini-api-key:latest" `
  --memory 512Mi `
  --cpu 1
```

First deploy takes ~5-8 min (builds container from source).

At the end you'll get a URL like:
`https://gemini-journal-XXXXXXXX-el.a.run.app`

## 8. Add the Cloud Run URL to Firebase Auth authorized domains

Firebase Console → Authentication → Settings → Authorized domains → Add domain → paste the Cloud Run host (e.g. `gemini-journal-xxx-el.a.run.app`, no `https://`, no trailing slash).

## 9. Test

Open the URL, click "Sign in with Google", chat with the journal, then click "📊 Weekly summary".

## 10. Verify the required service label

```powershell
gcloud run services describe gemini-journal --region $env:REGION --format="value(metadata.labels)"
```

You should see `dev-tutorial=cloud-run-ai-challenge` in the output. This label is a hard submission requirement.

## Troubleshooting

- **"Permission denied" accessing Secret Manager** → re-run step 5.
- **"Firebase not configured" in the browser** → env vars didn't reach the container; check step 7's `--set-env-vars` string.
- **Sign-in popup blocked / "auth/unauthorized-domain"** → run step 8.
- **Firestore permission denied on write** → run the second command in step 5 (datastore.user role).
- **Build fails** → check `gcloud builds log <BUILD_ID>` — usually a `requirements.txt` version mismatch.
