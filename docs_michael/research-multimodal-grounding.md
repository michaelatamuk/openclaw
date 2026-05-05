# Research Proposal: Multimodal Context Injection — Grounding Heterogeneous Inputs for Autonomous Agent Tasks in OpenClaw

**Author**: Technical Research Proposal
**Date**: 2026-05-05
**Domain**: Multimodal AI, Agent Systems, Computer Vision, Speech Processing, Information Extraction
**Code Analysis**: OpenClaw codebase (`openclaw/openclaw`)

---

## Table of Contents

1. [Abstract](#abstract)
2. [Problem Statement](#problem-statement)
3. [Research Objectives](#research-objectives)
4. [Background & Related Work](#background--related-work)
5. [Proposed Approach](#proposed-approach)
6. [Data Sources & Availability](#data-sources--availability)
7. [Multimodal Feature Taxonomy](#multimodal-feature-taxonomy)
8. [Model Architecture](#model-architecture)
9. [Integration with Skill Routing](#integration-with-skill-routing)
10. [Evaluation Methodology](#evaluation-methodology)
11. [Implementation Strategy](#implementation-strategy)
12. [Expected Outcomes & Contributions](#expected-outcomes--contributions)
13. [Challenges & Limitations](#challenges--limitations)
14. [Future Work](#future-work)
15. [References & Code Locations](#references--code-locations)

---

## Abstract

OpenClaw is a multi-channel autonomous agent system (CLI, Telegram, Discord, mobile) that operates exclusively on text. Yet every supported channel already carries richer inputs that the agent pipeline silently discards: **images** (screenshots, diagrams, charts), **audio** (voice messages, voice notes), **video** (screen recordings, GIF bug reproductions), and **structured file attachments** (CSV data, JSON configs, YAML, log dumps). Users must re-type or re-describe information the agent could directly process — a fundamental and growing capability gap as messaging platforms normalize non-text communication.

We propose **Multimodal Context Injection (MCI)**, a grounding pipeline that:

1. Ingests attachments from any OpenClaw channel through a channel-agnostic adapter layer covering all four non-text modalities
2. Classifies and decomposes each attachment into a typed, schema-validated **Multimodal Context Record (MCR)** — a structured intermediate representation tailored to the modality (visual, audio, video, structured file)
3. Injects MCRs into the agent task context alongside the user's text prompt, providing the LLM with structured, persistent multimodal evidence rather than raw bytes
4. Extends skill routing to incorporate cross-modal signals — a voice message describing a Node.js error, a screen recording of a crashing UI, or a CSV with anomalous values each carry routing signals absent from the text alone
5. Persists MCRs in trajectory logs and memory, enabling cross-session multimodal retrieval and creating a training data flywheel for future ML learning from non-text context

**Key contribution**: A production-viable multimodal grounding architecture for LLM agent systems that is (a) latency-constrained (<300ms for preprocessing, <600ms including audio transcription), (b) channel-agnostic across all four modalities, (c) structured rather than free-text enabling downstream ML use, and (d) grounded in OpenClaw's existing channel, memory, and trajectory infrastructure.

---

## Problem Statement

### Current Non-Text Handling in OpenClaw

**Channel layer** (`src/channels/`): Channel adapters normalize incoming messages to a shared `IncomingMessage` schema. Non-text content is available at the channel API level but not surfaced:

- **Images**: Present in Telegram (`PhotoSize[]`), Discord (`Attachment[]`), mobile — stripped or passed as opaque blobs
- **Voice/Audio**: Telegram voice messages (`Voice`, `.ogg` Opus), Discord audio attachments — universally stripped today
- **Video**: Telegram video notes, Discord screen recordings, GIF attachments — stripped
- **File attachments**: Telegram `Document`, Discord `Attachment` — passed as download URLs with no content analysis

**Auto-reply pipeline** (`src/auto-reply/reply/`): The agent receives `HandleCommandsParams` with only the text message body and session context. No non-text attachment data flows through this pipeline.

**Model call layer** (`src/agents/`): Claude Sonnet 4.6 supports image content blocks natively. It does not natively process audio or arbitrary file formats. Even images, when passed raw, consume model context budget with no persistent structured representation.

**Trajectory log schema** (`src/trajectory/types.ts`): `SessionMetadata` has no attachment field of any kind. All non-text context is invisible to the learning infrastructure.

### Critical Gaps

#### Gap 1: The Grounding Gap — Non-text inputs pass or die

For every non-text input, exactly two outcomes occur today:

- **Strip**: Channel adapter discards the attachment entirely. Agent sees only the text fragment ("fix this", "help", "what's wrong here") with no context.
- **Pass-raw** (images only, rare): Raw base64 is injected into the model context. The model spends its own reasoning budget on interpretation, cannot annotate or reuse the result, and the interpretation is invisible to downstream systems (routing, memory, trajectory).

Neither outcome produces a **structured, reusable, modality-typed representation** of what the attachment contained.

#### Gap 2: The Cross-Modal Routing Gap — Non-text signals ignored

Skill routing (currently alphabetical; see companion proposal `research-skill-routing-ml.md`) takes the user's text prompt as sole input. All four non-text modalities carry routing signals that often dominate the text:

| User text            | Attachment                                        | Correct skills                | Routed (text-only) |
| -------------------- | ------------------------------------------------- | ----------------------------- | ------------------ |
| "help"               | Screenshot: Python `ModuleNotFoundError`          | `debug`, `package-management` | alphabetical       |
| "check this" (audio) | Voice note: "the deploy keeps failing at step 3"  | `devops`, `ci`, `docker`      | alphabetical       |
| "this is broken"     | Screen recording: UI crash on button click        | `frontend-dev`, `debug`       | alphabetical       |
| "analyze"            | CSV file: sales data with a NaN spike in column B | `data-analysis`, `python`     | alphabetical       |
| "what do you think?" | Figma screenshot: incomplete UI layout            | `frontend-dev`, `css`         | alphabetical       |
| "broken config"      | YAML attachment: indentation error on line 14     | `config-management`, `yaml`   | alphabetical       |

#### Gap 3: The Memory Gap — Non-text context evaporates

OpenClaw's memory extension (`extensions/memory-*/`) stores text embeddings for retrieval. Screenshots, voice descriptions, screen recordings, and data files — often the most information-dense context in a task — are never stored, never indexed, never retrievable. A user who sent an architecture diagram last week, recorded a bug reproduction video, or shared a log file cannot reference it in a future session.

#### Gap 4: The Trajectory Gap — No multimodal training signal

Trajectory logs (`~/.openclaw/trajectories/*.jsonl`) capture text prompts and skill invocations for ML learning. When the actual task driver was a voice message or a CSV, the trajectory captures only the sparse text fragment, masking the true task signal and corrupting any ML model trained on trajectory data.

### Concrete Failure Scenarios

**Scenario A — Voice message (audio)**:
User sends a 15-second Telegram voice note: _"Hey, the production deploy failed again, same error as Tuesday — the Docker image can't find the config file, gives a file not found at `/etc/app/config.yaml`"_. User types: "same as before".

Today: Agent sees "same as before" with zero context. Cannot help without a clarifying exchange.

With MCI: Audio is transcribed, entities extracted — `{error_type: "FileNotFoundError", path: "/etc/app/config.yaml", service: "Docker"}`. Skills `docker` and `devops` are boosted. Agent responds with the targeted fix immediately.

**Scenario B — Screen recording (video)**:
User shares a 10-second `.mp4` of a React app crashing when a button is clicked. User types: "this is broken".

Today: Video is stripped entirely.

With MCI: Key frames extracted → visual MCR identifies a browser console error frame: `TypeError: Cannot read property 'map' of undefined`. Skills `frontend-dev` and `debug` are boosted. Agent diagnoses the null reference immediately.

**Scenario C — Structured file (CSV)**:
User attaches a 500-row `sales_data.csv` and types: "analyze".

Today: File URL is passed to agent raw (or stripped). Agent cannot read arbitrary URLs; task fails or requires the user to paste data manually.

With MCI: CSV is parsed — schema inferred, summary statistics computed, anomalies flagged (NaN spike in column `revenue`, row 247). `StructuredFileMCR` injected into context. Agent immediately says: "Your CSV has 500 rows with columns `[date, product, revenue, units]`. There's a NaN outlier in `revenue` at row 247. Want me to clean it?"

**Scenario D — Image (visual)**:
User sends a screenshot of a Python traceback. User types: "help".

With MCI: Error parsed → `{error_class: "ModuleNotFoundError", module: "pandas", file: "analysis.py", line: 3}`. Skills `python-dev`, `package-management` boosted. Agent says: "Run `pip install pandas`."

---

## Research Objectives

### Primary Objective

**Design, implement, and evaluate a latency-constrained multimodal grounding pipeline for LLM agent systems that converts heterogeneous non-text inputs (images, audio, video, structured files) into typed, schema-validated Multimodal Context Records (MCRs) within strict latency budgets, and demonstrates measurable improvement in task success rate across all four modality types.**

### Specific Research Questions

#### RQ1: Cross-Modal Type Classification

Can a unified classifier reliably assign incoming attachments to a closed modality+type taxonomy (terminal screenshot, voice error description, screen recording, CSV data file, etc.) using lightweight models?

- **Hypothesis**: A two-stage classifier (modality detection by MIME type + content-based sub-classification) achieves >90% accuracy on a developer-context attachment taxonomy
- **Why it matters**: Determines the downstream extraction strategy; misclassification leads to wrong extractors and corrupted MCRs

#### RQ2: Structured Extraction Fidelity per Modality

For each modality, how accurately can structured fields be extracted?

- **Image**: Error class, file path, language (terminal); code language, symbols (code screenshot)
- **Audio**: Transcription accuracy (WER), entity extraction precision/recall (error types, file paths, service names mentioned in voice)
- **Video**: Key frame selection quality (are the most informative frames chosen?), OCR accuracy on extracted frames
- **Structured file**: Schema inference accuracy, anomaly detection precision/recall

#### RQ3: Cross-Modal Skill Routing Improvement

Does injecting MCR features across all modalities improve NDCG@10 for skill routing versus text-only routing?

- **Hypothesis**: MCR features improve NDCG@10 by ≥5 points for image sessions, ≥7 points for audio sessions (audio text is often sparser than image context), ≥8 points for structured file sessions

#### RQ4: Latency Feasibility per Modality

Can each modality's preprocessing pipeline complete within its latency budget on commodity hardware?

| Modality                | Target p50 | Target p99 |
| ----------------------- | ---------- | ---------- |
| Image                   | ≤ 150ms    | ≤ 300ms    |
| Audio (≤30s)            | ≤ 400ms    | ≤ 600ms    |
| Video (≤60s)            | ≤ 500ms    | ≤ 800ms    |
| Structured file (≤10MB) | ≤ 100ms    | ≤ 200ms    |

#### RQ5: Audio-Specific Entity Extraction

Can named entities (file paths, error class names, service names, version numbers) be accurately extracted from transcribed developer voice notes?

- **Hypothesis**: Combining Whisper transcription with a developer-domain NER model achieves >80% entity-level F1 on voice descriptions of coding errors
- **Harder case**: Voice notes often contain filler words, corrections ("no wait, line 42, actually 43"), and acronyms ("the CI/CD pipeline" vs. "the CICD pipeline")

#### RQ6: Cross-Session Multimodal Memory Utility

Does storing MCRs in the memory system and retrieving them in future sessions improve task success on follow-up queries referencing prior non-text context?

- **Hypothesis**: MCR memory retrieval improves relevant skill recall by ≥10% on cross-session multimodal reference tasks (e.g., "same error as the voice note I sent last week")

#### RQ7: Modality Complementarity

Do different modalities contribute independent routing signal (are they complementary), or does one modality dominate when multiple are present?

- **Hypothesis**: Audio and image together provide more routing signal than either alone (a user who sends both a voice note and a screenshot is communicating from two angles, and the entities often align to confirm each other)

---

## Background & Related Work

### Multimodal LLMs

**GPT-4o, Claude Sonnet 3.7/4, Gemini 1.5 Pro**:

- Support image inputs natively via API content blocks
- GPT-4o and Gemini 1.5 Pro additionally support audio input
- **Limitation for agents**: Inputs are interpreted per-turn with no persistent structured representation; same attachment re-submitted in 10 sessions generates 10 independent interpretations with no learning or cross-session consistency
- **Limitation of audio in LLMs**: Full audio content blocks consume substantial context; there is no intermediate structured representation layer

**LLaVA, CogVLM, InternVL (open source vision)**:

- Strong vision-language understanding, deployable locally
- **Applicable**: Used as the visual extraction backbone for privacy-first deployments

**Whisper (OpenAI)**:

- State-of-the-art multilingual ASR
- Open-source, local-deployable, <1GB for `base` model
- **Applicable**: Core audio transcription component; `turbo` model achieves real-time factor ≈ 0.06 on CPU

### Structured Information Extraction

**PaddleOCR, TrOCR, Tesseract**: High-accuracy OCR for printed text in screenshots. TrOCR achieves near-human accuracy on clean terminal/code images.

**Donut (Document Understanding Transformer)**: End-to-end document → structured JSON without explicit OCR. Applicable to structured file previews.

**ScreenAI (Google DeepMind, 2024)**: Specialized for UI screenshot understanding — detects UI elements, labels, affordances.

**DePlot (Google)**: Chart → table conversion. Applicable to data chart MCR extraction.

**Named Entity Recognition for Code/Dev Domain**:

- General NER (spaCy, Stanza) has poor recall on developer entities (error class names, file paths, CLI flags, semver strings)
- CodeBERT and GraphCodeBERT fine-tuned on error messages achieves strong results on structured error entity extraction
- **Gap**: No NER model specifically for transcribed developer voice — a research opportunity

### Audio in Agent Systems

**Prior work on voice interfaces**: Most voice assistant research (Alexa, Siri, Google Assistant) focuses on ASR + intent classification for structured command grammars, not open-ended developer task descriptions.

**SpeechAgents (Microsoft, 2024)**: Multi-agent system using audio as primary modality for collaborative tasks. Focuses on speech synthesis coordination, not structured extraction.

**AudioPaLM (Google, 2023)**: Joint audio-text language model. Powerful but requires proprietary infrastructure; not locally deployable.

**Gap**: No prior work addresses audio grounding specifically for developer-domain agent tasks — extracting file paths, error class names, service identifiers, and version strings from informal developer voice notes.

### Video in Agent Systems

**WebAgent, SeeAct**: Use per-frame screenshots for web navigation. Single-frame, no temporal aggregation.

**AppAgent**: Screen video for mobile UI automation — but processes only a fixed action sequence, not free-form temporal content.

**Gap**: No agent system processes user-submitted screen recordings as an input modality for task understanding, with temporal key frame selection and cross-frame entity aggregation.

### Structured File Analysis

**OpenAI Assistants API (Code Interpreter)**: Accepts CSV/file attachments and provides a sandboxed Python environment. Does not produce structured MCRs; interpretation is generated per-response by the LLM.

**LlamaIndex, LangChain document loaders**: Parse PDFs, CSVs, JSONs as text chunks for RAG. Not designed for real-time agent task grounding with latency constraints.

**Gap**: No agent system converts structured file attachments to typed, schema-validated MCRs with extracted anomaly signals for skill routing.

### Novel Gap This Proposal Addresses

No prior work addresses the combination of:

1. **All four non-text modalities** (image, audio, video, structured file) in a unified grounding pipeline
2. **Structured intermediate representations** (typed MCRs, not free-text descriptions) for each modality
3. **Trajectory integration** for ML learning from non-text context
4. **Skill routing augmentation** across all modalities with cross-modal feature fusion
5. **Latency constraints** (<300–800ms depending on modality) for interactive agent systems
6. **Channel-agnostic** design across Telegram, Discord, CLI, and mobile

---

## Proposed Approach

### Overview: The MCI Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Channel Layer                               │
│  Telegram  Discord  CLI  Mobile  ──► Channel Adapter                 │
│                                       ↓                              │
│                           Attachment { type, bytes, metadata }       │
└──────────────────────────┬───────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────────────┐
│                   MCI Pipeline                                       │
│                                                                      │
│  Stage 1: Modality Router                                            │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ MIME type + magic bytes → modality branch                    │    │
│  │ image/* ──► Visual Branch                                    │    │
│  │ audio/* ──► Audio Branch                                     │    │
│  │ video/* ──► Video Branch                                     │    │
│  │ text/csv, application/json, text/plain → File Branch         │    │
│  └──────────────────────────────────────────────────────────────┘    │
│         ↓              ↓              ↓              ↓               │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────┐       │
│  │ Visual   │  │  Audio    │  │  Video   │  │ Structured   │       │
│  │ Branch   │  │  Branch   │  │  Branch  │  │ File Branch  │       │
│  │          │  │           │  │          │  │              │       │
│  │Classifier│  │  Whisper  │  │ Keyframe │  │ Schema       │       │
│  │  OCR     │  │  NER+     │  │ Extract  │  │ Inference    │       │
│  │  VLM     │  │  Prosody  │  │ VisualMCR│  │ Anomaly Det. │       │
│  └────┬─────┘  └─────┬─────┘  └────┬─────┘  └──────┬───────┘       │
│       └──────────────┴─────────────┴────────────────┘               │
│                            ↓                                         │
│  Stage 3: MCR Construction & Cross-Modal Fusion                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ Multimodal Context Record (MCR)                              │    │
│  │ { modality, type, entities, text_repr,                       │    │
│  │   confidence, source_channel,                                │    │
│  │   routing_features, embedding_vector }                       │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────┬───────────────────────────────────────────┘
                           ↓
         ┌─────────────────┴──────────────────────┐
         ▼                                        ▼
┌─────────────────────┐               ┌───────────────────────┐
│   Task Context      │               │  Memory + Trajectory  │
│   Injection         │               │  Persistence          │
│                     │               │                       │
│  MCR text_repr →    │               │  MCR text_repr →      │
│  structured block   │               │  embedding → memory   │
│  in agent prompt    │               │                       │
│                     │               │  MCR schema →         │
│  routing_features → │               │  trajectory JSONL     │
│  skill ranker       │               │                       │
└─────────────────────┘               └───────────────────────┘
```

### Stage 1: Modality Router

**Goal**: Classify each attachment into a modality branch in <5ms

**Method**: MIME type + magic byte inspection (no ML needed)

```typescript
function routeAttachment(attachment: RawAttachment): ModalityBranch {
  const mime = attachment.mimeType ?? detectMimeType(attachment.bytes.slice(0, 16));

  if (mime.startsWith("image/")) return "visual";
  if (mime.startsWith("audio/")) return "audio";
  if (mime.startsWith("video/") || mime === "image/gif") return "video";
  if (STRUCTURED_MIME_TYPES.has(mime)) return "structured_file";
  return "unsupported";
}

const STRUCTURED_MIME_TYPES = new Set([
  "text/csv",
  "application/json",
  "application/x-yaml",
  "text/yaml",
  "text/plain", // log files
  "application/x-ndjson", // JSONL logs
]);
```

### Stage 2A: Visual Branch

#### Visual Type Classifier

**Classes** (7):

| Class                  | Description                                 | Extraction Strategy        |
| ---------------------- | ------------------------------------------- | -------------------------- |
| `terminal_screenshot`  | Terminal/console with text output or errors | OCR → Error parser         |
| `code_screenshot`      | Code editor or IDE screenshot               | OCR → Language detector    |
| `ui_screenshot`        | Browser, app, or web UI screenshot          | ScreenAI element detection |
| `architecture_diagram` | System/component/sequence diagram           | VLM + graph extractor      |
| `data_chart`           | Bar/line/scatter/pie chart or table         | DePlot / ChartQA           |
| `handwritten`          | Whiteboard, paper sketch                    | VLM description only       |
| `other`                | Photo, meme, unclassifiable                 | VLM caption only           |

**Model**: CLIP ViT-B/32 zero-shot (<10ms CPU) → fine-tuned EfficientNet-B0 (<20ms CPU) → LLM fallback for low-confidence

**Key MCR types**:

```typescript
interface TerminalVisualMCR {
  modality: "visual";
  type: "terminal_screenshot";
  ocr_text: string;
  error?: {
    error_class: string;
    message: string;
    language?: string;
    file?: string;
    line?: number;
    module_or_symbol?: string;
  };
  shell?: string;
  confidence: number;
}

interface CodeVisualMCR {
  modality: "visual";
  type: "code_screenshot";
  ocr_text: string;
  language?: string;
  file_path?: string;
  visible_symbols?: string[];
  has_error_markers?: boolean;
  has_diff?: boolean;
  confidence: number;
}

interface DiagramVisualMCR {
  modality: "visual";
  type: "architecture_diagram";
  diagram_style?: "component" | "sequence" | "flow" | "erd" | "class" | "other";
  node_labels: string[];
  edge_labels: string[];
  caption?: string;
  confidence: number;
}

interface ChartVisualMCR {
  modality: "visual";
  type: "data_chart";
  chart_type?: "bar" | "line" | "scatter" | "pie" | "table" | "other";
  title?: string;
  x_axis_label?: string;
  y_axis_label?: string;
  series_names?: string[];
  anomalies?: string[];
  data_summary?: string;
  confidence: number;
}
```

### Stage 2B: Audio Branch

**Goal**: Convert voice messages and audio notes into structured developer-domain MCRs

#### Audio Type Classifier

**Classes** (4):

| Class                     | Description                                   | Channel Sources                    |
| ------------------------- | --------------------------------------------- | ---------------------------------- |
| `voice_error_description` | Developer describing a bug, error, or failure | Telegram voice note, Discord audio |
| `voice_instruction`       | Developer dictating a task or feature request | Mobile dictation, Telegram         |
| `voice_question`          | Developer asking a question verbally          | All voice-capable channels         |
| `other_audio`             | Music, ambient sound, non-speech audio        | Discord audio upload               |

**Classification method**: After Whisper transcription (always first), classify based on transcript content using a lightweight text classifier (logistic regression on TF-IDF of transcript). Audio-level classification (before transcription) is not needed — Whisper transcription is fast enough to be the first step for all audio.

#### Audio Extraction Pipeline

```
Audio bytes (OGG, MP3, WAV, M4A)
    ↓
Format normalization (ffmpeg → 16kHz mono WAV)
    ↓
Whisper transcription (base or turbo model)
    ↓
Audio type classification (text classifier on transcript)
    ↓
Developer NER (error classes, file paths, service names, version strings)
    ↓
Prosody feature extraction (duration, speaking rate, pause count)
    ↓
AudioMCR construction
```

#### AudioMCR Schema

```typescript
interface AudioMCR {
  modality: "audio";
  type: "voice_error_description" | "voice_instruction" | "voice_question" | "other_audio";

  // Transcription
  transcript: string; // Full Whisper transcript
  transcript_language: string; // Detected language (ISO 639-1)
  transcript_wer_estimate?: number; // Model confidence proxy

  // Developer entities extracted from transcript
  entities: {
    error_classes: string[]; // e.g. ["ModuleNotFoundError", "TypeError"]
    file_paths: string[]; // e.g. ["/etc/app/config.yaml"]
    service_names: string[]; // e.g. ["Docker", "Nginx", "PostgreSQL"]
    version_strings: string[]; // e.g. ["v2.3.1", "Node 22"]
    cli_commands: string[]; // e.g. ["npm install", "docker build"]
    urls: string[]; // URLs mentioned verbally
    line_numbers: number[]; // "line 42", "line forty-two"
  };

  // Prosody features (lightweight, no ML model needed)
  prosody: {
    duration_seconds: number;
    speaking_rate_wpm: number; // Words per minute
    pause_count: number; // Pauses > 0.5s
    urgency_score?: number; // 0–1, derived from speaking rate + pause pattern
  };

  confidence: number;
}
```

**Developer NER pipeline**:

Whisper transcripts of developers contain domain-specific entities that general NER models miss. We use a two-step approach:

1. **Pattern-based extraction** (regex library, zero latency):

```python
import re

DEVELOPER_ENTITY_PATTERNS = {
    "error_classes": [
        r"\b[A-Z][a-zA-Z]+(?:Error|Exception|Fault|Warning|Panic)\b",
        r"\bECONNREFUSED\b", r"\bENOENT\b", r"\bETIMEDOUT\b"  # POSIX errors
    ],
    "file_paths": [
        r"/?(?:\w+/)+\w+\.\w+",                     # Unix paths
        r"[A-Za-z]:\\(?:\w+\\)+\w+\.\w+"            # Windows paths
    ],
    "version_strings": [
        r"\bv?\d+\.\d+(?:\.\d+)?(?:-[a-z0-9.]+)?\b"
    ],
    "line_numbers": [
        r"\bline[s]?\s+(\d+)\b",
        r"\bat\s+(\d+)\b"
    ],
    "cli_commands": [
        r"\b(npm|pnpm|yarn|pip|docker|kubectl|git|cargo|go)\s+\w+\b"
    ]
}

def extract_developer_entities(transcript: str) -> dict:
    entities = {k: [] for k in DEVELOPER_ENTITY_PATTERNS}
    for entity_type, patterns in DEVELOPER_ENTITY_PATTERNS.items():
        for pattern in patterns:
            matches = re.findall(pattern, transcript, re.IGNORECASE)
            entities[entity_type].extend(matches)
    return {k: list(set(v)) for k, v in entities.items()}
```

2. **CodeBERT NER** (optional, higher accuracy, ~80ms): Fine-tuned CodeBERT for developer-domain NER on the same entity types. Activated for voice notes with low pattern-match confidence.

**Prosody features**:

```python
def extract_prosody(audio_bytes: bytes, transcript: str, duration_s: float) -> dict:
    words = transcript.split()
    word_count = len(words)

    # Detect pauses from Whisper segment timestamps (Whisper --word_timestamps flag)
    # Pauses = gaps > 0.5s between word timestamps
    pause_count = count_pauses_from_whisper_segments(audio_bytes)

    speaking_rate_wpm = (word_count / max(duration_s, 1)) * 60

    # Urgency heuristic: high speaking rate + few pauses = urgent delivery
    urgency_score = min(1.0, speaking_rate_wpm / 180) * (1 - pause_count / max(word_count, 1))

    return {
        "duration_seconds": duration_s,
        "speaking_rate_wpm": speaking_rate_wpm,
        "pause_count": pause_count,
        "urgency_score": urgency_score
    }
```

**Text representation** (injected into agent prompt):

```
[Voice Note — voice_error_description, 18s, English]
Transcript: "the production deploy failed again, Docker can't find the config at /etc/app/config.yaml, same FileNotFoundError as Tuesday"
Entities: errors=[FileNotFoundError], paths=[/etc/app/config.yaml], services=[Docker]
Urgency: moderate (142 wpm, 3 pauses)
---
```

### Stage 2C: Video Branch

**Goal**: Extract structured context from screen recordings and video bug reproductions in <800ms for clips up to 60 seconds

#### Video Processing Pipeline

```
Video bytes (MP4, WebM, MOV, GIF)
    ↓
Decode + uniform sampling (1 frame/sec for ≤60s clips)
    ↓
Visual significance scoring (frame-to-frame diff + entropy)
    ↓
Top-K frame selection (K=5 default)
    ↓
Per-frame VisualMCR extraction (reuse visual branch)
    ↓
Audio track extraction (if present) → AudioMCR
    ↓
Temporal aggregation → VideoMCR
```

#### VideoMCR Schema

```typescript
interface VideoMCR {
  modality: "video";
  type: "screen_recording" | "bug_reproduction" | "tutorial" | "other_video";

  duration_seconds: number;
  total_frames_sampled: number;
  key_frames_selected: number;

  // Aggregated across key frames
  visual_summary: {
    dominant_app?: string; // "Chrome", "VS Code", "Terminal" detected from chrome
    error_frames: {
      timestamp_s: number;
      error_mcr: TerminalVisualMCR | CodeVisualMCR | null;
    }[];
    state_transitions: string[]; // e.g. ["app launch", "button click", "crash"]
  };

  // Audio track if present
  audio_mcr?: AudioMCR;

  // Aggregated entities (union of all frame MCRs + audio)
  aggregated_entities: {
    error_classes: string[];
    file_paths: string[];
    service_names: string[];
  };

  caption: string; // VLM-generated: "Screen recording of a React app crashing on button click with TypeError in console"
  confidence: number;
}
```

**Key frame selection**:

```python
import cv2
import numpy as np

def select_key_frames(video_path: str, max_frames: int = 5) -> list[tuple[float, np.ndarray]]:
    """Select the most visually significant frames from a video."""
    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    frame_count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))

    # Sample 1 frame/sec uniformly
    candidate_frames = []
    for sec in range(int(frame_count / fps)):
        cap.set(cv2.CAP_PROP_POS_MSEC, sec * 1000)
        ret, frame = cap.read()
        if ret:
            candidate_frames.append((sec, frame))

    # Score by frame-to-frame diff (high diff = state change = interesting)
    scores = [0.0]
    for i in range(1, len(candidate_frames)):
        diff = cv2.absdiff(candidate_frames[i-1][1], candidate_frames[i][1])
        scores.append(float(diff.mean()))

    # Also score by entropy (frames with more information content)
    def entropy(frame):
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        hist = cv2.calcHist([gray], [0], None, [256], [0, 256])
        hist /= hist.sum()
        return -float(np.sum(hist * np.log2(hist + 1e-8)))

    entropy_scores = [entropy(f) for _, f in candidate_frames]

    # Combined score
    combined = [s + 0.5 * e for s, e in zip(scores, entropy_scores)]
    top_indices = sorted(range(len(combined)), key=lambda i: combined[i], reverse=True)[:max_frames]
    return [candidate_frames[i] for i in sorted(top_indices)]
```

### Stage 2D: Structured File Branch

**Goal**: Convert CSV, JSON, YAML, and log file attachments into structured MCRs in <200ms for files up to 10MB

#### File Type Sub-Classifier

| Sub-type      | Detection                                     | Extraction                                       |
| ------------- | --------------------------------------------- | ------------------------------------------------ |
| `csv_data`    | `.csv` extension or `text/csv` MIME           | Schema inference, stats, anomaly detection       |
| `json_config` | `.json` + object/array structure              | Key traversal, schema extract, validation errors |
| `yaml_config` | `.yaml` / `.yml`                              | Key traversal, indentation error detection       |
| `log_file`    | `.log` / `text/plain` with timestamp patterns | Error line extraction, frequency analysis        |
| `jsonl_log`   | `.jsonl` / `application/x-ndjson`             | Structured log parsing, error aggregation        |

#### StructuredFileMCR Schema

```typescript
interface StructuredFileMCR {
  modality: "structured_file";
  type: "csv_data" | "json_config" | "yaml_config" | "log_file" | "jsonl_log" | "other_text";

  file_name?: string;
  file_size_bytes: number;

  // CSV-specific
  csv?: {
    row_count: number;
    column_names: string[];
    column_types: Record<string, "string" | "number" | "boolean" | "date" | "mixed">;
    null_counts: Record<string, number>;
    anomalies: {
      // Statistical anomalies
      column: string;
      type: "null_spike" | "outlier" | "type_inconsistency";
      description: string;
      rows_affected: number;
    }[];
    sample_rows: Record<string, unknown>[]; // First 3 rows for context
  };

  // JSON/YAML-specific
  config?: {
    top_level_keys: string[];
    depth: number;
    validation_errors?: string[]; // e.g. schema violations, missing required keys
    sensitive_keys_detected?: string[]; // "password", "api_key", "secret" key names detected
  };

  // Log file-specific
  log?: {
    line_count: number;
    time_range?: { start: string; end: string };
    error_count: number;
    warning_count: number;
    top_error_patterns: {
      pattern: string;
      count: number;
      first_occurrence: string;
      last_occurrence: string;
    }[];
    services_mentioned?: string[];
  };

  summary: string; // Human-readable one-paragraph summary
  confidence: number;
}
```

**CSV anomaly detection**:

```python
import pandas as pd
import numpy as np

def detect_csv_anomalies(df: pd.DataFrame) -> list[dict]:
    anomalies = []

    for col in df.columns:
        # Null spike detection
        null_rate = df[col].isna().mean()
        if null_rate > 0.05:  # > 5% nulls
            anomalies.append({
                "column": col,
                "type": "null_spike",
                "description": f"{null_rate:.1%} null values",
                "rows_affected": int(df[col].isna().sum())
            })

        # Numeric outlier detection (IQR method)
        if pd.api.types.is_numeric_dtype(df[col]):
            Q1, Q3 = df[col].quantile(0.25), df[col].quantile(0.75)
            IQR = Q3 - Q1
            outliers = ((df[col] < Q1 - 3 * IQR) | (df[col] > Q3 + 3 * IQR))
            if outliers.any():
                anomalies.append({
                    "column": col,
                    "type": "outlier",
                    "description": f"{outliers.sum()} extreme outlier(s) beyond 3×IQR",
                    "rows_affected": int(outliers.sum())
                })

        # Type inconsistency detection
        if df[col].dtype == object:
            numeric_ratio = pd.to_numeric(df[col], errors='coerce').notna().mean()
            if 0.1 < numeric_ratio < 0.9:
                anomalies.append({
                    "column": col,
                    "type": "type_inconsistency",
                    "description": f"Mixed types: {numeric_ratio:.0%} numeric, rest string",
                    "rows_affected": int(len(df))
                })

    return anomalies
```

**Log error pattern extraction**:

```python
import re
from collections import Counter

LOG_ERROR_PATTERNS = [
    r"(?:ERROR|FATAL|CRITICAL|PANIC).*",
    r"Exception: .+",
    r"\b[A-Z][a-zA-Z]+Error: .+"
]

def extract_log_patterns(log_text: str) -> list[dict]:
    lines = log_text.splitlines()
    error_lines = []
    for line in lines:
        for pattern in LOG_ERROR_PATTERNS:
            if re.search(pattern, line):
                # Normalize: remove timestamps, IDs for pattern grouping
                normalized = re.sub(r"\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}[.,\d]*Z?", "<ts>", line)
                normalized = re.sub(r"\b[0-9a-f]{8}-[0-9a-f-]{27}\b", "<uuid>", normalized)
                error_lines.append((line, normalized))
                break

    pattern_counts = Counter(n for _, n in error_lines)
    top_patterns = pattern_counts.most_common(5)

    return [
        {"pattern": p, "count": c, "first_occurrence": next(
            l for l, n in error_lines if n == p
        )}
        for p, c in top_patterns
    ]
```

### Stage 3: MCR Construction

All branch outputs are wrapped in a common `MCR` envelope:

```typescript
interface MultimodalContextRecord {
  id: string; // UUID, stable across sessions
  created_at: string; // ISO timestamp
  source_channel: string; // "telegram", "discord", "cli", "mobile-ios", etc.
  session_id: string;
  content_hash: string; // SHA-256 of raw attachment bytes (dedup)

  // Typed payload (union discriminated by .modality + .type)
  payload:
    | TerminalVisualMCR
    | CodeVisualMCR
    | DiagramVisualMCR
    | ChartVisualMCR
    | AudioMCR
    | VideoMCR
    | StructuredFileMCR;

  // Cross-modal derived fields
  text_representation: string; // Compact human-readable summary for LLM injection
  embedding_vector?: Float32Array; // Embedding of text_representation for memory
  routing_features: Record<string, number>; // Feature vector for ML skill routing
}
```

**Text representations** (injected into agent prompt as structured blocks):

```
[Voice Note — voice_error_description, 18s, English]
Transcript: "...Docker can't find config at /etc/app/config.yaml..."
Entities: errors=[FileNotFoundError], paths=[/etc/app/config.yaml], services=[Docker]
---

[CSV Attachment — sales_data.csv, 500 rows × 4 columns]
Columns: date (date), product (string), revenue (number), units (number)
Anomalies: NULL spike in 'revenue' (12%), 1 outlier at row 247 (revenue=−99999)
---

[Screen Recording — 22s, screen_recording]
Key errors detected: TypeError: Cannot read property 'map' of undefined (frame 14s)
App: Chrome + VS Code visible
Audio: "when I click the button it just crashes"
---
```

---

## Data Sources & Availability

### 1. Non-Text Attachments from Channels

**Telegram**:

- Images: `PhotoSize[]` → largest variant via `getFile` API
- Voice: `Voice` object → `.ogg` Opus download
- Video notes: `VideoNote` object → `.mp4` download
- Documents: `Document` → download by `file_id`

**Discord**:

- Images/Video/Files: `Attachment[]` with `content_type`, `url`, `size`
- Audio: `Attachment` where `content_type` starts with `audio/`

**CLI**:

- Images/files: `--attachment <path>` flag or stdin pipe detection (magic bytes)
- Audio: `--voice <path>` flag for batch processing scenarios

**Mobile (iOS/Android)** (`apps/ios/`, `apps/android/`):

- Camera photos → React Native image picker → base64 upload
- Voice recording → in-app recorder → `.m4a` upload
- File picker → document picker → upload

**Privacy**: All processing runs locally by default. Cloud VLM/LLM extraction is opt-in per modality (`config.mci.{modality}.useCloudExtraction`).

### 2. Trajectory Logs (Extended Schema)

New `attachments` field added to `SessionMetadata`:

```typescript
interface SessionMetadata {
  // ... existing fields ...
  attachments?: {
    count: number;
    modality_counts: Record<string, number>; // {"visual": 2, "audio": 1}
    mcrs: MultimodalContextRecord[];
    total_processing_latency_ms: number;
  };
}
```

Training signal accumulated after MCI integration:

- MCR modality + type correlated with skill invocations
- MCR entity presence correlated with skill relevance (e.g. "Docker" entity → `docker` skill invoked)
- Task success correlated with MCR type + extracted entities
- Cross-modal correlation features (image + audio present together vs. either alone)

### 3. Developer-Context Benchmark Dataset (New Contribution)

No public dataset exists for developer-context multimodal grounding. We construct and release:

**Visual subset** (~70,000 images, 10,000 per class):

- Synthetic: rendered from real errors using `ansi2html`, Shiki, Mermaid, matplotlib
- Collected: anonymized from opt-in users and public GitHub issue screenshots

**Audio subset** (~5,000 clips):

- Synthetic: TTS (11labs, OpenAI TTS) reading sampled developer error descriptions
- Collected: volunteer recordings of scripted and ad-hoc bug descriptions

**Video subset** (~1,000 clips, 5–60s):

- Synthetic: scripted screen recordings of common bug patterns (null reference click, build failure, test failure)
- Collected: volunteer screen recordings of debugging sessions (anonymized)

**Structured file subset** (~10,000 files):

- Synthetic: generated CSVs with injected anomalies, JSONs with deliberate schema violations, logs with error patterns
- Real: public GitHub log files and anonymized user-contributed samples

**Annotation per sample**:

- Ground-truth modality + type label
- Ground-truth extracted entities (for extraction F1)
- Associated skill labels (which OpenClaw skills a developer would need)
- Consistency label (for audio+text pairs: consistent / inconsistent)

---

## Multimodal Feature Taxonomy

### Features by Modality for Skill Routing

#### Visual Features

| Feature                        | Type    | Example                              |
| ------------------------------ | ------- | ------------------------------------ |
| `has_visual`                   | Binary  | 1                                    |
| `visual_type_{class}`          | One-hot | `visual_type_terminal_screenshot: 1` |
| `visual_error_language_{lang}` | One-hot | `visual_error_language_python: 1`    |
| `visual_has_error`             | Binary  | 1 if error detected                  |
| `visual_code_language_{lang}`  | One-hot | `visual_code_language_typescript: 1` |
| `visual_confidence`            | Float   | 0.92                                 |

#### Audio Features

| Feature                       | Type     | Example                                      |
| ----------------------------- | -------- | -------------------------------------------- |
| `has_audio`                   | Binary   | 1                                            |
| `audio_type_{class}`          | One-hot  | `audio_type_voice_error_description: 1`      |
| `audio_has_error_entity`      | Binary   | 1 if any error class entity extracted        |
| `audio_error_class_embedding` | Float[N] | Embedding of "FileNotFoundError"             |
| `audio_has_file_path`         | Binary   | 1 if file path entity extracted              |
| `audio_has_service_name`      | Binary   | 1 if service name entity (Docker, K8s, etc.) |
| `audio_urgency_score`         | Float    | 0.73                                         |
| `audio_duration_seconds`      | Float    | 18.4                                         |
| `audio_transcript_length`     | Int      | 47 (words)                                   |

#### Video Features

| Feature                    | Type    | Example                                |
| -------------------------- | ------- | -------------------------------------- |
| `has_video`                | Binary  | 1                                      |
| `video_type_{class}`       | One-hot | `video_type_screen_recording: 1`       |
| `video_has_error_frame`    | Binary  | 1 if any frame contains detected error |
| `video_error_frame_count`  | Int     | 2                                      |
| `video_has_audio_track`    | Binary  | 1                                      |
| `video_duration_seconds`   | Float   | 22.0                                   |
| `video_dominant_app_{app}` | One-hot | `video_dominant_app_chrome: 1`         |

#### Structured File Features

| Feature                            | Type    | Example                 |
| ---------------------------------- | ------- | ----------------------- |
| `has_structured_file`              | Binary  | 1                       |
| `file_type_{class}`                | One-hot | `file_type_csv_data: 1` |
| `file_has_anomaly`                 | Binary  | 1 if anomaly detected   |
| `file_anomaly_count`               | Int     | 2                       |
| `file_csv_column_count`            | Int     | 4                       |
| `file_log_error_count`             | Int     | 17                      |
| `file_log_has_fatal`               | Binary  | 1                       |
| `file_config_has_validation_error` | Binary  | 1                       |

#### Cross-Modal Features

| Feature                       | Type    | Description                                             |
| ----------------------------- | ------- | ------------------------------------------------------- |
| `modality_count`              | Int     | How many distinct modalities in this message            |
| `visual_audio_entity_overlap` | Float   | Jaccard similarity of entities from visual + audio MCRs |
| `text_modal_consistency`      | Float   | 0=inconsistent, 1=consistent across present modalities  |
| `dominant_modality_{mod}`     | One-hot | Which modality carries the most routing signal          |

**Total feature vector dimension**: ~80–120 features (modality count × feature count per modality + cross-modal features)

### Modality-to-Skill Affinity Matrix (Prior for Cold-Start Routing)

| Modality + Entity                    | High-affinity skills                                       |
| ------------------------------------ | ---------------------------------------------------------- |
| Visual terminal + Python error       | `python-dev`, `debug`, `package-management`                |
| Visual terminal + Docker error       | `docker`, `devops`                                         |
| Visual code + TypeScript             | `typescript-dev`, `code-review`                            |
| Visual UI screenshot                 | `frontend-dev`, `css`                                      |
| Visual diagram                       | `design`, `docs`                                           |
| Audio + Docker/K8s entity            | `docker`, `devops`, `kubernetes`                           |
| Audio + file path entity             | `file-management`, matching language skill                 |
| Audio + urgency_score > 0.8          | Boost any language/debug skill (urgent = production issue) |
| Video screen recording + error frame | `debug` + language skill from error                        |
| CSV with anomalies                   | `data-analysis`, `python`, `r-stats`                       |
| JSON/YAML with validation error      | `config-management`                                        |
| Log file with error patterns         | `debug`, `logging`                                         |

---

## Model Architecture

### Audio Models

#### Whisper (ASR Core)

**Model**: OpenAI Whisper `turbo` (809M params, ~1.6GB)

- Real-time factor: ~0.06 on CPU M-series (6% of audio duration to process)
- A 30-second clip: ~1.8 seconds transcription time
- Supports 99 languages, automatic language detection
- Word-level timestamps via `--word_timestamps` flag (needed for prosody)

**Deployment**:

```python
import whisper

model = whisper.load_model("turbo")  # Load once at startup

def transcribe_audio(audio_bytes: bytes, language: str | None = None) -> dict:
    # Write to temp file (Whisper requires file path for streaming)
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        wav_bytes = convert_to_wav_16k(audio_bytes)
        f.write(wav_bytes)
        temp_path = f.name

    result = model.transcribe(
        temp_path,
        language=language,
        word_timestamps=True,
        fp16=False  # CPU inference
    )
    return result  # { text, segments, language }
```

**Latency benchmark targets**:

| Clip duration | Whisper turbo (CPU M3) | Whisper base (CPU t3.medium) |
| ------------- | ---------------------- | ---------------------------- |
| 10s           | ~600ms                 | ~1200ms                      |
| 30s           | ~1800ms                | ~3600ms                      |
| 60s           | ~3600ms                | ~7200ms                      |

For clips >30s, use async processing: inject preliminary MCR from first 10s while full transcription completes.

#### Audio Type Classifier

After transcription, a lightweight logistic regression on TF-IDF of transcript classifies into the 4 audio types. Training data: ~1,000 labeled transcripts from the benchmark dataset. Inference: <5ms.

### Visual Models

**Classifier**: CLIP ViT-B/32 zero-shot → EfficientNet-B0 fine-tuned → LLM fallback
**OCR**: TrOCR (microsoft/trocr-base-printed) for clean screenshots, PaddleOCR for complex layouts
**VLM extractor**: GPT-4o (cloud, opt-in) or InternVL-2-1B (local, 2GB, CPU-runnable)

### Video Models

**Frame extractor**: OpenCV `VideoCapture` — no ML, ~50ms for 60s clip
**Per-frame analysis**: Reuse visual branch models
**Audio track**: Extract with ffmpeg, process with audio branch

### Structured File Models

**CSV/JSON/YAML**: Pure algorithmic (pandas, standard library JSON/YAML parsers) — no ML
**Log pattern matching**: Regex library — no ML
**Schema inference**: Type heuristics + `pandas.read_csv` inference — no ML

No ML is needed in the structured file branch; the value is in the extraction, normalization, and anomaly detection logic, not learned models.

---

## Integration with Skill Routing

MCI integrates with the ML skill routing proposal (companion `research-skill-routing-ml.md`) as a **feature source extension**:

```
Text prompt ────────────────────────────────────────┐
                                                    ▼
Non-text attachments ──► MCI Pipeline ──► MCRs ──► routing_features (multimodal)
                                                    ↓
                                       skill_ranker.predict(
                                         text_features + multimodal_features
                                       )
                                                    ↓
                                       ranked_skills → truncate by budget
```

**Backwards compatibility**: When no attachments are present, all modality flags are 0 and feature vectors are zeroed. The routing model degrades gracefully to text-only behavior.

**Cold-start routing** (before ML model is trained on MCR-augmented trajectories): Use the Modality-to-Skill Affinity Matrix as a deterministic rule-based booster:

```typescript
function applyModalitySkillBoost(
  skills: SkillEntry[],
  mcrs: MultimodalContextRecord[],
  boostPositions: number = 3,
): SkillEntry[] {
  const boostedNames = new Set<string>();
  for (const mcr of mcrs) {
    for (const skillName of getAffinitySkills(mcr)) {
      boostedNames.add(skillName);
    }
  }
  const [boosted, rest] = partition(skills, (s) => boostedNames.has(s.skill.name));
  return [...boosted, ...rest];
}
```

**Multi-attachment handling**: When a message contains multiple attachments (e.g., a voice note + a CSV), MCR routing features are merged via element-wise max (not average), preserving the strongest signal from each modality.

---

## Evaluation Methodology

### Offline: Per-Modality Classification Accuracy

**Dataset**: Held-out 15% of the developer-context benchmark dataset

**Metrics**:

- Overall classification accuracy per modality
- Per-class F1
- Confusion matrix

**Targets**:

| Modality        | Metric                 | Target                     |
| --------------- | ---------------------- | -------------------------- |
| Visual          | Type accuracy          | ≥ 90%                      |
| Visual          | Terminal screenshot F1 | ≥ 0.92                     |
| Audio           | Type accuracy          | ≥ 88%                      |
| Video           | Type accuracy          | ≥ 85%                      |
| Structured file | Sub-type accuracy      | ≥ 95% (MIME-deterministic) |

### Offline: Per-Modality Extraction Fidelity

Ground-truth annotated fields for 500 samples per class. Field-level F1:

| Modality        | Key Field                    | Target F1                     |
| --------------- | ---------------------------- | ----------------------------- |
| Visual terminal | `error.error_class`          | ≥ 0.90                        |
| Visual terminal | `error.module_or_symbol`     | ≥ 0.85                        |
| Audio           | Transcript WER               | ≤ 8% (Whisper turbo baseline) |
| Audio           | `entities.error_classes`     | ≥ 0.80                        |
| Audio           | `entities.file_paths`        | ≥ 0.75                        |
| Video           | Error frame detection recall | ≥ 0.80                        |
| CSV             | `anomalies` precision        | ≥ 0.85                        |
| CSV             | `anomalies` recall           | ≥ 0.78                        |
| Log             | Top error pattern recall     | ≥ 0.88                        |

### Online: A/B Test — Task Success with Non-Text Inputs

**Setup**: Sessions with ≥1 non-text attachment (stratified by modality)

- **Control**: Attachment stripped or passed raw
- **Treatment A**: MCI with affinity-based skill boost (rule-based)
- **Treatment B**: MCI with ML routing (requires trained model)

**Primary metric**: Task success rate per modality arm

**Secondary metrics**:

- Clarifying question rate (lower = better: MCI understood the attachment first time)
- Skill routing precision@5 for attachment sessions
- Cross-session follow-up resolution rate (did memory retrieval of prior MCRs help?)

**Statistical**: Two-proportion z-test (α = 0.05), target n = 2,000 sessions per modality × arm

**Targets**:

| Modality        | Expected improvement in task success (Treatment A vs. Control)          |
| --------------- | ----------------------------------------------------------------------- |
| Visual (image)  | +3–5% absolute                                                          |
| Audio           | +5–8% absolute (larger gain: text alone is very sparse for voice notes) |
| Video           | +4–7% absolute                                                          |
| Structured file | +6–10% absolute (file content was previously completely invisible)      |

### End-to-End Latency Benchmark

```python
import time, statistics

for modality, sample_dir in SAMPLE_DIRS.items():
    latencies = []
    for sample_path in sample_dir.glob("*"):
        attachment_bytes = sample_path.read_bytes()
        start = time.perf_counter()
        mcr = process_attachment(attachment_bytes, config=MCIConfig(mode="fast"))
        latencies.append((time.perf_counter() - start) * 1000)

    print(f"{modality}: p50={statistics.median(latencies):.0f}ms, "
          f"p99={sorted(latencies)[int(0.99 * len(latencies))]:.0f}ms")
```

**Targets**:

| Modality              | p50     | p99     |
| --------------------- | ------- | ------- |
| Image                 | ≤ 150ms | ≤ 300ms |
| Audio ≤30s            | ≤ 400ms | ≤ 600ms |
| Video ≤60s            | ≤ 500ms | ≤ 800ms |
| Structured file ≤10MB | ≤ 100ms | ≤ 200ms |

### Cross-Modal Consistency Evaluation

For 200 hand-labeled (text, attachment) pairs with ground-truth consistency labels:

- **Metric**: Consistency classifier AUC
- **Target**: AUC ≥ 0.80

### Visual Memory Retrieval

For 200 cross-session query-MCR pairs per modality:

- **Metric**: Recall@5 (does correct MCR appear in top-5 memory results?)
- **Target**: Recall@5 ≥ 0.82 for same-modality queries

---

## Implementation Strategy

### Phase 1: Channel Adapter Extension

**Target files**: `src/channels/telegram/`, `src/channels/discord/`, `src/channels/cli/`

**New `IncomingMessage` field**:

```typescript
interface IncomingMessage {
  // ... existing fields ...
  attachments?: RawAttachment[];
}

interface RawAttachment {
  id: string;
  source_type: "image" | "audio" | "video" | "document";
  mime_type?: string;
  file_name?: string;
  file_size_bytes?: number;
  download: () => Promise<Buffer>; // Lazy download
}
```

Telegram voice (`Voice`), photos (`PhotoSize[]`), video notes, and documents all map to `RawAttachment`. Discord `Attachment[]` similarly. CLI: flag-based injection.

**Deliverable**: `IncomingMessage.attachments` populated for all channels.

### Phase 2: MCI Core Pipeline

**New module**: `src/multimodal-context/`

```
src/multimodal-context/
  index.ts                  # Public API: processAttachments()
  pipeline.ts               # Orchestration + Stage 1 router
  types.ts                  # MCR interfaces
  branches/
    visual/
      classifier.ts
      extractors/
        terminal.ts
        code.ts
        ui.ts
        diagram.ts
        chart.ts
        generic.ts
    audio/
      transcriber.ts         # Whisper wrapper
      ner.ts                 # Developer NER
      prosody.ts
    video/
      keyframes.ts
      aggregator.ts
    structured-file/
      csv.ts
      json-yaml.ts
      log.ts
  affinity.ts               # Modality-to-skill affinity matrix
  fusion.ts                 # Cross-modal feature fusion
  cache.ts                  # Content-hash dedup + MCR cache
```

**Rollout order** (highest impact first):

1. Structured file branch (pure algorithmic, lowest risk, highest user value for CSV/log)
2. Visual branch (builds on existing image-handling groundwork, most mature ecosystem)
3. Audio branch (Whisper is production-ready; main work is developer NER)
4. Video branch (most complex, builds on frames from visual + audio pipeline)

### Phase 3: Command Handler Integration

New handler `commands-multimodal-context.ts` in `src/auto-reply/reply/`:

```typescript
export const multimodalContextHandler: CommandHandler = async (
  params: HandleCommandsParams,
  _allowTextCommands: boolean,
): Promise<CommandHandlerResult | null> => {
  const { attachments } = params.message;
  if (!attachments?.length) return null;

  const mcrs = await processAttachments(attachments, params.config.mci);

  // Inject MCR summaries into effective prompt
  const modalBlock = mcrs.map(formatMCRForPrompt).join("\n---\n");
  params.effectivePrompt = `${modalBlock}\n\n${params.effectivePrompt}`;

  // Attach routing features for skill ranker
  params.multimodalRoutingFeatures = buildRoutingFeatures(mcrs);

  // Persist to trajectory
  for (const mcr of mcrs) {
    params.trajectoryLogger.logMCR(mcr);
  }

  // Persist to memory (async, non-blocking)
  persistMCRsToMemory(mcrs, params.config).catch((err) =>
    logger.warn("MCR memory persist failed", err),
  );

  return null; // Continue pipeline
};
```

Register in `src/auto-reply/reply/commands-handlers.runtime.ts` before the main LLM dispatch handler.

### Phase 4: Trajectory & Memory Integration

**Trajectory** (`src/trajectory/`):

- Add `MultimodalContextAttachedEvent` event type to the JSONL stream
- Extend `session.metadata` export to include `attachments.mcrs`

**Memory** (`extensions/memory-*/`):

- Implement `MCRMemoryAdapter`: embed `mcr.text_representation` and store with modality metadata
- Retrieval augments future sessions with `[MCR from 3 sessions ago: ...]` blocks

### Phase 5: ML Routing Integration

Extend skill routing feature vector with multimodal features from MCRs. Retrain LightGBM ranker on MCR-augmented trajectories. Separate ablation study isolating contribution of each modality (visual only, audio only, structured file only, all modalities).

### Phase 6: Benchmark Dataset Release

Publish developer-context multimodal benchmark to HuggingFace Datasets. Include evaluation scripts, MCR schema definition, and baseline model weights.

### Phase 7: Online A/B Test

Deploy MCI with feature flag to 10% of sessions with attachments. Collect data for statistical significance. Report results; iterate on modalities showing below-target improvement.

---

## Expected Outcomes & Contributions

### Quantitative Outcomes

**Classifier accuracy** (offline):

- Visual type: ≥ 90%; Audio type: ≥ 88%; Structured file sub-type: ≥ 95%

**Extraction fidelity** (offline):

- Visual terminal error_class F1: ≥ 0.90
- Audio entity extraction F1: ≥ 0.80
- CSV anomaly detection precision: ≥ 0.85

**Task success rate** (online A/B):

- Visual: +3–5% absolute; Audio: +5–8%; Video: +4–7%; Structured file: +6–10%

**Latency** (end-to-end):

- Image p50 ≤ 150ms; Audio p50 ≤ 400ms; Video p50 ≤ 500ms; File p50 ≤ 100ms

### Qualitative Outcomes

1. **Users can communicate naturally**: Send a voice note, a screenshot, a CSV, or a screen recording without re-typing its content
2. **Structured MCRs are durable**: Unlike raw pixels or audio blobs, MCRs persist in memory and trajectories and remain useful across sessions
3. **Agent needs fewer clarifying questions**: MCR injection pre-answers "what error is shown?", "what does the CSV contain?", "what do you see in the recording?"
4. **Cross-modal corroboration**: When a user sends both a voice note and a screenshot, the agent can confirm consistency between the two, increasing its confidence in the diagnosis

### Research Contributions

1. **MCI architecture**: First channel-agnostic, four-modality grounding pipeline for multi-channel LLM agent systems — covering image, audio, video, and structured files in a unified MCR framework

2. **Multimodal Context Record (MCR) schema**: A typed, versioned, modality-discriminated intermediate representation bridging raw attachment bytes and agent reasoning, designed to be ML-trainable and memory-storable

3. **Developer-context multimodal benchmark**: A labeled dataset spanning all four modalities with developer-context-specific annotations — the first such benchmark for grounding non-text developer inputs to agent skills

4. **Cross-modal skill routing**: Demonstration that multimodal features improve NDCG@10 across all four modality types, with per-modality ablations isolating each modality's contribution

5. **Multimodal trajectory learning**: Integration of MCRs into trajectory logs establishes the data flywheel for future ML models to learn from non-text developer context — a novel capability absent from existing agent learning systems

6. **Developer NER for voice**: A regex + fine-tuned model pipeline for extracting developer entities (error classes, file paths, service names, version strings) from informal spoken audio — addressing a gap in existing NER research

7. **Latency-constrained multimodal design**: Explicit per-modality latency budgets, async processing patterns, and fast/slow path fallbacks — engineering contributions absent from research-prototype multimodal agent papers

---

## Challenges & Limitations

### 1. Audio Quality Degradation

**Problem**: Telegram OGG voice notes are recorded on mobile microphones in noisy environments (office, transit). WER for Whisper on noisy audio can degrade from 4% to 20%+.

**Mitigation**:

- Noise suppression preprocessing (RNNoise or DeepFilterNet, ~30ms overhead)
- Confidence gating: if Whisper segment confidence < 0.6, request re-recording or clarification
- Partial extraction: even noisy transcripts often preserve technical entity strings (error class names, file paths) which are more phonetically distinct than common words

### 2. Privacy Sensitivity

**Problem**: Voice notes contain vocal biometrics; screenshots may contain credentials; CSV/JSON files may contain personal or proprietary data.

**Mitigation**:

- All processing runs locally by default; cloud extraction is opt-in per modality per config
- Credential detection in structured file MCRs (flag `sensitive_keys_detected` and do not log values)
- Audio: transcripts stored, raw audio discarded immediately after transcription
- Image: raw bytes not stored; only structured MCR fields are persisted

### 3. Video Processing Latency on Long Clips

**Problem**: A 60-second screen recording at 30fps = 1,800 frames. Even sampling at 1fps = 60 frames, and running visual branch on 5 key frames, still takes ~5×150ms = 750ms for visual extraction alone, exceeding p99 targets if audio track is also processed.

**Mitigation**:

- Adaptive sampling: for clips >30s, reduce to 1 frame/5sec (12 candidates for 60s, top-3 selected)
- Parallel processing: visual branch and audio track run concurrently
- Progressive injection: inject VideoMCR from first 10s immediately; update with full MCR when complete

### 4. Structured File Size Limits

**Problem**: Users may send large CSVs (>100k rows), enormous log files (>100MB), or deeply nested JSON configs. Full parsing would exceed both latency and memory budgets.

**Mitigation**:

- Size limit enforced at adapter layer: >10MB files get header-only MCR (schema inference from first 1k rows)
- Row sampling for large CSVs: anomaly detection on a stratified 10k-row sample
- Log files: only last 10k lines analyzed (most recent errors most relevant)

### 5. Cross-Modal Inconsistency Handling

**Problem**: User sends a screenshot of a Python error but voice note says "the JavaScript is broken". Which should the agent trust?

**Mitigation**:

- `text_modal_consistency` feature flags the inconsistency in the routing feature vector
- Agent prompt includes a consistency warning when inconsistency is detected: `[Note: voice note (JavaScript) and image (Python error) may refer to different issues]`
- Log inconsistencies for analysis; they are a research signal for ambiguous user intent

### 6. Whisper Latency on CPU-Only Cloud Instances

**Problem**: On a 2-core t3.small (common deployment target), Whisper `turbo` real-time factor on CPU is ~0.3–0.4 (3–4× slower than M-series). A 30s clip takes 9–12 seconds — far beyond the 600ms target.

**Mitigation**:

- Async audio processing: kick off Whisper immediately, inject text-only context into agent, inject AudioMCR when ready (within the same session turn if Whisper completes before model responds)
- Use Whisper `base` (smaller, 4× faster) for latency-critical paths; `turbo` for background/batch processing
- Quantized Whisper (int8) via `faster-whisper` library: ~3× speedup with minimal WER degradation

### 7. Developer NER Generalization

**Problem**: Developer entity vocabulary is vast and evolving. New error class names (from new frameworks), new CLI tools, new service names — a static regex library cannot keep up.

**Mitigation**:

- Regex patterns cover structural invariants (capitalized CamelCase followed by "Error", Unix-style paths, semver strings) which generalize to novel entities
- Periodic regex library updates from new error pattern collections
- CodeBERT NER fine-tuned on a broad error corpus handles novel entities through token-level pattern learning

---

## Future Work

### 1. Modality Synthesis: MCR-to-Image

Reverse direction: given an MCR (e.g. a diagram MCR extracted from an architecture image), generate a new image representation enriched with agent annotations. Useful for explaining discovered issues back to the user visually.

### 2. Real-Time Modality Streaming

Continuous MCI mode: agent monitors a live screen capture stream (every N seconds) and a live audio stream (wake-word activated), maintaining a rolling `current_context_mcrs` set that is always available — analogous to a pair programmer who can see the screen and hear the developer without being explicitly told what is happening.

### 3. Cross-Modal Consistency Training

Train a classifier on (text embedding, MCR embedding) pairs to predict modality-text consistency. Build a labeled dataset of consistent vs. inconsistent multimodal messages. Study what percentage of real-world agent messages exhibit cross-modal inconsistency and what the common confusion patterns are.

### 4. Multimodal Fine-Tuning Data Flywheel

MCI trajectories (MCRs + skill invocations + task success) constitute a labeled multimodal training dataset for future fine-tuning of smaller, faster extraction models calibrated specifically to developer-context inputs. The flywheel: more usage → more labeled MCRs → better fine-tuned extractors → better MCRs → better task success.

### 5. Audio Generation (Output Modality)

Complement audio input with audio output: agent can respond via synthesized speech through the same channel (Telegram voice reply, Discord TTS). Enables truly voice-first developer workflows on mobile. Research question: do audio responses to audio inputs improve user satisfaction vs. text replies?

### 6. Sketch and Annotation Input

Extend structured file branch to accept hand-drawn whiteboard photos with annotation overlays (arrows, circles drawn in messaging apps). Detect annotations as additional signal: "user circled line 42" → entity `{annotated_region: {element: "line_42", annotation_type: "circle"}}`.

---

## References & Code Locations

### Key Files in OpenClaw Codebase

| Component                 | File Path                                           |
| ------------------------- | --------------------------------------------------- |
| Channel layer (Telegram)  | `src/channels/telegram/`                            |
| Channel layer (Discord)   | `src/channels/discord/`                             |
| Auto-reply pipeline       | `src/auto-reply/reply/commands-handlers.runtime.ts` |
| Command handler interface | `src/auto-reply/reply/`                             |
| Skill workspace           | `src/agents/skills/workspace.ts`                    |
| Trajectory export         | `src/trajectory/export.ts`                          |
| Trajectory types          | `src/trajectory/types.ts`                           |
| Embedding infrastructure  | `src/memory-host-sdk/host/embeddings.ts`            |
| Memory extension          | `extensions/memory-*/`                              |
| Config schema             | `src/config/types.openclaw.ts`                      |
| Mobile iOS                | `apps/ios/`                                         |
| Mobile Android            | `apps/android/`                                     |

### External Libraries

| Library                  | Modality        | Purpose                                          |
| ------------------------ | --------------- | ------------------------------------------------ |
| `clip` (OpenAI ViT-B/32) | Visual          | Zero-shot image type classifier                  |
| `trocr` (Microsoft)      | Visual          | High-accuracy printed-text OCR                   |
| PaddleOCR                | Visual          | Multi-layout OCR fallback                        |
| ScreenAI (Google)        | Visual          | UI element detection                             |
| DePlot (Google)          | Visual          | Chart → table conversion                         |
| InternVL-2-1B            | Visual          | Local VLM for privacy-first deployments          |
| EfficientNet-B0          | Visual          | Fine-tuned visual type classifier                |
| `faster-whisper`         | Audio           | Quantized Whisper ASR (3× faster than original)  |
| `openai-whisper`         | Audio           | Reference Whisper implementation                 |
| RNNoise / DeepFilterNet  | Audio           | Audio noise suppression preprocessing            |
| CodeBERT (Microsoft)     | Audio           | Developer-domain NER for transcripts             |
| OpenCV (`cv2`)           | Video           | Frame extraction and visual difference scoring   |
| ffmpeg                   | Video           | Video/audio decode, format normalization         |
| `pandas`                 | Structured file | CSV parsing, schema inference, anomaly detection |
| `@xenova/transformers`   | All             | ONNX transformer inference in Node.js            |

### Academic References

1. **CLIP** (Radford et al., 2021): "Learning Transferable Visual Models From Natural Language Supervision". ICML 2021.

2. **Whisper** (Radford et al., 2022): "Robust Speech Recognition via Large-Scale Weak Supervision". ICML 2023.

3. **TrOCR** (Li et al., 2021): "TrOCR: Transformer-based Optical Character Recognition with Pre-trained Models". AAAI 2023.

4. **ScreenAI** (Baechler et al., 2024): "ScreenAI: A Vision-Language Model for UI and Infographics Understanding". arXiv 2024.

5. **DePlot** (Liu et al., 2022): "DePlot: One-shot visual language reasoning by plot-to-table translation". ACL 2023.

6. **CodeBERT** (Feng et al., 2020): "CodeBERT: A Pre-Trained Model for Programming and Natural Language". EMNLP 2020 Findings.

7. **SeeAct** (Zheng et al., 2024): "GPT-4V(ision) is a Generalist Web Agent, if Grounded". ICML 2024.

8. **WebAgent** (Gur et al., 2023): "A Real-World WebAgent with Planning, Long Context Understanding, and Program Synthesis". ICLR 2024.

9. **AppAgent** (Zhang et al., 2023): "AppAgent: Multimodal Agents as Smartphone Users". arXiv 2023.

10. **SpeechAgents** (Liang et al., 2024): "SpeechAgents: Human-Communication Simulation with Multi-Modal Multi-Agent Systems". arXiv 2024.

11. **AudioPaLM** (Rubenstein et al., 2023): "AudioPaLM: A Large Language Model That Can Speak and Listen". arXiv 2023.

12. **EfficientNet** (Tan & Le, 2019): "EfficientNet: Rethinking Model Scaling for Convolutional Neural Networks". ICML 2019.

13. **InternVL** (Chen et al., 2024): "InternVL: Scaling up Vision Foundation Models and Aligning for Generic Visual-Linguistic Tasks". CVPR 2024.

14. **faster-whisper** (SYSTRAN, 2023): CTranslate2-based reimplementation of Whisper with int8 quantization. https://github.com/SYSTRAN/faster-whisper

15. **DeepFilterNet** (Schröter et al., 2022): "DeepFilterNet: A Low Complexity Speech Enhancement Framework for Full-Band Audio based on Deep Filtering". INTERSPEECH 2022.

---

**End of Research Proposal**
