# Structured Voice Memos — Design

A cross-platform (iOS + Android) Expo app that turns voice memos into **structured rows**. You define a table's columns once. Each time you record, the app walks you through the columns step by step until the row is complete. It keeps the audio as evidence and adds the row to a local table you can export.

This document describes **what we want and the desired end state**. It is not an implementation plan or release schedule.

---

## 1. Vision

- **Any structure, by voice.** If you can describe it as columns, you can capture it by talking: daily task logs, behaviour tracking, expenses, inspections, visits, observations.
- **Complete by design.** The app knows which columns are still empty and asks for them. A row is never saved half-filled unless you explicitly skip a field.
- **Private and local-first.** Everything works on the phone with no account, no server and no network: recording, transcription, AI, storage and export.
- **Swappable intelligence.** Speech recognition, voice output and LLMs are pluggable. The defaults are on-device. You can bring a third-party provider with your own API key, or download a different local model.
- **Trustworthy data.** The AI only points at what you said. Deterministic code derives, validates, decides and saves. Every value can be traced back to the audio it came from.

---

## 2. Design principles

1. **Guided first.** Step-by-step questioning is the default and the backbone. Free speech and AI extraction are enhancements on top of it, and the app always falls back to Guided.
2. **Deterministic core, narrow LLM.** The LLM has a small, closed list of jobs with fixed contracts. Completeness, conditions, type conversion, validation, question order and storage are plain code.
3. **Validate everything the model outputs.** Model output passes schema checks, evidence checks and business rules before it touches a row.
4. **Everything behind an interface.** STT, TTS and LLM each sit behind an abstraction with interchangeable backends. Swapping one is a settings change.
5. **Audio is the source of truth.** Recordings are always kept with their row, and any row can be re-transcribed or re-extracted later. Re-runs produce a proposed diff; they never silently overwrite the row.
6. **Local by default, cloud by choice.** Nothing leaves the device unless you configure a network provider. That includes OS speech services that may use the network. Collections can forbid network providers entirely.

---

## 3. Core concepts

```
Collection (columns)  ──►  Recording session  ──►  One complete row
 "Daily task log"           App: "What did you work on?"      Date | Task          | Project | Time  | Status
  Task, Project,            You: "Fixed the export bug"       5 Oct| Fix export bug| App     | 1h30m | Done
  Time spent, Status…       App: "How long did it take?" …
```

- **Collection:** a named table with a versioned, ordered list of column definitions (a template).
- **Column:** stable id, name, type, required, question, options, condition, validation.
- **Row:** the result of one recording session. Values, audio clips, transcripts and provenance, all keyed by column id.

---

## 4. Data model

### 4.1 Collection
| Property | Description |
|---|---|
| `id`, `name`, `icon`, `color` | e.g. "Daily task log" |
| `schema_version` | incremented on every column change; each row records the version it was captured with |
| `columns[]` | ordered column definitions (including archived ones) |
| `mode` | `guided` (default) / `free` / `hybrid` |
| `language` | `en`, later `hi`, `hi-en` (Hinglish) |
| `output_script` | script values are stored in (e.g. `latin`, `devanagari`); normalised by code after STT |
| `privacy` | `default` or `on-device-only` (blocks every provider that may use the network) |
| `voice` | TTS voice, speed, prompts spoken on/off |
| `intro_prompt` | optional hint shown when recording starts |

### 4.2 Column
| Property | Description | Example |
|---|---|---|
| `id` | stable, never reused; all row data is keyed by it, so renames are safe | `col_7f3a` |
| `name` | header (can be renamed freely) | `Status` |
| `type` | `text`, `long_text`, `number`, `currency`, `boolean`, `date`, `time`, `duration`, `single_select`, `multi_select`, `rating`, `location`, `audio_only` | `single_select` |
| `required` | must be filled or explicitly skipped, **evaluated only while the column is active** (its condition holds) | `true` |
| `prompt` | question asked in Guided mode / when missing | "What's the status?" |
| `options` | for select types; each option has a stable id and label. New options are only added after you confirm (§5.6) | `Done`, `In progress`, `Blocked` |
| `condition` | the column is active only if another column matches; must reference an active column | active if `Status = Blocked` |
| `auto` | pre-filled value: session start date/time, location, last-used value. Shown and editable on the review card; never asked in Guided mode | session date |
| `validation` | min/max, pattern, unit | `Time spent ≤ 24h` |
| `description` | hint for interpreting speech | "Duration; '1 and a half' = 1h30m" |
| `archived` | soft-deleted; hidden but data kept | `false` |

