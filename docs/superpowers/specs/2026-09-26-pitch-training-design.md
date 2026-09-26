# Pitch Training: design (spec 1 of 2)

**Status:** approved in brainstorming, awaiting spec review
**Date:** 2026-09-26
**Scope:** the shared core, mode A (rehearse prepared answers) and mode C (delivery practice).
Mode B (live mock interviewer) is spec 2.

## TL;DR

A native iPhone app for practising interview answers out loud. It plays a recruiter question,
records the spoken answer, transcribes it on the device and gives feedback in two layers:
delivery numbers computed on the phone (time, pace, fillers, pauses) and a content grade from
one Claude call (which planned points were hit, whether the closing line landed, which guard
rules were broken, up to three rephrasings). Content comes from the existing prep docs in
InterviewPreparationsBot through a new exporter. The practice loop runs hands-free with
earbuds and a locked screen, and the detail waits on screen. A 4-box Leitner scheduler decides
what comes back and when.

---

## 1. Problem

The prep pipeline (`me:prep`) turns per-company markdown into reading tabs and drill tapes.
That is passive practice: reading and listening. Nothing checks whether the answer comes out
of Nick's mouth the way the doc has it: the points in order, the closing line, inside the time,
without the phrases the doc says to avoid.

### Related projects

| Project | Relation to this app |
|---|---|
| InterviewPreparationsBot | Source of all content. Gains one new file, `tools/export_deck.py`. No existing file changes |
| interview-drill (OpenClaw, Telegram) | Daily improvised questions with a 1-5 score. Keeps running; it trains improvising, this app trains prepared answers |
| Interview Simulator | Spec for a live AI interviewer. Its settled decisions 1-4 feed spec 2 of this app. That repo is not changed |

## 2. Decisions

| # | Question | Decision | Reason |
|---|---|---|---|
| 1 | Which training loops | A (rehearse) and C (delivery) now; B (mock interviewer) in spec 2 | A and C share one request/response loop. B needs streaming voice in both directions |
| 2 | Where practice happens | Hands-free on the move, detail on screen afterwards | Reps happen while walking; reading feedback happens later |
| 3 | Platform | Native SwiftUI, iOS 26 and later | Background mic, lock-screen and earbud controls, on-device speech (`SpeechAnalyzer`) |
| 4 | Distribution | TestFlight | Nick has a paid Apple Developer account |
| 5 | Grader model | Claude Opus 5 (`claude-opus-5`), effort `low` | Best judgement on paraphrased points. One env var to change |
| 6 | Scheduling | 4-box Leitner plus a before-a-call mode | The horizon is days to a known call, not months |
| 7 | Privacy | Audio never leaves the phone. Decks are never committed to git and are served only behind a token | This repo is public, and prep docs hold personal details |

---

## 3. Architecture

```
InterviewPreparationsBot                   this repo
┌────────────────────────┐   ┌──────────────────────────────────────────┐
│ applications/*.md      │   │ grader/   Vercel, TypeScript             │
│ build/<slug>/*/segments│   │   GET  /api/decks/<slug>/…  deck + clips │
│          │             │   │   POST /api/grade           grade JSON   │
│ tools/export_deck.py ──┼──▶│   bearer token on every request          │
└────────────────────────┘   └───────────▲──────────────────▲───────────┘
                                          │ deck, clips      │ transcript only
                             ┌────────────┴──────────────────┴───────────┐
                             │ ios/   SwiftUI app                        │
                             │  DeckStore  RepEngine  Transcriber        │
                             │  DeliveryAnalyzer  GraderClient  Speaker  │
                             │  Scheduler  History                       │
                             └───────────────────────────────────────────┘
```

### Units

