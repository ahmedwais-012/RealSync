# RealSync Full Code Review — 13 April 2026

> Comprehensive review of all 3 services + live health check + transcription flow trace
> Reviewed by: Claude (5 parallel agents)
> Scope: Frontend (30 files), Backend (25+ files), AI Service (20+ files)

---

## Executive Summary

| Service | CRITICAL | HIGH | MEDIUM | LOW | Total |
|---------|----------|------|--------|-----|-------|
| Frontend (Front-End-v2) | 3 | 5 | 10 | 7 | 25 |
| Backend (realsync-backend) | 2 | 5 | 8 | 5 | 20 |
| AI Service (RealSync-AI-Prototype) | 1 | 5 | 9 | 8 | 23 |
| **Total** | **6** | **15** | **27** | **20** | **68** |

### Live Services Status

| Service | Status | Details |
|---------|--------|---------|
| Frontend (real-sync.app) | **UP** | 200 OK on `/` and `/app` |
| Backend (api.real-sync.app) | **UP** | PM2 online, 13h uptime, 120MB RAM |
| AI Service (RunPod) | **UP** | Running since Apr 10 (111h+), GPU 0% idle |
| Oracle VPS | **HEALTHY** | 8% disk, 4% RAM, load 0.00 |

### Transcription to PDF Verdict

**TRANSCRIPTION DOES NOT APPEAR IN PDFs.** The pipeline has a gap:
- Recall.ai to backend to Supabase: **works**
- API endpoint `/api/sessions/:id/transcript`: **works**
- Frontend fetches transcript for PDF: **MISSING** — `Reports.tsx` never calls the transcript API
- PDF generator has `TranscriptPage` component: **ready but never receives data**

---

## CRITICAL Issues (Fix Before Demo)

### Frontend

**F-1. No validation of WebSocket messages — type confusion/injection risk**
- Files: `WebSocketContext.tsx:115`, `NotificationContext.tsx:206`, `Dashboard.tsx:206`
- All WS messages are blindly cast via `as` assertions. A compromised WS server could inject arbitrary data rendered in UI and OS notifications.
- Fix: Add Zod schema or type guard validation for every inbound WS message type.

**F-2. OAuth redirect URL mismatch — `/app` vs `/app/`**
- Files: `Login.tsx:151,161`, `SignUp.tsx:602,612`
- Hardcoded `${origin}/app` but Vite base is `/app/` (trailing slash). OAuth callback token may be lost during redirect chain.
- Fix: Change to `${origin}/app/` or create a dedicated `/auth/callback` route.

**F-3. WebSocket auth token sent in plaintext message body**
- File: `WebSocketContext.tsx:97`
- JWT sent as JSON message over WS. Falls back to `ws:` (non-TLS) when page is HTTP.
- Fix: Ensure backend rejects non-WSS connections. Consider token in URL param (encrypted over TLS).

### Backend

**B-1. No authentication on Recall.ai WebSocket endpoint (`/ws/recall`)**
- File: `recallWs.js:48-87`
- Token is a random UUID in query param — no HMAC, no expiry, no IP allowlist. Anyone with the token can inject frames/audio/transcripts into a live session.
- Fix: Add HMAC-signed tokens with expiration. Verify Recall.ai IP ranges.

**B-2. Hardcoded secrets in `.env` file on disk**
- File: `realsync-backend/.env:5-7`
- Real `AI_API_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` values present. `.gitignore` covers it, but no pre-commit hook protection.
- Fix: Rotate keys. Add pre-commit hook to block `.env` files.

### AI Service

**A-1. `.env` file not in `.gitignore` — secret leak risk**
- File: `RealSync-AI-Prototype/.env` and `.gitignore`
- `.gitignore` only excludes `data/`. The `.env` with `AI_API_KEY=` is trackable by git.
- Fix: Add `.env` to `.gitignore`. Add standard Python entries (`__pycache__/`, `*.pyc`, etc.)

---

## HIGH Issues (Fix This Week)

### Frontend

**F-4. Password change does not require current password** — `Settings.tsx:542-567`
**F-5. Session delete only removes from UI, not server** — `Sessions.tsx:601-613`
**F-6. Blocked domain check is client-side only** — `Login.tsx:132`, `blockedDomains.ts`
**F-7. No CSRF protection on state-changing API calls** — `api.ts:41-54`
**F-8. handleEndSession clears local state even if API calls fail** — `SessionContext.tsx:173-184`

