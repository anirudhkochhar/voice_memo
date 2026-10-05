# Structured Voice Memos — Implementation Stages

Companion to `focus-log-design_1.md`. **§ numbers refer to that doc.** This is a rough roadmap, not a spec. Expect it to change as we learn.

**Rule for every stage:** it ships as a build you can install (TestFlight / Play internal track) and use day to day, it is small enough to review in one sitting, and it ends with a short feedback note that shapes the next stage.

---

## Overview

| # | Stage | Track | You can… |
|---|---|---|---|
| 1 | Templates + typed rows | Core | create your own collections, fill rows by typing, see a table, export CSV |
| 2 | Richer columns + safe schema changes | Core | use every column type, conditions and auto values, and change a collection that already has rows |
| 3 | Typed UX polish | Core | log quickly and comfortably without voice |
| 4 | Guided flow (typed) | Core | fill a row one question at a time, with drafts you can resume |
| 5 | Native on-device voice | Voice | answer and hear questions by voice using the OS (iOS / Android) speech engines |
| 6 | Quick launch | Voice | start a recording from Siri, the Action Button, widgets or Android shortcuts |
| 7 | Cloud STT / TTS (your own key) | Voice | use third-party speech providers |
| 8 | Local Whisper | Voice | transcribe on-device with a downloaded model |
| 9 | Voice polish | Voice | rely on voice day to day |
| 10 | LLM foundation + platform models | AI | get help on answers the parsers can't resolve, using Apple FM / Gemini Nano |
| 11 | Free & Hybrid modes | AI | speak one memo and have the app fill the row |
| 12 | Local LLMs | AI | run downloaded open-weights models |
| 13 | Cloud LLMs + custom endpoint | AI | use your own keys or an Ollama / LM Studio server |
| 14 | Medium-tier AI features | AI | describe a template in words; get themes and summaries |
| 15 | Quality loop + AI polish | AI | see per-engine quality and re-extract rows with a newer setup |
| 16 | Hindi / Hinglish | Language | log in Hindi or Hinglish |

**Templates and rows are the core of the app.** Everything later (guided flow, voice, AI) is just another way to fill the same row of the same template.

Stages 1–4 have no STT, TTS or LLM. Stages 5–9 add voice but no LLM. The order within a track can shift. Stage 6 (quick launch) is a good candidate to pull earlier or later.

---

## Core: no voice, no AI

### Stage 1: Templates + typed rows
The essence of the app: define columns, fill rows. No session engine, no guided flow. You just type into a form.
- Expo + TypeScript + Expo Router, SQLite + Drizzle (§9, §10)
- Data model: Collection / Column / Row, keyed by stable column id from day one (§4)
- Simple template builder (§6 item 1, §7.7): create a collection, then add, name and reorder columns, and set `required` and `prompt` (shown as the field label/hint for now)
- Basic types: text, long_text, number, date, duration, boolean, single_select (§4.2), with the options for single_select managed in the builder
- Daily task log as a starter template you can copy and edit (§6 item 3), without its Blocker condition for now
- Row entry: one form with all columns, typed input per type (pickers where natural). Save is blocked while required fields are empty
- Edit and delete rows
- Screens: Collections, Template builder, Row form, simple Table (list) view (§7.1, 7.4, 7.7)
- CSV export (§8.5)
- Limited schema changes once rows exist: rename, reorder and add columns only. Other changes come in Stage 2

**Usable when:** you can set up whatever you want to track, log rows by typing, and export to a spreadsheet.

### Stage 2: Richer columns + safe schema changes
- Remaining column types except `location` and `audio_only` (§4.2): currency, time, multi_select, rating
- `auto` values, `condition` (§5.5), `validation`, `description`
- Deterministic parsers, so free-typed values like "1h30" or "yesterday" work as well as pickers (§9 Parsing)
- Inline "add 'Website' as a new option?" from the row form (§5.6)
- Schema versioning and the full change rules table (§6 *Schema changes*): archive instead of delete, type conversion, condition guards

**Usable when:** you can model anything you can describe as columns, and edit a collection without breaking old rows.

### Stage 3: Typed UX polish (based on feedback from 1–2)
Placeholder. Likely candidates:
- Faster entry: smart keyboards per type, last-used values, option chips, one-tap "same as last time"
- Table view: sort, filter, search, inline edit (§7.4); Row detail (§7.5)
- Full backup and restore (§8.5)
- Accessibility basics (§8.7): labels, font scaling, list/card view
- `location` column type

**Usable when:** logging by typing feels quick, not like a chore.

### Stage 4: Guided flow (typed)
The step-by-step backbone that voice and AI will plug into later. Still typing only.
- Session engine as a state machine, minus the speaking/listening states (§5.1): question queue, re-evaluated after every answer
- Guided mode: one question at a time from each column's `prompt` (§5.3), with skip / back / cancel as buttons
- Draft after every answer, Drafts screen to resume (§5.1, §7.6)
- Review card before save (§5.7, without audio playback)
- Collection `mode`: form (from Stage 1) or guided

**Usable when:** you can fill a row question by question, and an interrupted entry is never lost.
**Keep in mind:** shape the session engine so that voice plugs in as another input/output, with no rewrite.

---

## Voice: STT / TTS, still no LLM

