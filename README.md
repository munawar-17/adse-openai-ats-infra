# Tkxel ATS

Tkxel ATS is a self-hosted recruitment automation system that replaces external ATS dependencies with a native PostgreSQL application, FastAPI backend, React frontend, and OpenAI-powered agent runtime.

This project intentionally uses:

- FastAPI OpenAPI docs at `/docs` and `/openapi.json`
- OpenAI Responses API for the AI agent layer
- PostgreSQL 15+ with pgvector as the ATS system of record
- Celery + Redis for async agent and analytics tasks
- React 18 + TypeScript for the recruiter, hiring manager, HOD, and admin UI

## Implemented Scope

The current app implements the native Tkxel ATS workflow from the widget prompts:

- Jobs: create talent requests, generate OpenAI-backed JDs, approve JDs, and publish approved roles to the local careers feed or configured job-board adapters.
- Pipeline Kanban: dynamic role selector, structured filters, SLA chips, stage movement, assessment invites, and grouped Kanban API output.
- Candidate Profile: candidate summary, fit breakdown, stage history, agent outputs, screening, phone screen, assessment, interview, offer, tags, and audit history.
- Discovery: semantic plus structured search, LinkedIn headhunt launch, reviewed-lead import, job/source/score filters, profile drill-in, and add-to-pipeline action.
- Talent Pool: reusable candidate pool with tags and ReMatch-style shortlist cards.
- Analytics: dynamic funnel, source conversion, time-to-hire, bottlenecks, recruiter workload, headhunting dependency, and JD quality trend.
- Admin/Compliance: users, audit logs, agent config, integration readiness, published postings, and assessment invite status.

The agent roster is configured for `IntakeAgent`, `JDAgent`, `PostingAgent`, `DiscoveryAgent`, `DuplicateAgent`, `ScreeningAgent`, `PhoneScreenAgent`, `AssessmentAgent`, `SchedulingAgent`, `FeedbackAgent`, `ReMatchAgent`, `CommsAgent`, `AnalyticsAgent`, and `ComplianceAgent`.

## Project Layout

```text
adse-openai-ats/
  backend/
    app/
      agents/        OpenAI runner, prompts, orchestrator
      api/           FastAPI routers
      db/            SQLAlchemy session/base
      models/        Core SQLAlchemy models
      schemas/       Pydantic contracts
      services/      Embedding and ranking services
      tasks/         Celery app and async task stubs
    schema.sql       PostgreSQL + pgvector schema
  frontend/
    src/
      api/           React Query API client
      components/    Shell, metrics, fit score UI
      pages/         Dashboard, jobs, pipeline, discovery, talent pool, profile, analytics, admin
  infra/
    docker-compose.yml
```

## Run Everything Locally

Use these commands from a fresh terminal session. Keep the backend and frontend commands running in separate terminals.

### 1. Open the Project

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats
```

### 2. Start PostgreSQL and Redis

If Docker is installed:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/infra
docker compose up -d
```

If Docker is not installed, start PostgreSQL locally with any PostgreSQL tool you already use, then make sure this database URL works:

```text
postgresql://adse:adse@localhost:5432/adse
```

The database must support these extensions:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;
```

For a GUI database client, use a PostgreSQL client such as DataGrip, DBeaver, pgAdmin, TablePlus, or Postico. MySQL Workbench is for MySQL and is not the right viewer for this PostgreSQL database.

### 3. Install Backend Dependencies

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
cp -n .env.example .env
```

Edit `backend/.env` and set at least:

```env
DATABASE_URL=postgresql+asyncpg://adse:adse@localhost:5432/adse
OPENAI_API_KEY=your-openai-api-key
CODEX_MODEL=gpt-4o
CODEX_MODEL_COMPLEX=o3
EMBEDDING_MODEL=text-embedding-3-large
EMBEDDING_DIMENSIONS=1536
JWT_SECRET_KEY=change-this-to-a-long-random-secret
AUTH_DEV_AUTO_VERIFY_EMAIL=false
```