### Backend

**B-3. No ping/pong keepalive on Recall.ai WebSocket** — `index.js:197-198` (1-line fix: add `setupWsPingPong(wssRecall, "recall")`)
**B-4. Missing env var validation for RECALL_API_KEY and AI_API_KEY at startup** — `index.js:10-14`
**B-5. RecallBotAdapter status polling never cleans up on leave()** — `RecallBotAdapter.js:159-285`
**B-6. Ingest WebSocket lacks byte-level rate limiting on audio_pcm** — `ingest.js:213-222`
**B-7. POST /api/metrics allows sessionId to bypass ownership for null-user sessions** — `sessions.js:241-286`

### AI Service

**A-2. SPRTDetector not thread-safe — _sessions dict mutated from pool workers** — `sprt_detector.py:49-113`
**A-3. Session state dicts grow unboundedly under concurrent load** — `inference.py:105-113`
**A-4. No rate limit on /api/analyze/frame endpoint** — `app.py:213`
**A-5. _no_face_counters eviction deletes by insertion order, not staleness** — `inference.py:233-236`
**A-6. Trust score formula matches tests algebraically but documents wrong formula** — `inference.py:368-369`

---

## MEDIUM Issues (27 total)

### Frontend (10)
- useIsMobile reads window.innerWidth during SSR — `useIsMobile.ts:4`
- Notification fetch race condition with initialFetchDone ref — `NotificationContext.tsx:139-170`
- Avatar upload has no server-side file type validation — `CompleteProfile.tsx:27-47`
- Missing onClose in useEffect dependency array — `MobileSidebar.tsx:25`
- ErrorBoundary missing componentDidCatch — `App.tsx:73-91`
- Duplicated Orb/ScanLine/Logo components across Login/SignUp
- OAuth handlers missing try/catch — `Login.tsx:147-165`
- authFetch forces JSON Content-Type on all bodies — `api.ts:49-51`
- Supabase client silently uses placeholder when env vars missing — `supabaseClient.ts:7-10`
- alert() used for password reset confirmation — `Login.tsx:176`

### Backend (8)
- Transcript not included in report generation (only count) — `persistence.js:252-311`
- Recall.ai captions + Whisper create duplicate transcripts — `audioHandler.js:137-153`
- AlertFusionEngine _consecutiveLow map leaks memory — `alertFusion.js:58`
- broadcastInterval callback does not wrap in try/catch — `index.js:157-168`
- _smoothedVisual/_smoothedAudio maps not cleaned on session GC — `frameHandler.js:16`, `audioHandler.js:15`
- Express error handler missing request path in logs — `data.js:71-103`
- trust proxy hardcoded to 1 — `index.js:61`
- Async transcription request returned 405 on bot disconnect (Recall.ai timing issue)

### AI Service (9)
- Dual session_id validation (UUID in app.py, relaxed regex in inference.py) — `inference.py:221`
- Whisper transcribe_audio does not validate base64 — `whisper_model.py:48-50`
- Thread pool has 2 workers but submits 4 parallel futures — `inference.py:62`
- Frequency analyzer GAN detection has O(n) loop — `frequency_analyzer.py:91-93`
- .env.example identical to .env — should have placeholders
- No-face response returns authenticityScore 1.0 instead of null — `inference.py:426-429`
- Ensemble adaptive weights sum to 1.05 before normalization — `inference.py:296-303`
- SPRT scores list grows unboundedly per session — `sprt_detector.py:59`
- Training script references undefined DATASET_ID — `finetune_text_classifier.py:349`

---

## LOW Issues (20 total)

### Frontend (7)
- Unused jspdf and jspdf-autotable in package.json
- Tailwind CSS installed but never used
- LABEL_STYLE uses React.CSSProperties without importing React — `tokens.ts:48`
- CommandPalette search input does not filter — `CommandPalette.tsx:78-85`
- Multiple icon buttons missing aria-label
- ScanLine never updates height on resize — `Login.tsx:64-86`
- fetchReports empty dependency array is fragile — `Reports.tsx:514`

