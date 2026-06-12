# RealSync Presentation Script — Demo First

**Format:** 10-min presentation + 5-min Q&A
**Approach:** Lead with live demo, then explain how it works
**Pacing:** ~2 min demo, ~8 min slides

---

## Opening — Live Demo FIRST (~2 minutes)

Good morning everyone. We are Innovation Labs. Before we show you any slides, we want to show you what our system actually does.

*[Screen share: open real-sync.app dashboard, have Zoom meeting already running]*

What you're looking at is RealSync — a real-time deepfake detection dashboard. Right now, I'm in a live Zoom call. Our bot joined the meeting automatically. The dashboard is showing a Trust Score — you can see it's sitting at around 85 to 92%. All signals are green. That means the system is confident this is a real person.

Now watch what happens.

*[Switch Zoom camera to OBS Virtual Camera with Deep Live Cam face swap]*

The trust score is dropping. You can see it falling to 22 to 38%. The deepfake alert just fired — high severity. The alert feed shows exactly what triggered it and when. The system detected the face swap in 2 to 3 frames — that's about one second.

*[Switch back to real webcam]*

And now I switch back to my real face. The score recovers. Alerts clear.

That's RealSync. It joins your meeting, watches in real time, and tells you who's real and who's not. No one in the meeting had to install anything. The bot joined on its own.

Now let us show you how this works under the hood.

---

## Slide 1 — Title (~20 seconds)

We are Innovation Labs — Mohammed Atwani, Mohamed Ghazi, Yousef Kanjo, Aws Diab, and myself, Ahmed Sarhan. Supervised by Dr. May El Barachi. This is our CSIT321 capstone project.

---

## Slide 2 — The Problem (~1.5 minutes)

What you just saw matters because deepfake fraud is already happening at scale.

In February 2024, a finance worker in Hong Kong was on a live Zoom call with his CFO and several colleagues. They instructed him to transfer 25 million dollars. Every person on that call was a deepfake. He transferred the money.

This is not an edge case. Sumsub reported a 40% year-on-year increase in deepfake fraud in 2024, with 2.7 billion dollars in estimated losses in 2023. The tools to do this — Deep Live Cam, voice cloning services — are free and run on consumer hardware.

The gap is not in detection research. Models like FaceForensics++ and CLIP-based detectors work well on benchmarks. The gap is that every existing tool is offline and post-hoc — you upload a recording after the call, and they tell you what happened after the damage is done.

No commercial product — not Zoom, not Teams, not Meet — detects deepfakes during a live call. That's the gap RealSync fills, and you just saw it working.

---

## Slide 3 — Our Solution (~1 minute)

RealSync is a SaaS bot that joins any virtual meeting through Recall.ai's native API. No screen sharing, no browser automation, nothing the host needs to configure.

Four things make it work. The Recall.ai bot captures isolated per-participant video and audio streams. Multi-modal fusion combines visual, audio, and behavioural signals into one Trust Score. GPU inference on RunPod gives us sub-110 millisecond latency per frame. And it's completely non-intrusive — the host monitors a separate dashboard while the meeting runs normally.

---

## Slide 4 — System Architecture (~1.5 minutes)

Three independent services, split for a reason.

The frontend is a React TypeScript dashboard on Cloudflare Pages. Trust score gauge, confidence bars, alert feed, transcript viewer — all updating over WebSocket in real time. That's what you saw in the demo.

The backend is Node.js on an Oracle Cloud VPS. It handles session management, the Recall.ai webhook ingestion, the Alert Fusion Engine, and SPRT statistical decisions. It routes incoming frames and audio to the AI service.

The AI service is Python FastAPI on a RunPod GPU — an RTX 4000 Ada. This runs CLIP ViT-L/14 for deepfake detection, EfficientNet-B2 for emotion, WavLM for voice authenticity, and DeBERTa for social engineering detection in transcripts. Processing takes 40 to 60 milliseconds per frame.

Why split them? The backend doesn't need a GPU. Paying GPU rates to run Node.js wastes money. RunPod is on-demand at 20 cents an hour — we only pay during active meetings. Total infrastructure cost for six months of development was under 50 dollars.

