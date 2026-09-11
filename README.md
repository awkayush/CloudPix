# CloudPix — Day 1

FastAPI core: JWT auth + upload endpoint + job status tracking, images stored in MinIO.

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env   # edit SECRET_KEY at minimum

docker-compose up -d   # starts MinIO (and Redis, for Day 2)

uvicorn app.main:app --reload
```

API docs: http://localhost:8000/docs
MinIO console: http://localhost:9001 (minioadmin / minioadmin)

## Endpoints (Day 1)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/auth/register` | — | `{email, password}` → creates user |
| POST | `/auth/login` | — | form data `username`, `password` → JWT |
| POST | `/upload` | Bearer token | multipart file upload → creates Job, stores original in MinIO |
| GET | `/jobs/{id}` | Bearer token | poll job status |
| GET | `/jobs` | Bearer token | list jobs, optional `?status=` filter |

## Test flow

```bash
# register
curl -X POST localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"you@test.com","password":"yourpassword"}'

# login
curl -X POST localhost:8000/auth/login \
  -d "username=you@test.com&password=yourpassword"
# copy the access_token from the response

# upload (replace TOKEN)
curl -X POST localhost:8000/upload \
  -H "Authorization: Bearer TOKEN" \
  -F "file=@/path/to/image.jpg"

# check job status (replace TOKEN and JOB_ID from the upload response)
curl localhost:8000/jobs/JOB_ID -H "Authorization: Bearer TOKEN"
```

A fresh upload will sit at `status: queued`, `progress: 25` — there's no worker yet.
That's Day 2.

## Verified locally (no MinIO needed)
- App imports cleanly, all routes register
- Register → login → JWT → protected `/jobs` route: full round trip passes
- Unauthenticated request to `/jobs` correctly returns 401

## Known dependency gotcha
`passlib[bcrypt]` pulls a bcrypt version that breaks passlib's internal version
probe (`ValueError: password cannot be longer than 72 bytes...` on the *first*
hash call, unrelated to actual password length). Fixed by pinning
`bcrypt==4.0.1` explicitly — already reflected in `requirements.txt`.

## Day 2 preview
Replace the "job just sits at QUEUED" comment in `app/routers/upload.py` with
an actual Celery task enqueue, and add a worker that:
1. Downloads the original from MinIO
2. Runs Pillow processing (resize/compress/thumbnail)
3. Uploads the result to the processed bucket
4. Updates the Job row's `status`, `progress`, `processed_key`, `processed_size`
