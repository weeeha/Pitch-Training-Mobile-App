# Pitch Training Mobile App

**Problem.** Interview prep that stops at reading and listening never checks whether the spoken answer hits its points in order, lands the closing line, fits the time and avoids the phrases marked for avoidance.

**Idea.** The app plays a recruiter question, records the spoken answer and transcribes it on the device. Feedback comes in two layers: delivery numbers computed on the phone, and a content grade from one Claude call.

**Built.** Spec only so far: a design spec and two implementation plans (an iOS risk spike, and a deck exporter plus grader service), with no app code. The planned stack is SwiftUI on iOS 26 and later, with a TypeScript grader on Vercel. 5 commits on 3 days, Sep 26 to Oct 12, 2026.

**Decision.** Rehearsing prepared answers and delivery practice come first, and the live mock interviewer waits for a second spec. The first two share one request and response loop, while the interviewer needs streaming voice in both directions.

| | |
|---|---|
| **What it is** | A native iPhone app for practising interview answers out loud. Design and plans only so far |
| **Status** | exploration (spec approved 2026-09-26, no app code yet) |
| **Live URL** | none yet |

The app plays a recruiter question, records the spoken answer, transcribes it on the device and gives feedback in two layers: delivery numbers computed on the phone (time, pace, fillers, pauses) and a content grade from one Claude call (planned points hit, closing line, guard rules broken, up to three rephrasings). Audio stays on the phone. The loop works hands-free with earbuds and a locked screen, and a 4-box Leitner scheduler decides which cards come back.

## Planned stack

- iOS app: SwiftUI, iOS 26 and later, on-device speech with `SpeechAnalyzer`, SwiftData history, TestFlight distribution
- Grader: Vercel project in TypeScript with `POST /api/grade` and token-protected deck endpoints
- Deck exporter: a Python script that turns existing interview prep markdown into deck JSON and audio clips

## What is in the repo

This repo has documents only. There is nothing to build or run yet.

- [`docs/superpowers/specs/2026-09-26-pitch-training-design.md`](docs/superpowers/specs/2026-09-26-pitch-training-design.md): design spec 1 of 2 (rehearse prepared answers and delivery practice; the live mock interviewer is spec 2)
- [`docs/superpowers/plans/2026-09-26-plan-1-ios-risk-spike.md`](docs/superpowers/plans/2026-09-26-plan-1-ios-risk-spike.md): a throwaway iOS spike that answers the spec's first four risks
- [`docs/superpowers/plans/2026-09-26-plan-2-exporter-and-grader.md`](docs/superpowers/plans/2026-09-26-plan-2-exporter-and-grader.md): the deck exporter and the grader service