| Unit | Where | Responsibility | Depends on |
|---|---|---|---|
| `export_deck.py` | InterviewPreparationsBot `tools/` | Prep docs and cached audio segments → deck JSON and m4a clips | `drill_tape.parse()`, ffmpeg, ffprobe |
| `api/grade` | grader | Card and transcript → verified grade JSON | Anthropic TypeScript SDK, `lib/verify` |
| `api/decks` | grader | Serves deck files behind the token | bundled `decks/` folder |
| `lib/verify` | grader | Checks every quote the model returns against the source text | nothing |
| `DeckStore` | iOS | Downloads decks when `version` changes, caches them for offline use | grader HTTP |
| `RepEngine` | iOS | State machine for one attempt. Owns the audio session, playback, recording, end-of-answer detection, remote commands | Transcriber, DeliveryAnalyzer, GraderClient, Speaker |
| `Transcriber` | iOS | Audio → words with timings, on device, with custom vocabulary | `SpeechAnalyzer` / `SpeechTranscriber` |
| `DeliveryAnalyzer` | iOS | Pure function: words and target → delivery metrics | nothing |
| `GraderClient` | iOS | Sends grade requests, queues them while offline | grader HTTP |
| `Speaker` | iOS | Speaks verdicts with the system voice | `AVSpeechSynthesizer` |
| `Scheduler` | iOS | Pure logic: card states, attempts, clock, call time → next session, box moves | nothing |
| `History` | iOS | Stores attempts and card states, deletes old audio | SwiftData |

Each unit can be tested alone: `DeliveryAnalyzer`, `Scheduler` and `lib/verify` are pure
functions; the grader is testable with curl before the app exists; the exporter is a CLI.

### Repository layout

```
ios/                       Xcode project, SwiftUI app
grader/
  api/grade.ts
  api/decks/[...path].ts
  lib/verify.ts  lib/prompt.ts  lib/schema.ts
  decks/                   gitignored; written by the exporter, deployed with the grader
  test/fixtures/
docs/superpowers/specs/
```

---

## 4. Content: the deck

### 4.1 Exporter

Run from the InterviewPreparationsBot root:

```bash
python3 tools/export_deck.py <slug> --out "<this repo>/grader/decks/<slug>"
```

**Inputs**
- `applications/<slug>-listen-config.json`, field `tapes[]`: each entry gives an answers-style
  `doc` and its audio `dir` (with `segments/`). This file is already written by `new_prep.py`.
- `applications/<slug>-hr-bank.md` and `applications/<slug>-intros.md`, when present
  (naming convention from `new_prep.py`).

**Parsing**
- Spoken fields (question, Say, Long version, follow-ups) come from `drill_tape.parse()`, so the
  practised text is exactly the text the audio was generated from.
- Coaching fields (Spine, Last line, If you lose the thread, Guard) come from a second pass over
  the same `### N.` sections. `parse()` stops at those labels on purpose (`STOP_BOLD`), because
  they are not spoken on the tape.
- Spine beats are the Spine text split on `→`.
- Talk cards come from each `##` section of the intros doc that contains a blockquote. The
  target comes from the heading ("Thirty seconds" → 30, "Ninety seconds" → 90,
  "Ten minutes" → 600). The Guard comes from the section's `**Guard.**` paragraph.
- A section containing ⚠️ gets `has_gaps: true`.
- An HR-bank question whose normalised text matches an answers-doc question merges into that
  answer card instead of creating a duplicate.
- Cards under the HR-bank section that records questions from a real call get
  `asked_for_real: true`. The exact markup of that section is confirmed against the current
  HR-bank files during planning (see §12).

**Clips** (no new text-to-speech)
- The segment cache is content-addressed: `segments/q{n:02d}_{h}.wav`, where
  `h = sha1(f"{model}|{voice}|{style}|{text}")[:12]`. Voices and styles are imported from
  `drill_tape.py`, which reads the voices from `TAPE_RECRUITER_VOICE` and `TAPE_NICK_VOICE`.
  The model is a CLI flag in `drill_tape.py`, so the exporter takes the same
  `--model` flag with the same default (`gemini-2.5-flash-preview-tts`). A tape made with
  other settings needs the same flag and env vars at export time.
- `prompt_clip`: the recruiter segment with text `Question {n}. {question}`. It says the doc
  number out loud, so the practice screen shows that same number (for example "Q14").
- `reference_clip`: the Say segment in Nick's voice.
- A missing segment sets the field to `null` and prints a warning. The phone then reads the
  question with the system voice and offers no reference clip.
- Every clip is converted to m4a and checked with ffprobe for a duration above 0.5 s.

**Version:** `version` is the first 12 hex characters of a sha1 over the canonical cards JSON.
The phone downloads only when it changes.

### 4.2 Deck JSON

Sample values are placeholders.

