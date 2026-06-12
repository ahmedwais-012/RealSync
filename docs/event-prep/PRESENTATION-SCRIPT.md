# RealSync Presentation Script

**Format:** 10-min presentation + 5-min Q&A
**Slides:** 7
**Pacing:** ~1.5 min per slide (with demo slide getting more time)

---

## Slide 1 — Title (~30 seconds)

Good morning everyone. We are Innovation Labs, and this is RealSync — a real-time deepfake detection system for virtual meetings.

Our team: Mohammed Atwani, Mohamed Ghazi, Yousef Kanjo, Aws Diab, and myself, Ahmed Sarhan. Supervised by Dr. May El Barachi.

What you're about to see is a system that can tell you — live, during a Zoom call — whether the person you're talking to is real or fake. And we'll show you a live demo at the end.

---

## Slide 2 — The Problem (~1.5 minutes)

Let's start with why this matters.

In February 2024, a finance worker at a multinational company in Hong Kong was instructed by his CFO to transfer 25 million dollars. He was on a live Zoom call. Every person on that call — including the CFO — was a deepfake. He transferred the money. It was gone.

This is not science fiction. The technology to do this is freely available. Tools like Deep Live Cam can swap your face in real time on a consumer laptop. Voice cloning is a few clicks away. Sumsub reported a 40% year-on-year increase in deepfake fraud globally, with estimated losses of 2.7 billion dollars in 2023 alone.

Now here's the gap. Deepfake detection research has produced good models. FaceForensics++, Face X-Ray, CLIP-based detectors — they all work well on benchmark datasets. But they all operate offline. You upload a recording after the meeting, and they tell you what happened. By then, the money is gone.

No commercial product — not Zoom, not Teams, not Meet — performs real-time deepfake detection during a live call. No academic benchmark has ever been deployed as a live meeting tool. That's the gap we're filling.

---

## Slide 3 — Our Solution (~1.5 minutes)

RealSync is a SaaS bot that joins any virtual meeting — Zoom, Google Meet, or Microsoft Teams — analyzes each participant's audio and video in real time, and surfaces a live Trust Score on a secure dashboard.

Four things make this work:

First, the Recall.ai bot. It joins through a native API — no screen sharing, no browser automation, nothing the host needs to do. It captures isolated per-participant streams at 640 by 360 resolution at 2 frames per second, plus clean PCM audio at 16 kilohertz. Each person gets their own stream — no more trying to crop faces from a gallery view screenshot.

Second, multi-modal fusion. We don't rely on a single signal. We combine visual analysis — that's our deepfake ensemble — with audio analysis using WavLM for voice authenticity, and behavioral analysis using DeBERTa for social engineering detection. All three signals feed into a single Trust Score from 0 to 100.

Third, sub-110 millisecond latency. Our AI service runs on an RTX 4000 Ada GPU through RunPod. CLIP ViT-L/14 inference takes 50 to 70 milliseconds per frame. The entire pipeline from frame capture to dashboard update is under 200 milliseconds.

Fourth, it's non-intrusive. Participants don't install anything. The meeting host creates a session, pastes the meeting URL, and the bot handles the rest. The host monitors a separate dashboard.

---

## Slide 4 — System Architecture (~1.5 minutes)

The system runs as three independent services, and there's a deliberate reason for that split.

Tier 1 is the frontend — a React TypeScript dashboard deployed on Cloudflare Pages at real-sync.app. It shows the trust score gauge, per-signal confidence bars, a live alert feed, and a transcript viewer. Everything updates over WebSocket in real time.

Tier 2 is the backend — Node.js on an Oracle Cloud VPS, accessible at api.real-sync.app through a Cloudflare Tunnel. This is where session management, the Alert Fusion Engine, and SPRT session decisions happen. It also handles the Recall.ai webhook ingestion — the bot streams data here, and the backend routes it to the AI service.

Tier 3 is the AI service — Python FastAPI running on a RunPod GPU. This is where the heavy lifting happens. CLIP ViT-L/14 for deepfake detection, EfficientNet-B2 for emotion classification, WavLM for audio authenticity, and DeBERTa for social engineering detection in transcripts. Processing time is 40 to 60 milliseconds per frame.

Why not put it all on one server? Because the backend is lightweight Node.js — it doesn't need a GPU. Paying for GPU time to run a Node server wastes money. RunPod is on-demand at 20 cents an hour. We only pay for compute during active meetings. The Oracle VPS is free. Total infrastructure cost for six months of development was under 50 dollars.

The Trust Score formula combines the three signals: 45% visual, 35% audio, 20% behavioral. When the camera is off, it falls back to audio-only scoring.

---

## Slide 5 — Key Technical Features (~2 minutes)

Let me walk you through the six features that make the detection reliable.

**SPRT Session Decisions.** We don't rely on single-frame scores — those are noisy. We use Wald's Sequential Probability Ratio Test, a statistical framework from 1947, to accumulate evidence across frames. It reaches 95% statistical confidence in 2 to 8 frames. Real faces converge to "authentic" in 5 to 8 frames. Raw deepfakes converge to "fake" in 2 to 3 frames. It doesn't guess — if the evidence is ambiguous, it says "undecided," which is the correct answer.

