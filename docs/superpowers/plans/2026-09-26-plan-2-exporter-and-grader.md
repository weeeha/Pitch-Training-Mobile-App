# Exporter and Grader Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Export a company's prep docs as a practice deck, serve it behind a token, and grade a spoken-answer transcript with quotes the server has checked.

**Architecture:** The exporter is one Python module in InterviewPreparationsBot that reuses `drill_tape.parse()` and the existing TTS segment cache; it is pure functions plus a CLI, tested with pytest on invented fixtures. The grader is a Vercel project in `grader/` (TypeScript, Node runtime): pure modules in `lib/` (quote check, auth, schemas, prompt, grading, handlers) tested with vitest, and two thin endpoints in `api/`.

**Tech Stack:** Python 3.14, pytest 9, ffmpeg/ffprobe; Node 26, TypeScript, vitest, zod, `@anthropic-ai/sdk`, `@vercel/node`, Vercel CLI.

**Spec:** `docs/superpowers/specs/2026-09-26-pitch-training-design.md` (§3 architecture, §4 deck, §6.2 grader, §10 failures, §11 testing, §12 items 5-6)

## Global Constraints

- **Two repos.** Exporter: `/Users/nickv/ClaudeCode Projects/InterviewPreparationsBot` (local git, no remote), branch `feat/export-deck`. Grader: `/Users/nickv/ClaudeCode Projects/Pitch Training Mobile` (public GitHub repo `weeeha/Pitch-Training-Mobile-App`), branch `feat/exporter-grader` cut from `spec/pitch-training-design`. State the repo and path before every commit, push or deploy. Never push to `main`.
- **This repo is public.** Fixtures and test data are invented. Never commit a deck, a real prep answer, a company name from the prep docs, or a recruiter's name. `grader/decks/` is gitignored.
- **The exporter never calls TTS.** Clips come only from existing segment files.
- **Exporter commands run from the InterviewPreparationsBot root**, because the listen config holds repo-relative paths.
- **Grader model:** env `GRADER_MODEL`, default `claude-opus-5`; `output_config.effort: "low"`; `max_tokens: 8000`; non-streaming; `fallbacks: "default"` with beta `server-side-fallback-2026-07-01`; check `stop_reason` before reading content.
- **Secrets:** the executor never reads, types or stores `ANTHROPIC_API_KEY` or `APP_TOKEN`. Nick creates and sets them (Task 12).
- **Grader HTTP contract** is exactly spec §6.2: `POST /api/grade`, `GET /api/decks/<slug>/deck.json`, `GET /api/decks/<slug>/clips/<file>`, bearer token on all.
- **Paid API calls** (Task 11) run only after Nick says go.

---

### Task 1: Exporter: section splitter and coaching fields

**Files:**
- Create: `InterviewPreparationsBot/tools/export_deck.py`
- Create: `InterviewPreparationsBot/tools/test_export_deck.py`
- Create: `InterviewPreparationsBot/tools/test_fixtures/applications/acme-answers-full.md`