```json
{
  "slug": "acme",
  "title": "Acme",
  "version": "3f9c2a1b7d04",
  "exported_at": "2026-09-26T18:00:00Z",
  "vocabulary": ["Acme"],
  "cards": [
    {
      "id": "answers-full#1",
      "kind": "answer",
      "doc_number": 1,
      "question": "Where are you based?",
      "prompt_clip": "clips/answers-full-01-q.m4a",
      "say": "…",
      "say_hash": "a41c09e2",
      "say_words": 58,
      "spine": ["City", "Staying", "Work authorisation", "Hybrid is easy"],
      "last_line": "…",
      "rescue": "…",
      "guard": "…",
      "follow_ups": [{ "label": "Follow-up", "text": "…" }],
      "reference_clip": "clips/answers-full-01-say.m4a",
      "reference": null,
      "target_seconds": null,
      "vocabulary": ["…"],
      "has_gaps": false,
      "asked_for_real": false
    },
    {
      "id": "intros#short",
      "kind": "talk",
      "doc_number": null,
      "question": "Your short intro",
      "prompt_clip": null,
      "reference": "…",
      "guard": "…",
      "target_seconds": 30,
      "vocabulary": ["…"],
      "has_gaps": false,
      "asked_for_real": false
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `kind` | `answer` (mode A, has a script) or `talk` (mode C) |
| `say_hash` | Stored on every attempt, so edited answers show "practised an older version" instead of inheriting mastery |
| `target_seconds` | Set for talk cards. `null` for answer cards: the phone computes `say_words ÷ pace` |
| `vocabulary` | Deck-level names plus capitalised words in Say that do not start a sentence. Passed to the transcriber |

**Target pace** is 150 words per minute until 10 attempts exist, then the median pace across
Nick's attempts.

**Freestyle** (mode C without a deck): Nick types a topic and picks a length (30 s, 90 s,
2 min, 5 min). Freestyle attempts are stored but never scheduled.

---

## 5. The practice loop

One attempt runs through four states. Earbud presses map to the iOS media commands that
AirPods send (1 press = play/pause, 2 = next track, 3 = previous track).

| State | What happens | 1 press | 2 presses | 3 presses |
|---|---|---|---|---|
| Asking | Prompt clip plays (or the system voice reads the question). Question text and doc number appear | Skip to Listening | | |
| Listening | A tone, then the mic opens. A time bar fills toward the target and turns amber past it. No live transcript | End the answer | | |
| Processing | The delivery half of the verdict is spoken at once ("Thirty-four seconds, nine over"). Grade requested | | | |
| Verdict | The grader's one-sentence verdict is spoken | Try again | Next card | Hear the target (reference clip) |

- **End of answer:** 4.0 s of silence after the first word (setting, 2-8 s), or one press.
- **No answer:** no speech within 10 s of the tone ends the attempt as a miss and speaks the
  rescue line.
- **Stalled or finished:** an attempt ended by silence with `last_line: "missing"` counts as a
  stall, and the rescue line is spoken after the verdict.
- **Spine:** hidden by default. Peeking on screen marks the attempt `assisted`.
- **Slow grade:** if the grade has not arrived 8 s after the answer ends, the phone says
  "Content grade later" and continues. The grade fills in when it arrives.
- **Audio session:** one `playAndRecord` session stays active for the whole practice session,
  so a new attempt can start from a lock-screen or earbud command (see §12, item 4).
- **Interruptions** (phone call, Siri): the partial attempt is discarded, not graded, and the
  session resumes on the same card.

---

## 6. Feedback

### 6.1 Delivery (on the phone)

| Metric | Definition |
|---|---|
| `duration_s` | First word start to last word end |
| `over_target` | `duration_s ÷ target` |
| `pace_wpm` | Word count ÷ duration in minutes |
| `fillers` | Count of: um, uh, er, erm, "you know", "I mean", "kind of", "sort of", "basically". "Like" is left out in v1 because word matching cannot separate it from real uses |
| `pauses` | Gaps above 1.5 s between words: count and longest |

### 6.2 Content grade (grader)

**Request**

```
POST /api/grade
Authorization: Bearer <APP_TOKEN>
```

```json
{
  "card": {
    "kind": "answer",
    "question": "…",
    "say": "…",
    "spine": ["…"],
    "last_line": "…",
    "guard": "…",
    "reference": null
  },
  "transcript": "…",
  "duration_s": 34.2,
  "target_s": 25
}
```

Talk cards send `reference` and no `say`, `spine` or `last_line`. Freestyle sends the topic as
`question` and nothing else.

**Response 200**

```json
{
  "verdict": "Three of four beats. You skipped work authorisation. Last line landed.",
  "beats": [{ "beat": "…", "hit": true, "evidence": "…", "verified": true }],
  "missing_points": [{ "point": "…", "verified": true }],
  "last_line": "landed",
  "guard": [{ "rule": "…", "violated": false, "evidence": null, "verified": true }],
  "rephrases": [{ "original": "…", "better": "…", "verified": true }],
  "model": "claude-opus-5"
}
```

| Field | Rule |
|---|---|
| `verdict` | 20 words or fewer, plain spoken English. The only field that is spoken |
| `beats` | Answer cards only. Each hit needs an `evidence` quote from the transcript |
| `missing_points` | Talk cards with a reference: up to 3 points from the reference that were left out, quoted from the reference |
| `last_line` | `landed`, `paraphrased`, `missing`, or `null` when the card has none |
| `guard` | One entry per guard rule. A violation needs an `evidence` quote |
| `rephrases` | At most 3. `original` is Nick's exact words. `better` reuses the card's Say wording where it can, and appears only where the change affects how the answer comes across |

There is no overall score. In mode A the beat count is the score. In mode C progress shows in
the delivery numbers over time.

**Verification** (`lib/verify.ts`): text is lowercased, punctuation is stripped and whitespace
collapsed. Each `evidence` and `original` must then be a substring of the transcript, and each
`point` a substring of the reference. A beat or guard entry that fails keeps the model's value
but gets `verified: false`, and the phone treats an unverified hit as a miss. A rephrase that
fails is dropped.

**Model call**
- Anthropic TypeScript SDK, model from env `GRADER_MODEL` (default `claude-opus-5`).
- `output_config: { effort: "low", format: <JSON schema from lib/schema.ts> }`. Adaptive
  thinking stays on (the Opus 5 default). Non-streaming, `max_tokens: 8000`.
- The server-side refusal fallback is enabled (`fallbacks: "default"` with the
  `server-side-fallback-2026-07-01` beta). `stop_reason` is checked before reading content.
  A refusal or output that fails the schema returns `502 { "error": "ungraded" }`.
- The system prompt says: the transcript comes from speech recognition, so ignore misheard
  names; a beat counts as hit when its meaning is present, even when paraphrased; keep
  rephrases in Nick's voice.
- Expected size: about 2.5K input and 400 output tokens, roughly $0.02-0.03 per attempt.

**Auth and spend**
- One bearer token (env `APP_TOKEN`) is compiled into the app. Every endpoint returns 401
  without it.
- The Anthropic key lives in a dedicated workspace with a monthly spend limit. This replaces
  the "300 calls a day" cap from brainstorming: a daily counter needs a database, and the
  workspace limit gives the same protection with no extra service.
- If the token leaks: rotate `APP_TOKEN` in Vercel and ship a new TestFlight build.

**Decks endpoint:** `GET /api/decks/<slug>/deck.json` and `GET /api/decks/<slug>/clips/<file>`,
token required. Files come from the gitignored `grader/decks/` folder, bundled into the
function and deployed with `vercel deploy --prod` from the local checkout.

---

## 7. Scheduler

| Box | Comes back after |
|---|---|
| 1 | 1 day |
| 2 | 2 days |
| 3 | 4 days |
| 4 (mastered) | 10 days |

- **Box moves:** only the first graded attempt per card per calendar day moves it. Clean → up
  one box (maximum 4, or 3 for cards with `has_gaps`). Not clean → box 1. Later same-day
  attempts are practice and move nothing, so cramming one card five times does not fake
  mastery. Ungraded attempts never move a box. A grade that arrives late (offline attempt)
  applies on arrival, dated to the attempt.
- **Clean attempt:** `isClean(attempt, card) -> Bool`. Nick writes this function during
  implementation, because it sets his own bar for "ready". Starting proposal for answer cards:
  every beat hit and verified, last line landed or paraphrased, no guard violated,
  `over_target ≤ 1.25`, not assisted. For talk cards: no guard violated, `over_target ≤ 1.25`,
  not assisted. Missing points are shown but do not block, because a long reference always has
  more points than one telling covers.
- **New cards** enter box 1 when first practised. At most 3 new cards per session.
- **Session:** due cards ordered by box (lowest first), then oldest due date, then new cards.
  The session stops at 8 cards or 10 minutes. A missed card comes back once, three cards later.
- **Before-a-call mode:** set `callAt` and due dates are ignored. Order: `asked_for_real` cards,
  then talk cards with a target of 90 s or less, then lowest box, then the lowest share of
  beats hit on the last attempt.
  The mode switches off at `callAt`.
- **Ready for the call:** cards in box 4 ÷ cards without gaps, shown per deck.

---

## 8. History

Stored on the phone with SwiftData. No cloud sync in v1; the normal iPhone backup covers it.

- **Attempt:** id, card id, deck slug, `say_hash`, kind (answer, talk, freestyle), started at,
  duration, transcript, word timings, delivery metrics, grade JSON, grade status (pending,
  graded, ungraded), assisted, starred, audio file path.
- **CardState:** card id, deck slug, box, due at, last attempt at.
- **Retention:** audio is deleted after 14 days unless the attempt is starred. Text, metrics and
  grades are kept.

---

## 9. Screens

| Screen | Content |
|---|---|
| Practice | The four states from §5 |
| Feedback | Spine checklist, delivery tiles, guard result, rephrases, the previous attempt's result, buttons: Try again, Hear target, Next |
| Decks | Deck list, ready-for-the-call number, call date, start session |
| Freestyle | Topic field, length picker, start |
| Settings | Silence cut-off, speech model status |

UI work follows the standing rule: run `me:unslop` before building screens and again before
calling them done.

---

## 10. Failure handling

| Failure | Behaviour |
|---|---|
| No signal or grader down | Delivery verdict still spoken. The attempt saves as `pending` and grades when back online |
| Grader returns `ungraded` | Stored as ungraded, retried once, never counted for box moves |
| Phone call or Siri during an answer | Partial attempt discarded, same card resumes |
| Deck download fails | Cached deck keeps working |
| Speech model not yet on the device | First launch downloads it with a progress bar; practice waits for it |
| Microphone permission denied | Explanation screen with a link to Settings; practice cannot start |
| Token leaks | Rotate `APP_TOKEN`, ship a build. The workspace spend limit caps the damage meanwhile |

---

## 11. Testing

| Unit | Test |
|---|---|
| `DeliveryAnalyzer` | Unit tests on fixture word lists: pace, fillers, pauses, over-target |
| `Scheduler` | Unit tests: box moves, first-attempt-per-day rule, gap cap, session order, before-a-call order |
| `lib/verify` | Unit tests: exact quote, punctuation and case differences, invented quote rejected |
| Exporter | Run on the first real deck: card counts match the docs, every non-null clip exists and lasts over 0.5 s, no duplicate ids |
| Grader | About 10 fixtures: the card's own Say text hits every beat; an off-topic answer hits none; a paraphrase still hits; a guard phrase is flagged. Costs about $0.30 per run, and is run only with Nick's go-ahead |
| App | The full loop in the iOS simulator, fed recorded audio files instead of the mic |
| On Nick's iPhone | A short checklist the simulator cannot cover: earbud presses 1, 2 and 3 in Verdict; an attempt started and finished with the screen locked; a phone call mid-answer; an offline attempt graded later; prompt clips clear over AirPods |

---

## 12. Risks to check first

The implementation plan starts with a short spike on items 1-4, because each can change the
design.

1. **Fillers:** `SpeechTranscriber` may drop "um" and "uh". If it does, the filler count shows
   "n/a" and the plan adds a fallback.
2. **Custom vocabulary:** confirm how `SpeechAnalyzer` accepts contextual strings on iOS 26.
3. **AirPods microphone:** recording through AirPods uses the Bluetooth hands-free profile,
   which lowers playback quality while the session is active. Confirm prompt clips stay clear.
4. **Starting from the lock screen:** confirm that with one audio session kept active, an
   earbud press can start a new recording while the screen is locked.
5. **Deck upload:** confirm `vercel deploy` uploads the gitignored `grader/decks/` folder
   (using `.vercelignore`) and that the function can read it.
6. **Real-question markup:** confirm how the HR-bank docs mark questions from a real call.

---

## 13. Spec 2: mode B

Spec 2 designs the live mock interviewer on top of this core: it reuses decks, history and the
grader contract, adds a conversation-level grade, and feeds the cards a mock call asked back
into the scheduler. It starts from the Interview Simulator spec's settled decisions 1-4 and its
open questions 4-8.

## Out of scope for spec 1

Mode B, Android, cloud sync, multiple users, editing cards in the app (edit the markdown and
re-export), a web version.