The Trust Score combines three signals: 45% visual, 35% audio, 20% behavioural. When the camera is off, it falls back to audio-only.

---

## Slide 5 — Key Technical Features (~1.5 minutes)

Six features that make detection reliable.

**SPRT Session Decisions.** We use Wald's Sequential Probability Ratio Test to accumulate evidence across frames. It doesn't guess on a single frame — it builds confidence. 95% statistical confidence in 2 to 8 frames. If the evidence is ambiguous, it says "undecided," which is the honest answer.

**Recall.ai Integration.** We originally built a 2,000-line Puppeteer bot. It had unsolvable problems — audio feedback, gallery view lock, speaker attribution failures. We replaced it with a 280-line Recall.ai adapter. Same interface, zero downstream changes, and all capture problems disappeared. Now we support Zoom, Meet, and Teams through one adapter.

**EMA Smoothing.** Exponential moving average with 15-second hold and decay prevents the trust score from flickering on brief silences or camera glitches.

**Live Transcription and AI Coaching.** DeBERTa analyzes transcripts in zero-shot NLI mode against six social engineering hypotheses — credential requests, urgency pressure, authority impersonation. Zero-shot scored 89% on Zoom transcripts versus 50% from a fine-tuned email classifier.

**Alert Fusion Engine.** Deduplicates low-confidence alerts, requires 3 consecutive frames before firing, applies 30-second cooldowns, and escalates multi-signal anomalies to critical severity.

**Global Decay Ticker.** When someone mutes or turns off their camera, the Trust Score correctly fades to zero instead of holding a stale number.

---

## Slide 6 — Results & Future Work (~1.5 minutes)

You saw the live demo. Here are the numbers behind it.

Real faces on 360p frames scored 85 to 92% — correctly authentic. DeepLiveCam deepfakes scored 22 to 38% — alerts triggered every time. Score separation is about 31 points on average. End-to-end latency was 85 to 107 milliseconds per frame. WavLM voice authenticity scored 65 to 75%. The system ran stable across 30-minute demo sessions.

For future work — five priorities. Higher resolution frames at 720p, which we expect will add 14 points of score separation. GAN and diffusion artifact detection for fully generated faces. Voice clone testing against ElevenLabs and Resemble.ai. Per-participant identity profiles with persistent face embeddings. And webhook integrations to push alerts directly to Slack or Teams SOC channels.

---

## Slide 7 — Live Demo Slide / Closing (~30 seconds)

You saw the demo at the start. The landing page is live at real-sync.app. The dashboard, the bot, the detection pipeline — it's all deployed and running.

RealSync joins your meeting, tells you in real time who's real and who's not, and gives you a full report when the meeting ends.

Thank you. We're happy to take questions.

---

## Q&A Cheat Sheet (not spoken)

- **GAN faces:** "Known limitation. Ensemble targets face-swap artifacts. StyleGAN scores 70-73%, too close to real for SPRT. Future work: zero-shot CLIP text prompts."
- **Accuracy:** "31-point separation on 360p. Real 85-92%, fake 22-38%. SPRT converges at 95% confidence in 2-8 frames."
- **Cost:** "Under $50 total for 6 months. GPU $0.20/hr on-demand. Backend free. Frontend free."
- **Privacy:** "Bot joins visibly. Only host sees data. Row-Level Security in Supabase. No raw video stored."
- **Why CLIP over CNN:** "EfficientNet-B4 failed on Zoom — H.264 compression destroys pixel artifacts. CLIP semantic features survive compression."
- **Puppeteer migration:** "2,072 lines replaced by 280. Solved beeping, gallery lock, attribution, Whisper hallucination."
- **Ensemble 7.6x improvement:** "CLIP alone: 0.028 separation on post-processed swaps. Ensemble (50% CLIP + 30% frequency + 20% boundary): 0.214 separation. 7.6x better."
- **If you don't know the answer:** "That's a great question — we haven't tested that scenario yet, but our approach would be..."

---

*Total script time: ~9 minutes (1 min buffer before Q&A)*