### 4.3 Row
| Field | Description |
|---|---|
| `id`, `collection_id`, `schema_version` | — |
| `session_started_at`, `timezone` | reference point for relative dates ("yesterday", "this morning"), so a session crossing midnight stays consistent |
| `location?` | if enabled |
| `values{col_id}` | typed values |
| `status` | `draft` / `complete` |
| `skipped[col_id]` | explicitly skipped |
| `locked[col_id]` | confirmed or edited by you; never changed without your approval |
| `flags{col_id}` | review flags: `inferred`, `ambiguous`, `correction`, `negation`, `low_confidence`, `unverifiable_type` |
| `recordings[]` | audio clips, each linked to a column (Guided / follow-up answer) or to the whole memo (Free / Hybrid opening) |
| `transcripts[]` | per clip: text, word timings, language, STT provider, on-device or network |
| `field_sources{col_id}` | clip and word span(s) the value came from (tap a cell → hear it) |
| `extraction_meta` | when an LLM was involved: tier, engine, model + version, prompt version, repairs, timings, skipped engines + reasons |

---

## 5. Recording experience

### 5.1 Session engine (deterministic state machine)

```
idle → preparing → [ speaking → listening → processing ]* → reviewing → saved
                         ▲            │             │
                         └── re-ask ◄─┘             └─► (next column | follow-up | review)
any state ──interruption──► paused ──resume──► re-ask current question
any state ──cancel (confirmed)──► discarded (or kept as draft)
```

- **Draft after every answer:** the row is saved as a `draft` after each processed answer. A crash, OS kill or force-quit loses at most the answer in progress, and Drafts lets you resume.
- **Interruptions:** phone calls, Siri, alarms and audio route changes (e.g. headphones unplugged) pause the session. On resume, the current question is asked again.
- **Background / screen lock:** the session pauses. Recording does not continue in the background, so the mic is never left open unattended.
- **Question queue:** the session recomputes it after every merge and every edit. It contains active, required, unfilled, unskipped columns in column order (see §5.5).

### 5.2 Turn-taking (half-duplex)
1. **Speak:** play the prompt with TTS. The mic is closed.
2. **Gap:** wait for TTS to finish, plus a short gap (around 300 ms). This avoids the prompt echoing into the mic on the speaker.
3. **Listen:** open the mic. The answer ends after a configurable silence (default around 1.5 s), when you tap "done", or at a maximum length.
4. **No barge-in by default.** Tapping the mic button skips the rest of the prompt and starts listening.
- **Audio session:** on iOS, a play-and-record session is held for the whole session, so switching between speaking and listening doesn't glitch. Route changes are handled as interruptions.
- **Bluetooth headsets:** supported. In call-mode (HFP) the mic quality drops, and the app shows a hint suggesting the phone's mic instead.

### 5.3 Guided mode (default)
The app asks each active column's `prompt` in order and records **one clip per answer**.
```
App: "What did you work on?"   → You: "Fixed the CSV export bug"
App: "Which project?"           → You: "The app"
App: "How long did it take?"    → You: "About an hour and a half"
App: "Status?"                  → You: "Done"
→ Review card → Save
```
- Questions are spoken and shown. Spoken prompts can be turned off per collection.
- **Voice commands:** "skip", "back", "repeat", "cancel". An answer counts as a command **only if the whole utterance matches the command grammar**, so "cancel the meeting" is an answer. "Cancel" always asks for confirmation.
- **Parsing an answer:** deterministic parsers first. If they can't resolve it, the `normalize` LLM task helps (§8.2). If that also fails, the app asks again once, then offers skip. There's a text-input fallback for every question.
- **Extra information in one answer** (e.g. "fixed the bug, took an hour, done"): only the current column is filled. Values for other columns are offered as **suggestions** when those columns come up ("Use 'an hour'?"). They're never filled silently.
- **No LLM is required** for Guided mode to work.

### 5.4 Free and Hybrid modes
- **Free:** you speak naturally in one recording. The `extract` task finds evidence for every column, and the app then asks only for required active columns that are still missing, Guided-style, one per turn.
- **Hybrid:** starts with an open "tell me about it" recording, then switches to Guided for whatever is missing.
- **Recordings in both modes:** one memo clip, plus one clip per follow-up answer.