`EMBEDDING_DIMENSIONS=1536` keeps `text-embedding-3-large` compatible with the current PostgreSQL `vector(1536)` columns. If you change the database vectors to 3072 dimensions, update this value and regenerate stored embeddings.

To configure the OpenAI key without putting it in shell history:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
chmod +x scripts/configure_openai_key.sh
./scripts/configure_openai_key.sh
```

### 4. Apply Database Migrations

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
alembic upgrade head
```

Check the database tables:

```bash
psql postgresql://adse:adse@localhost:5432/adse -c "\dt"
psql postgresql://adse:adse@localhost:5432/adse -c "\d users"
```

### 5. Configure and Test Gmail SMTP

For Gmail or Google Workspace SMTP, create a Google App Password first. Use the app password, not your normal Gmail password.

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
chmod +x scripts/configure_gmail_smtp.sh
./scripts/configure_gmail_smtp.sh
python scripts/test_smtp.py --to your-email@tkxel.io
```

Expected success:

```text
SMTP test email sent to your-email@tkxel.io.
```

If SMTP works in the test but registration still says `smtp_delivery_failed`, restart the backend so it reloads `.env`.

### 6. Configure Interview Calendar

Interview scheduling creates a real Google Calendar event with a Google Meet conference link. Store one of these in `GOOGLE_CALENDAR_CREDENTIALS`:

- a service account JSON string
- a base64-encoded service account JSON string
- a path to a local service account JSON file
- an authorized user JSON that contains a refresh token

Example local configuration:

```env
GOOGLE_CALENDAR_CREDENTIALS=/absolute/path/to/google-calendar-credentials.json
GOOGLE_CALENDAR_ID=primary
GOOGLE_CALENDAR_TIMEZONE=Asia/Karachi
GOOGLE_CALENDAR_IMPERSONATED_USER=
```

For a Google Workspace service account, enable domain-wide delegation and set `GOOGLE_CALENDAR_IMPERSONATED_USER` to the recruiter mailbox that owns the calendar. After changing these values, restart the backend. Candidate profile pages can then schedule interviews from the Interview Calendar panel; the app stores the event id, Meet link, attendee emails, and email delivery status.

If you do not have Google Workspace admin access, use a Google OAuth Desktop client instead. In Google Cloud, enable the Calendar API, create an OAuth client with application type `Desktop app`, download the JSON file, then run:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
python scripts/configure_google_oauth_calendar.py --client-secrets /absolute/path/to/oauth-desktop-client.json
python scripts/test_calendar.py --to your-email@tkxel.io
```