**Interfaces:**
- Consumes: `drill_tape.clean(md) -> str`.
- Produces: `sections_raw(md_path) -> dict[int, list[str]]` (every `### N.` section's lines, heading first); `coaching(lines) -> {"spine": list[str], "last_line": str|None, "rescue": str|None, "guard": str|None, "has_gaps": bool}`; `DEFAULT_MODEL`; `warn(msg)`.

- [ ] **Step 1: Create the branch**

```bash
git -C "/Users/nickv/ClaudeCode Projects/InterviewPreparationsBot" switch -c feat/export-deck
```

- [ ] **Step 2: Write the fixture**

`tools/test_fixtures/applications/acme-answers-full.md` (invented content):

```markdown
# Acme answers

## Logistics

### 1. "Where are you based?"

**Spine:** Lisbon → staying → hybrid easy.

**Say:** "Yes, Lisbon, and I'm staying. I moved here from Porto as a choice. Hybrid is easy."

**Last line:** "...Hybrid is easy."

**If you lose the thread:** "Short version: Lisbon, staying."

**Guard:** never name a salary. Do not say
"relocation" at all.

**Follow-up (if she asks about travel):** "Travel is fine, up to a week a month."

---

### 2. "Why this role?" ⚠️

**Spine:** product fit → team.

**Say:** "Because the product is the kind I have built at Zentrix. ⚠️ team detail missing."

**Last line:** "...the kind I have built."

---
```

- [ ] **Step 3: Write the failing tests**

`tools/test_export_deck.py`:

```python
import hashlib, json, shutil, sys, wave
from pathlib import Path

HERE = Path(__file__).resolve().parent
sys.path.insert(0, str(HERE))
import export_deck as ed  # noqa: E402

FIX = HERE / "test_fixtures" / "applications"
ANSWERS = FIX / "acme-answers-full.md"
INTROS = FIX / "acme-intros.md"
HR = FIX / "acme-hr-bank.md"
MODEL = "test-model"


def make_wav(path, seconds):
    path.parent.mkdir(parents=True, exist_ok=True)
    with wave.open(str(path), "wb") as w:
        w.setnchannels(1); w.setsampwidth(2); w.setframerate(24000)
        w.writeframes(b"\x00\x00" * int(24000 * seconds))


def test_sections_raw_splits_numbered_sections():
    raw = ed.sections_raw(ANSWERS)
    assert sorted(raw) == [1, 2]
    assert raw[1][0].startswith("### 1.")


def test_coaching_reads_the_fields_parse_skips():
    c = ed.coaching(ed.sections_raw(ANSWERS)[1])
    assert c["spine"] == ["Lisbon", "staying", "hybrid easy"]
    assert c["last_line"] == "Hybrid is easy."
    assert c["rescue"] == "Short version: Lisbon, staying."
    assert c["guard"] == 'never name a salary. Do not say "relocation" at all.'
    assert c["has_gaps"] is False


def test_coaching_flags_gaps():
    assert ed.coaching(ed.sections_raw(ANSWERS)[2])["has_gaps"] is True
```

- [ ] **Step 4: Run to verify failure**

Run: `cd "/Users/nickv/ClaudeCode Projects/InterviewPreparationsBot" && python3 -m pytest tools/test_export_deck.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'export_deck'`.

- [ ] **Step 5: Write the module header and the two functions**

`tools/export_deck.py`:

```python
"""Export one company's prep docs as a practice deck for the Pitch Training app.

Usage: python3 tools/export_deck.py <slug> --out <dir> [--model <tts model>] [--title <name>] [--root <repo root>]

Reads applications/<slug>-listen-config.json (its tapes), <slug>-hr-bank.md and <slug>-intros.md.
Writes <out>/deck.json and <out>/clips/*.m4a. Never generates speech: clips come from the
drill_tape segment cache, found by the same content hash drill_tape used to name them.
"""
import argparse, datetime, hashlib, json, re, shutil, subprocess, sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
from drill_tape import (NICK_STYLE, NICK_VOICE, RECRUITER_STYLE, RECRUITER_VOICE,  # noqa: E402
                        clean, parse, to_m4a)

DEFAULT_MODEL = "gemini-2.5-flash-preview-tts"  # drill_tape.py's default for --model

SECTION_RE = re.compile(r"^### (\d+)\. ")
COACH_RE = re.compile(r"^\*\*(Spine|Last line|If you lose the thread|Guard)[:.]\*\*:?\s*(.*)$")
COACH_KEYS = {"Spine": "spine", "Last line": "last_line", "If you lose the thread": "rescue", "Guard": "guard"}


def warn(msg):
    print(f"warning: {msg}", file=sys.stderr)


def sections_raw(md_path):
    """Lines of every '### N.' section, heading first. A '#' or '##' heading ends a section."""
    out, cur = {}, None
    for ln in Path(md_path).read_text(encoding="utf-8").splitlines():
        m = SECTION_RE.match(ln)
        if m:
            cur = int(m.group(1))
            out[cur] = [ln]
            continue
        if ln.startswith(("# ", "## ")):
            cur = None
            continue
        if cur is not None:
            out[cur].append(ln)
    return out


def _line(s):
    """A quoted one-liner with its quotes and leading '...' removed."""
    if not s:
        return None
    return clean(s).lstrip(".… ").strip() or None


def coaching(lines):
    """Spine, last line, rescue line and guard. drill_tape.parse() stops at these on purpose."""
    raw, key, buf = {}, None, []

    def flush():
        nonlocal key, buf
        if key:
            raw[key] = " ".join(buf).strip()
        key, buf = None, []

    for ln in lines[1:]:
        m = COACH_RE.match(ln)
        if m:
            flush()
            key, buf = COACH_KEYS[m.group(1)], [m.group(2)]
        elif key and (not ln.strip() or ln.startswith(("**", "---", "#", "{{"))):
            flush()
        elif key:
            buf.append(ln)
    flush()
    spine = [clean(b).rstrip(".").strip() for b in raw.get("spine", "").split("→")]
    return {
        "spine": [b for b in spine if b],
        "last_line": _line(raw.get("last_line")),
        "rescue": _line(raw.get("rescue")),
        "guard": clean(raw["guard"]) if raw.get("guard") else None,
        "has_gaps": any("⚠️" in ln for ln in lines),
    }
```

- [ ] **Step 6: Run to verify pass**

Run: `python3 -m pytest tools/test_export_deck.py -v`
Expected: 3 passed.

- [ ] **Step 7: Commit** (repo: InterviewPreparationsBot)

```bash
git add tools/export_deck.py tools/test_export_deck.py tools/test_fixtures/applications/acme-answers-full.md
git commit -m "export_deck: section splitter and coaching fields"
```

---

### Task 2: Exporter: answer cards, vocabulary and clips

**Files:**
- Modify: `InterviewPreparationsBot/tools/export_deck.py` (append)
- Modify: `InterviewPreparationsBot/tools/test_export_deck.py` (append)

**Interfaces:**
- Consumes: `sections_raw`, `coaching`, `warn` (Task 1); `drill_tape.parse(md_path) -> [{"n", "q", "blocks": [(label, text)]}]` where label `""` is the Say block; `drill_tape.to_m4a(wav, m4a)`.
- Produces: `vocabulary(text) -> list[str]`; `say_hash(text) -> str` (8 hex); `segment_path(segments_dir, n, model, voice, style, text) -> Path`; `export_clip(wav, clips_dir, name) -> "clips/<name>.m4a" | None`; `answer_cards(doc_path, doc_id, segments_dir, clips_dir, model) -> list[card]`; `SCRIPT_FREE_TARGET_S = 60`.
- Card shapes (spec §4.2): answer cards carry `id, kind="answer", doc_number, question, prompt_clip, say, say_hash, say_words, spine, last_line, rescue, guard, follow_ups, reference_clip, reference=None, target_seconds=None, vocabulary, has_gaps, asked_for_real`. A section with no Say becomes `kind="talk"` with `reference=None, target_seconds=60`.

- [ ] **Step 1: Write the failing tests** (append to `tools/test_export_deck.py`)

```python
SAY_1 = "Yes, Lisbon, and I'm staying. I moved here from Porto as a choice. Hybrid is easy."


def test_vocabulary_skips_sentence_starts_and_I():
    assert ed.vocabulary(SAY_1) == ["Lisbon", "Porto"]


def test_segment_path_matches_drill_tape_naming(tmp_path):
    h = hashlib.sha1("m|v|s|Question 3. Hi?".encode()).hexdigest()[:12]
    assert ed.segment_path(tmp_path, 3, "m", "v", "s", "Question 3. Hi?") == tmp_path / f"q03_{h}.wav"


def test_answer_cards_with_clips(tmp_path):
    seg = tmp_path / "segments"
    make_wav(ed.segment_path(seg, 1, MODEL, ed.RECRUITER_VOICE, ed.RECRUITER_STYLE,
                             "Question 1. Where are you based?"), 1.0)
    make_wav(ed.segment_path(seg, 1, MODEL, ed.NICK_VOICE, ed.NICK_STYLE, SAY_1), 2.0)
    cards = ed.answer_cards(ANSWERS, "answers-full", seg, tmp_path / "clips", MODEL)
    c1 = cards[0]
    assert c1["id"] == "answers-full#1" and c1["kind"] == "answer" and c1["doc_number"] == 1
    assert c1["question"] == "Where are you based?"
    assert c1["say"] == SAY_1 and c1["say_words"] == 16 and len(c1["say_hash"]) == 8
    assert c1["prompt_clip"] == "clips/answers-full-01-q.m4a"
    assert c1["reference_clip"] == "clips/answers-full-01-say.m4a"
    assert (tmp_path / "clips" / "answers-full-01-q.m4a").exists()
    assert c1["follow_ups"] == [{"label": "Follow-up (if she asks about travel)",
                                 "text": "Travel is fine, up to a week a month."}]
    assert c1["vocabulary"] == ["Lisbon", "Porto"]
    assert c1["asked_for_real"] is False
    assert cards[1]["prompt_clip"] is None and cards[1]["has_gaps"] is True


def test_short_clip_is_dropped(tmp_path):
    seg = tmp_path / "segments"
    make_wav(ed.segment_path(seg, 1, MODEL, ed.RECRUITER_VOICE, ed.RECRUITER_STYLE,
                             "Question 1. Where are you based?"), 0.1)
    cards = ed.answer_cards(ANSWERS, "answers-full", seg, tmp_path / "clips", MODEL)
    assert cards[0]["prompt_clip"] is None


def test_answer_cards_without_audio():
    cards = ed.answer_cards(ANSWERS, "answers-full", None, None, MODEL)
    assert cards[0]["prompt_clip"] is None and cards[0]["reference_clip"] is None
```

- [ ] **Step 2: Run to verify failure**

Run: `python3 -m pytest tools/test_export_deck.py -v`
Expected: the 5 new tests FAIL with `AttributeError: module 'export_deck' has no attribute ...`.

- [ ] **Step 3: Implement** (append to `tools/export_deck.py`)

```python
STOP_CAPS = {"I", "I'm", "I'd", "I've", "I'll", "OK"}
MIN_CLIP_S = 0.5
SCRIPT_FREE_TARGET_S = 60


def vocabulary(text):
    """Capitalised words that do not start a sentence: names the transcriber should expect."""
    out = []
    for sentence in re.split(r"(?<=[.!?])\s+", text or ""):
        for w in re.findall(r"[A-Za-z][\w'-]*", sentence)[1:]:
            if w[0].isupper() and w not in STOP_CAPS and w not in out:
                out.append(w)
    return out


def say_hash(text):
    return hashlib.sha1(text.encode()).hexdigest()[:8]


def segment_path(segments_dir, n, model, voice, style, text):
    """Where drill_tape.py cached this segment: q{n:02d}_{sha1(model|voice|style|text)[:12]}.wav."""
    h = hashlib.sha1(f"{model}|{voice}|{style}|{text}".encode()).hexdigest()[:12]
    return Path(segments_dir) / f"q{n:02d}_{h}.wav"


def probe_seconds(path):
    r = subprocess.run(["ffprobe", "-v", "error", "-show_entries", "format=duration", "-of", "csv=p=0", str(path)],
                       capture_output=True, text=True)
    try:
        return float(r.stdout.strip())
    except ValueError:
        return 0.0


def export_clip(wav, clips_dir, name):
    """Cached segment -> clips/<name>.m4a. None when the segment is missing or under MIN_CLIP_S."""
    if wav is None or not Path(wav).exists():
        return None
    Path(clips_dir).mkdir(parents=True, exist_ok=True)
    out = Path(clips_dir) / f"{name}.m4a"
    to_m4a(str(wav), str(out))
    if probe_seconds(out) < MIN_CLIP_S:
        out.unlink(missing_ok=True)
        warn(f"clip {name} is shorter than {MIN_CLIP_S}s, dropped")
        return None
    return f"clips/{name}.m4a"


def answer_cards(doc_path, doc_id, segments_dir, clips_dir, model):
    """One card per '### N.' section. Spoken text comes from drill_tape.parse() so it matches the audio."""
    raw = sections_raw(doc_path)
    cards = []
    for s in parse(str(doc_path)):
        n, q = s["n"], s["q"]
        say = next((t for lab, t in s["blocks"] if lab == ""), None)
        c = coaching(raw.get(n, []))
        q_wav = segment_path(segments_dir, n, model, RECRUITER_VOICE, RECRUITER_STYLE, f"Question {n}. {q}") \
            if segments_dir else None
        prompt = export_clip(q_wav, clips_dir, f"{doc_id}-{n:02d}-q")
        if segments_dir and prompt is None:
            warn(f"{doc_id}#{n}: no prompt clip in {segments_dir}")
        base = {"id": f"{doc_id}#{n}", "doc_number": n, "question": q, "prompt_clip": prompt,
                "guard": c["guard"], "has_gaps": c["has_gaps"], "asked_for_real": False}
        if say is None:
            cards.append({**base, "kind": "talk", "reference": None,
                          "target_seconds": SCRIPT_FREE_TARGET_S, "vocabulary": vocabulary(q)})
            continue
        say_wav = segment_path(segments_dir, n, model, NICK_VOICE, NICK_STYLE, say) if segments_dir else None
        cards.append({
            **base, "kind": "answer", "say": say, "say_hash": say_hash(say), "say_words": len(say.split()),
            "spine": c["spine"], "last_line": c["last_line"], "rescue": c["rescue"],
            "follow_ups": [{"label": lab, "text": t} for lab, t in s["blocks"]
                           if lab.lower().startswith(("follow-up", "if she asks"))],
            "reference_clip": export_clip(say_wav, clips_dir, f"{doc_id}-{n:02d}-say"),
            "reference": None, "target_seconds": None, "vocabulary": vocabulary(say),
        })
    return cards
```

- [ ] **Step 4: Run to verify pass**

Run: `python3 -m pytest tools/test_export_deck.py -v`
Expected: 8 passed.

- [ ] **Step 5: Commit** (repo: InterviewPreparationsBot)

```bash
git add tools/export_deck.py tools/test_export_deck.py
git commit -m "export_deck: answer cards with cached clips"
```

---

### Task 3: Exporter: talk cards from the intros doc

**Files:**
- Create: `InterviewPreparationsBot/tools/test_fixtures/applications/acme-intros.md`
- Modify: `tools/export_deck.py`, `tools/test_export_deck.py` (append)

**Interfaces:**
- Consumes: `clean`, `vocabulary`, `warn`.
- Produces: `talk_cards(intros_path) -> list[card]` with `id="intros#<name>", kind="talk", doc_number=None, question="Your <name> intro", prompt_clip=None, reference, guard, target_seconds, vocabulary, has_gaps, asked_for_real=False`; `slugify(s)`.

- [ ] **Step 1: Write the fixture**

`tools/test_fixtures/applications/acme-intros.md`:

```markdown
# Acme intros

**The frame.** Say one thing well.

---

## Short. Thirty seconds.

> I'm a product designer. I work on complex tools, and I make them
> coherent.

**Guard.** Full stop after "coherent." Do not add a pitch.

---

## Medium. Ninety seconds.

> Product designer, three chapters.
>
> First, a startup. Then, larger teams.

---

## Notes

No blockquote here, so no card.
```

- [ ] **Step 2: Write the failing test** (append)

```python
def test_talk_cards_from_intros():
    cards = ed.talk_cards(INTROS)
    assert [c["id"] for c in cards] == ["intros#short", "intros#medium"]
    short = cards[0]
    assert short["kind"] == "talk" and short["question"] == "Your short intro"
    assert short["target_seconds"] == 30
    assert short["reference"] == "I'm a product designer. I work on complex tools, and I make them coherent."
    assert short["guard"] == 'Full stop after "coherent." Do not add a pitch.'
    assert cards[1]["target_seconds"] == 90 and cards[1]["guard"] is None
```

- [ ] **Step 3: Run to verify failure**

Run: `python3 -m pytest tools/test_export_deck.py -v -k talk`
Expected: FAIL with `AttributeError: module 'export_deck' has no attribute 'talk_cards'`.

- [ ] **Step 4: Implement** (append)

```python
TARGETS = {"thirty seconds": 30, "ninety seconds": 90, "two minutes": 120, "five minutes": 300, "ten minutes": 600}
DEFAULT_TALK_TARGET_S = 90
GUARD_RE = re.compile(r"^\*\*Guard[.:]\*\*:?\s*")


def slugify(s):
    return re.sub(r"[^a-z0-9]+", "-", s.lower()).strip("-")


def talk_cards(intros_path):
    """One card per '##' section that holds a blockquote. The length comes from the heading."""
    cards, cur = [], None

    def finish():
        if not cur or not cur["quote"]:
            return
        name = cur["heading"].split(".")[0].strip()
        low = cur["heading"].lower()
        target = next((v for k, v in TARGETS.items() if k in low), None)
        if target is None:
            warn(f"no length in intro heading '{cur['heading']}', using {DEFAULT_TALK_TARGET_S}s")
            target = DEFAULT_TALK_TARGET_S
        ref = clean(" ".join(cur["quote"]))
        cards.append({"id": f"intros#{slugify(name)}", "kind": "talk", "doc_number": None,
                      "question": f"Your {name.lower()} intro", "prompt_clip": None, "reference": ref,
                      "guard": clean(" ".join(cur["guard"])) or None, "target_seconds": target,
                      "vocabulary": vocabulary(ref), "has_gaps": cur["gaps"], "asked_for_real": False})

    for ln in Path(intros_path).read_text(encoding="utf-8").splitlines():
        if ln.startswith("## "):
            finish()
            cur = {"heading": ln[3:].strip(), "quote": [], "guard": [], "in_guard": False, "gaps": "⚠️" in ln}
            continue
        if cur is None:
            continue
        if "⚠️" in ln:
            cur["gaps"] = True
        if ln.startswith(">"):
            cur["quote"].append(ln[1:].strip())
            cur["in_guard"] = False
        elif GUARD_RE.match(ln):
            cur["in_guard"] = True
            cur["guard"].append(GUARD_RE.sub("", ln))
        elif cur["in_guard"] and ln.strip() and not ln.startswith(("---", "#", "**")):
            cur["guard"].append(ln)
        else:
            cur["in_guard"] = False
    finish()
    return cards
```

- [ ] **Step 5: Run to verify pass**

Run: `python3 -m pytest tools/test_export_deck.py -v`
Expected: 9 passed.

- [ ] **Step 6: Commit** (repo: InterviewPreparationsBot)

```bash
git add tools/export_deck.py tools/test_export_deck.py tools/test_fixtures/applications/acme-intros.md
git commit -m "export_deck: talk cards from the intros doc"
```

---

### Task 4: Exporter: HR bank merges and real-question flags

Resolves spec §12 item 6: the HR bank marks a real call's questions as a numbered list `N. "question"` under a `##` heading that contains `REAL`, and pulls answers in with `{{include <doc>.md <n> <url>}}`.

**Files:**
- Create: `InterviewPreparationsBot/tools/test_fixtures/applications/acme-hr-bank.md`
- Modify: `tools/export_deck.py`, `tools/test_export_deck.py` (append)

**Interfaces:**
- Consumes: `sections_raw`, `answer_cards`, `vocabulary`, `SCRIPT_FREE_TARGET_S`.
- Produces: `doc_id(path, slug) -> str` (`acme-answers-full.md` → `answers-full`); `real_questions(md_path) -> list[str]`; `matches(real, question) -> bool`; `hr_bank_cards(md_path, existing) -> list[card]` which also sets `asked_for_real=True` on matching cards in `existing`. Unmatched real questions become `id="real#<i>"` talk cards.

- [ ] **Step 1: Write the fixture**

`tools/test_fixtures/applications/acme-hr-bank.md`:

```markdown
# Acme recruiter questions

## REAL questions from the screen (transcript 2026-01-01)

1. "Where are you based these days?"
2. "What would you change about our onboarding?"
   She followed up on this one.

## Logistics

### 1. "Where are you based?"

{{include acme-answers-full.md 1 /answers#q1}}

### 2. "Are you open to a background check?"

**Say:** "Yes, and references are ready."

**Guard:** do not volunteer past employers.

### 3. "What is your notice period?"

Answer in the moment.
```

- [ ] **Step 2: Write the failing tests** (append)

```python
def test_doc_id_strips_the_slug():
    assert ed.doc_id("applications/acme-answers-full.md", "acme") == "answers-full"
    assert ed.doc_id("notes.md", "acme") == "notes"


def test_matches_handles_alternatives_and_containment():
    assert ed.matches("Where are you based these days?", "Where are you based?")
    assert ed.matches("So you're here?", "Where are you located? Or, So you're here?")
    assert not ed.matches("Why?", "Why this role?")


def test_hr_bank_merges_includes_and_flags_real_questions():
    answers = ed.answer_cards(ANSWERS, "answers-full", None, None, MODEL)
    extra = ed.hr_bank_cards(HR, answers)
    assert [c["id"] for c in extra] == ["hr-bank#2", "hr-bank#3", "real#2"]
    assert answers[0]["asked_for_real"] is True
    assert extra[0]["kind"] == "answer" and extra[0]["prompt_clip"] is None
    assert extra[1]["kind"] == "talk" and extra[1]["target_seconds"] == 60
    assert extra[2]["question"] == "What would you change about our onboarding?"
    assert extra[2]["asked_for_real"] is True
```

- [ ] **Step 3: Run to verify failure**

Run: `python3 -m pytest tools/test_export_deck.py -v -k "doc_id or matches or hr_bank"`
Expected: 3 FAIL with `AttributeError`.

- [ ] **Step 4: Implement** (append)

```python
INCLUDE_RE = re.compile(r"\{\{include\s+(\S+)\s+(\d+)")
REAL_ITEM_RE = re.compile(r'^\d+\.\s+"(.+?)"')


def doc_id(path, slug):
    stem = Path(path).stem
    return stem[len(slug) + 1:] if stem.startswith(slug + "-") else stem


def norm(s):
    return re.sub(r"\s+", " ", re.sub(r"[^a-z0-9 ]", " ", s.lower())).strip()


def matches(real, question):
    """Same question, or one contains the other (at least 3 words). Alternatives are split on ' Or, '."""
    r = norm(real)
    for alt in question.split(" Or, "):
        a = norm(alt)
        if a == r:
            return True
        short, long_ = sorted((a, r), key=len)
        if len(short.split()) >= 3 and short in long_:
            return True
    return False


def real_questions(md_path):
    """Numbered '1. "..."' items under a '##' heading that contains REAL."""
    qs, inside = [], False
    for ln in Path(md_path).read_text(encoding="utf-8").splitlines():
        if ln.startswith("## "):
            inside = "REAL" in ln
            continue
        m = REAL_ITEM_RE.match(ln) if inside else None
        if m:
            qs.append(m.group(1))
    return qs


def hr_bank_cards(md_path, existing):
    """HR-bank cards that are not {{include}}s of an answer, plus real-question flags and cards."""
    included = {n for n, lines in sections_raw(md_path).items() if any(INCLUDE_RE.search(ln) for ln in lines)}
    own = [c for c in answer_cards(md_path, "hr-bank", None, None, None) if c["doc_number"] not in included]
    pool, extra = existing + own, []
    for i, q in enumerate(real_questions(md_path), start=1):
        hit = next((c for c in pool if matches(q, c["question"])), None)
        if hit:
            hit["asked_for_real"] = True
        else:
            extra.append({"id": f"real#{i}", "kind": "talk", "doc_number": None, "question": q,
                          "prompt_clip": None, "reference": None, "guard": None,
                          "target_seconds": SCRIPT_FREE_TARGET_S, "vocabulary": vocabulary(q),
                          "has_gaps": False, "asked_for_real": True})
    return own + extra
```

- [ ] **Step 5: Run to verify pass**

Run: `python3 -m pytest tools/test_export_deck.py -v`
Expected: 12 passed.

- [ ] **Step 6: Commit** (repo: InterviewPreparationsBot)

```bash
git add tools/export_deck.py tools/test_export_deck.py tools/test_fixtures/applications/acme-hr-bank.md
git commit -m "export_deck: HR bank merges and real-question flags"
```

---

### Task 5: Exporter: deck assembly and CLI

**Files:**
- Modify: `tools/export_deck.py`, `tools/test_export_deck.py` (append)

**Interfaces:**
- Consumes: everything above; `applications/<slug>-listen-config.json` with `tapes: [{"doc", "dir", ...}]`.
- Produces: `build_deck(root, slug, out_dir, model, title=None) -> deck dict` (writes `<out>/deck.json`, rebuilds `<out>/clips/`); `summary(deck) -> str`; `main(argv=None)`. Deck shape per spec §4.2: `slug, title, version (12 hex), exported_at, vocabulary, cards`.

- [ ] **Step 1: Write the failing tests** (append)

```python
def make_root(tmp_path):
    apps = tmp_path / "applications"
    apps.mkdir()
    for f in ["acme-answers-full.md", "acme-hr-bank.md", "acme-intros.md"]:
        shutil.copy(FIX / f, apps / f)
    (apps / "acme-listen-config.json").write_text(json.dumps({"tapes": [
        {"doc": "applications/acme-answers-full.md", "dir": "build/acme/audio", "pattern": "q{n:02d}.m4a"}]}))
    seg = tmp_path / "build" / "acme" / "audio" / "segments"
    make_wav(ed.segment_path(seg, 1, MODEL, ed.RECRUITER_VOICE, ed.RECRUITER_STYLE,
                             "Question 1. Where are you based?"), 1.0)
    return tmp_path


def test_build_deck_end_to_end(tmp_path):
    root, out = make_root(tmp_path), tmp_path / "deck"
    deck = ed.build_deck(root, "acme", out, MODEL, "Acme")
    assert [c["id"] for c in deck["cards"]] == [
        "answers-full#1", "answers-full#2", "hr-bank#2", "hr-bank#3", "real#2", "intros#short", "intros#medium"]
    assert deck["title"] == "Acme" and deck["vocabulary"] == ["Acme"] and len(deck["version"]) == 12
    assert json.loads((out / "deck.json").read_text())["version"] == deck["version"]
    assert (out / "clips" / "answers-full-01-q.m4a").exists()
    assert ed.build_deck(root, "acme", out, MODEL, "Acme")["version"] == deck["version"]


def test_build_deck_removes_stale_clips(tmp_path):
    root, out = make_root(tmp_path), tmp_path / "deck"
    (out / "clips").mkdir(parents=True)
    (out / "clips" / "old.m4a").write_bytes(b"x")
    ed.build_deck(root, "acme", out, MODEL, "Acme")
    assert not (out / "clips" / "old.m4a").exists()


def test_cli_prints_a_summary(tmp_path, capsys):
    root = make_root(tmp_path)
    ed.main(["acme", "--out", str(tmp_path / "deck"), "--model", MODEL, "--root", str(root)])
    out = capsys.readouterr().out
    assert out.startswith("acme v") and "7 cards" in out and "1 prompt clips" in out
```

- [ ] **Step 2: Run to verify failure**

Run: `python3 -m pytest tools/test_export_deck.py -v -k "build_deck or cli"`
Expected: 3 FAIL with `AttributeError`.

- [ ] **Step 3: Implement** (append)

```python
def build_deck(root, slug, out_dir, model, title=None):
    root, out_dir = Path(root), Path(out_dir)
    cfg = json.loads((root / "applications" / f"{slug}-listen-config.json").read_text(encoding="utf-8"))
    clips = out_dir / "clips"
    shutil.rmtree(clips, ignore_errors=True)
    cards = []
    for tape in cfg.get("tapes", []):
        doc = root / tape["doc"]
        cards += answer_cards(doc, doc_id(doc, slug), root / tape["dir"] / "segments", clips, model)
    hr = root / "applications" / f"{slug}-hr-bank.md"
    if hr.exists():
        cards += hr_bank_cards(hr, cards)
    intros = root / "applications" / f"{slug}-intros.md"
    if intros.exists():
        cards += talk_cards(intros)
    ids = [c["id"] for c in cards]
    dupes = sorted({i for i in ids if ids.count(i) > 1})
    if dupes:
        raise SystemExit(f"duplicate card ids: {dupes}")
    title = title or slug.capitalize()
    body = json.dumps(cards, sort_keys=True, ensure_ascii=False)
    deck = {"slug": slug, "title": title, "version": hashlib.sha1(body.encode()).hexdigest()[:12],
            "exported_at": datetime.datetime.now(datetime.UTC).strftime("%Y-%m-%dT%H:%M:%SZ"),
            "vocabulary": [title], "cards": cards}
    out_dir.mkdir(parents=True, exist_ok=True)
    (out_dir / "deck.json").write_text(json.dumps(deck, ensure_ascii=False, indent=2), encoding="utf-8")
    return deck


def summary(deck):
    cards = deck["cards"]
    kinds = ", ".join(f"{k} {sum(1 for c in cards if c['kind'] == k)}" for k in ("answer", "talk"))
    return (f"{deck['slug']} v{deck['version']}: {len(cards)} cards ({kinds}), "
            f"{sum(1 for c in cards if c.get('prompt_clip'))} prompt clips, "
            f"{sum(1 for c in cards if c.get('reference_clip'))} reference clips, "
            f"{sum(1 for c in cards if c['has_gaps'])} with gaps, "
            f"{sum(1 for c in cards if c['asked_for_real'])} asked for real")


def main(argv=None):
    ap = argparse.ArgumentParser(description=__doc__.splitlines()[0])
    ap.add_argument("slug")
    ap.add_argument("--out", required=True)
    ap.add_argument("--model", default=DEFAULT_MODEL)
    ap.add_argument("--title")
    ap.add_argument("--root", default=".")
    a = ap.parse_args(argv)
    print(summary(build_deck(a.root, a.slug, a.out, a.model, a.title)))


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run to verify pass**

Run: `python3 -m pytest tools/test_export_deck.py -v`
Expected: 15 passed.

- [ ] **Step 5: Commit** (repo: InterviewPreparationsBot)

```bash
git add tools/export_deck.py tools/test_export_deck.py
git commit -m "export_deck: deck assembly and CLI"
```

---

### Task 6: Grader: scaffold, quote check and auth

**Files:**
- Create: `grader/package.json` (via npm), `grader/tsconfig.json`, `grader/.gitignore`, `grader/.vercelignore`
- Create: `grader/lib/verify.ts`, `grader/lib/auth.ts`
- Test: `grader/test/verify.test.ts`, `grader/test/auth.test.ts`

**Interfaces:**
- Produces: `normalize(s: string): string`; `quoted(needle: string | null | undefined, haystack: string): boolean` (whole-word phrase match after normalising); `authorized(header: string | undefined, token = process.env.APP_TOKEN): boolean`.

- [ ] **Step 1: Create the branch and the package** (repo: Pitch Training Mobile)

```bash
cd "/Users/nickv/ClaudeCode Projects/Pitch Training Mobile" && git switch -c feat/exporter-grader
mkdir -p grader && cd grader
npm init -y
npm pkg set type=module private=true scripts.test="vitest run" scripts.typecheck="tsc --noEmit" scripts.live="tsx test/live/run-fixtures.ts"
npm install @anthropic-ai/sdk zod
npm install -D typescript vitest tsx @vercel/node @types/node
```

- [ ] **Step 2: Write the config files**

`grader/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["api", "lib", "test"]
}
```

`grader/.gitignore`:

```
node_modules/
decks/
.vercel/
.env*
```

`grader/.vercelignore` (when present, the Vercel CLI uses it instead of `.gitignore`, so `decks/` is uploaded):

```
node_modules
test
.env*
```

- [ ] **Step 3: Write the failing tests**

`grader/test/verify.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { normalize, quoted } from "../lib/verify.js";