### 5.5 Conditions
- Conditions are re-evaluated after **every** merge (extraction or answer) and **every** edit on the review card.
- A condition is satisfied only by a value that **passed validation and isn't flagged**. A flagged value has to be confirmed first.
- Values in columns that become inactive are hidden and excluded from the row (kept in the draft until save, in case you switch back).
- Required columns that become active are added to the question queue. On the review card, they're highlighted as missing.

### 5.6 Select options
- Unmatched spoken values are first fuzzy-matched against existing options.
- If nothing matches, the app asks: **"Add 'Website' as a new project?"** (yes / pick existing / skip). The LLM can never create options.
- Option ids are stable, so renaming an option relabels all rows.

### 5.7 Review card
- Every active value is shown and editable, including `auto` values (e.g. change the date to yesterday).
- Missing and flagged values are highlighted, and each flag shows its reason (e.g. "inferred from 'finished it'", "you corrected yourself").
- Tap a value to play its source audio.
- Save is blocked while required active columns are missing (unless skipped) or flagged values are unconfirmed. Save as draft is always possible.

---

## 6. Templates

Ways to create a collection:
1. **Manual builder:** add, reorder, type and configure columns.
2. **Describe it** ("I want to log gym sessions: exercise, sets, reps, weight"). The `draft_template` LLM task proposes columns, and you review and edit them.
3. **Starter templates.** The first is the daily task log:

| Column | Type | Required | Prompt | Notes |
|---|---|---|---|---|
| Date | date | auto | — | session start date; editable on review |
| Task | text | yes | "What did you work on?" | |
| Project | single_select | yes | "Which project or area?" | new options added after confirmation (§5.6) |
| Time spent | duration | yes | "How long did it take?" | |
| Status | single_select | yes | "What's the status?" | Done / In progress / Blocked |
| Blocker | text | yes, when active | "What's blocking it?" | active only if Status = Blocked |
| Next step | text | no | "Anything to do next?" | |
| Notes | long_text | no | "Anything else?" | |

*(Starting point. Columns are fully editable.)*

### Schema changes
Every change bumps `schema_version`. All row data is keyed by column id.

| Change | Effect on existing rows |
|---|---|
| Add column | existing rows: empty (not "missing"), so old rows don't become incomplete |
| Rename column / option | label change only; data untouched |
| Reorder | display/question order only |
| Change type | deterministic conversion where possible (e.g. text → single_select via option matching, number → text). Values that fail are kept as the original text and flagged for review |
| Delete column | archived: hidden, data kept, restorable |
| Delete option | option archived; rows keep it, and it's shown as archived |
| Condition referencing an archived column | not allowed; the builder asks you to fix or remove the condition first |

**Re-extracting an old row** uses its original `schema_version` by default, or the current schema if you choose. Either way, the result is a proposed diff (§7).

---

## 7. Screens

1. **Collections (home):** each collection has a big record button, row count and storage used.
2. **Record:** current question, waveform, live transcript, and skip / back / pause / done controls.
3. **Review card:** see §5.7.
4. **Table view:** spreadsheet-style grid with sort, filter, search and inline edit; also a list/card view (the accessible default for screen readers).
5. **Row detail:** values, all clips with transcripts, flags, provenance, and **re-transcribe / re-extract with…**. A re-run never writes to the row: it shows a **proposed diff** (old vs new per column) that you accept per field or all at once. Locked values are shown but untouched unless you approve.
6. **Drafts:** incomplete or interrupted sessions; resume at the current question.
7. **Template builder:** includes schema change warnings (§6).
8. **Settings:**
   - **AI models:** engine chain per tier, presets, local model downloads, memory/storage
   - **Speech:** STT and TTS providers, voices, on-device language packs
   - **Providers & keys:** bring your own key, consent, usage limits
   - **Data:** export, full backup / restore, audio storage
   - **Quality:** per-engine stats
   - Language
9. **Quick launch:** iOS Siri / App Intent and Action Button ("Record daily task log"), home-screen widgets, Android app shortcuts and quick-settings tile. These open straight into the recording screen. The guided Q&A always runs in-app.

---

## 8. Architecture