If you do have Workspace admin access and want service-account delegation, configure and test it with:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
chmod +x scripts/configure_google_calendar.sh
./scripts/configure_google_calendar.sh
python scripts/test_calendar.py --to your-email@tkxel.io
```

### 7. Configure LinkedIn

LinkedIn credentials stay in `backend/.env`; never put the client secret in the frontend. The current integration is a compliant recruiter workflow: Discovery builds a LinkedIn people-search URL from your keywords and selected role, recruiters review profiles on LinkedIn, then import selected leads into the ATS and optionally into a job pipeline.

```env
LINKEDIN_CLIENT_ID=your-linkedin-client-id
LINKEDIN_CLIENT_SECRET=your-linkedin-client-secret
LINKEDIN_COMPANY_PAGE_URL=https://www.linkedin.com/company/your-company/
LINKEDIN_POSTER_EMAIL=recruiting@your-company.com
```

After editing LinkedIn values, restart the backend. Open Discovery, use the LinkedIn headhunt form, click `Open LinkedIn`, then import reviewed leads with name, email, LinkedIn URL, skills, and notes.

### 8. Run the Backend API

### 8. Configure Phone Screen Modes

The ATS supports two phone screen paths:

- Recruiter call: free local workflow. The recruiter calls the candidate, follows the generated script, enters notes, then submits a human decision.
- Bot call: development mock by default, or real outbound phone calls when Vapi is configured.

Local/dev defaults:

```env
VOICE_PROVIDER=mock
VOICE_AGENT_PROVIDER=mock
PHONE_SCREEN_REQUIRE_CONSENT=true
PHONE_SCREEN_MAX_DURATION_MINUTES=12
PHONE_SCREEN_MIN_SCORE_TO_PASS=70
```

For real outbound calls, expose the backend with a public HTTPS URL and set:

```env
VOICE_PROVIDER=vapi
VOICE_AGENT_PROVIDER=vapi
VAPI_API_KEY=your-vapi-api-key
VAPI_PHONE_NUMBER_ID=your-vapi-phone-number-id
VAPI_WEBHOOK_BASE_URL=https://your-public-backend-url
VAPI_WEBHOOK_BEARER_TOKEN=your-shared-webhook-token
VAPI_SERVER_CREDENTIAL_ID=your-vapi-server-credential-id
VAPI_API_BASE_URL=https://api.vapi.ai
VAPI_MODEL_PROVIDER=anthropic
VAPI_MODEL=claude-haiku-4-5-20251001
VAPI_MODEL_MAX_TOKENS=250
VAPI_VOICE_PROVIDER=11labs
VAPI_VOICE_ID=dN8hviqdNrAsEcL57yFj
VAPI_VOICE_MODEL=eleven_turbo_v2_5
VAPI_TRANSCRIBER_PROVIDER=deepgram
VAPI_TRANSCRIBER_MODEL=flux-general-en
VAPI_TRANSCRIBER_LANGUAGE=en
VAPI_START_WAIT_SECONDS=0.4
VAPI_SMART_ENDPOINTING_PROVIDER=livekit
VAPI_SMART_ENDPOINTING_WAIT_FUNCTION=2000 / (1 + exp(-10 * (x - 0.5)))
VAPI_STOP_NUM_WORDS=0
VAPI_STOP_VOICE_SECONDS=0.2
VAPI_STOP_BACKOFF_SECONDS=1
```

Create the Vapi server credential in the Vapi dashboard and point it at the same bearer token you set in `VAPI_WEBHOOK_BEARER_TOKEN`. ATS creates a Vapi assistant per phone-screen session using the model, voice, transcriber, and speaking-plan settings above, places the outbound call through `VAPI_PHONE_NUMBER_ID`, and receives events at `/api/v1/voice/vapi/phone-screen/{session_id}/webhook`.

Assessment invites are gated behind a passed phone screen. Either recruiter-call or bot-call mode can satisfy that gate, but a recruiter must still click `Pass` before CodeForge assessment links can be sent.

### 9. Run the Backend API

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

Backend URLs:

- API root: `http://127.0.0.1:8000`
- OpenAPI docs: `http://127.0.0.1:8000/docs`
- OpenAPI JSON: `http://127.0.0.1:8000/openapi.json`
- Health: `http://127.0.0.1:8000/api/v1/health`

Health check:

```bash
curl http://127.0.0.1:8000/api/v1/health
```

### 10. Run the Frontend App

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/frontend
npm install
cp -n .env.example .env
npm run dev
```

Frontend URL:

```text
http://127.0.0.1:5173
```

Production build check:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/frontend
npm run build
```

### 11. Optional: Run the Celery Worker

Run this only if you are using background task queues. Redis must be running first.

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
celery -A app.tasks.celery_app.celery_app worker --loglevel=info -Q agents
```

### 12. Stop Local Servers

```bash
kill $(lsof -ti tcp:8000) 2>/dev/null || true
kill $(lsof -ti tcp:5173) 2>/dev/null || true
```

If you started Docker infrastructure:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/infra
docker compose down
```

## Authentication and Email Verification