const t = "So, um, Lisbon, and I'm staying. Hybrid is easy.";

describe("quoted", () => {
  it("finds an exact phrase", () => expect(quoted("Hybrid is easy", t)).toBe(true));
  it("ignores case and punctuation", () => expect(quoted("lisbon and I'm STAYING", t)).toBe(true));
  it("treats curly apostrophes as straight", () => expect(quoted("I’m staying", t)).toBe(true));
  it("matches whole words only", () => expect(quoted("is easy", "this easy")).toBe(false));
  it("rejects an invented phrase", () => expect(quoted("I love remote work", t)).toBe(false));
  it("rejects empty evidence", () => {
    expect(quoted("", t)).toBe(false);
    expect(quoted(null, t)).toBe(false);
  });
});

it("normalize collapses punctuation and whitespace", () => expect(normalize("  A,\n b ")).toBe("a b"));
```

`grader/test/auth.test.ts`:

```ts
import { expect, it } from "vitest";
import { authorized } from "../lib/auth.js";

it("rejects when no token is configured", () => expect(authorized("Bearer x", undefined)).toBe(false));
it("rejects a missing header", () => expect(authorized(undefined, "secret")).toBe(false));
it("rejects a wrong token", () => expect(authorized("Bearer nope", "secret")).toBe(false));
it("rejects a non-bearer scheme", () => expect(authorized("Basic secret", "secret")).toBe(false));
it("accepts the right token", () => expect(authorized("Bearer secret", "secret")).toBe(true));
```

- [ ] **Step 4: Run to verify failure**

Run: `cd grader && npm test`
Expected: FAIL, cannot resolve `../lib/verify.js` and `../lib/auth.js`.

- [ ] **Step 5: Implement**

`grader/lib/verify.ts`:

```ts
/** Lowercase, straight apostrophes, punctuation to spaces, single spaces. */
export function normalize(s: string): string {
  return s
    .toLowerCase()
    .replace(/[’‘]/g, "'")
    .replace(/[^\p{L}\p{N}\s']/gu, " ")
    .replace(/\s+/g, " ")
    .trim();
}

/** True when `needle` appears in `haystack` as whole words after normalising both.
 *  Every quote the grader returns is checked with this before the phone trusts it. */
export function quoted(needle: string | null | undefined, haystack: string): boolean {
  if (!needle) return false;
  const n = normalize(needle);
  return n.length > 0 && ` ${normalize(haystack)} `.includes(` ${n} `);
}
```

`grader/lib/auth.ts`:

```ts
import { timingSafeEqual } from "node:crypto";

/** Bearer-token check against APP_TOKEN, in constant time. */
export function authorized(header: string | undefined, token = process.env.APP_TOKEN): boolean {
  if (!token || !header?.startsWith("Bearer ")) return false;
  const given = Buffer.from(header.slice(7));
  const expected = Buffer.from(token);
  return given.length === expected.length && timingSafeEqual(given, expected);
}
```

- [ ] **Step 6: Run to verify pass**

Run: `npm test && npm run typecheck`
Expected: 12 tests passed; typecheck exits 0.

- [ ] **Step 7: Commit** (repo: Pitch Training Mobile, branch `feat/exporter-grader`)

```bash
git add grader/package.json grader/package-lock.json grader/tsconfig.json grader/.gitignore grader/.vercelignore grader/lib grader/test
git commit -m "grader: scaffold, quote check and bearer auth"
```

---

### Task 7: Grader: request and output schemas, prompt

**Files:**
- Create: `grader/lib/schema.ts`, `grader/lib/prompt.ts`, `grader/test/fixtures.ts`
- Test: `grader/test/schema.test.ts`, `grader/test/prompt.test.ts`

**Interfaces:**
- Produces: zod `GradeRequest` (+ type), zod `ModelOutput` (+ type), `MODEL_OUTPUT_SCHEMA` (JSON schema object for `output_config.format`), `interface GradeResponse` (spec §6.2 response); `SYSTEM_PROMPT: string`; `buildUserMessage(r: GradeRequest): string`; test fixtures `answerReq`, `talkReq`.

- [ ] **Step 1: Write the shared fixtures**

`grader/test/fixtures.ts` (invented content):

```ts
import type { GradeRequest } from "../lib/schema.js";

export const answerReq: GradeRequest = {
  card: {
    kind: "answer",
    question: "Where are you based?",
    say: "Yes, Lisbon, and I'm staying. I moved here from Porto as a choice. Hybrid is easy.",
    spine: ["Lisbon", "staying", "hybrid easy"],
    last_line: "Hybrid is easy.",
    guard: "never name a salary.",
    reference: null,
  },
  transcript: "So, um, Lisbon, and I'm staying. Hybrid is easy.",
  duration_s: 12.4,
  target_s: 7,
};

export const talkReq: GradeRequest = {
  card: {
    kind: "talk",
    question: "Your short intro",
    reference: "I'm a product designer. I work on complex tools, and I make them coherent.",
    guard: null,
    last_line: null,
  },
  transcript: "I'm a product designer and I work on complex tools.",
  duration_s: 20,
  target_s: 30,
};
```

- [ ] **Step 2: Write the failing tests**

`grader/test/schema.test.ts`:

```ts
import { expect, it } from "vitest";
import { GradeRequest, MODEL_OUTPUT_SCHEMA, ModelOutput } from "../lib/schema.js";
import { answerReq, talkReq } from "./fixtures.js";

it("accepts answer and talk requests", () => {
  expect(GradeRequest.safeParse(answerReq).success).toBe(true);
  expect(GradeRequest.safeParse(talkReq).success).toBe(true);
});

it("rejects a request without a transcript", () => {
  const { transcript, ...rest } = answerReq;
  expect(GradeRequest.safeParse(rest).success).toBe(false);
});

it("JSON schema and zod schema list the same fields, all required", () => {
  const keys = Object.keys(MODEL_OUTPUT_SCHEMA.properties).sort();
  expect(Object.keys(ModelOutput.shape).sort()).toEqual(keys);
  expect([...MODEL_OUTPUT_SCHEMA.required].sort()).toEqual(keys);
});

it("parses a valid model reply", () => {
  const reply = {
    verdict: "All three beats. Last line landed.",
    beats: [{ beat: "Lisbon", hit: true, evidence: "Lisbon" }],
    missing_points: [],
    last_line: "landed",
    guard: [],
    rephrases: [],
  };
  expect(ModelOutput.safeParse(reply).success).toBe(true);
  expect(ModelOutput.safeParse({ ...reply, last_line: "maybe" }).success).toBe(false);
});
```

`grader/test/prompt.test.ts`:

```ts
import { expect, it } from "vitest";
import { buildUserMessage, SYSTEM_PROMPT } from "../lib/prompt.js";
import { answerReq, talkReq } from "./fixtures.js";

it("numbers the beats and ends with the transcript", () => {
  const m = buildUserMessage(answerReq);
  expect(m).toContain("<beats>\n1. Lisbon\n2. staying\n3. hybrid easy\n</beats>");
  expect(m).toContain("<timing>spoke for 12 s, target 7 s</timing>");
  expect(m.trim().endsWith("</transcript>")).toBe(true);
  expect(m).not.toContain("<reference>");
});

it("marks a missing guard and last line as none and includes the reference", () => {
  const m = buildUserMessage(talkReq);
  expect(m).toContain("<guard>(none)</guard>");
  expect(m).toContain("<last_line>(none)</last_line>");
  expect(m).toContain("<reference>I'm a product designer.");
  expect(m).not.toContain("<beats>");
});

it("system prompt states the verbatim-quote rule", () => {
  expect(SYSTEM_PROMPT).toContain("character for character");
});
```

- [ ] **Step 3: Run to verify failure**

Run: `npm test`
Expected: FAIL, cannot resolve `../lib/schema.js` and `../lib/prompt.js`.

- [ ] **Step 4: Implement**

`grader/lib/schema.ts`:

```ts
import { z } from "zod";

export const GradeRequest = z.object({
  card: z.object({
    kind: z.enum(["answer", "talk"]),
    question: z.string().min(1),
    say: z.string().nullable().optional(),
    spine: z.array(z.string()).optional(),
    last_line: z.string().nullable().optional(),
    guard: z.string().nullable().optional(),
    reference: z.string().nullable().optional(),
  }),
  transcript: z.string(),
  duration_s: z.number().nonnegative(),
  target_s: z.number().positive().nullable().optional(),
});
export type GradeRequest = z.infer<typeof GradeRequest>;

/** What the model returns. Empty strings stand in for "no evidence" so every field stays required. */
export const ModelOutput = z.object({
  verdict: z.string(),
  beats: z.array(z.object({ beat: z.string(), hit: z.boolean(), evidence: z.string() })),
  missing_points: z.array(z.object({ point: z.string() })),
  last_line: z.enum(["landed", "paraphrased", "missing", "none"]),
  guard: z.array(z.object({ rule: z.string(), violated: z.boolean(), evidence: z.string() })),
  rephrases: z.array(z.object({ original: z.string(), better: z.string() })),
});
export type ModelOutput = z.infer<typeof ModelOutput>;

const str = { type: "string" };
const bool = { type: "boolean" };
const obj = (properties: Record<string, unknown>) => ({
  type: "object",
  additionalProperties: false,
  required: Object.keys(properties),
  properties,
});

/** JSON schema for output_config.format. Mirrors ModelOutput (a test keeps them in step). */
export const MODEL_OUTPUT_SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["verdict", "beats", "missing_points", "last_line", "guard", "rephrases"],
  properties: {
    verdict: str,
    beats: { type: "array", items: obj({ beat: str, hit: bool, evidence: str }) },
    missing_points: { type: "array", items: obj({ point: str }) },
    last_line: { type: "string", enum: ["landed", "paraphrased", "missing", "none"] },
    guard: { type: "array", items: obj({ rule: str, violated: bool, evidence: str }) },
    rephrases: { type: "array", items: obj({ original: str, better: str }) },
  },
};

/** The HTTP response, spec §6.2. */
export interface GradeResponse {
  verdict: string;
  beats: { beat: string; hit: boolean; evidence: string | null; verified: boolean }[];
  missing_points: { point: string; verified: boolean }[];
  last_line: "landed" | "paraphrased" | "missing" | null;
  guard: { rule: string; violated: boolean; evidence: string | null; verified: boolean }[];
  rephrases: { original: string; better: string; verified: true }[];
  model: string;
}
```

`grader/lib/prompt.ts`:

```ts
import type { GradeRequest } from "./schema.js";

export const SYSTEM_PROMPT = `You grade one spoken practice answer for a job interview. The candidate is rehearsing answers they prepared in advance. English is not their first language; they speak it at an advanced level.

The transcript comes from on-device speech recognition. Names and product words may be misheard: judge meaning, and never count a misheard name as a missed point.

Rules for each field:
- beats: one entry per planned beat, in the given order. hit is true when the meaning of the beat is present, even when paraphrased. evidence is the shortest exact phrase from the transcript that shows it, copied character for character, or "" when hit is false.
- last_line: "landed" when the answer ends with the planned last line almost word for word, "paraphrased" when it ends on the same idea in other words, "missing" otherwise, "none" when no last line is given.
- guard: one entry per rule in the guard text. violated is true only when the transcript breaks that rule. evidence is the exact phrase that breaks it, copied character for character, or "".
- missing_points: only when a reference is given. Up to 3 important points from the reference that the answer left out, each copied character for character from the reference. Otherwise an empty list.
- rephrases: at most 3. original is copied character for character from the transcript. better keeps the candidate's own voice and reuses the planned wording where it fits. Include one only when it changes how the answer comes across: unclear, a grammar slip a listener would notice, or weaker than the planned line. Do not polish for its own sake.
- verdict: one sentence of 20 words or fewer, to be read aloud. With beats: how many were hit, the most important miss, and whether the last line landed. Without beats: the single most useful thing to fix next time. No other numbers.`;

export function buildUserMessage(r: GradeRequest): string {
  const c = r.card;
  const parts = [`<question>${c.question}</question>`];
  if (c.say) parts.push(`<planned_answer>${c.say}</planned_answer>`);
  if (c.spine?.length) parts.push(`<beats>\n${c.spine.map((b, i) => `${i + 1}. ${b}`).join("\n")}\n</beats>`);
  parts.push(`<last_line>${c.last_line ?? "(none)"}</last_line>`);
  parts.push(`<guard>${c.guard ?? "(none)"}</guard>`);
  if (c.reference) parts.push(`<reference>${c.reference}</reference>`);
  const target = r.target_s ? `, target ${Math.round(r.target_s)} s` : "";
  parts.push(`<timing>spoke for ${Math.round(r.duration_s)} s${target}</timing>`);
  parts.push(`<transcript>${r.transcript}</transcript>`);
  return parts.join("\n\n");
}
```

- [ ] **Step 5: Run to verify pass**

Run: `npm test && npm run typecheck`
Expected: 19 tests passed; typecheck exits 0.

- [ ] **Step 6: Commit** (repo: Pitch Training Mobile)

```bash
git add grader/lib/schema.ts grader/lib/prompt.ts grader/test/fixtures.ts grader/test/schema.test.ts grader/test/prompt.test.ts
git commit -m "grader: request and output schemas, grading prompt"
```

---

### Task 8: Grader: the grading call and quote verification

**Files:**
- Create: `grader/lib/grade.ts`
- Test: `grader/test/grade.test.ts`

**Interfaces:**
- Consumes: `quoted` (Task 6); `GradeRequest`, `ModelOutput`, `MODEL_OUTPUT_SCHEMA`, `GradeResponse`, `SYSTEM_PROMPT`, `buildUserMessage` (Task 7).
- Produces: `type CreateMessage = (params: Anthropic.Beta.Messages.MessageCreateParamsNonStreaming) => Promise<Anthropic.Beta.Messages.BetaMessage>`; `class UngradedError extends Error`; `gradeAttempt(req, create, model): Promise<GradeResponse>`; `toResponse(out, req, model): GradeResponse`.

- [ ] **Step 1: Write the failing tests**

`grader/test/grade.test.ts`:

```ts
import type Anthropic from "@anthropic-ai/sdk";
import { expect, it } from "vitest";
import { gradeAttempt, UngradedError, type CreateMessage } from "../lib/grade.js";
import type { ModelOutput } from "../lib/schema.js";
import { answerReq, talkReq } from "./fixtures.js";

type Params = Anthropic.Beta.Messages.MessageCreateParamsNonStreaming;

function fake(reply: unknown, stop = "end_turn") {
  const calls: Params[] = [];
  const create: CreateMessage = async (p) => {
    calls.push(p);
    return {
      model: "claude-opus-5",
      stop_reason: stop,
      content: [{ type: "text", text: JSON.stringify(reply) }],
    } as unknown as Anthropic.Beta.Messages.BetaMessage;
  };
  return { create, calls };
}

const good: ModelOutput = {
  verdict: "Two of three beats confirmed. Last line landed.",
  beats: [
    { beat: "Lisbon", hit: true, evidence: "Lisbon, and I'm staying" },
    { beat: "staying", hit: true, evidence: "I'm staying" },
    { beat: "hybrid easy", hit: true, evidence: "hybrid is really easy" },
  ],
  missing_points: [],
  last_line: "landed",
  guard: [{ rule: "never name a salary", violated: false, evidence: "" }],
  rephrases: [
    { original: "So, um, Lisbon", better: "Lisbon" },
    { original: "I adore Lisbon", better: "Lisbon suits me" },
  ],
};

it("sends model, low effort, the JSON schema and the refusal fallback", async () => {
  const { create, calls } = fake(good);
  await gradeAttempt(answerReq, create, "claude-opus-5");
  const p = calls[0] as unknown as Record<string, any>;
  expect(p.model).toBe("claude-opus-5");
  expect(p.max_tokens).toBe(8000);
  expect(p.output_config.effort).toBe("low");
  expect(p.output_config.format.type).toBe("json_schema");
  expect(p.fallbacks).toBe("default");
  expect(p.betas).toEqual(["server-side-fallback-2026-07-01"]);
});

it("marks unfound evidence unverified and drops invented rephrases", async () => {
  const out = await gradeAttempt(answerReq, fake(good).create, "claude-opus-5");
  expect(out.beats.map((b) => b.verified)).toEqual([true, true, false]);
  expect(out.rephrases).toEqual([{ original: "So, um, Lisbon", better: "Lisbon", verified: true }]);
  expect(out.guard[0]).toEqual({ rule: "never name a salary", violated: false, evidence: null, verified: true });
  expect(out.last_line).toBe("landed");
  expect(out.model).toBe("claude-opus-5");
});

it("maps last_line none to null and checks missing points against the reference", async () => {
  const reply = {
    ...good, beats: [], last_line: "none",
    missing_points: [{ point: "I make them coherent" }, { point: "I ship fast" }],
    rephrases: [],
  };
  const out = await gradeAttempt(talkReq, fake(reply).create, "m");
  expect(out.last_line).toBeNull();
  expect(out.missing_points).toEqual([
    { point: "I make them coherent", verified: true },
    { point: "I ship fast", verified: false },
  ]);
});

it("keeps at most three rephrases", async () => {
  const r = { original: "Hybrid is easy", better: "Hybrid works well" };
  const out = await gradeAttempt(answerReq, fake({ ...good, rephrases: [r, r, r, r] }).create, "m");
  expect(out.rephrases).toHaveLength(3);
});

it("throws UngradedError on a refusal, a cut-off reply or a reply that fails the schema", async () => {
  await expect(gradeAttempt(answerReq, fake(good, "refusal").create, "m")).rejects.toBeInstanceOf(UngradedError);
  await expect(gradeAttempt(answerReq, fake(good, "max_tokens").create, "m")).rejects.toBeInstanceOf(UngradedError);
  await expect(gradeAttempt(answerReq, fake({ verdict: 3 }).create, "m")).rejects.toBeInstanceOf(UngradedError);
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL, cannot resolve `../lib/grade.js`.

- [ ] **Step 3: Implement**

`grader/lib/grade.ts`:

```ts
import type Anthropic from "@anthropic-ai/sdk";
import { buildUserMessage, SYSTEM_PROMPT } from "./prompt.js";
import { type GradeRequest, type GradeResponse, MODEL_OUTPUT_SCHEMA, ModelOutput } from "./schema.js";
import { quoted } from "./verify.js";

export type CreateMessage = (
  params: Anthropic.Beta.Messages.MessageCreateParamsNonStreaming,
) => Promise<Anthropic.Beta.Messages.BetaMessage>;

/** The attempt could not be graded: refusal, cut-off or malformed output. The phone stores it as ungraded. */
export class UngradedError extends Error {}

export async function gradeAttempt(req: GradeRequest, create: CreateMessage, model: string): Promise<GradeResponse> {
  const response = await create({
    model,
    max_tokens: 8000,
    betas: ["server-side-fallback-2026-07-01"],
    fallbacks: "default",
    output_config: { effort: "low", format: { type: "json_schema", schema: MODEL_OUTPUT_SCHEMA } },
    system: SYSTEM_PROMPT,
    messages: [{ role: "user", content: buildUserMessage(req) }],
  });
  if (response.stop_reason !== "end_turn") throw new UngradedError(`stop_reason ${response.stop_reason}`);
  const block = response.content.find((b) => b.type === "text");
  if (!block || block.type !== "text") throw new UngradedError("no text block");
  let parsed: ModelOutput;
  try {
    parsed = ModelOutput.parse(JSON.parse(block.text));
  } catch {
    throw new UngradedError("reply failed the schema");
  }
  return toResponse(parsed, req, response.model);
}

/** Applies the quote checks: a claim whose quote is not in the source text is marked unverified. */
export function toResponse(o: ModelOutput, req: GradeRequest, model: string): GradeResponse {
  const t = req.transcript;
  const ref = req.card.reference ?? "";
  return {
    verdict: o.verdict,
    beats: o.beats.map((b) => ({
      beat: b.beat, hit: b.hit, evidence: b.evidence || null, verified: !b.hit || quoted(b.evidence, t),
    })),
    missing_points: o.missing_points.map((p) => ({ point: p.point, verified: quoted(p.point, ref) })),
    last_line: o.last_line === "none" ? null : o.last_line,
    guard: o.guard.map((g) => ({
      rule: g.rule, violated: g.violated, evidence: g.evidence || null, verified: !g.violated || quoted(g.evidence, t),
    })),
    rephrases: o.rephrases
      .filter((r) => quoted(r.original, t))
      .slice(0, 3)
      .map((r) => ({ original: r.original, better: r.better, verified: true as const })),
    model,
  };
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test && npm run typecheck`
Expected: 24 tests passed; typecheck exits 0.
If typecheck rejects only the `fallbacks: "default"` line, run `npm install @anthropic-ai/sdk@latest` and retry. If it still fails, the installed SDK types predate the scalar form: add `// @ts-expect-error fallbacks: "default" is newer than the SDK types` on the line above `fallbacks` and note it in the commit message. If it rejects a type name (`MessageCreateParamsNonStreaming` or `BetaMessage`), use the name the compiler suggests.

- [ ] **Step 5: Commit** (repo: Pitch Training Mobile)

```bash
git add grader/lib/grade.ts grader/test/grade.test.ts grader/package.json grader/package-lock.json
git commit -m "grader: grading call with verified quotes"
```

---

### Task 9: Grader: HTTP endpoints

**Files:**
- Create: `grader/lib/handlers.ts`, `grader/api/grade.ts`, `grader/api/deck.ts`, `grader/vercel.json`
- Test: `grader/test/handlers.test.ts`

**Interfaces:**
- Consumes: `authorized`, `GradeRequest`, `gradeAttempt`, `UngradedError`, `CreateMessage`.
- Produces: `makeGradeHandler(create, model: () => string)`; `deckFile(slug, path, root): string | null`; `makeDeckHandler(root)`. Status codes: 405 wrong method, 401 bad token, 400 bad body or path, 404 missing deck file, 502 `{"error":"ungraded"}`, 503 `{"error":"unavailable"}` on Anthropic API errors.

- [ ] **Step 1: Write the failing tests**

`grader/test/handlers.test.ts`:

```ts
import type Anthropic from "@anthropic-ai/sdk";
import type { VercelRequest, VercelResponse } from "@vercel/node";
import { beforeEach, expect, it, vi } from "vitest";
import type { CreateMessage } from "../lib/grade.js";
import { deckFile, makeDeckHandler, makeGradeHandler } from "../lib/handlers.js";
import { answerReq } from "./fixtures.js";

function res() {
  const r = { statusCode: 0, body: undefined as unknown, headers: {} as Record<string, string> } as any;
  r.status = (c: number) => ((r.statusCode = c), r);
  r.json = (b: unknown) => ((r.body = b), r);
  r.setHeader = (k: string, v: string) => ((r.headers[k] = v), r);
  return r as VercelResponse & { statusCode: number; body: any };
}

const req = (x: Record<string, unknown>) =>
  ({ method: "POST", headers: {}, query: {}, ...x }) as unknown as VercelRequest;
const auth = { authorization: "Bearer t" };

const reply = {
  verdict: "Three of three beats.", beats: [], missing_points: [], last_line: "landed", guard: [], rephrases: [],
};
const create = (stop = "end_turn"): CreateMessage => async () =>
  ({ model: "claude-opus-5", stop_reason: stop, content: [{ type: "text", text: JSON.stringify(reply) }] }) as unknown as
    Anthropic.Beta.Messages.BetaMessage;

beforeEach(() => vi.stubEnv("APP_TOKEN", "t"));

it("grade: 405, 401 and 400 before any model call", async () => {
  const h = makeGradeHandler(create(), () => "m");
  const a = res(); await h(req({ method: "GET", headers: auth }), a); expect(a.statusCode).toBe(405);
  const b = res(); await h(req({ body: answerReq }), b); expect(b.statusCode).toBe(401);
  const c = res(); await h(req({ headers: auth, body: { nope: 1 } }), c); expect(c.statusCode).toBe(400);
});

it("grade: 200 with the graded body, 502 when ungraded", async () => {
  const ok = res();
  await makeGradeHandler(create(), () => "m")(req({ headers: auth, body: answerReq }), ok);
  expect(ok.statusCode).toBe(200);
  expect(ok.body.verdict).toBe("Three of three beats.");
  const bad = res();
  await makeGradeHandler(create("refusal"), () => "m")(req({ headers: auth, body: answerReq }), bad);
  expect(bad.statusCode).toBe(502);
  expect(bad.body).toEqual({ error: "ungraded" });
});

it("deckFile accepts only deck.json and clips/*.m4a under a simple slug", () => {
  expect(deckFile("acme", "deck.json", "/r")).toBe("/r/acme/deck.json");
  expect(deckFile("acme", ["clips", "a-01-q.m4a"], "/r")).toBe("/r/acme/clips/a-01-q.m4a");
  expect(deckFile("acme", "../secret.json", "/r")).toBeNull();
  expect(deckFile("ac/me", "deck.json", "/r")).toBeNull();
  expect(deckFile("acme", "clips/x.wav", "/r")).toBeNull();
  expect(deckFile(undefined, "deck.json", "/r")).toBeNull();
});

it("deck: 401 without token, 400 bad path, 404 missing file", () => {
  const h = makeDeckHandler("/nonexistent-root");
  const a = res(); h(req({ method: "GET", query: { slug: "acme", path: "deck.json" } }), a); expect(a.statusCode).toBe(401);
  const b = res(); h(req({ method: "GET", headers: auth, query: { slug: "acme", path: "x.txt" } }), b); expect(b.statusCode).toBe(400);
  const c = res(); h(req({ method: "GET", headers: auth, query: { slug: "acme", path: "deck.json" } }), c); expect(c.statusCode).toBe(404);
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test`
Expected: FAIL, cannot resolve `../lib/handlers.js`.

- [ ] **Step 3: Implement**

`grader/lib/handlers.ts`:

```ts
import Anthropic from "@anthropic-ai/sdk";
import type { VercelRequest, VercelResponse } from "@vercel/node";
import { createReadStream, existsSync } from "node:fs";
import { join } from "node:path";
import { authorized } from "./auth.js";
import { type CreateMessage, gradeAttempt, UngradedError } from "./grade.js";
import { GradeRequest } from "./schema.js";

export function makeGradeHandler(create: CreateMessage, model: () => string) {
  return async function handler(req: VercelRequest, res: VercelResponse) {
    if (req.method !== "POST") return res.status(405).json({ error: "method_not_allowed" });
    if (!authorized(req.headers.authorization)) return res.status(401).json({ error: "unauthorized" });
    const body = GradeRequest.safeParse(req.body);
    if (!body.success) return res.status(400).json({ error: "bad_request" });
    try {
      return res.status(200).json(await gradeAttempt(body.data, create, model()));
    } catch (e) {
      if (e instanceof UngradedError) return res.status(502).json({ error: "ungraded" });
      if (e instanceof Anthropic.APIError) return res.status(503).json({ error: "unavailable" });
      throw e;
    }
  };
}

const SLUG = /^[a-z0-9-]+$/;
const FILE = /^(deck\.json|clips\/[A-Za-z0-9._-]+\.m4a)$/;

/** Maps a request to a file under root/<slug>/, or null for anything outside deck.json and clips/*.m4a. */
export function deckFile(slug: unknown, path: unknown, root: string): string | null {
  const p = Array.isArray(path) ? path.join("/") : path;
  if (typeof slug !== "string" || typeof p !== "string") return null;
  if (!SLUG.test(slug) || !FILE.test(p) || p.includes("..")) return null;
  return join(root, slug, p);
}

export function makeDeckHandler(root: string) {
  return function handler(req: VercelRequest, res: VercelResponse) {
    if (req.method !== "GET") return res.status(405).json({ error: "method_not_allowed" });
    if (!authorized(req.headers.authorization)) return res.status(401).json({ error: "unauthorized" });
    const file = deckFile(req.query.slug, req.query.path, root);
    if (!file) return res.status(400).json({ error: "bad_path" });
    if (!existsSync(file)) return res.status(404).json({ error: "not_found" });
    res.setHeader("Content-Type", file.endsWith(".json") ? "application/json" : "audio/mp4");
    res.setHeader("Cache-Control", "private, no-store");
    createReadStream(file).pipe(res);
  };
}
```

`grader/api/grade.ts`:

```ts
import Anthropic from "@anthropic-ai/sdk";
import type { CreateMessage } from "../lib/grade.js";
import { makeGradeHandler } from "../lib/handlers.js";

let client: Anthropic | undefined;
const create: CreateMessage = (params) => (client ??= new Anthropic()).beta.messages.create(params);

export default makeGradeHandler(create, () => process.env.GRADER_MODEL ?? "claude-opus-5");
```

`grader/api/deck.ts`:

```ts
import { join } from "node:path";
import { makeDeckHandler } from "../lib/handlers.js";

export default makeDeckHandler(join(process.cwd(), "decks"));
```

`grader/vercel.json`:

```json
{
  "rewrites": [{ "source": "/api/decks/:slug/:path*", "destination": "/api/deck?slug=:slug&path=:path*" }],
  "functions": { "api/deck.ts": { "includeFiles": "decks/**" } },
  "headers": [{ "source": "/(.*)", "headers": [{ "key": "X-Robots-Tag", "value": "noindex" }] }]
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test && npm run typecheck`
Expected: 28 tests passed; typecheck exits 0.

- [ ] **Step 5: Commit** (repo: Pitch Training Mobile)

```bash
git add grader/lib/handlers.ts grader/api grader/vercel.json grader/test/handlers.test.ts
git commit -m "grader: grade and deck endpoints behind the token"
```

---

### Task 10: First real deck export

Produces a real deck in the gitignored `grader/decks/`. Nothing from it is committed or pasted into this repo.

**Files:** none committed.

- [ ] **Step 1: Ask Nick which prep to export**

Ask for the slug and display title (the newest `applications/*-listen-config.json` is the likely one). Below, `<slug>` and `<Title>` stand for his answer.

- [ ] **Step 2: Run the exporter**

```bash
cd "/Users/nickv/ClaudeCode Projects/InterviewPreparationsBot"
python3 tools/export_deck.py <slug> --title "<Title>" --out "/Users/nickv/ClaudeCode Projects/Pitch Training Mobile/grader/decks/<slug>"
```

Expected: one summary line and a warning per missing clip. If every card warns "no prompt clip", the tape was made with another model or voice: ask Nick which `--model` and `TAPE_*_VOICE` values he used and rerun with them.

- [ ] **Step 3: Check the counts against the docs**

```bash
for f in $(python3 -c "import json;print(' '.join(t['doc'] for t in json.load(open('applications/<slug>-listen-config.json'))['tapes']))"); do echo "$f $(grep -c '^### [0-9]' $f)"; done
python3 -c "import json;d=json.load(open('/Users/nickv/ClaudeCode Projects/Pitch Training Mobile/grader/decks/<slug>/deck.json'));import collections;print(collections.Counter(c['id'].split('#')[0] for c in d['cards']))"
```

Expected: each tape doc's `###` count equals its card count in the counter.

- [ ] **Step 4: Confirm the deck stays out of git**

Run: `cd "/Users/nickv/ClaudeCode Projects/Pitch Training Mobile" && git check-ignore -v grader/decks/<slug>/deck.json && git status --short grader/`
Expected: `check-ignore` names the `grader/.gitignore` rule, and `git status` lists nothing under `grader/decks/`.

- [ ] **Step 5: Report to Nick**

Report the summary line, the per-doc counts and any missing-clip warnings. Do not paste card content into chat or any file in this repo.

---

### Task 11: Live grader check [Nick's go-ahead, about $0.20]

**Files:**
- Create: `grader/test/live/cases.ts`, `grader/test/live/run-fixtures.ts`, `grader/test/live/sample-request.json`

**Interfaces:**
- Consumes: `gradeAttempt`, `GradeRequest`, `GradeResponse`.
- Produces: `npm run live`: six real calls to the model, each checked against its expectations. Exit code 1 on any failure.

- [ ] **Step 1: Write the cases** (invented content)

`grader/test/live/cases.ts`:

```ts
import type { GradeRequest, GradeResponse } from "../../lib/schema.js";

export interface Case { name: string; request: GradeRequest; expect: (o: GradeResponse) => string[] }

const based: GradeRequest["card"] = {
  kind: "answer",
  question: "Where are you based?",
  say: "Yes, Lisbon, and I'm staying. I moved here from Porto as a choice. Hybrid is easy.",
  spine: ["Lisbon", "staying", "hybrid easy"],
  last_line: "Hybrid is easy.",
  guard: "never name a salary number.",
  reference: null,
};

const hits = (o: GradeResponse) => o.beats.filter((b) => b.hit && b.verified).length;
const short = (o: GradeResponse) => (o.verdict.split(/\s+/).length <= 20 ? [] : ["verdict over 20 words"]);

export const cases: Case[] = [
  {
    name: "own Say hits every beat",
    request: { card: based, transcript: based.say!, duration_s: 7, target_s: 7 },
    expect: (o) => [
      ...(hits(o) === 3 ? [] : [`verified hits ${hits(o)}/3`]),
      ...(o.last_line === "landed" ? [] : [`last_line ${o.last_line}`]),
      ...short(o),
    ],
  },
  {
    name: "off-topic hits nothing",
    request: { card: based, transcript: "I really enjoy cooking pasta on weekends and trying new sauces.", duration_s: 5, target_s: 7 },
    expect: (o) => [
      ...(o.beats.some((b) => b.hit) ? ["a beat was hit"] : []),
      ...(o.last_line === "missing" ? [] : [`last_line ${o.last_line}`]),
      ...short(o),
    ],
  },
  {
    name: "paraphrase still counts",
    request: {
      card: based,
      transcript: "I live in Lisbon now and I plan to stay. I came over from Porto on purpose. Coming into the office a couple of days a week is no problem.",
      duration_s: 9, target_s: 7,
    },
    expect: (o) => [
      ...(hits(o) >= 2 ? [] : [`verified hits ${hits(o)}/3`]),
      ...(o.last_line === "landed" ? ["paraphrase counted as landed"] : []),
      ...short(o),
    ],
  },
  {
    name: "guard violation is flagged with a real quote",
    request: {
      card: based,
      transcript: "Lisbon, and I'm staying. I'm looking for about ninety thousand a year. Hybrid is easy.",
      duration_s: 8, target_s: 7,
    },
    expect: (o) => [...(o.guard.some((g) => g.violated && g.verified) ? [] : ["no verified violation"]), ...short(o)],
  },
  {
    name: "talk card lists missing points from the reference",
    request: {
      card: {
        kind: "talk",
        question: "Your short intro",
        reference: "I'm a product designer with ten years in complex tools. I make large products coherent. Lately I build design systems that people and AI agents both follow.",
        guard: null,
        last_line: null,
      },
      transcript: "Hi, I'm a product designer, and I've spent about ten years on complex tools.",
      duration_s: 6, target_s: 30,
    },
    expect: (o) => [
      ...(o.missing_points.length >= 1 ? [] : ["no missing points"]),
      ...(o.missing_points.every((p) => p.verified) ? [] : ["unverified missing point"]),
      ...short(o),
    ],
  },
  {
    name: "rephrases quote the transcript",
    request: {
      card: {
        kind: "answer",
        question: "What do you work on?",
        say: "I've worked on admin tools for eight years, mostly dashboards for the people who run the platform.",
        spine: ["admin tools", "eight years", "dashboards"],
        last_line: "mostly dashboards for the people who run the platform.",
        guard: null,
        reference: null,
      },
      transcript: "I am working in this domain since eight years and I did many dashboards for the admins.",
      duration_s: 6, target_s: 6,
    },
    expect: (o) => [...(o.rephrases.length >= 1 ? [] : ["no rephrases"]), ...short(o)],
  },
];
```

`grader/test/live/run-fixtures.ts`:

```ts
import Anthropic from "@anthropic-ai/sdk";
import { gradeAttempt } from "../../lib/grade.js";
import { cases } from "./cases.js";

const client = new Anthropic();
const model = process.env.GRADER_MODEL ?? "claude-opus-5";
let failed = 0;

for (const c of cases) {
  const started = Date.now();
  const out = await gradeAttempt(c.request, (p) => client.beta.messages.create(p), model);
  const problems = c.expect(out);
  const ms = Date.now() - started;
  if (problems.length === 0) {
    console.log(`PASS ${c.name} (${ms} ms): ${out.verdict}`);
  } else {
    failed++;
    console.log(`FAIL ${c.name} (${ms} ms): ${problems.join("; ")}\n  ${JSON.stringify(out)}`);
  }
}
console.log(`${cases.length - failed}/${cases.length} passed`);
process.exit(failed ? 1 : 0);
```

`grader/test/live/sample-request.json` (used by the deploy check in Task 12):

```json
{
  "card": {
    "kind": "answer",
    "question": "Where are you based?",
    "say": "Yes, Lisbon, and I'm staying. I moved here from Porto as a choice. Hybrid is easy.",
    "spine": ["Lisbon", "staying", "hybrid easy"],
    "last_line": "Hybrid is easy.",
    "guard": "never name a salary number.",
    "reference": null
  },
  "transcript": "So, um, Lisbon, and I'm staying. Hybrid is easy.",
  "duration_s": 6,
  "target_s": 7
}
```

- [ ] **Step 2: Typecheck and commit**

```bash
cd grader && npm run typecheck && npm test
git add test/live
git commit -m "grader: live check with six invented cases"
```

- [ ] **Step 3: Ask Nick for the go-ahead**

"The live check makes 6 calls to `claude-opus-5`, about $0.20. Go?" Wait for yes.

- [ ] **Step 4: Run it**

Run `ant auth status`. If it shows an active credential, run `cd grader && npm run live`. If not, give Nick the command to run in his own terminal with his key exported: `cd "/Users/nickv/ClaudeCode Projects/Pitch Training Mobile/grader" && npm run live`.
Expected: `6/6 passed`, each call under about 8 s. On a FAIL, show Nick the printed JSON and propose a prompt change in `lib/prompt.ts` before rerunning. Do not loosen a test to make it pass.

---

### Task 12: Deploy [Nick + executor]

**Files:** none (Vercel project settings and env vars live in Vercel).

- [ ] **Step 1: [Nick] Anthropic workspace and key**

In the Anthropic Console: create a workspace `pitch-training`, set a monthly spend limit (suggested $30), create an API key in it, and keep the key in the password manager.

- [ ] **Step 2: Confirm the Vercel scope, then link**

Ask Nick which Vercel team or account owns `pitch-grader`. Then, from `grader/`:
`vercel link --yes --scope <team> --project pitch-grader`

- [ ] **Step 3: [Nick] Set the secrets**

In his own terminal, from `grader/`:

```bash
vercel env add ANTHROPIC_API_KEY production
openssl rand -hex 32
vercel env add APP_TOKEN production
```

Paste the key at the first prompt. Save the `openssl` output in the password manager as the Pitch Training app token, and paste it at the `APP_TOKEN` prompt. The iOS app gets the same token in Plan 3.

- [ ] **Step 4: Deploy**

State: repo `weeeha/Pitch-Training-Mobile-App`, path `/Users/nickv/ClaudeCode Projects/Pitch Training Mobile/grader`, branch `feat/exporter-grader`, target Vercel production for `pitch-grader`. Then:

```bash
cd "/Users/nickv/ClaudeCode Projects/Pitch Training Mobile/grader" && npm test && npm run typecheck && vercel deploy --prod --yes
```

Expected: a production URL. The alias can lag a few seconds; retry before believing a 404.

- [ ] **Step 5: [Nick] Check the deployment** (spec §12 item 5)

In his terminal, with `APP_TOKEN` exported and `PT_URL` set to the production URL:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "$PT_URL/api/decks/<slug>/deck.json"
curl -s -H "Authorization: Bearer $APP_TOKEN" "$PT_URL/api/decks/<slug>/deck.json" | head -c 120; echo
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" -H "Authorization: Bearer $APP_TOKEN" "$PT_URL/api/decks/<slug>/clips/$(curl -s -H "Authorization: Bearer $APP_TOKEN" "$PT_URL/api/decks/<slug>/deck.json" | python3 -c "import json,sys;print(next(c['prompt_clip'] for c in json.load(sys.stdin)['cards'] if c.get('prompt_clip')).split('/',1)[1])")"
curl -s -X POST -H "Authorization: Bearer $APP_TOKEN" -H "Content-Type: application/json" --data @test/live/sample-request.json "$PT_URL/api/grade" | head -c 200; echo
```

Expected, in order: `401`; JSON starting `{"slug"`; `200` with a size over 10000 bytes; JSON starting `{"verdict"`.
If the second line is a 404 JSON, the deck folder was not uploaded: stop and report. The fallback (a private Vercel Blob store) is a design change for Nick to approve.

- [ ] **Step 6: Report**

Tell Nick the production URL, the four check results, and that Plan 3 (iOS app) can be written once the spike report is in.