```
                    ┌──────────────── App (all on-device) ────────────────┐
 Mic ─► Capture ─►  │  STT layer ─► Session engine (deterministic) ─► SQLite + audio files
                    │                 │   ▲                                    │
 Speaker ◄─ TTS layer ◄───────────────┘   │                                    ▼
                    │            LLM layer (router(tier) → engines)     Table / CSV / backup
                    │            + validation pipeline                         │
                    └──────────────────────────────────────────────────────────┘
                     optional network providers (your own key) ─► third-party STT / TTS / LLM APIs
```

The **session engine** (§5.1) owns question order, conditions, completeness, turn-taking, retries, and when to call STT, TTS or the LLM.

### 8.1 Voice layer (STT + TTS)

```ts
interface SttProvider {
  id: string;                               // "native", "whisper-local", "deepgram", ...
  capabilities: { streaming: boolean; languages: string[]; vocabularyHints: boolean;
                  wordTimestamps: boolean;
                  network: "never" | "maybe" | "always" };  // drives privacy rules
  isAvailable(lang: string): Promise<Availability>;          // reason: missing language pack, no key, …
  startStream(opts: SttOptions): SttStream;                  // live transcript (partial / final)
  transcribeFile(uri: string, opts: SttOptions): Promise<Transcript>;
}

interface TtsProvider {
  id: string;
  capabilities: { network: "never" | "maybe" | "always"; streaming: boolean;
                  languages: string[]; voices: Voice[] };
  speak(text: string, opts: TtsOptions): Promise<void>;     // resolves when playback finishes
  synthesize?(text: string, opts: TtsOptions): Promise<AudioFile>;
  stop(): void;
}

type Transcript = { text: string; language: string; provider: string; onDevice: boolean;
                    words?: { w: string; start: number; end: number }[]; confidence?: number };
```

| | STT | TTS |
|---|---|---|
| **Default** | `native` in **forced on-device mode**: iOS Speech with on-device recognition required; Android with an offline language pack | `native`: iOS AVSpeechSynthesizer / Android TextToSpeech (on-device voices) |
| On-device alternative | Whisper (whisper.cpp) with a downloadable model; transcribes saved clips; useful for Hindi/Hinglish | — |
| Network (your own key) | e.g. OpenAI, Deepgram, AssemblyAI, Google, Azure, ElevenLabs | e.g. ElevenLabs, OpenAI, Google, Azure |

**On-device guarantee**
- The OS recognisers can silently use the network (Apple's servers by default, Google's cloud when no offline pack is installed). The `native` provider therefore always requests **on-device-only** recognition.
- If on-device recognition isn't possible for a language, `native` reports itself as unavailable and names the reason, e.g. "Download the English offline speech pack". It never quietly switches to the network.
- A separate, opt-in `native-network` provider exists for users who accept OS cloud recognition. It is classed as a network provider.
- `on-device-only` collections can only use providers with `network: "never"`.

**Capture: one mic owner per session**
- Running a recorder and an OS recogniser on the mic at the same time isn't reliable, so each session has exactly one capture owner:
  - **Preferred: the recogniser owns the mic and saves its own audio** (iOS: tap the shared audio session's buffers into the clip file; Android: the recogniser's audio persistence where the OS supports it).
  - **Fallback: record, then transcribe.** `expo-audio` records the clip, and on-device Whisper (or another file-capable provider) transcribes it right after the answer ends. There's no live transcript in this mode, and silence detection uses the recorder's levels.
- The app chooses the strategy per device and platform at startup and records it in transcripts.
- **Audio format:** AAC (or Opus), mono, 16–24 kHz, roughly 1 MB per few minutes. Storage per collection is shown, and any cleanup is explicit and opt-in (e.g. "delete audio older than…").

**Other behaviour**
- **Selection:** follows the configured chain (collection override → app setting). Fallback only goes to providers in that chain that satisfy the collection's privacy rule.
- **Vocabulary hints:** option labels and recent values are passed to providers that support phrase boosting.
- **TTS caching:** fixed column prompts are synthesised once per voice and cached.

### 8.2 LLM layer

#### Tier: one source of truth
- The tier is a **required parameter of every `LlmRouter`**, chosen at the call site in code. A task has no tier of its own.
- The table below shows which tier each call site uses. The user never changes it.