### Stage 5: Native on-device voice
- `SttProvider` / `TtsProvider` interfaces (§8.1) with `native` providers in **forced on-device mode**, plus availability reasons ("download the English speech pack")
- Turn-taking: speak → gap → listen, silence end, tap to skip the prompt (§5.2)
- Capture strategy with one mic owner; the audio clip is kept per answer (§8.1 *Capture*, §4.3 `recordings`, `transcripts`)
- Voice commands with the whole-utterance grammar (§5.3)
- Interruptions, background → pause (§5.1)
- Tap a value to hear its source audio (§5.7, `field_sources`)
- Typed input stays available for every question

**Usable when:** you can complete a row hands-free on supported devices, offline.

### Stage 6: Quick launch
- iOS App Intents / Siri / Action Button, widgets; Android app shortcuts and the quick-settings tile (§7.9)

**Usable when:** "Hey Siri, record daily task log" opens straight into the session.

### Stage 7: Cloud STT / TTS (bring your own key)
- Settings → Providers & keys: secure storage, consent screen, usage limits, key redaction, error mapping (§8.3)
- One or two STT and TTS providers to start (e.g. OpenAI, Deepgram, ElevenLabs) (§8.1 table)
- Provider chains with fallback; collection `privacy: on-device-only` (§2.6, §4.1); opt-in `native-network`

**Usable when:** you can pick a better or different voice/STT, knowing exactly what leaves the device.

### Stage 8: Local Whisper
- Local model manager, Whisper only for now: catalog, resumable download, checksum, storage view (§8.2 *Local model manager*)
- Record-then-transcribe capture fallback (§8.1)
- Row detail: **re-transcribe with…** as a proposed diff (§2.5, §7.5)

**Usable when:** you get better on-device transcription, and can re-run old clips.

### Stage 9: Voice polish (based on feedback from 5–8)
Placeholder. Likely candidates:
- Silence/timeout tuning, Bluetooth / HFP hint (§5.2)
- Vocabulary hints, TTS prompt caching (§8.1 *Other behaviour*)
- "Extra info in one answer" offered as suggestions (§5.3)
- Screen reader interplay, haptic listen cues (§8.7)
- Audio storage view and opt-in cleanup (§8.1 *Audio format*)

**Usable when:** voice is the default way you log, not a demo.

---

## AI: LLM

### Stage 10: LLM foundation + platform on-device models
- `LlmEngine`, `LlmRouter` with a call-site tier, tier chains, runtime resolution (§8.2)
- Validation pipeline, including evidence checks and deterministic flags (§8.2 *Validation pipeline*)
- Engines: `apple-fm`, `gemini-nano`
- Tasks: `normalize` (shown as a suggestion, §5.3) and optional `phrase_question`
- `extraction_meta` recorded on rows (§4.3)
- Settings → AI models, with the "nothing on-device available" explanation (§8.2)

**Usable when:** answers like "pretty much wrapped up" become a "Did you mean Done?" suggestion, on capable devices. Everything else falls back to Stage 1–9 behaviour.

### Stage 11: Free and Hybrid modes
- `extract` task, collection `mode` (§4.1, §5.4)
- Review card flags with reasons (§5.7), condition re-evaluation after merges (§5.5)
- Missing columns asked Guided-style; the memo is kept for later extraction when no engine is available

**Usable when:** you can say "fixed the export bug, about an hour and a half, done" and only get asked for what's missing.

### Stage 12: Local LLMs
- `local` engine via `llama.rn` with grammar-constrained output (§8.2 *Engines*)
- Extend the model manager to LLMs: device fit checks, memory budget, model sequencing with Whisper, add by URL with the sandboxed template (§8.2)
- Presets: All on-device (default), Balanced, Best quality, Custom

**Usable when:** devices without Apple FM / Gemini Nano get AI features fully offline.

### Stage 13: Cloud LLMs + custom endpoint
- `cloud` (Anthropic, OpenAI, Google…) and `custom-endpoint` engines, reusing Stage 7's keys/consent (§8.3)

**Usable when:** you can put a stronger model in any tier with your own key, or point at your own server.

### Stage 14: Medium-tier AI features
- `draft_template`: "Describe it" in the template builder (§6 item 2)
- `themes` / `summary` on a collection (§8.2 task table)

**Usable when:** you can create a collection by describing it and get a summary of your logs.

### Stage 15: Quality loop + AI polish (based on feedback from 10–14)
- Correction log and per-engine stats, Settings → Quality (§8.6)
- **Re-extract with…** as a proposed diff, original or current schema (§6, §7.5)
- Developer eval runner with sample/synthetic memos (§8.6, `eval/`)
- Prompt tweaks and engine defaults driven by the data

**Usable when:** you can see which engine works best for you and upgrade old rows safely.

---

## Language

### Stage 16: Hindi / Hinglish
- `language`, `output_script`, transliteration (§4.1, §8.4)
- Hindi/Hinglish parsers for numbers, dates, durations; option aliases
- Language-aware provider/engine selection; Whisper or network STT for Hinglish

**Usable when:** you can log a full row in Hinglish and get clean, consistent values.

---

## Notes
- **Can move earlier if needed:** backup/restore (if you switch phones), quick launch, `location`.
- **Each "polish" stage (3, 9, 15) is deliberately open.** Fill it in from real usage of the previous stages.
- **Not planned:** anything in §11 *Not part of the design*.