Authentication is self-service and database-backed. Create an account in the UI, receive a six-digit verification code through SMTP, enter the code on the Verify Email screen, then sign in.

Verification codes are stored as hashes, expire after 10 minutes by default, and are limited by attempts and resend cooldowns. Set `SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD`, and `SMTP_FROM_EMAIL` in `backend/.env` before using email verification.

Example SMTP configuration for a TLS SMTP provider:

```env
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USE_TLS=true
SMTP_TIMEOUT_SECONDS=15
SMTP_USER=recruiting@tkxel.io
SMTP_PASSWORD=your-smtp-password-or-app-password
SMTP_FROM_EMAIL=recruiting@tkxel.io
AUTH_DEV_AUTO_VERIFY_EMAIL=false
AUTH_VERIFICATION_CODE_MINUTES=10
AUTH_VERIFICATION_MAX_ATTEMPTS=5
AUTH_VERIFICATION_RESEND_COOLDOWN_SECONDS=60
AUTH_VERIFICATION_MAX_RESENDS_PER_WINDOW=5
```

After changing SMTP values, restart the backend. New account registrations will email a six-digit verification code.

Interview calendar delivery uses both Google Calendar and SMTP: Google Calendar creates the Meet event and sends calendar invitations to attendees, while SMTP sends the candidate-facing interview email. If SMTP is not configured, the Meet event is still created and the profile records the email delivery status.

Admin integration status is visible inside the Admin screen. LinkedIn, CodeForge Assessment, email, and interview calendar flows report their current readiness. CodeForge is native to this project and does not need third-party assessment credentials.

Current local workflows:

- Jobs: create a role, generate a JD, approve the JD, then publish it to the local career page or a configured LinkedIn adapter.
- Discovery: search native candidates, launch LinkedIn headhunt searches from keywords, and import reviewed LinkedIn leads into the ATS.
- Candidates: create candidate records through the API, career page, or LinkedIn import, then search/filter them in Discovery.
- Pipeline: move candidates through gated stages, run recruiter or bot phone screens, send assessment invites after phone-screen pass, and open full candidate profiles.
- Candidate Profile: review phone screen scripts, notes, transcripts, scores, pass/hold/reject decisions, schedule Google Meet interviews, and send candidate interview invites from the Interview Calendar panel.
- Talent Pool: review reusable candidates and ReMatch tags for future roles.
- Admin: review users, agent config, integration readiness, published postings, assessment invites, and audit logs.

## Useful API Checks

Use these after signing in from the UI or when testing with a bearer token:

```bash
curl http://127.0.0.1:8000/api/v1/health
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/v1/jobs
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/v1/analytics/dashboard
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/v1/talent-pool
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/v1/admin/overview
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/v1/pipeline/{job_id}/kanban
```

Quality checks before handing off changes:

```bash
cd /Users/tk-lpt-1276/Documents/adse-openai-ats/backend
source .venv/bin/activate
ruff check app alembic/versions scripts
python -m compileall -q app alembic/versions scripts

cd /Users/tk-lpt-1276/Documents/adse-openai-ats/frontend
npm run build
```

## Agent Runtime

All agents return this contract:

```json
{
  "agent": "ScreeningAgent",
  "action": "screening_processed",
  "result": {},
  "confidence": 0.72,
  "human_review_required": false,
  "review_reason": null,
  "db_writes": [
    { "table": "screening_results", "operation": "insert", "payload": {} }
  ],
  "next_agent": "PhoneScreenAgent",
  "next_action": "Route to PhoneScreenAgent."
}
```

Manual test:

```bash
curl -X POST http://127.0.0.1:8000/api/v1/agents/run \
  -H "Content-Type: application/json" \
  -d '{
    "event_name": "application.received",
    "payload": {
      "full_name": "Candidate Name",
      "email": "candidate@example.com",
      "resume_text": "Python FastAPI PostgreSQL backend engineer"
    }
  }'
```