#### What the LLM is allowed to do: a closed task list
Each task has a fixed contract: input → output schema → validator → fallback.

| Task | Call-site tier | Purpose | Output | Fallback |
|---|---|---|---|---|
| `normalize` | `ultraSmall` | Point to the meaningful part of one Guided answer that the parsers couldn't resolve, or map it to an option | `{ evidence, option_id?, polarity? }` | ask again → skip |
| `extract` | `budget` | Free/Hybrid: point to evidence for each column in a memo | `{ col_id: { evidence[], option_ids[]?, polarity? } }` | Guided questions for missing columns |
| `phrase_question` | `ultraSmall` | Optional natural rewording of a follow-up | `{ question }` | `column.prompt` |
| `draft_template` | `medium` | Propose columns from a description | `Column[]` (no ids; ids are assigned by code) | manual builder |
| `themes` / `summary` | `medium` | Group free-text values, summarise a collection | structured themes / text | feature hidden |

**The LLM only points; code derives values.**
- The LLM returns **evidence spans** (quotes from the transcript), **existing option ids**, and **polarity** for booleans. It never returns typed or computed values.
- Dates, numbers, durations and currency are always parsed by code **from the evidence span**, against `session_started_at` and `timezone`. "Tuesday" can't arrive as "2026-10-06" from the model.

**The LLM never:**
- produces typed or converted values, or does date maths
- decides completeness or evaluates conditions
- creates options or column ids
- chooses the next question
- writes to storage

#### Responsibility split
| LLM | Deterministic code |
|---|---|
| Quote the words that answer a column (**evidence**) | Build the output schema from the active columns |
| Pick **existing** option ids (with evidence for each) | Derive typed values from evidence with parsers |
| Indicate polarity for booleans ("not blocked") | Validate: schema, option ids exist, ranges, conditions |
| Reword a question (optional) | Detect ambiguity, corrections and negations; set flags |
| Draft templates, themes, summaries (non-critical) | Decide what's missing and what to ask next; merge; save |

#### Validation pipeline (every LLM call)
```
input
 ├─► 1. Rules pre-pass (no LLM): parsers, exact option matches, yes/no, negation words
 │      everything resolved? → done, skip the LLM
 ▼
 2. Budget check: prompt + schema must fit the engine's context; otherwise split by column
    group or skip the engine (reason logged)
 3. LLM call, with structured output enforced as strongly as the engine allows
 4. JSON parse ............................. fail → repair once
 5. Schema check (generated per collection)  fail → repair once
 6. Evidence check: every span must appear in the transcript (fuzzy); option ids must exist;
    multi_select needs evidence per option. A value with no valid evidence is dropped
 7. Deterministic derivation: parsers run on the evidence span ... fail → stays missing
 8. Flagging (deterministic):
      - option label not contained in its evidence        → `inferred`
      - several candidate spans for one column             → `ambiguous`
      - correction marker near the span ("no wait", "I mean", "actually", "sorry")
                                                           → `correction`
      - negation near a boolean / select span              → `negation` (unless polarity agrees with code)
      - types the evidence check can't vouch for (rating, long_text summaries)
                                                           → `unverifiable_type`
 9. Business rules: validation ranges, conditions (only unflagged valid values satisfy conditions)
10. Merge (locked values untouched) → re-evaluate conditions → recompute question queue
```
- **Repair:** one retry with the validator's errors. After that, the next engine in the router's chain is tried. When the chain is exhausted, the task's fallback is used.
- **Confidence is computed by code:** evidence quality, parser success, exact vs inferred option, flags, engine. Flagged or low-confidence values must be confirmed on the review card before save.
- **Guided `normalize`:** the whole answer is the transcript, so the evidence check is weak. The result is always shown as a suggestion in the Guided flow ("Did you mean In progress?") rather than filled silently.