**Recall.ai Bot Integration.** We originally built a 2,000-line Puppeteer bot that joined Zoom through a headless browser. It worked, but had unsolvable problems — audio beeping, gallery view lock, speaker attribution failures. We migrated to Recall.ai's native SDK, which gave us isolated per-participant streams. That 2,000 lines became 280 lines, and all those capture-layer problems disappeared. We support Zoom, Meet, and Teams through one adapter.

**EMA Smoothing.** Raw frame scores fluctuate. We apply exponential moving average smoothing with a 15-second hold and 15-second decay. This prevents the trust score from flickering on brief silences or camera glitches, while still responding quickly to real changes.

**Live Transcription and AI Coaching.** Transcripts are analyzed by DeBERTa in a zero-shot NLI setup with six social engineering hypotheses — things like credential requests, urgency pressure, authority impersonation. We chose zero-shot over fine-tuning because a fine-tuned email classifier scored 50% on Zoom transcripts, while our zero-shot approach scored 89%.

**Alert Fusion Engine.** Not every anomaly should trigger an alert. The engine deduplicates low-confidence signals, requires 3 consecutive frames before firing a deepfake alert, applies 30-second cooldowns per alert type, and escalates multi-signal anomalies — if both a deepfake signal and a text fraud signal occur together, it escalates to critical severity.

**Global Decay Ticker.** When a participant mutes or turns off their camera, the system correctly fades the Trust Score toward zero rather than holding a stale number. When they come back, it recovers.

---

## Slide 6 — Results & Future Work (~1.5 minutes)

We tested the full pipeline live — Deep Live Cam performing a face swap, fed through OBS into a Zoom call, with our bot capturing and analyzing in real time.

The results: real faces on 360p Recall.ai frames scored 85 to 92% — correctly classified as authentic. DeepLiveCam deepfakes scored 22 to 38% — alerts triggered immediately. The score separation is about 31 points on average. Our target was 45, so there's room to improve, but the system reliably distinguishes real from fake at 95% confidence.

End-to-end latency was 85 to 107 milliseconds per frame — well under our 200-millisecond design target. WavLM voice authenticity scored 65 to 75% on real speech. The system ran stable across 30-minute demo sessions with no crashes or data loss.

For future work, five priorities. First, moving to 720p frames — Recall.ai's 360p limits our score separation, and we expect a 14-point improvement at higher resolution. Second, GAN and diffusion artifact detection — our current ensemble targets face swaps specifically, not fully generated faces. Third, voice clone testing against ElevenLabs and Resemble.ai outputs. Fourth, per-participant identity profiles with persistent face embeddings across sessions. And fifth, webhook integrations to push alerts to Slack or Teams SOC channels.

---

## Slide 7 — Live Demo (~1.5 minutes)

Now let me show you RealSync working in a live Zoom meeting.

*[Navigate to real-sync.app and open the dashboard]*

What you see here is our landing page — "See what's real." When a host clicks Launch Dashboard, they log in, create a session, and paste a meeting URL.

*[Show the dashboard with live metrics]*

The bot joins the meeting. The dashboard lights up with live metrics. You can see the Trust Score gauge here — with a real face, it sits at 62 to 92%, all signals are green.

*[If doing live deepfake demo]*

Now watch what happens when I switch to a deepfake. The trust score drops to 22 to 38%. The deepfake alert fires. The alert feed shows the severity, the category, and a timestamp. When I switch back to my real face — the score recovers.

That's RealSync. A bot joins your meeting, tells you in real time who's real and who's not, and gives you a full report when the meeting ends.

Thank you. We're happy to take questions.

---

## Q&A Prep Reminders (not spoken)

- **If asked about GAN faces:** "Known limitation. Our ensemble targets face-swap artifacts. StyleGAN faces score 70-73%, too close to real for SPRT to converge. Future work addresses this with zero-shot CLIP text prompts."
- **If asked about accuracy:** "31-point score separation on 360p. Real faces 85-92%, deepfakes 22-38%. SPRT converges at 95% confidence in 2-8 frames."
- **If asked about cost:** "Under $50 total for six months. GPU at $0.20/hr on-demand. Backend free on Oracle Cloud. Frontend free on Cloudflare."
- **If asked about privacy:** "Bot joins visibly. Only the authenticated host sees data. Row-Level Security in Supabase. No raw video stored."
- **If asked why CLIP over CNN:** "EfficientNet-B4 failed on Zoom video — scored real faces as fake because H.264 compression destroys pixel-level artifacts. CLIP captures semantic features that survive compression."
- **If asked about the pivot from Puppeteer:** "2,072 lines of Puppeteer replaced by 280 lines of Recall.ai adapter. Same interface, same message format, zero downstream changes. Solved audio beeping, gallery view lock, speaker attribution, and Whisper hallucination problems."

---

*Total script time: ~9 minutes (leaves 1 minute buffer before Q&A)*
