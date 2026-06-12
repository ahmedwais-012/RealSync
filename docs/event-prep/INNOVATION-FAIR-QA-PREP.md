# RealSync Innovation Fair Q&A Preparation Guide

**Event:** CSIT321 Innovation Fair (Virtual, Remo Platform)
**Date:** 15 April 2026, 11:00 AM -- 2:15 PM UAE
**Format:** 10-min presentation + 5-min Q&A, 5 judges per panel
**Team:** Ahmed Sarhan (Project Leader), Mohammed Atwani (Scribe), Mohamed Ghazi, Yousef Kanjo, Aws Diab
**Supervisor:** Dr. May El Barachi

> Log in by 10:45 AM. One person shares screen. This document prepares the team for likely judge questions.

---

## Table of Contents

1. [Project Overview Questions](#1-project-overview)
2. [Technical Architecture Questions](#2-technical-architecture)
3. [AI & Detection Pipeline Questions](#3-ai--detection-pipeline)
4. [Statistical Methods Questions](#4-statistical-methods-sprt)
5. [Meeting Bot & Capture Questions](#5-meeting-bot--capture-layer)
6. [Frontend & UX Questions](#6-frontend--user-experience)
7. [Security & Privacy Questions](#7-security--privacy)
8. [Testing & Validation Questions](#8-testing--validation)
9. [Deployment & Infrastructure Questions](#9-deployment--infrastructure)
10. [Business & Market Questions](#10-business--market-viability)
11. [Limitations & Honesty Questions](#11-limitations--honesty)
12. [Academic & Research Questions](#12-academic--research-rigor)
13. [Team & Process Questions](#13-team--process)
14. [Curveball / Stress-Test Questions](#14-curveball--stress-test-questions)
15. [Quick Reference: Key Numbers](#15-quick-reference-key-numbers)
16. [Final Report Highlights to Reference](#16-final-report-highlights)

---

## 1. Project Overview

### Q: What is RealSync in one sentence?

**A:** RealSync is a real-time deepfake detection system that joins video meetings as an automated bot, analyzes video, audio, and text using a multi-modal AI ensemble, and displays results on a live security dashboard -- all without participants installing anything.

### Q: What problem does this solve?

**A:** Video call fraud is a real and growing threat. In February 2024, a Hong Kong finance worker transferred $25.6 million after a Zoom call where every participant -- including the apparent CFO -- was a deepfake. Sumsub reported a 40% year-on-year increase in deepfake fraud in 2024, with $2.7 billion in estimated losses in 2023. No existing product detects deepfakes in real time during a live call. Tools like Reality Defender operate post-call. Zoom, Teams, and Meet have no built-in detection. RealSync fills that gap.

### Q: Who is the target user?

**A:** Meeting hosts in corporate, financial, and institutional settings who need to verify that participants are who they claim to be during a live call. The host creates a session, pastes a meeting URL, and the bot handles the rest. No IT integration or endpoint software required.

### Q: What makes this different from existing deepfake detectors?

**A:** Three things:
1. **Real-time** -- processes frames at 85-107 ms, under 200 ms target. Most detectors are forensic (after the fact).
2. **Compression-resilient** -- uses CLIP semantic features that survive H.264 Zoom compression, unlike pixel-level detectors that break on compressed video.
3. **Multi-modal** -- analyzes video (deepfake, emotion, identity), audio (voice authenticity), and text (social engineering detection) simultaneously.

### Q: Can you give us a quick demo walkthrough?

**A:** Yes. The flow is:
1. Sign up at real-sync.app (corporate email required)
2. Create a session -- give it a title, paste the Zoom/Meet/Teams URL
3. Click "Join" -- the Recall.ai bot joins the meeting in 15-30 seconds
4. Open the dashboard -- see live trust score, per-layer confidence bars, deepfake risk, alert feed
5. If someone is using a face swap, the trust score drops, alerts fire in real-time
6. When the meeting ends, click Stop -- download a PDF security report

---

## 2. Technical Architecture

### Q: What is the system architecture?

**A:** Three independent services:

| Service | Tech | Where | Why |
|---------|------|-------|-----|
| **Frontend** | React 18 + TypeScript + Vite + Tailwind + shadcn/ui | Cloudflare Pages (real-sync.app) | Static, zero cost, CDN-distributed |
| **Backend** | Node.js + Express 5 + WebSocket | Oracle Cloud VPS (api.real-sync.app via Cloudflare Tunnel) | Stable uptime, WebSocket management, free tier |
| **AI Service** | Python + FastAPI + PyTorch | RunPod GPU (RTX 4000 Ada) | GPU needed for CLIP inference, on-demand to save cost |

Plus:
- **Recall.ai** -- third-party meeting bot API for Zoom/Meet/Teams capture
- **Supabase** -- PostgreSQL database + JWT authentication + Row-Level Security

### Q: Why not run everything on one server?

**A:** Cost and separation of concerns. The backend is lightweight Node.js -- it doesn't need a GPU. The AI service needs CUDA for CLIP inference. Renting a GPU server 24/7 to run both wastes money on GPU time during idle periods. RunPod's on-demand model means we pay $0.20/hr only during active meetings. Oracle Cloud's Always Free ARM tier gives us stable backend hosting at zero cost.

### Q: How do the services communicate?

**A:**
- **Recall.ai -> Backend**: WebSocket (`/ws/recall`) -- per-participant video frames (PNG 640x360, ~2 fps) and audio (PCM16 16kHz)
- **Backend -> AI Service**: HTTP REST (`/api/analyze/frame`, `/api/analyze/audio`, `/api/analyze/text`)
- **Backend -> Frontend**: WebSocket (`/ws`) -- real-time metrics and alert broadcast, throttled to 1 push per 1.5 seconds
- **Backend -> Supabase**: PostgreSQL via Supabase JS client for persistent storage

### Q: What database schema do you use?

**A:** Supabase PostgreSQL with 6 tables:
1. **sessions** -- meeting records (title, type, URL, user, timestamps)
2. **transcript_lines** -- utterance text with speaker names, linked to sessions
3. **alerts** -- every fired alert with severity, category, confidence, source model
4. **suggestions** -- contextual recommendations driven by meeting type
5. **metrics_snapshots** -- periodic JSONB snapshots for reports
6. **session_reports** -- aggregated post-session summaries

All tables use Row-Level Security -- users can only access their own data, enforced at the database level.

---

## 3. AI & Detection Pipeline

### Q: What AI models do you use?

**A:** Six models loaded at startup:

| Model | Size | Purpose |
|-------|------|---------|
| **CLIP ViT-L/14 (GenD)** | ~1.8 GB | Face-swap deepfake detection (semantic features) |
| **MediaPipe** | ~224 KB | Face detection and localization |
| **EfficientNet-B2** | 31 MB | Six-class emotion recognition |
| **WavLM-base** | 361 MB | Voice authenticity / audio deepfake detection |
| **Whisper (base)** | ~140 MB | Audio transcription |
| **DeBERTa-v3-base** | ~440 MB | Social engineering / phishing detection via zero-shot NLI |

Plus two signal-processing analyzers (no model weights): **Frequency analyzer** (DCT high-frequency) and **Boundary analyzer** (face boundary texture).

### Q: Why did you choose CLIP over CNN-based detectors?

**A:** We started with EfficientNet-B4-SBI (a CNN detector). It scored 95%+ AUC on benchmarks. But on actual Zoom video, it failed completely -- scoring real faces as fake and fake faces as real. The reason: CNN detectors look for pixel-level blending artifacts at the face-swap boundary. Zoom's H.264 compression at low bitrates destroys those artifacts before the detector sees the frame.

CLIP ViT-L/14 captures semantic features instead -- does the lighting match the environment? Are facial geometry proportions consistent? Is the skin texture distribution natural? These higher-level cues survive compression. We confirmed this empirically: CLIP maintained clear separation between real and fake faces on Zoom-captured frames where EfficientNet produced essentially random scores.

### Q: What is the ensemble and why do you need it?

**A:** CLIP alone struggles with **post-processed** face swaps. An attacker who applies bilateral filtering + sharpening after a swap can shrink CLIP's score separation. In our tests, a raw swap scored 0.21 CLIP authenticity. The same swap post-processed scored 0.64 -- dangerously close to real faces at 0.89.

The ensemble adds two more signals:

1. **Frequency analyzer** -- inswapper processes faces at 128x128 then upsamples. This strips high-frequency texture. We measure DCT high-frequency energy loss. Post-processing actually makes this worse (bilateral filter further kills high-frequency detail).

2. **Boundary analyzer** -- face swaps paste a resized crop into the original image, creating a noise pattern discontinuity at the boundary. We measure Laplacian variance and cross-channel noise across inner, boundary ring, and outer regions.

**Ensemble formula:** `0.50 x CLIP + 0.30 x frequency + 0.20 x boundary`

**Result:** Post-processed swap score separation improved from **0.028 to 0.214 -- a 7.6x improvement**. This is our main technical contribution.

### Q: How does the frequency analyzer work specifically?

**A:** We apply a 2D Discrete Cosine Transform (DCT) to a grayscale 256x256 face crop and measure energy in high-frequency bands (normalized DCT distance > 0.50). The log-scale high-frequency ratio is the primary discriminator:

- Real face (good lighting): log-high-freq ~ -4.95
- Raw inswapper swap: ~ -6.30
- Post-processed swap: ~ -9.58

We also run an FFT-based GAN fingerprint detector that looks for periodic spectral peaks from GAN upsampling layers. This is based on Wang et al.'s (CVPR 2020) finding that CNN-generated images have consistent spectral artifacts in the Fourier domain.

### Q: How does the boundary analyzer work?

**A:** We define three regions using elliptical distance from the face center: inner (<65% of face extent), boundary ring (65-90%), and outer (>90%). We measure:
- Laplacian variance per region
- High-pass noise level
- Per-channel noise variance

Key calibration finding:
- Real faces: cross-channel noise variance at boundary ~ 0.13
- Post-processed swaps: ~ 0.92

Camera sensor noise is spatially uniform across a real image. A face swap breaks this uniformity at the paste boundary.

### Q: How does the text analysis work? Why zero-shot NLI instead of fine-tuning?

**A:** We use DeBERTa-v3-base in zero-shot natural language inference mode. We evaluate 6 behavioral hypotheses against each transcript segment:
1. Asking for passwords / credentials / 2FA codes
2. Demanding urgent wire transfers / payments
3. Impersonating IT support / executives / authorities
4. Telling someone to keep things secret / not verify
5. Threatening consequences (suspension, legal action, arrest)
6. Asking to install remote access software / click suspicious links

Scores above 0.65 trigger medium-severity alerts; above 0.80 trigger high.

**Why not fine-tune?** We tried. A model fine-tuned on email/spam data got 99.2% accuracy in-domain but only **50% on actual Zoom transcripts** because phishing emails sound nothing like meeting conversations. Zero-shot NLI with our 6 hypotheses got **89% on Zoom-style conversation samples**. The lesson: domain mismatch kills fine-tuned models. Zero-shot generalizes across registers.

### Q: What is the trust score formula?

**A:**
- **With audio signal (RMS > 100):** `trust = 0.45 x video + 0.35 x audio + 0.20 x behavior`
- **Without audio (silence):** `trust = 0.55 x video + 0.45 x behavior`

Where `behavior_conf = 0.5 + emotion_confidence x 0.5`.

---

## 4. Statistical Methods (SPRT)

### Q: What is SPRT and why do you use it?

**A:** SPRT (Sequential Probability Ratio Test) is Wald's statistical framework from 1947 for sequential decision-making. It accumulates evidence over time and commits to a decision only when it has 95% confidence.

We use it because single-frame scores are noisy. A single frame might score 0.65 -- is that real or fake? SPRT doesn't guess. It accumulates log-likelihood ratios across frames until the evidence crosses a threshold. Clear cases converge fast; ambiguous cases take longer (which is itself informative).

### Q: How does SPRT work mathematically?

**A:** For each frame with ensemble score x, we compute:

```
llr_increment = -0.5 x ((x - fake_mean) / std)^2 + 0.5 x ((x - real_mean) / std)^2
session_llr += llr_increment
```

Parameters: real_mean=0.65, fake_mean=0.32, std=0.12 (calibrated for 360p Recall.ai frames).

Decision boundaries (alpha = beta = 0.05 for 5% error rates):
- FAKE when session_llr > log(19) ~ 2.94
- REAL when session_llr < log(0.053) ~ -2.94

### Q: How fast does SPRT converge?

**A:**
- Real faces: **5-8 frames** (~3-4 seconds at 2 fps)
- Raw inswapper swaps: **2-3 frames** (~1-1.5 seconds)
- Post-processed swaps: **3-5 frames** (~1.5-2.5 seconds)
- StyleGAN-generated faces: **Did not converge** (borderline, which is correct -- the system is genuinely uncertain)

### Q: What happens after SPRT converges? Can it be fooled by switching faces?

**A:** Once SPRT converges, the decision is sticky -- it won't re-evaluate. This prevents an attacker from "averaging out" a fake detection by alternating real and fake footage. However, this means if someone starts with a real face (SPRT converges to REAL) then switches to a deepfake, SPRT won't catch the switch directly.

We compensate with the **temporal anomaly detector**: if the trust score suddenly drops more than 0.40 below the 15-frame rolling mean, a high-severity alert fires regardless of SPRT state. Future work includes per-participant SPRT tracking and SPRT reset on identity switch events.

---

## 5. Meeting Bot & Capture Layer

### Q: How does the bot join meetings?

**A:** We use Recall.ai's meeting bot API. The backend creates a bot via REST API with the meeting URL. Recall.ai's bot joins as a native SDK participant (not a browser). It appears as "RealSync Bot" with a branded camera tile. The bot connects to our backend's `/ws/recall` WebSocket endpoint and streams:
- Per-participant video frames (PNG, 640x360, ~2 fps)
- Per-participant audio (PCM16, 16kHz mono)
- Participant metadata (names, join/leave, speech events)

### Q: What platforms are supported?

**A:** Zoom, Google Meet, and Microsoft Teams -- all through Recall.ai's native SDK integration.

### Q: Why did you switch from Puppeteer to Recall.ai?

**A:** The Puppeteer bot (2,072 lines of code) worked in controlled testing but had four unsolvable problems:
1. **Audio beeping** -- PulseAudio loopback injected artifacts; participants could hear feedback
2. **Gallery View lock** -- couldn't reliably switch to Speaker View; Zoom's DOM changed frequently
3. **Speaker attribution** -- multiple faces in Gallery View caused oscillation between tiles
4. **Whisper hallucination** -- PulseAudio captured ambient noise; Whisper transcribed it as phantom speech, generating false "SCAM" alerts

Recall.ai solved all four: per-participant streams (no Gallery View), clean PCM audio (no beeping), participant names from API (no DOM scraping), speech events for accurate speaker detection. The migration replaced 2,072 lines of Puppeteer with a **280-line RecallBotAdapter** -- same interface, same ingest message format, zero downstream changes needed.

### Q: What does Recall.ai cost?

**A:** $0.50 per bot-hour, with 5 free hours on signup. A typical 2-hour meeting costs $1.00 in bot time. We budgeted ~$30-40 total for the capstone project.

### Q: What's your fallback if Recall.ai goes down?

**A:** An environment variable (`BOT_ADAPTER=recall` or `puppeteer`) controls which adapter loads. The Puppeteer code is preserved on a `puppeteer-backup` git branch. Switching is instant -- no code changes needed.

---

## 6. Frontend & User Experience

### Q: What does the dashboard show?

**A:** The main dashboard screen displays:
- **Trust score gauge** (animated, real-time)
- **Per-signal confidence bars** (Visual / Audio / Emotion)
- **Deepfake risk indicator** with severity level
- **Live alert feed** with category tags and timestamps
- **Trust score timeline chart** (Recharts)
- **Participant count** and **source connection status**

Everything updates over WebSocket with sub-second latency.

### Q: How many screens does the app have?

**A:** Eight screens:
1. **Login / Sign Up** -- Supabase auth, Google/Microsoft OAuth, corporate email enforcement
2. **Complete Profile** -- post-registration onboarding (name, job title, avatar)
3. **Sessions** -- session management with stat cards, table/card views
4. **Dashboard** -- main analysis view (described above)
5. **Reports** -- session report with trust curve, alert timeline, export to PDF/CSV/JSON
6. **Settings** -- profile, detection toggles, notification preferences, security (password change, 2FA coming soon)
7. **FAQ** -- expandable sections, working contact link
8. **Command palette** (Cmd+K) for keyboard navigation

### Q: How do you handle real-time updates?

**A:** React context + WebSocket:
- **WebSocketContext** -- persistent connection with exponential backoff reconnection and heartbeat ping/pong
- **SessionContext** -- auth state and active session tracking
- **NotificationContext** -- alert delivery with desktop notifications and audio cues

The backend throttles broadcasts to 1 push per 1.5 seconds to prevent overwhelming the frontend at high frame rates.

### Q: Can users export reports?

**A:** Yes, three formats:
- **PDF** -- via @react-pdf/renderer (vector text, clean images, native React JSX). Contains session metadata, severity breakdown, trust score timeline, alert history with category tags, executive summary.
- **CSV** -- sanitized against formula injection
- **JSON** -- full data export

All generated client-side.

---

## 7. Security & Privacy

### Q: How do you handle authentication?

**A:** Supabase Auth with JWT tokens:
- Email/password with email verification
- Google and Microsoft OAuth
- Corporate email domain enforcement (personal Gmail/Yahoo blocked by policy)
- Password strength validation
- MFA/TOTP marked as "coming soon"
- Row-Level Security (RLS) at the database level -- users can only access their own sessions and data

### Q: What security measures are in place?

**A:**
- **Helmet.js** for HTTP security headers
- **Rate limiting** -- tiered: 100 req/min global, 20/min session creation, 10/min settings, 30/min notifications
- **CORS** origin validation against a configurable allowlist
- **UUID validation** middleware on all session ID parameters
- **API key authentication** on the AI service (HMAC timing-safe comparison)
- **Production startup guard** -- backend refuses to start without Supabase credentials
- **Meeting URL validation** -- only \*.zoom.us, \*.zoom.com, \*.google.com, or \*.microsoft.com URLs accepted

### Q: Is this a surveillance tool? What about privacy?

**A:** RealSync is designed for meeting hosts who want to verify ongoing calls, not for mass surveillance. Key privacy considerations:
- The host must explicitly create a session and enter the meeting URL
- The bot joins visibly as "RealSync Bot" -- it's not hidden
- Only the authenticated host can view their own session data (RLS)
- Transcripts are stored in Supabase with RLS; no cross-user access
- No video/audio is stored permanently -- only analysis results and transcripts
- The system processes data in real-time and doesn't retain raw frames or audio

**Limitation acknowledged:** Full GDPR consent flow is deferred. A production deployment would need participant consent mechanisms.

---

## 8. Testing & Validation

### Q: How did you test the system?

**A:** Three levels:

1. **Unit/Integration tests** -- 34+ test cases across Jest (backend) + pytest (AI service), covering:
   - Health endpoints, session CRUD, auth enforcement
   - Alert fusion logic (thresholds, cooldowns, consecutive-frame gating, escalation)
   - Fraud detector keyword scoring
   - Full inference pipeline integration
   - EWMA smoothing, anomaly detection
   - Audio/text endpoint edge cases

2. **Live end-to-end tests (April 6-8, 2026):**
   - Deep Live Cam (inswapper) -> OBS Virtual Camera -> Zoom -> RealSync bot -> AI service -> dashboard
   - Multiple sessions testing real faces, raw swaps, post-processed swaps, StyleGAN faces

3. **Code review** -- 5 reviewers, 40 issues found, 33 fixed before code freeze

### Q: What were the live test results?

**A:**

| Scenario | CLIP Score | Ensemble Score | SPRT Decision | Frames to Decision | Latency |
|----------|-----------|----------------|---------------|-------------------|---------|
| Real face (good lighting) | 0.76-0.89 | 0.72-0.84 | REAL 95% | 5-8 | 85-107 ms |
| Raw inswapper swap | 0.21-0.35 | 0.28-0.50 | FAKE 95% | 2-3 | 85-107 ms |
| Post-processed swap | 0.64 | 0.53 | FAKE 95% | 3-5 | 85-107 ms |
| StyleGAN-generated | 0.73 | 0.70-0.73 | Undecided | -- | 85-107 ms |

### Q: What about the audio test results?

**A:** Voice authenticity scored 0.76 (low risk) on real speech and 0.44 (medium risk) on silence. The three-signal trust score (video + audio + behavior) was validated working end-to-end.

---

## 9. Deployment & Infrastructure

### Q: What's the total infrastructure cost?

**A:**
| Component | Cost |
|-----------|------|
| AI GPU (RunPod) | $0.20/hr on-demand (~AED 30/month) |
| Backend (Oracle Cloud) | Free (Always Free ARM tier) |
| Frontend (Cloudflare Pages) | Free |
| Database (Supabase) | Free tier |
| Domain (real-sync.app) | ~AED 40/year |
| Meeting bot (Recall.ai) | $0.50/hr per session |
| **Total development period (6 months)** | **Under $50 USD** |

vs. original estimate of AED 220-270/month. **Actual cost: ~AED 30-50/month.**

### Q: What are the live URLs?

**A:**
- Frontend: **real-sync.app** (landing) + **real-sync.app/app** (dashboard)
- Backend: **api.real-sync.app** (via Cloudflare Tunnel)
- AI Service: RunPod GPU pod (started on-demand)

### Q: What does the Oracle Cloud VPS provide?

**A:** Always Free ARM instance with 4 OCPU and 24 GB RAM. PM2 manages the Node.js process with auto-restart. Cloudflare Tunnel provides HTTPS without exposing ports. We migrated from Railway due to persistent Docker cache invalidation bugs.

---

## 10. Business & Market Viability

### Q: Is there a market for this?

**A:** Yes:
- **$2.7 billion** in estimated deepfake fraud losses in 2023 (Sumsub)
- **40% year-on-year** increase in deepfake fraud (Sumsub 2024)
- **$25.6 million** stolen in a single deepfake Zoom call (Hong Kong, Feb 2024)
- Voice deepfake fraud documented since 2019 ($243K wire transfer CEO impersonation)
- No existing commercial product does real-time in-call detection
- Reality Defender and Attestiv are post-call forensic tools

### Q: What would a pricing model look like?

**A:** A freemium SaaS model:
- **Free tier** -- limited sessions per month, basic detection
- **Business tier** -- unlimited sessions, advanced alerts, PDF reports, priority support
- **Enterprise** -- custom deployment, SSO, compliance features, SLA

Per-session costs are low ($0.20-0.70/hr total for GPU + bot), so margins are healthy even at moderate pricing.

### Q: Who are your competitors?

**A:** No direct competitor does exactly what RealSync does. Adjacent products:
- **Reality Defender** -- post-call forensic analysis, not real-time
- **Attestiv** -- document and media authenticity verification, not live meeting detection
- **Pindrop** -- voice authentication for call centers, not video meeting deepfake detection
- **Zoom/Teams/Meet** -- no built-in deepfake detection whatsoever

### Q: What's the competitive advantage?

**A:**
1. **Zero installation** -- participants don't install anything
2. **Real-time** -- sub-200 ms latency vs. forensic tools that run post-call
3. **Multi-modal** -- video + audio + text, not just one signal
4. **Platform-agnostic** -- Zoom, Meet, Teams via Recall.ai
5. **Cost-efficient** -- on-demand GPU + free-tier hosting, under $50 total for development

---

## 11. Limitations & Honesty

> Judges respect teams that know their limitations. Don't hide these -- own them.

### Q: What can't RealSync detect?

**A:**
1. **GAN-generated faces (StyleGAN)** -- scored 0.70-0.73, too close to real faces for SPRT to converge. GenD was trained on face-swap artifacts, not GAN-generated faces. This is a known limitation with a clear mitigation path (zero-shot CLIP text prompts).

2. **Audio deepfakes through Zoom compression** -- WavLM was trained on clean ASVspoof 2019 data. Zoom's Opus codec degrades audio at variable bitrates. We treat audio as supporting evidence (35% weight), not a standalone verdict.

3. **Low-light environments** -- real face scores drop to ~0.67 in dark rooms (vs. 0.89 in good lighting). CLIP features are partly luminance-dependent.

### Q: What are the single points of failure?

**A:**
- **Recall.ai dependency** -- if Recall.ai goes down, fallback to Puppeteer bot (instant via env var switch)
- **RunPod GPU pod** -- community pods can be reclaimed. Mitigation: use secure pods ($0.59/hr) for important sessions. AI service is stateless and re-initializes on restart.
- **Supabase** -- backend degrades gracefully (continues with null metrics, doesn't crash)

### Q: SPRT can be tricked by starting with a real face?

**A:** Yes, if someone shows their real face first and SPRT converges to REAL, then switches to a deepfake, SPRT won't catch it directly. Our **temporal anomaly detector** partially compensates -- it flags sudden trust drops exceeding 0.40 below the rolling mean. Future work: per-participant SPRT tracking and SPRT reset on identity switch events.

### Q: How robust is this to new deepfake methods?

**A:** The ensemble's three signals target different physical properties:
- CLIP: semantic consistency (generalizes across manipulation types)
- Frequency: high-frequency energy loss from upsampling (any encoder-decoder swap)
- Boundary: noise pattern discontinuity at the paste boundary (any compositing method)

New face-swap methods that still upscale and paste will likely trigger at least two signals. However, fundamentally new generation methods (e.g., full-frame generation without compositing) could evade all three. The system would need to be re-evaluated for each new threat.

---

## 12. Academic & Research Rigor

### Q: What papers did you reference?

**A:** 13 references including:
- FaceForensics++ (Rossler et al., ICCV 2019) -- benchmark dataset
- Face X-Ray (Li et al., CVPR 2020) -- blending boundary detection
- Wang et al. (CVPR 2020) -- CNN spectral artifacts in Fourier domain
- Ojha et al. (CVPR 2023) -- universal fake image detection via CLIP
- CLIP (Radford et al., ICML 2021) -- foundation model
- ViT (Dosovitskiy et al., ICLR 2021) -- vision transformer architecture
- Wald (1947) -- SPRT statistical framework
- DeBERTa (He et al., ICLR 2021) -- disentangled attention for text
- WavLM (Chen et al., IEEE 2022) -- self-supervised speech model

### Q: What is your main research contribution?

**A:** The **ensemble deepfake detector** that combines CLIP ViT-L/14 with frequency-domain and boundary-texture analysis for face-swap detection on Zoom-compressed video. The 7.6x improvement in score separation on post-processed swaps (0.028 -> 0.214) is the core technical contribution.

Secondary contributions:
- Novel application of SPRT for session-level deepfake decision-making
- Empirical demonstration that zero-shot NLI (89%) outperforms fine-tuned classification (50%) for meeting transcript social engineering detection
- End-to-end system architecture for real-time multi-modal meeting analysis

### Q: Why didn't you train your own model?

**A:** Three reasons:
1. **No labeled Zoom deepfake dataset exists** -- we'd have to create one
2. **Transfer learning from CLIP is stronger** -- CLIP's pretraining on 400M image-text pairs gives it semantic understanding no small training set could match
3. **GenD was published at WACV 2026** -- a recent, peer-reviewed model finetuned specifically for face-swap detection on CLIP features

We added the frequency and boundary analyzers as pure signal processing (no training needed) to cover CLIP's blind spots.

### Q: How does this relate to the literature?

**A:** Our boundary analyzer applies the same principle as Face X-Ray (Li et al., CVPR 2020) -- detecting where a face was pasted rather than what the face looks like. Our frequency analyzer is based on Wang et al.'s (CVPR 2020) finding about spectral artifacts in generated images. SPRT comes from Wald's 1947 sequential analysis. We combined established techniques in a novel way for a new application (real-time compressed video).

---

## 13. Team & Process

### Q: How did you divide the work?

**A:** Five team members:
- **Ahmed Sarhan** -- Project Leader, AI pipeline, backend architecture, deployment, integration
- **Mohammed Atwani** -- Scribe, documentation, Zoom bot development
- **Mohamed Ghazi** -- Frontend development
- **Yousef Kanjo** -- Frontend development
- **Aws Diab** -- AI model research and integration

### Q: What was the development timeline?

**A:** Six months, five phases:
1. **Oct-Nov 2025** -- Architecture decisions, Zoom bot prototype, initial AI pipeline with EfficientNet-B4
2. **Nov-Dec 2025** -- Backend WebSocket infrastructure, Supabase integration, frontend core screens
3. **Jan-Feb 2026** -- CLIP model integration (replaced failing EfficientNet), frequency/boundary analyzers, SPRT
4. **Feb-Mar 2026** -- Frontend dashboard completion, alert fusion engine, PDF reports
5. **Mar-Apr 2026** -- Live E2E testing, Recall.ai migration, calibration, bug fixes, documentation

### Q: What was the biggest challenge?

**A:** Discovering that our original detection model (EfficientNet-B4-SBI) completely failed on Zoom video in January 2026. It scored real faces as fake and fake faces as real because Zoom's H.264 compression destroys the pixel-level artifacts CNN detectors rely on. We had to rethink the entire detection approach mid-project -- pivoting to CLIP, adding the ensemble, implementing SPRT. That pivot is documented in the change report (Section 8 of the final report).

### Q: What tools did you use?

**A:** React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui, Recharts (frontend); Node.js, Express 5, ws library, PM2 (backend); Python, FastAPI, PyTorch, CLIP, MediaPipe, WavLM, Whisper, DeBERTa (AI); Supabase (database/auth); Git + GitHub (version control); Recall.ai (meeting bot API); Cloudflare Pages, Oracle Cloud VPS, RunPod GPU (deployment).

---

## 14. Curveball / Stress-Test Questions

### Q: What happens if someone uses a deepfake that you've never seen before?

**A:** The ensemble's three signals target physical properties (semantic inconsistency, frequency energy loss, boundary noise discontinuity) rather than specific deepfake tools. Any face-swap that involves upsampling and compositing should trigger at least two signals. However, a completely novel generation method (e.g., full-frame generation without compositing) could potentially evade detection. SPRT would return "undecided" rather than a false "real" verdict, which is the correct behavior under uncertainty.

### Q: Could an attacker reverse-engineer your detection to evade it?

**A:** Partially. If an attacker knows the exact ensemble weights and thresholds, they could try to optimize their post-processing to maximize the ensemble score. However:
- Improving CLIP score (add texture detail) worsens frequency score (more high-freq = looks more real to freq analyzer but CLIP catches it)
- Improving boundary score (smooth the paste boundary) worsens CLIP score (over-smoothing is detectable semantically)
- The three signals create opposing pressures that make simultaneous evasion difficult

This is a cat-and-mouse problem inherent to adversarial ML. We don't claim our system is unbreakable -- we claim it raises the cost of attack significantly.

### Q: Why not just use a webcam integrity check instead?

**A:** That only catches virtual cameras, not legitimate webcams showing a deepfaked face. Deep Live Cam generates the face swap first, then OBS captures and presents it as a virtual camera feed. Even if Zoom blocked virtual cameras (which it doesn't), an attacker could use an HDMI capture card to loop the deepfaked output through a "real" camera device. You need content-level detection, not device-level detection.

### Q: What about latency? Does 85-107 ms per frame actually matter?

**A:** Yes. At ~2 fps capture rate, we have ~500 ms between frames. Processing at 85-107 ms means we complete analysis well before the next frame arrives. The entire pipeline (capture -> network -> inference -> broadcast -> render) stays under 200 ms. This is genuinely real-time -- the dashboard updates are perceived as instant during live testing.

### Q: How do you handle network failures mid-meeting?

**A:** Multiple resilience layers:
- WebSocket has exponential backoff reconnection
- If the AI service is unavailable, the backend continues with null metrics (doesn't crash)
- Session state is in-memory with periodic Supabase snapshots
- The bot maintains its own connection to the meeting independently of backend health

### Q: Is 34 test cases enough?

**A:** For a capstone project, 34+ test cases covering the critical paths is reasonable. Each test covers a distinct behavior (not just line coverage). The more important validation is the live E2E testing with real Zoom calls and real deepfakes. We tested the full pipeline end-to-end across multiple sessions with measurable, reproducible results.

---

## 15. Quick Reference: Key Numbers

| Metric | Value |
|--------|-------|
| Frame processing latency | 85-107 ms |
| Latency target (NFR) | < 200 ms |
| Real face ensemble score | 0.72-0.84 |
| Raw deepfake ensemble score | 0.28-0.50 |
| Post-processed deepfake score | 0.53 |
| Score separation improvement (ensemble vs CLIP-only) | **7.6x** (0.028 -> 0.214) |
| SPRT confidence level | 95% |
| Real face convergence | 5-8 frames |
| Raw deepfake convergence | 2-3 frames |
| Post-processed deepfake convergence | 3-5 frames |
| Zero-shot NLI accuracy (social engineering) | 89% |
| Fine-tuned email classifier accuracy on Zoom data | 50% |
| Total test cases | 34+ |
| Lines of code replaced (Puppeteer -> Recall) | 2,072 -> 280 |
| Infrastructure cost (6 months development) | < $50 USD |
| GPU cost per hour | $0.20 (community) / $0.59 (secure) |
| Bot cost per hour | $0.50 (Recall.ai) |
| Models loaded at startup | 6 ML models + 2 signal processors |
| Database tables | 6 with Row-Level Security |
| Frontend screens | 8 |
| Capture resolution | 640x360 PNG, ~2 fps per participant |
| Team size | 5 members |
| Development duration | 6 months (Oct 2025 - Apr 2026) |
| Academic references | 13 papers |
| Ensemble weights | 0.50 CLIP + 0.30 freq + 0.20 boundary |
| Trust score weights (with audio) | 0.45 video + 0.35 audio + 0.20 behavior |

---

## 16. Final Report Highlights to Reference

When judges ask deeper questions, point them to these report sections:

| Topic | Report Section |
|-------|---------------|
| Threat landscape & real incidents | Section 4.1-4.2 (Introduction) |
| Literature & research foundations | Section 5 (Literature Review) |
| Functional requirements (FR1-FR10) | Section 6.1 |
| Non-functional requirements (latency, security) | Section 6.2 |
| Feasibility analysis (technical, economic, operational) | Section 7 |
| All architecture changes from semester 1 | Section 8.1 (Change Report) |
| Puppeteer -> Recall.ai migration story | Section 8.5 |
| 360p calibration details | Section 8.5.1 |
| System architecture & data flow | Section 9 (8.1-8.4 in report) |
| AI pipeline implementation details | Section 10 (9.1 in report) |
| Ensemble formula & improvement metrics | Section 10 (9.1 under "Ensemble detection") |
| SPRT math & convergence | Section 10 (9.1 under "SPRT accumulator") |
| Live E2E test results table | Section 11 (10.2 in report) |
| Deployment architecture rationale | Section 12 (11.4 in report) |
| Discussion of results | Section 13 |
| Known limitations (honest assessment) | Section 14 (13.1) |
| Future work | Section 14 (13.2) |

---

## Tips for the Team

1. **Lead with the demo, not the slides.** If you can show the dashboard live with a real Zoom call, do it. The visual impact of watching a trust score drop when a deepfake activates is worth more than any slide.

2. **Own your limitations.** Judges respect honesty. When asked about GAN detection or audio under compression, say "we know, here's why, and here's how we'd fix it."

3. **Emphasize the 7.6x number.** The ensemble improvement is your strongest technical contribution. Make sure judges understand that post-processed swaps are the hard case and you solved it.

4. **The pivot story is compelling.** The mid-project discovery that EfficientNet completely failed on Zoom video, and the recovery through CLIP + ensemble + SPRT, shows real engineering maturity. Don't hide it -- it's a strength.

5. **Know the cost numbers.** Under $50 total infrastructure cost for 6 months of development is impressive. Judges from industry will appreciate the efficiency.

6. **If you don't know, say so.** "That's a great question -- we haven't tested that scenario yet, but our approach would be..." is always better than guessing.

---

*Generated 14 April 2026. Good luck tomorrow!*