#### `LlmEngine` and `LlmRouter`
```ts
type LlmTier = "ultraSmall" | "budget" | "medium" | "frontier";

interface LlmEngine {
  id: string;                     // "apple-fm", "gemini-nano", "local:qwen3-1.7b-q4", "anthropic:claude-haiku"
  kind: "apple-fm" | "gemini-nano" | "local" | "cloud" | "custom-endpoint";
  capabilities: { structuredOutput: "native-schema" | "grammar" | "json-schema" | "prompt-only";
                  network: "never" | "always"; contextTokens: number; languages: string[] };
  isAvailable(): Promise<Availability>;
  // Availability = { ok: true } | { ok: false, reason: "unsupported-device" | "os-feature-disabled"
  //   | "region-or-language" | "model-not-downloaded" | "downloading" | "no-key" | "offline"
  //   | "foreground-only" | "memory" }
  warmup?(): Promise<void>;
  generate(req: { prompt: PromptBundle; schema: JsonSchema; maxTokens: number;
                  signal?: AbortSignal }): Promise<{ text: string; latencyMs: number }>;
  unload?(): Promise<void>;
}

class LlmRouter {
  constructor(opts: { tier: LlmTier; escalateTo?: LlmTier });  // tier required, fixed for its lifetime
  run<T>(task: TaskId, input: TaskInput, ctx: { collection: Collection }): Promise<TaskResult<T>>;
}

// The tier is chosen in code at the call site, never by the user:
const router = new LlmRouter({ tier: "budget" });
await router.run("extract", input, { collection });
```
- Engines only turn text into text. **Prompts, schemas and validation live in the task layer**, so they're identical across engines.
- Structured output is enforced by each engine's strongest mechanism:
  - Apple: `DynamicGenerationSchema`
  - local: grammar-constrained decoding
  - cloud: JSON schema / tool use
  - Gemini Nano: prompt-only

  The same validators run regardless.

#### Engines
| Engine | Runs | Notes |
|---|---|---|
| `apple-fm` | On-device (iOS) | Apple Foundation Models. Needs iOS 26+ on an Apple Intelligence device with the feature enabled and a supported region/language. Small context (~4K tokens), so long memos or schemas are split or skipped |
| `gemini-nano` | On-device (Android) | ML Kit GenAI Prompt API. Supported devices only, foreground only, may need a feature download. Prompt-only structured output |
| `local` | On-device (both) | Downloaded open-weights model (GGUF) via `llama.rn` / llama.cpp, with grammar-constrained output |
| `cloud` | Provider API | Anthropic, OpenAI, Google…, called directly with **your own key** |
| `custom-endpoint` | Your server | Any OpenAI-compatible URL (e.g. Ollama / LM Studio), optional key |