### Backend (5)
- Puppeteer still in production dependencies (200MB+) — `package.json:25`
- CORS allows no-origin requests — `index.js:73-76`
- POST /api/metrics bypasses rehydration — `sessions.js:264`
- generateAvatarVideo.js uses --no-sandbox — line 52
- Unnecessary defensive guard on _behavioralCooldowns — `fraudDetector.js:327`

### AI Service (8)
- SRC_DIR computed twice — `config.py:11`, `inference.py:28`
- Emotion model loads TTA on every frame — `emotion_model.py:164-167`
- CORS wildcard risk if env var empty in production — `app.py:130-134`
- Benchmark script has hardcoded URL — `scripts/benchmark.py:18-19`
- Missing Pillow in requirements.txt
- Missing httpx in requirements.txt
- Missing pytest-anyio in dev requirements
- .gitignore missing standard Python entries

---

## Transcription to PDF Flow (Detailed)

```
Recall.ai Bot (transcript.data WebSocket message)
    |
RecallBotAdapter._handleTranscript() [RecallBotAdapter.js:412]
    | emits caption message
ingest.js: processIngestMessage() [line 31]
    | type="caption"
transcriptHandler.js: handleTranscript() [line 17]
    |-- persistence.insertTranscriptLine() --> Supabase  [WORKS]
    |-- broadcastToSession({ type: "transcript" }) --> Frontend WS  [WORKS]
    
ALSO: audioHandler.js --> Whisper --> handleTranscript() [DUPLICATE transcripts]

Backend API:
    GET /api/sessions/:id/transcript --> returns lines[]  [WORKS]

Frontend:
    Reports.tsx: fetchReports() --> fetches report + alerts  [WORKS]
    Reports.tsx: fetchReports() --> DOES NOT fetch transcript  [GAP]
    Reports.tsx: downloadPdfV2() --> DOES NOT pass transcript to PDF  [GAP]
    
    generateReportReactPdf.tsx:
        ReportInput.transcript?: ReportTranscriptLine[]  [FIELD EXISTS]
        TranscriptPage()  [READY TO RENDER]
        BUT: never receives data --> page silently omitted  [GAP]
```

### Fix Required (3 changes in Reports.tsx):
1. Add `transcript: ReportTranscriptLine[]` to `ReportData` interface
2. In `fetchReports()`, call `/api/sessions/:id/transcript` and store result
3. In `downloadPdfV2()`, pass `transcript` field to `generateReportReactPdf()`

---

## What is Done Well

### Frontend
- Zero console.log statements — production clean
- Zero unsafe HTML injection — no XSS via innerHTML
- Excellent WebSocket reconnection (exponential backoff, jitter, heartbeat, dead connection detection)
- Proper useEffect cleanup throughout
- Immutable state updates everywhere
- sanitizeCsvCell prevents CSV injection
- Notification cap at 200 prevents memory leaks

### Backend
- Structured JSON logging throughout
- Session garbage collection on 10-minute timer
- Rate limiting on all HTTP + WS endpoints
- Proper Supabase RLS integration
- WebSocket heartbeat for subscribe/ingest connections
- Graceful shutdown handling

### AI Service
- hmac.compare_digest for timing-safe API key comparison
- UUID validation prevents path traversal
- Graceful model degradation (returns "unavailable" if model fails to load)
- Async semaphore prevents frame analysis concurrency issues
- Thread-safe singleton loading with double-checked locking for all models
- Comprehensive test coverage (edge cases, oversized payloads, invalid inputs)
- Session TTL eviction prevents unbounded growth

---

## Recommendations Priority

### Before Innovation Fair Demo (Apr 15)
1. **Know the transcript gap exists** — do not promise transcript in PDF during demo
2. **Verify RunPod is running** before demo starts (it is now, but check morning of)
3. **Test the full flow** — create session, join Zoom, detect deepfake, view report

### After Fair (If Fixing)
1. Fix transcript to PDF pipeline (3 code changes in Reports.tsx)
2. Add WS message validation (frontend)
3. Add ping/pong to Recall.ai WS (1-line backend fix)
4. Add SPRT thread safety (AI service)
5. Clean up memory leaks in alertFusion, frameHandler, audioHandler

---

*Review completed 13-04-2026 at ~06:00 UAE time. 68 issues found across 3 services. 5 parallel agents used.*