#### Tiers
- **Code decides how capable a model a call needs** (the router's tier).
- **The user decides which engines serve each tier**, in Settings → AI models. Each tier holds an ordered engine chain: the first available engine is used, and the rest are fallbacks.

| Tier | Meant for |
|---|---|
| `ultraSmall` | Tiny single-field interpretation, phrasing |
| `budget` | Routine structured extraction |
| `medium` | Generation and multi-step reasoning (templates, themes, summaries) |
| `frontier` | The hardest tasks, e.g. cross-row analysis |

```yaml
# Default preset: All on-device
tiers:
  ultraSmall: [apple-fm, gemini-nano, local:<small model>]
  budget:     [apple-fm, gemini-nano, local:<small model>]
  medium:     [local:<larger model>, apple-fm, gemini-nano]
  frontier:   [local:<larger model>]

# Example custom setup with your own keys
tiers:
  ultraSmall: [apple-fm, gemini-nano]
  budget:     [apple-fm, anthropic:claude-haiku]
  medium:     [anthropic:claude-sonnet]
  frontier:   [anthropic:claude-opus]
```
- Presets: **All on-device (default)**, Balanced, Best quality, Custom.
- An engine can sit in several tiers. A tier may be empty, in which case its calls use the task fallback.
- **Runtime resolution:** take the tier's chain → drop engines that are unavailable (reason logged) → if the collection is `on-device-only`, drop network engines → drop engines lacking the collection's language → try each in order → the task fallback.
- **Escalation** to a higher tier happens only if the call site declares `escalateTo` in code.
- If observed quality for an engine on a tier is poor (§8.6), Settings shows a hint, but it doesn't block the choice.

**When nothing on-device is available** (no Apple Intelligence or Gemini Nano, no downloaded model):
- The app is fully usable in **Guided mode**, with parsers only.
- Free and Hybrid fall back to Guided after the opening memo. The memo is kept, so it can be extracted later as a proposed diff.
- Settings and the first-run screen explain why (using the `Availability` reason) and suggest a model that fits the device.

#### Local model manager
- **Users can download models.** The catalog shows models that fit the device (RAM and storage checks), with size, licence and suggested tiers. The same manager handles Whisper models.
- **Catalog:** a static manifest file with checksums and a signature. It's fetched without telemetry and can list new models without an app update.
- **Add a model by URL** (GGUF + chat template). The chat template is treated as untrusted input: it's rendered in a sandboxed, restricted template engine and validated with a test prompt before use.
- Downloads are resumable, with progress, a Wi-Fi-only option and checksum verification.
- Models are stored in app storage and excluded from OS backups. Settings shows the space used with a delete button.
- **Memory budget:** only one large model (LLM or Whisper) is loaded at a time. The session engine sequences them: transcribe, unload if needed, then extract. Models are unloaded on memory pressure or when the app goes to the background.
- Model weights are downloaded as data, not executable code.

### 8.3 Network providers & keys (bring your own key)
- Third-party STT, TTS and LLM providers are optional and configured in Settings → Providers & keys.
- Keys are entered by the user, stored in the device keychain (`expo-secure-store`), and used to call the provider **directly from the device**. No backend or account is needed.
- **Consent:** before a provider receives data for the first time, the app shows exactly what will be sent (audio / transcript text / prompts) and to whom, and asks for explicit consent. Each collection shows which providers may receive its data.
- **Key safety:** keys are never written to logs, crash reports, exports or backups, and they're redacted in any error message.
- **Usage limits (optional):** per-provider caps on requests per day and audio minutes, to prevent runaway costs.
- **Errors:** invalid key (401), quota or rate limits (429) and outages map to clear messages and trigger the normal chain fallback.
- `on-device-only` collections never use network providers.

### 8.4 Languages
- English first. The design accommodates **Hindi** and **Hinglish** (code-mixed):
  - `language` and `output_script` are set per collection and passed to STT, TTS and LLM.
  - STT output is normalised by code to the collection's script (e.g. Devanagari → Latin with a deterministic transliterator). This keeps values consistent even when a model switches scripts.
  - Every engine and provider declares its supported languages, and the router skips unsupported ones.
  - Hindi/Hinglish needs **dedicated parsers** for numbers, dates and durations (e.g. "dedh ghanta" = 1h30m, "kal" depends on context and is flagged). These are separate from the English date parser.
  - Hinglish is likely to need Whisper-class or network STT. The STT abstraction makes that a configuration choice.
  - Select option labels stay as defined (e.g. English), and spoken Hindi is matched to them via aliases.

### 8.5 Offline, storage, export & backup
- Everything works offline with the on-device defaults.
- Storage is SQLite (rows, collections, schema versions, metadata) plus audio files on the device. Nothing syncs anywhere.
- **Fallback** always follows the configured chains (§8.1, §8.2). If nothing in the chain can process a memo right now, it's saved as a **draft**.

**CSV export**
| Aspect | Rule |
|---|---|
| Encoding | UTF-8 with BOM (so Excel shows Hindi and other scripts correctly); RFC 4180 quoting |
| Columns | one per active column, by current name; optional metadata columns (row id, created, schema version, flags) |
| Dates / times | ISO 8601 (`2026-10-05`, `14:32`, with timezone offset for timestamps) |
| Duration | ISO 8601 (`PT1H30M`) plus an optional minutes column |
| multi_select | labels joined with `; ` (labels containing `;` are quoted/escaped) |
| boolean | `true` / `false` |
| Location | `lat,lon` |
| Empty vs skipped | empty cell for empty; `—skipped—` marker (configurable) for skipped |
| Drafts / archived columns | excluded by default; toggles to include |

**Full backup & restore**
- Settings → Data → Backup creates a single versioned archive (zip): SQLite database, all audio clips, and a manifest (app version, schema versions, checksums). It can be saved to Files, Drive or anywhere via the share sheet.
- Restore merges or replaces, with conflict handling by row id. Old archive versions are migrated forward.
- **OS backups:** iCloud and Android backups include rows and settings. Audio is included where size limits allow (Android auto-backup caps at about 25 MB). Models and API keys are never included. Settings states this, so the archive backup is the reliable way to move phones.

### 8.6 Quality: measured and improved continuously
- Every correction you make on the review card is stored locally as a pair (what was proposed → what you saved). Over time this becomes a **personal evaluation set**, built from real usage.
- Per engine, tier and task, the app tracks: valid-output rate, repair rate, values dropped for missing evidence, flag rates, how often you corrected a value, and latency. These are visible in Settings → Quality.
- **Eval runner:** compares engines and prompt versions by replaying memos. On a phone it uses that device's own data. In development it uses sample or synthetic memos. User data never leaves the device for evaluation.
- Prompts are versioned, and every row records which prompt and engine produced it. Any row can be re-extracted with a newer setup, as a proposed diff.

### 8.7 Accessibility
- **Screen readers:** when VoiceOver or TalkBack is on, prompts are announced through the screen reader instead of TTS (or queued after it), so they never talk over each other.
- Every control has an accessible label. Recording state is announced.
- Dynamic Type / font scaling throughout. The list/card view is the accessible alternative to the grid.
- Haptic and visual cues mark when the app starts and stops listening, for users who can't hear the prompts.
- Every voice step has a text-input equivalent.

---

## 9. Tech stack

| Concern | Choice |
|---|---|
| Framework | Expo (dev client) + TypeScript, Expo Router |
| Audio | `expo-audio`, `expo-file-system`; a custom native module where buffer access or audio-session control is needed |
| STT / TTS (defaults) | `expo-speech-recognition` (on-device mode), `expo-speech` |
| Local LLM / Whisper | `llama.rn` (llama.cpp), whisper.cpp binding |
| Platform LLMs | Custom Expo modules: Swift (Apple Foundation Models), Kotlin (ML Kit GenAI Prompt API) |
| Siri / shortcuts | Expo module for iOS App Intents; Android app shortcuts / quick-settings tile |
| Storage | `expo-sqlite` + Drizzle ORM |
| Keys | `expo-secure-store` |
| Validation | zod (schemas generated from columns) |
| Parsing | `chrono-node` (English dates), custom Hindi/Hinglish number/date/duration parsers, transliterator, fuzzy matching (e.g. `fuse.js`) |
| Table UI | FlashList-based grid |
| Export / backup | CSV + zip archive via `expo-sharing` |
| Builds | EAS Build / EAS Update |

---

## 10. Code structure

```
app/                         # screens (Expo Router)
  index.tsx                  # collections
  collection/[id]/index.tsx  # table
  collection/[id]/record.tsx # recording session
  collection/[id]/edit.tsx   # template builder
  row/[id].tsx               # row detail / review / re-run diff
  drafts.tsx  settings/
src/
  schema/      # column types, ids, versions, conditions, migrations, zod generation
  session/     # state machine, turn-taking, interruptions, question queue, command grammar
  parsers/     # dates, numbers, durations, currency, booleans, negation/correction markers,
               # options, transliteration (language-aware)
  voice/       # SttProvider / TtsProvider, providers/, capture strategies, TTS cache
  llm/
    engine.ts  router.ts  tiers.ts
    engines/   # apple-fm, gemini-nano, local, cloud (anthropic, openai, …), custom-endpoint
    models/    # catalog, downloader, checksum, template sandbox, memory budget
  tasks/       # normalize, extract, phrase_question, draft_template, themes
    prompts/   # versioned
    pipeline.ts  flags.ts
  quality/     # correction log, per-engine stats
  keys/        # secure key storage, consent, usage limits, redaction
  data/        # db, export (csv), backup/restore
  a11y/  components/
modules/       # native: audio-capture, app-intents, apple-fm, gemini-nano
eval/          # developer eval runner (sample/synthetic data)
```

---

## 11. Decisions

| Topic | Decision |
|---|---|
| Default recording mode | **Guided** |
| Audio granularity | Both supported: **one clip per answer** in Guided; in Free/Hybrid, one memo clip plus one clip per follow-up answer |
| Third-party STT / TTS / LLM | Optional, configurable in Settings, using the user's **own API keys**, called directly from the device; no backend |
| Bring your own key | **Allowed** |
| Languages | **English**; Hindi and Hinglish accommodated (§8.4) |
| Output | **Local table + CSV export** (plus full backup archive) |
| Starter template | **Daily task log** |
| App lock | **No** |
| Default AI preset | **All on-device** |
| Local models | **Users can download models** |
| Quality | Evaluated and improved continuously from real use (§8.6), not gated by fixed thresholds |

### Not part of the design
- Accounts, multi-user or shared collections
- Cloud sync / hosted backend
- Links between collections
- Non-audio media
