# Research Proposal: Visual Context Injection — Multimodal Grounding for Autonomous Agent Tasks in OpenClaw

**Author**: Technical Research Proposal
**Date**: 2026-05-03
**Domain**: Multimodal AI, Agent Systems, Computer Vision, Information Extraction
**Code Analysis**: OpenClaw codebase (`openclaw/openclaw`)

---

## Table of Contents

1. [Abstract](#abstract)
2. [Problem Statement](#problem-statement)
3. [Research Objectives](#research-objectives)
4. [Background & Related Work](#background--related-work)
5. [Proposed Approach](#proposed-approach)
6. [Data Sources & Availability](#data-sources--availability)
7. [Visual Feature Taxonomy](#visual-feature-taxonomy)
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

OpenClaw is a multi-channel autonomous agent system (CLI, Telegram, Discord, mobile) that operates exclusively on text. However, every one of its supported channels already carries image inputs — screenshots of errors, UI mockups, architecture diagrams, charts, annotated code — that the agent pipeline silently discards. This creates a fundamental capability gap: users must re-type information the agent can see, if only it were equipped to look.

We propose **Visual Context Injection (VCI)**, a multimodal grounding pipeline that:

1. Ingests images from any OpenClaw channel (Telegram photo, Discord attachment, CLI paste, mobile camera) through a channel-agnostic adapter layer
2. Classifies and decomposes images into structured **Visual Context Records (VCRs)** — typed, schema-validated representations of image content (error traces, code snippets, UI states, data charts, architectural diagrams)
3. Injects VCRs into the agent task context alongside the user's text prompt, providing the LLM with structured, persistent visual evidence rather than raw pixels
4. Extends skill routing to incorporate visual signals — a screenshot of a Python traceback should surface debugging and test-runner skills even if the user's text is just "help"
5. Persists VCRs in trajectory logs and memory, enabling cross-session visual retrieval and learning

**Key contribution**: A production-viable multimodal grounding architecture for LLM agent systems that is (a) latency-constrained (<300ms end-to-end for visual preprocessing), (b) channel-agnostic, (c) structured rather than free-text, enabling downstream ML use, and (d) grounded in the actual OpenClaw codebase's channel, memory, and trajectory infrastructure.

---

## Problem Statement

### Current Multimodal Handling in OpenClaw

**Channel layer** (`src/channels/`): Channel adapters receive incoming messages and normalize them to a shared `IncomingMessage` schema. Image attachments are present in Telegram (`PhotoSize[]`), Discord (`Attachment[]`), and mobile app messages, but are either stripped or passed as opaque URLs/base64 blobs without structured processing.

**Auto-reply pipeline** (`src/auto-reply/reply/`): The agent receives a `HandleCommandsParams` object containing the text message body and session context. No image data flows through this pipeline today.

**Model call layer** (`src/agents/`): Claude (Sonnet 4.6, Sonnet 3.7) supports multimodal inputs natively via the Anthropic API (`image` content blocks). However, even when image data reaches this layer, it is passed raw with no preprocessing, classification, or structured extraction — the model must do all interpretive work from pixels alone, in the same context budget as the task itself.

**Trajectory log schema** (`src/trajectory/types.ts`): The `SessionMetadata` schema has no image attachment field. Visual content is entirely invisible to the trajectory learning infrastructure.

### Critical Gaps

#### Gap 1: The Grounding Gap — Images pass or die

When a Telegram user sends a screenshot of a traceback alongside "fix this", one of two things happens today:

- The channel adapter **strips** the image (most common): the agent sees only "fix this" with no context
- The image is passed as a raw base64 blob to the model context: the model must spend its own reasoning budget interpreting the image, cannot annotate or re-use it, and the interpretation is invisible to downstream systems (routing, memory, trajectory)

Neither outcome produces a **structured, reusable representation** of what the image contained.

#### Gap 2: The Skill Routing Gap — Visual signals ignored

Skill routing (currently alphabetical; see companion proposal `research-skill-routing-ml.md`) takes the user's text prompt as its sole input. But images carry task intent signals that often exceed the text:

| User text            | Image content                          | Correct skill             | Routed skill (text-only) |
| -------------------- | -------------------------------------- | ------------------------- | ------------------------ |
| "help"               | Python traceback                       | `debug`, `test-runner`    | random/alphabetical      |
| "what do you think?" | Figma UI screenshot                    | `frontend-dev`, `css`     | random/alphabetical      |
| "make this work"     | Architecture diagram with missing edge | `design`, `docs`          | random/alphabetical      |
| "check the numbers"  | Bar chart with anomalous spike         | `data-analysis`, `python` | random/alphabetical      |

#### Gap 3: The Memory Gap — Visual context evaporates

OpenClaw's memory system (`extensions/memory-*/`) stores text embeddings for retrieval. Screenshots and diagrams — often the richest context for a task — are never stored, never indexed, never retrievable. A user who sent a system architecture diagram last week cannot ask "remember that diagram I shared?" today.

#### Gap 4: The Trajectory Gap — No visual training signal

Trajectory logs (`~/.openclaw/trajectories/*.jsonl`) capture skill invocations and task prompts for learning. If a user sent a screenshot, the trajectory captures only "help" — masking the true task signal. This corrupts any ML model trained on trajectory data (including the skill routing ML proposed in the companion research).

### Concrete Failure Scenario

**User action**: Opens OpenClaw mobile app (iOS/Android), takes a photo of a terminal showing:

```
ModuleNotFoundError: No module named 'pandas'
Traceback (most recent call last):
  File "analysis.py", line 3, in <module>
    import pandas as pd
```

Sends it with the message: "why isn't this working"

**Current behavior**:

- Image is stripped by the channel adapter
- Agent receives: "why isn't this working" (4 words, no context)
- Agent asks a clarifying question, or tries to help generically
- No debugging skill is loaded; `pandas` import issue is not diagnosed

**Desired behavior (VCI)**:

- Channel adapter extracts image
- VCI pipeline classifies: `image_type: "terminal_screenshot"`, extracts OCR text, parses error type: `ModuleNotFoundError`, identifies module: `pandas`, file: `analysis.py`
- Structured VCR is injected into task context: `{ type: "python_error", error_class: "ModuleNotFoundError", module: "pandas", file: "analysis.py" }`
- Skill router boosts `python-dev`, `package-management`, `debug` skills
- Agent immediately responds: "Your `analysis.py` is missing the pandas package. Run `pip install pandas`"

---

## Research Objectives

### Primary Objective

**Design, implement, and evaluate a latency-constrained multimodal grounding pipeline for LLM agent systems that converts heterogeneous image inputs into structured, reusable Visual Context Records while respecting a <300ms preprocessing budget.**

### Specific Research Questions

#### RQ1: Visual Type Classification

Can we reliably classify developer-context images into a closed taxonomy (terminal screenshot, code editor screenshot, UI/browser screenshot, architecture diagram, data chart, handwritten note, other) using a lightweight vision model?

- **Hypothesis**: A fine-tuned EfficientNet or CLIP-based classifier achieves >90% accuracy on a developer-context image taxonomy
- **Why it matters**: Classification determines the downstream extraction strategy; misclassification leads to wrong extractors and corrupted VCRs

#### RQ2: Structured Extraction Fidelity

For each image type, how accurately can we extract the relevant structured fields (error class, file path, module name, code language, UI element labels, axis labels, etc.)?

- **Hypothesis**: A combined OCR + LLM extraction pipeline achieves >85% field-level F1 on ground-truth annotations for terminal screenshots and code screenshots
- **Harder case**: Architecture diagrams and hand-drawn mockups require spatial understanding beyond OCR

#### RQ3: Visual-Text Consistency

When a user sends image + text, how often are they consistent, and how should inconsistencies be handled?

- **Hypothesis**: 15-20% of image+text pairs contain inconsistencies (user describes different issue than image shows)
- **Impact**: Inconsistencies can mislead the agent; detecting them is a research problem

#### RQ4: Cross-Modal Skill Routing Improvement

Does injecting visual features into the skill routing model improve task success rate versus text-only routing?

- **Hypothesis**: Visual features (image type + extracted entities) improve NDCG@10 by ≥5 points versus text-only routing for tasks with images
- **Evaluation**: A/B test on sessions with image inputs

#### RQ5: Latency Feasibility

Can the full VCI pipeline (channel adapter → classifier → extractor → VCR construction) complete in <300ms on commodity hardware (MacBook M-series, cloud t3.medium)?

- **Constraint**: Agent must not feel slower due to vision preprocessing
- **Trade-off**: Accuracy vs. latency; may require async processing with progressive context injection

#### RQ6: Visual Memory Utility

Does storing VCRs in the memory system and retrieving them in future sessions improve task success on visually-grounded follow-up queries?

- **Hypothesis**: Visual memory retrieval improves relevant skill recall by ≥10% on cross-session visual reference tasks

---

## Background & Related Work

### Multimodal LLMs

**GPT-4o, Claude Sonnet 3.7/4, Gemini 1.5 Pro**:

- All support native image inputs via `image` content blocks in their API
- **Limitation for agents**: They interpret images "on demand" with no persistent structured representation; the same image passed in 10 sessions generates 10 independent interpretations with no learning or cross-session consistency

**LLaVA, CogVLM, InternVL (open source)**:

- Strong vision-language understanding
- Can be deployed locally (<8GB VRAM for quantized versions)
- **Applicable**: Used as the extraction backbone in the VCI pipeline for latency-sensitive deployments

### Structured Information Extraction from Images

**PaddleOCR, Tesseract, TrOCR**:

- High-accuracy OCR for printed text
- TrOCR (Microsoft) achieves near-human accuracy on clean terminal/code screenshots
- **Applicable**: Stage 2 extraction for terminal and code screenshots

**Donut (Document Understanding Transformer)**:

- End-to-end document understanding without explicit OCR
- Encodes image → structured JSON directly
- **Applicable**: Promising for structured form extraction but not code-specific

**ScreenAI (Google DeepMind, 2024)**:

- Specialized for UI screenshot understanding
- Detects UI elements, labels, affordances
- **Applicable**: UI/browser screenshot extraction in VCI

**Code Screenshot Understanding**:

- Limited prior work on structured extraction from code editor screenshots
- Existing work focuses on code retrieval from images (Wang et al., 2022) but not structured entity extraction
- **Gap**: No benchmark for developer-context image grounding in agent systems — this is a research opportunity

### Agent Multimodality

**WebAgent (Google, 2023)**:

- Web navigation agent using screenshots of browser state
- Passes raw screenshots to LLM, no preprocessing
- **Limitation**: No structured extraction; all interpretation left to LLM per turn

**SeeAct (OSU, 2024)**:

- Augments web agents with annotated screenshots
- Draws bounding boxes around interactive elements
- **Applicable**: Spatial annotation approach relevant for UI screenshot VCRs

**AppAgent (Tencent, 2023)**:

- Smartphone UI automation via screenshots
- Uses XML accessibility trees + screenshots
- **Gap**: No cross-session memory or trajectory learning from visual context

### Memory Systems for Agents

**MemGPT (Berkeley, 2023)**:

- Hierarchical memory for long-context agents
- Stores text only
- **Gap**: No vision support

**OpenClaw Memory Extension** (`extensions/memory-*/`):

- Text embedding + retrieval (OpenAI, Voyage, Google providers)
- **Extension point**: VCR text representations can be embedded using existing infrastructure

### Novel Gap This Proposal Addresses

No prior work addresses the combination of:

1. **Channel-agnostic** image ingestion for multi-channel agent systems
2. **Structured** (not free-text) extraction into a typed schema
3. **Trajectory integration** for ML learning from visual context
4. **Skill routing augmentation** via cross-modal features
5. **Latency constraints** (<300ms) for interactive agent systems

---

## Proposed Approach

### Overview: The VCI Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Channel Layer                                │
│  Telegram  Discord  CLI  Mobile  ──► Channel Adapter                │
│                                       ↓                             │
│                              ImageAttachment[]                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────────────┐
│                    VCI Pipeline (< 300ms budget)                     │
│                                                                      │
│  Stage 1: Classifier         Stage 2: Extractor                      │
│  ┌─────────────────┐         ┌──────────────────────────────────┐    │
│  │ EfficientNet-B0 │ ──type─►│ Terminal  → OCR + Error Parser   │    │
│  │ CLIP zero-shot  │         │ Code      → OCR + Language Detect │    │
│  │ (< 20ms)        │         │ UI        → ScreenAI / GPT-4o    │    │
│  └─────────────────┘         │ Diagram   → VLM + Graph Extract  │    │
│                              │ Chart     → ChartQA / Deplot     │    │
│                              │ Other     → VLM caption only     │    │
│                              └──────────────────────────────────┘    │
│                                          ↓                           │
│  Stage 3: VCR Construction   ┌──────────────────────────────────┐    │
│                              │ Visual Context Record (VCR)       │    │
│                              │ { type, entities, text_repr,     │    │
│                              │   confidence, source_channel,    │    │
│                              │   embedding_vector }              │    │
│                              └──────────────────────────────────┘    │
└──────────────────────────┬───────────────────────────────────────────┘
                           ↓
         ┌─────────────────┴──────────────────────┐
         ▼                                        ▼
┌─────────────────────┐               ┌───────────────────────┐
│   Task Context      │               │  Memory + Trajectory  │
│   Injection         │               │  Persistence          │
│                     │               │                       │
│  VCR → structured   │               │  VCR text repr →      │
│  JSON block in      │               │  embedding → memory   │
│  agent prompt       │               │                       │
│                     │               │  VCR schema →         │
│  VCR entities →     │               │  trajectory JSONL     │
│  skill routing      │               │                       │
│  feature vector     │               └───────────────────────┘
└─────────────────────┘
```

### Stage 1: Visual Type Classifier

**Goal**: Assign each incoming image to one of 7 classes in <20ms

**Classes**:

| Class                  | Description                               | Extraction Strategy                   |
| ---------------------- | ----------------------------------------- | ------------------------------------- |
| `terminal_screenshot`  | Terminal/console with text output, errors | OCR → Error parser                    |
| `code_screenshot`      | Code editor or IDE screenshot             | OCR → Language detector + AST snippet |
| `ui_screenshot`        | Browser, app, or web UI screenshot        | ScreenAI / element detection          |
| `architecture_diagram` | System/component/sequence diagram         | VLM + graph structure extractor       |
| `data_chart`           | Bar chart, line chart, scatter, table     | DePlot / ChartQA / tabular extractor  |
| `handwritten`          | Whiteboard, paper sketch, sticky note     | VLM description only                  |
| `other`                | Photo, meme, unclassifiable               | VLM caption only                      |

**Model Approaches** (in order of latency):

1. **CLIP zero-shot** (OpenAI ViT-B/32): Encode image and class descriptions, pick argmax similarity. ~10ms on CPU, ~3ms on GPU. No fine-tuning needed. Expected accuracy: ~75-80% on developer context taxonomy.

2. **Fine-tuned EfficientNet-B0**: Train on labeled developer-context images. ~15ms CPU. Expected accuracy: ~90-92%.

3. **LLM-based classification** (fallback): Send thumbnail to GPT-4o with classification prompt. ~200ms (API latency). Used when confidence is low.

**Decision Logic**:

```python
def classify_image(image: bytes, config: VCIConfig) -> ImageClassification:
    # Fast path: CLIP zero-shot
    clip_result = clip_classify(image, CLASS_DESCRIPTIONS)
    if clip_result.confidence > config.confidence_threshold:
        return clip_result

    # Medium path: EfficientNet (if local model available)
    if config.local_classifier_path:
        return efficientnet_classify(image, config.local_classifier_path)

    # Slow path: LLM classification
    return llm_classify(image, config.llm_provider)
```

### Stage 2: Type-Specific Extractors

Each image type has a specialized extractor producing typed entities.

#### Terminal Screenshot Extractor

**Input**: Terminal screenshot (error, output, command)
**Output**: `TerminalVCR`

```typescript
interface TerminalVCR {
  type: "terminal_screenshot";
  ocr_text: string;
  command?: string; // Last command run
  exit_code?: number;
  error?: {
    error_class: string; // e.g. "ModuleNotFoundError"
    message: string; // e.g. "No module named 'pandas'"
    language?: "python" | "node" | "bash" | "go" | "rust" | "other";
    file?: string; // e.g. "analysis.py"
    line?: number;
    stacktrace_depth?: number;
    module_or_symbol?: string; // e.g. "pandas"
  };
  shell?: "bash" | "zsh" | "fish" | "powershell" | "cmd";
  confidence: number;
}
```

**Pipeline**:

1. TrOCR / PaddleOCR → raw text
2. Error pattern matching (regex library of ~200 patterns per language)
3. Language detection from error format
4. Structured field extraction

**Example patterns**:

```python
ERROR_PATTERNS = {
    "python": [
        r"(?P<error_class>\w+Error): (?P<message>.+)",
        r"ModuleNotFoundError: No module named '(?P<module>[^']+)'",
        r"File \"(?P<file>[^\"]+)\", line (?P<line>\d+)"
    ],
    "node": [
        r"(?P<error_class>\w+Error): (?P<message>.+)",
        r"at (?P<location>\S+) \((?P<file>[^:]+):(?P<line>\d+):\d+\)"
    ],
    "rust": [
        r"error\[(?P<code>E\d+)\]: (?P<message>.+)",
        r"--> (?P<file>[^:]+):(?P<line>\d+):(?P<col>\d+)"
    ]
}
```

#### Code Screenshot Extractor

**Input**: Code editor / IDE screenshot
**Output**: `CodeVCR`

```typescript
interface CodeVCR {
  type: "code_screenshot";
  ocr_text: string;
  language?: string; // Detected programming language
  file_path?: string; // From title bar / breadcrumb
  visible_symbols?: string[]; // Function names, class names visible
  ide?: "vscode" | "vim" | "emacs" | "jetbrains" | "other";
  has_error_markers?: boolean; // Red underlines, error gutter icons
  has_diff?: boolean; // Git diff / comparison view
  confidence: number;
}
```

**Pipeline**:

1. OCR to get code text
2. Language detection (linguist-like heuristics on OCR output)
3. Symbol extraction (regex for `def `, `class `, `function `, `fn `, etc.)
4. IDE detection from UI chrome elements (CLIP similarity to IDE template images)

#### UI Screenshot Extractor

**Input**: Browser/app UI screenshot
**Output**: `UIScreenVCR`

```typescript
interface UIScreenVCR {
  type: "ui_screenshot";
  url?: string; // From browser address bar
  page_title?: string;
  ui_framework_hint?: string; // React, Vue, Tailwind classes visible
  visible_text_blocks: string[]; // Labels, headings, buttons
  error_messages?: string[]; // Alert/error text visible on screen
  form_fields?: { label: string; value?: string }[];
  confidence: number;
}
```

**Pipeline**: ScreenAI element detection + OCR for text labels.

#### Architecture Diagram Extractor

**Input**: Component/sequence/flow diagram
**Output**: `DiagramVCR`

```typescript
interface DiagramVCR {
  type: "architecture_diagram";
  diagram_style?: "component" | "sequence" | "flow" | "erd" | "class" | "other";
  node_labels: string[]; // Component/service names
  edge_labels: string[]; // Relationship labels
  caption?: string; // VLM-generated description
  confidence: number;
}
```

**Pipeline**: VLM (GPT-4o / Claude) with structured extraction prompt targeting nodes and edges.

#### Data Chart Extractor

**Input**: Bar chart, line chart, scatter, table
**Output**: `ChartVCR`

```typescript
interface ChartVCR {
  type: "data_chart";
  chart_type?: "bar" | "line" | "scatter" | "pie" | "table" | "other";
  title?: string;
  x_axis_label?: string;
  y_axis_label?: string;
  series_names?: string[];
  anomalies?: string[]; // Noted by the extractor
  data_summary?: string; // VLM: "Sales peaked in Q3 at $2.1M"
  confidence: number;
}
```

**Pipeline**: DePlot (Google) for chart → table conversion, then tabular summarizer.

### Stage 3: Visual Context Record (VCR)

All extractor outputs are wrapped in a common `VCR` envelope:

```typescript
interface VisualContextRecord {
  id: string; // UUID, stable across sessions
  created_at: string; // ISO timestamp
  source_channel: string; // "telegram", "discord", "cli", "mobile-ios", etc.
  session_id: string;
  image_hash: string; // SHA-256 of raw image bytes (dedup)

  // Typed payload (union discriminated by .type)
  payload: TerminalVCR | CodeVCR | UIScreenVCR | DiagramVCR | ChartVCR | GenericVCR;

  // Cross-modal fields derived from payload
  text_representation: string; // Human-readable summary for LLM prompt injection
  embedding_vector?: Float32Array; // Embedding of text_representation for memory
  skill_routing_features: Record<string, number>; // Feature vector for ML routing
}
```

**Text representation** (injected into agent prompt as structured block):

```
[Visual Context: terminal_screenshot]
Error: ModuleNotFoundError — No module named 'pandas'
File: analysis.py, line 3
Language: Python
Shell: zsh
---
```

This is compact, structured, and far more useful to the LLM than re-describing the raw image.

---

## Data Sources & Availability

### 1. Image Inputs from Channels

**Telegram**: `PhotoSize[]` in `Message.photo`; largest size (~1280px) available via `getFile` API
**Discord**: `Attachment[]` in `Message.attachments`; direct URL download
**CLI**: stdin pipe, drag-and-drop to terminal (iTerm2 / Kitty protocol), or explicit `--image` flag
**Mobile (iOS/Android)**: Camera roll access via React Native image picker, camera capture

**Privacy constraint**: Images contain potentially sensitive content (code, data, personal interfaces). VCI pipeline must operate locally by default, with opt-in cloud VLM processing.

### 2. Trajectory Logs (Extended Schema)

**Extended schema** (new fields added to `SessionMetadata`):

```typescript
interface SessionMetadata {
  // ... existing fields ...
  visual_attachments?: {
    count: number;
    vcrs: VisualContextRecord[];
    processing_latency_ms: number;
  };
}
```

**Training signal**: After VCI integration, trajectories will contain:

- VCR type (image category)
- Extracted entities (error class, language, etc.)
- Whether skill invocations matched VCR-predicted skill needs
- Task success correlated with visual context type

### 3. Developer-Context Image Dataset (Synthetic + Collected)

No public dataset exists specifically for developer-context image grounding. We construct one:

**Synthetic generation**:

- Terminal screenshots: render known error messages using `ansi2html` + screenshot tool
- Code screenshots: render code snippets in VS Code theme using Shiki + headless browser
- Architecture diagrams: export from Mermaid.js with known node/edge sets
- Data charts: generate from pandas/matplotlib with known data

**Size**: ~10,000 images per class (70,000 total), synthetic + manual collection

**Annotation**: Each image labeled with:

- Ground-truth class
- Ground-truth extracted fields (for extraction F1 evaluation)
- Associated skill labels (which OpenClaw skills a developer would need given this image)

### 4. Embedding Infrastructure (Reuse)

**Location**: `src/memory-host-sdk/host/embeddings.ts`

VCR `text_representation` strings are embedded using the existing embedding infrastructure (OpenAI, Voyage, etc.) for storage in the memory system. No new embedding infrastructure needed.

---

## Visual Feature Taxonomy

### Features for Skill Routing Augmentation

When a VCR is produced, it generates a feature vector that extends the skill routing model (complementary to the ML routing proposal):

| Feature                       | Type             | Source                   | Example                             |
| ----------------------------- | ---------------- | ------------------------ | ----------------------------------- |
| `has_image`                   | Binary           | VCI pipeline             | 1 if any image in request           |
| `image_type_{class}`          | Binary (one-hot) | Classifier               | `image_type_terminal_screenshot: 1` |
| `error_language_{lang}`       | Binary (one-hot) | Terminal extractor       | `error_language_python: 1`          |
| `error_class_embedding`       | Float[N]         | Embed error class string | embedding of "ModuleNotFoundError"  |
| `has_error_in_image`          | Binary           | Terminal extractor       | 1 if error detected                 |
| `code_language_{lang}`        | Binary (one-hot) | Code extractor           | `code_language_typescript: 1`       |
| `ui_has_error_message`        | Binary           | UI extractor             | 1 if error alert visible            |
| `diagram_node_count`          | Integer          | Diagram extractor        | 5 (number of components)            |
| `chart_has_anomaly`           | Binary           | Chart extractor          | 1 if anomaly detected               |
| `image_classifier_confidence` | Float            | Classifier               | 0.92                                |
| `vcr_extraction_confidence`   | Float            | Extractor                | 0.87                                |

**Dimension added to routing feature vector**: ~30-50 additional features (manageable)

### Visual-to-Skill Affinity Matrix

A learned or hand-crafted prior for skill routing:

| Image Type + Entity                   | High-affinity skills                                       |
| ------------------------------------- | ---------------------------------------------------------- |
| `terminal_screenshot` + Python error  | `python-dev`, `debug`, `test-runner`, `package-management` |
| `terminal_screenshot` + Node/TS error | `typescript-dev`, `debug`, `npm`                           |
| `terminal_screenshot` + Docker error  | `docker`, `devops`                                         |
| `code_screenshot` + TypeScript        | `typescript-dev`, `code-review`                            |
| `code_screenshot` + diff/PR           | `code-review`, `git`, `github-pr`                          |
| `ui_screenshot` + CSS/HTML visible    | `frontend-dev`, `css`, `accessibility`                     |
| `architecture_diagram`                | `design`, `docs`, `diagram`                                |
| `data_chart`                          | `data-analysis`, `python`, `r-stats`                       |

This matrix is the **prior** used for cold-start routing; the ML model learns from trajectory data to refine it.

---

## Model Architecture

### Component 1: Image Type Classifier

**Architecture**: CLIP ViT-B/32 + linear probe (fine-tuned on developer-context dataset)

**Input**: 224×224 image
**Output**: Softmax over 7 classes + confidence

**Training**:

```python
import torch
import clip
from torch import nn

class DeveloperImageClassifier(nn.Module):
    def __init__(self, num_classes=7):
        super().__init__()
        self.clip, self.preprocess = clip.load("ViT-B/32")
        self.classifier = nn.Sequential(
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(256, num_classes)
        )

    def forward(self, images):
        with torch.no_grad():
            features = self.clip.encode_image(images)
        return self.classifier(features.float())
```

**Zero-shot fallback** (no fine-tuning):

```python
CLASS_PROMPTS = {
    "terminal_screenshot": "a screenshot of a computer terminal showing command output or error messages",
    "code_screenshot": "a screenshot of a code editor or IDE showing source code",
    "ui_screenshot": "a screenshot of a web browser or mobile app user interface",
    "architecture_diagram": "a technical architecture or system design diagram with boxes and arrows",
    "data_chart": "a data visualization chart such as a bar chart, line graph, or scatter plot",
    "handwritten": "a photo of handwritten notes, whiteboard, or paper sketch",
    "other": "a photograph or image that is not a screenshot or diagram"
}

def clip_zero_shot_classify(image: Image.Image) -> dict:
    text_tokens = clip.tokenize(list(CLASS_PROMPTS.values()))
    image_features = model.encode_image(preprocess(image).unsqueeze(0))
    text_features = model.encode_text(text_tokens)
    similarities = (image_features @ text_features.T).softmax(dim=-1)
    return dict(zip(CLASS_PROMPTS.keys(), similarities[0].tolist()))
```

### Component 2: OCR Engine (Terminal + Code)

**Architecture**: TrOCR (microsoft/trocr-base-printed) for clean terminal/code screenshots; PaddleOCR as fallback for complex layouts

**Latency**: ~50-100ms on CPU for a 1280×800 screenshot

**Optimization**: Run OCR asynchronously while classifier runs; results available before stage 2 needs them

### Component 3: VLM Extractor (Diagrams, UI, Charts)

**Architecture**: Two deployment modes

**Mode A (Cloud/Opt-in)**: GPT-4o or Claude with structured extraction prompt

```python
EXTRACTION_PROMPTS = {
    "architecture_diagram": """
Analyze this architecture diagram. Extract as JSON:
{
  "diagram_style": "<component|sequence|flow|erd|class|other>",
  "node_labels": ["<list of component/service names>"],
  "edge_labels": ["<list of relationship/connection labels>"],
  "caption": "<one sentence description of what this architecture shows>"
}
Respond with ONLY the JSON, no explanation.
""",
    "data_chart": """
Analyze this chart. Extract as JSON:
{
  "chart_type": "<bar|line|scatter|pie|table|other>",
  "title": "<chart title if visible>",
  "x_axis_label": "<x-axis label>",
  "y_axis_label": "<y-axis label>",
  "series_names": ["<series names if legend visible>"],
  "data_summary": "<one sentence summarizing the key insight>",
  "anomalies": ["<list any obvious anomalies or outliers>"]
}
Respond with ONLY the JSON, no explanation.
"""
}
```

**Mode B (Local/Default)**: InternVL-2-1B (1B params, ~2GB, runs on CPU at ~800ms) for privacy-first deployments

### Component 4: VCR Persistence

**Trajectory integration** (new event type):

```typescript
// New event in trajectory JSONL stream
interface VCRAttachedEvent {
  type: "visual.vcr_attached";
  ts: string;
  seq: number;
  data: {
    vcr_id: string;
    image_hash: string;
    vcr_type: string;
    processing_latency_ms: number;
    skill_routing_features_count: number;
  };
}
```

**Memory integration** (`extensions/memory-*/`):

VCR `text_representation` is treated as a memory fragment:

```typescript
// Insert VCR as memory entry
await memoryPlugin.insertMemory({
  id: vcr.id,
  content: vcr.text_representation,
  metadata: {
    source: "visual_context",
    image_hash: vcr.image_hash,
    vcr_type: vcr.payload.type,
    channel: vcr.source_channel,
    session_id: vcr.session_id,
  },
  embedding: vcr.embedding_vector,
});
```

---

## Integration with Skill Routing

VCI integrates with the ML skill routing proposal as a **feature source extension**:

```
Text prompt ──────────────────────────────┐
                                          ▼
Visual attachments ──► VCI Pipeline ──► VCR ──► visual_routing_features
                                                    ↓
                                          skill_ranker.predict(
                                            text_features + visual_features
                                          )
                                                    ↓
                                          ranked_skills → truncate by budget
```

**Backwards compatibility**: When no image is present, all `has_image` = 0 and visual features are zeroed. The routing model degrades gracefully to text-only behavior.

**Cold-start routing** (before ML model is trained): Use the Visual-to-Skill Affinity Matrix as a deterministic rule-based booster:

```typescript
function applyVisualSkillBoost(
  skills: SkillEntry[],
  vcr: VisualContextRecord,
  boostAmount: number = 3, // Boost matching skills by 3 rank positions
): SkillEntry[] {
  const boostedSkillNames = getAffinitySkills(vcr);
  const [boosted, rest] = partition(skills, (s) => boostedSkillNames.has(s.skill.name));
  return [...boosted, ...rest];
}
```

---

## Evaluation Methodology

### Offline Evaluation: Classifier

**Dataset**: Held-out 15% of synthetic+collected developer-context image dataset

**Metrics**:

- **Accuracy**: Overall classification accuracy across 7 classes
- **Per-class F1**: F1 score per image class (some classes may be harder)
- **Confusion matrix**: Identify common misclassification pairs (e.g., terminal vs. code)
- **Latency**: Median and p99 classification time on MacBook M-series CPU

**Targets**:

- Overall accuracy ≥ 90%
- Terminal screenshot F1 ≥ 0.92 (most common and most impactful class)
- Median latency ≤ 20ms

### Offline Evaluation: Extraction Fidelity

**Ground truth**: Annotated fields for 500 images per class

**Metric**: Field-level F1 (per extracted field)

```
field_precision = correctly_extracted_fields / total_extracted_fields
field_recall = correctly_extracted_fields / total_ground_truth_fields
field_F1 = 2 * (precision * recall) / (precision + recall)
```

**Targets**:

| Class                | Key Field                 | Target F1 |
| -------------------- | ------------------------- | --------- |
| Terminal screenshot  | `error.error_class`       | ≥ 0.90    |
| Terminal screenshot  | `error.module_or_symbol`  | ≥ 0.85    |
| Code screenshot      | `language`                | ≥ 0.88    |
| Architecture diagram | `node_labels` (set match) | ≥ 0.75    |
| Data chart           | `chart_type`              | ≥ 0.85    |

### Online Evaluation: Task Success with Visual Inputs

**Setup**: A/B test on sessions with ≥1 image attachment

- **Control**: Image passed raw to LLM (current behavior, or image stripped)
- **Treatment A**: VCI pipeline with affinity-based skill boost (no ML routing)
- **Treatment B**: VCI pipeline with ML routing (requires training data)

**Primary metric**: Task success rate (`finalStatus == "success"`)

**Secondary metrics**:

- User follow-up questions on same visual context (lower = better: VCI made image clear first time)
- Skill routing precision for sessions with images (are the right skills loaded?)
- Clarifying question rate (agent asks "can you describe the error?" — lower is better with VCI)

**Statistical test**: Two-proportion z-test (α = 0.05), target n = 2,000 sessions with images per group

### End-to-End Latency Benchmark

**Measure**: Time from image received at channel adapter to VCR injected into agent context

**Target**: p50 ≤ 150ms, p99 ≤ 300ms

**Setup**:

```python
import time
from vci.pipeline import process_image

latencies = []
for image_path in benchmark_images:
    image_bytes = open(image_path, 'rb').read()
    start = time.perf_counter()
    vcr = process_image(image_bytes, config=VCIConfig(mode="fast"))
    latencies.append((time.perf_counter() - start) * 1000)

print(f"p50: {np.percentile(latencies, 50):.1f}ms")
print(f"p99: {np.percentile(latencies, 99):.1f}ms")
```

### Visual Memory Retrieval Evaluation

**Task**: Given a new query that references a previous image ("same error as yesterday's screenshot"), does memory retrieval surface the correct VCR?

**Metric**: Recall@1 and Recall@5 (does correct VCR appear in top-1 or top-5 retrieved results?)

**Evaluation set**: 200 cross-session query-VCR pairs, manually constructed

**Target**: Recall@5 ≥ 0.85 for same-class VCR types

---

## Implementation Strategy

### Phase 1: Channel Adapter Extension

**Target files**: `src/channels/telegram/`, `src/channels/discord/`, `src/channels/cli/`

**Telegram**:

```typescript
// In Telegram channel adapter
interface TelegramIncomingMessage extends IncomingMessage {
  imageAttachments?: {
    fileId: string;
    width: number;
    height: number;
    fileSize: number;
    downloadUrl?: string;
  }[];
}

async function downloadTelegramPhoto(fileId: string, botToken: string): Promise<Buffer> {
  const fileInfo = await telegram.getFile(fileId);
  const url = `https://api.telegram.org/file/bot${botToken}/${fileInfo.file_path}`;
  const response = await fetch(url);
  return Buffer.from(await response.arrayBuffer());
}
```

**Discord**: Similar pattern using `Attachment.url` with authenticated download.

**CLI**: Accept image path via `--image <path>` flag or detect piped image data (PNG/JPEG magic bytes on stdin).

**Deliverable**: `IncomingMessage.imageAttachments?: ImageAttachment[]` field populated for all image-capable channels.

### Phase 2: VCI Core Pipeline

**New module**: `src/visual-context/`

```
src/visual-context/
  index.ts               # Public API
  pipeline.ts            # Main orchestration
  classifier.ts          # Stage 1: type classifier
  extractors/
    terminal.ts          # Terminal screenshot extractor
    code.ts              # Code screenshot extractor
    ui.ts                # UI screenshot extractor
    diagram.ts           # Architecture diagram extractor
    chart.ts             # Data chart extractor
    generic.ts           # Fallback VLM extractor
  types.ts               # VCR interfaces
  affinity.ts            # Visual-to-skill affinity matrix
  cache.ts               # Image dedup + VCR cache
```

**Main orchestration**:

```typescript
// src/visual-context/pipeline.ts
export async function processImageAttachments(
  attachments: ImageAttachment[],
  config: VCIConfig,
): Promise<VisualContextRecord[]> {
  return Promise.all(
    attachments.map(async (attachment) => {
      const imageBytes = await attachment.download();

      // Stage 1: Classify
      const classification = await classifyImage(imageBytes, config);

      // Stage 2: Extract (type-specific)
      const payload = await extractPayload(imageBytes, classification, config);

      // Stage 3: Construct VCR
      return constructVCR({
        attachment,
        classification,
        payload,
        config,
      });
    }),
  );
}
```

**Deliverable**: Working VCI pipeline with terminal screenshot support (most impactful class first).

### Phase 3: Task Context Injection

**Target file**: `src/auto-reply/reply/` (command handler pipeline)

Add a new handler `commands-visual-context.ts` that:

1. Checks `params.message.imageAttachments`
2. Runs VCI pipeline
3. Prepends VCR text representations to the effective task prompt
4. Attaches `skill_routing_features` to routing context

```typescript
// src/auto-reply/reply/commands-visual-context.ts
export const visualContextHandler: CommandHandler = async (
  params: HandleCommandsParams,
  _allowTextCommands: boolean,
): Promise<CommandHandlerResult | null> => {
  const attachments = params.message.imageAttachments;
  if (!attachments?.length) {
    return null; // No images, pass through
  }

  const vcrs = await processImageAttachments(attachments, params.config.vci);

  // Inject VCR summaries into prompt context
  const visualBlock = vcrs.map(formatVCRForPrompt).join("\n---\n");
  params.effectivePrompt = `${visualBlock}\n\n${params.effectivePrompt}`;

  // Attach routing features
  params.visualRoutingFeatures = buildRoutingFeatures(vcrs);

  // Persist to trajectory
  for (const vcr of vcrs) {
    params.trajectoryLogger.logVCR(vcr);
  }

  return null; // Continue pipeline
};
```

**Deliverable**: Agent receives structured visual context for all image-bearing messages.

### Phase 4: Trajectory & Memory Integration

**Trajectory** (`src/trajectory/`): Add `VCRAttachedEvent` to the event schema and export pipeline.

**Memory** (`extensions/memory-*/`): Implement `VCRMemoryAdapter` that converts VCRs to memory fragments with embeddings.

**Deliverable**: VCRs persisted in trajectories (training data for future ML) and retrievable from memory.

### Phase 5: ML Routing Integration

Extend the skill routing feature vector (from the companion ML routing proposal) with visual features from VCRs. Retrain the LightGBM ranker on trajectories that now include visual signals.

**Deliverable**: Routing model that outperforms text-only routing on image-bearing sessions.

### Phase 6: Evaluation & A/B Test

Run online A/B test measuring task success rate with and without VCI. Collect 2,000+ sessions per arm.

**Deliverable**: A/B test results report, decision to ship or iterate.

---

## Expected Outcomes & Contributions

### Quantitative Outcomes

**Classifier** (offline):

- Type classification accuracy: ≥ 90%
- Terminal screenshot F1: ≥ 0.92
- Median classification latency: ≤ 20ms

**Extraction fidelity** (offline):

- Terminal error field F1: ≥ 0.87 average across key fields
- Code language detection: ≥ 0.88

**Task success (online A/B test)**:

- Sessions with images, VCI treatment vs. control: +3-7% absolute improvement in task success rate
- Clarifying question rate: -20-30% reduction (agent needs less clarification)
- Skill routing precision@5 for image sessions: +8-12 points NDCG improvement

**End-to-end latency**:

- p50 ≤ 150ms, p99 ≤ 300ms (fast-path CLIP + OCR mode)

### Qualitative Outcomes

1. **Users can share images instead of typing**: A common developer pain point (re-typing error messages) is eliminated
2. **Agent understands visual context without a free-form describe-the-image roundtrip**: No extra LLM turn needed; VCI is pre-processing not in the agent's reasoning loop
3. **Visual context persists across sessions**: First multimodal memory capability in an open-source agent system of this architecture
4. **Structured VCRs are ML-trainable**: Unlike raw pixel data, VCRs are tabular-compatible features that future models can learn from

### Research Contributions

1. **Novel architecture**: First channel-agnostic, structured multimodal grounding pipeline for multi-channel LLM agent systems — as opposed to monolithic single-interface approaches (WebAgent, SeeAct)

2. **Visual Context Record (VCR) schema**: A typed, schema-validated intermediate representation for developer-context images, bridging computer vision and agent reasoning

3. **Developer-context image benchmark**: A labeled dataset (synthetic + collected) for developer-context image classification and extraction, filling a gap in existing benchmarks (which focus on natural images, documents, or web UI — not terminal/code screenshots)

4. **Cross-modal skill routing**: First demonstration that visual features improve skill/tool routing in LLM agent systems, with quantified NDCG improvement

5. **Visual trajectory learning**: VCR integration into trajectory logs enables future ML models to learn from visual task context — establishing the data flywheel for improving multimodal agent behavior from usage

6. **Latency-constrained design**: Explicit <300ms budget and multi-path (fast/slow) design is a concrete engineering contribution absent from research-prototype multimodal agent papers

---

## Challenges & Limitations

### 1. OCR Quality on Low-Resolution Screenshots

**Problem**: Mobile camera photos of screens, or low-DPI screenshots, degrade OCR accuracy significantly below benchmark conditions.

**Mitigation**:

- Super-resolution preprocessing (Real-ESRGAN) for mobile photos before OCR
- Confidence gating: if OCR confidence < 0.6, fall back to VLM description
- Prompt users to send higher-quality screenshots when confidence is low

### 2. Privacy and Data Sensitivity

**Problem**: Screenshots may contain API keys, credentials, personal data, proprietary code.

**Mitigation**:

- VCI runs locally by default; cloud VLM is opt-in only
- Credential detection: scan OCR text for patterns matching API keys, tokens, passwords before persisting
- Redact detected credentials from VCRs before storing in trajectory or memory
- GDPR/privacy: VCRs stored locally, never uploaded without explicit consent

### 3. Hallucination in VLM Extractors

**Problem**: VLM-based extractors (for diagrams, charts) may hallucinate node labels, relationships, or data values not visible in the image.

**Mitigation**:

- Confidence calibration: VLM extractors output `confidence` scores; low-confidence VCRs are marked and treated as soft hints rather than hard facts
- Structured output parsing with validation: reject VCR fields that fail schema validation
- Human-readable `text_representation` shown to user for confirmation on low-confidence extractions

### 4. Latency Budget Pressure

**Problem**: The 300ms budget is tight for the full pipeline on CPU-only hardware (cloud t2.small, older MacBooks).

**Mitigation**:

- **Async processing**: Start VCI pipeline immediately upon image receipt, in parallel with text processing; inject VCR if ready before model call, otherwise skip and log
- **Fast path**: CLIP zero-shot (10ms) + OCR (80ms) for terminal/code screenshots fits budget easily; slow path (VLM, 200ms+) only for diagrams/charts
- **Progressive context**: If VCI completes after model call starts, VCR is stored for next turn

### 5. Diagram Extraction Quality

**Problem**: Architecture diagrams vary enormously in style (Mermaid, Lucidchart, hand-drawn, Visio). No single extractor handles all styles well.

**Mitigation**:

- Style classifier (sub-classifier within the diagram extractor): detect diagram tool from visual signature
- Tool-specific parsers where format is machine-generated (Mermaid diagrams often have text labels at regular grid positions)
- VLM caption as fallback: even if structured extraction fails, a one-sentence VLM description is useful

### 6. Channel Fragmentation

**Problem**: Each channel (Telegram, Discord, CLI, mobile) has different image APIs, size limits, formats, and authentication models.

**Mitigation**:

- Channel adapter abstraction (`ImageAttachment` interface) hides channel-specific download logic
- Normalization to JPEG/PNG before VCI pipeline; all downstream components are format-agnostic
- Size limits enforced at adapter layer: images >5MB are downsampled before processing

### 7. Cross-Modal Inconsistency

**Problem**: User text says "python error" but image shows a Node.js traceback. Which should the agent trust?

**Mitigation**:

- Flag inconsistencies in VCR metadata (`text_image_consistency: "low"`)
- When inconsistent, present both interpretations to the agent and let it resolve
- Log inconsistencies for analysis; they may indicate a category of confused user requests worth addressing in UX

---

## Future Work

### 1. Video / Screen Recording Support

Extend VCI to handle short screen recordings (GIF, MP4) — a common format for sharing bug reproductions. Key frame extraction → per-frame VCRs → temporal summary VCR.

### 2. Annotated Screenshot Protocol

User can annotate a screenshot with circles/arrows before sending (common in Slack/Discord). Detect annotations as additional signal: "user pointed at line 42" → VCR entity `{annotated_region: {y: 42, annotation_type: "circle"}}`.

### 3. Cross-Modal Consistency Training

Train a small classifier that takes (text embedding, VCR embedding) and predicts consistency. Use this to:

- Flag inconsistencies for user clarification
- Build a dataset of consistent vs. inconsistent multimodal requests
- Study what percentage of real-world agent image+text requests are actually consistent

### 4. Visual Skill Generation

Go beyond routing existing skills: if VCI detects an unfamiliar UI or technology (novel framework, internal tool), suggest to the user that a new skill could be created. Use VCR entities as seed content for a SKILL.md template.

### 5. Multimodal Fine-Tuning Data Collection

VCI trajectories (with VCRs + skill invocations + task success) constitute a labeled multimodal training dataset for future fine-tuning of smaller, faster extraction models specifically calibrated to developer-context images.

### 6. Real-Time Screen Sharing Mode

A continuous VCI mode where the agent monitors a live screen capture stream (every N seconds), maintaining a rolling `current_screen_vcr` that is always available as context — analogous to how a human pair programmer can see the screen without being explicitly told what is on it.

### 7. Spatial Reasoning Integration

Extend UI screenshot VCRs with bounding box coordinates for detected elements. Enable queries like "fix the button in the top-right" by grounding spatial references to actual UI element coordinates. Required for full UI automation use cases.

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

| Library                | Purpose                                 | Notes                            |
| ---------------------- | --------------------------------------- | -------------------------------- |
| `clip` (OpenAI)        | Zero-shot image classifier              | Python; JS bindings via ONNX     |
| `trocr` (Microsoft)    | High-accuracy printed-text OCR          | HuggingFace; ~300MB model        |
| PaddleOCR              | Multi-language OCR fallback             | Strong on mixed layouts          |
| ScreenAI (Google)      | UI element detection                    | Available via Vertex AI          |
| DePlot (Google)        | Chart → table conversion                | HuggingFace                      |
| InternVL-2-1B          | Local VLM for privacy-first deployments | 2GB, CPU-runnable                |
| Real-ESRGAN            | Super-resolution for low-quality inputs | Optional preprocessing           |
| `@xenova/transformers` | ONNX transformer inference in Node.js   | Enables local CLIP in TypeScript |

### Academic References

1. **CLIP** (Radford et al., 2021): "Learning Transferable Visual Models From Natural Language Supervision". ICML 2021.

2. **TrOCR** (Li et al., 2021): "TrOCR: Transformer-based Optical Character Recognition with Pre-trained Models". AAAI 2023.

3. **ScreenAI** (Baechler et al., 2024): "ScreenAI: A Vision-Language Model for UI and Infographics Understanding". arXiv 2024.

4. **DePlot** (Liu et al., 2022): "DePlot: One-shot visual language reasoning by plot-to-table translation". ACL 2023.

5. **SeeAct** (Zheng et al., 2024): "GPT-4V(ision) is a Generalist Web Agent, if Grounded". ICML 2024.

6. **WebAgent** (Gur et al., 2023): "A Real-World WebAgent with Planning, Long Context Understanding, and Program Synthesis". ICLR 2024.

7. **AppAgent** (Zhang et al., 2023): "AppAgent: Multimodal Agents as Smartphone Users". arXiv 2023.

8. **EfficientNet** (Tan & Le, 2019): "EfficientNet: Rethinking Model Scaling for Convolutional Neural Networks". ICML 2019.

9. **InternVL** (Chen et al., 2024): "InternVL: Scaling up Vision Foundation Models and Aligning for Generic Visual-Linguistic Tasks". CVPR 2024.

10. **Multimodal Grounding Survey** (Gan et al., 2022): "Vision-Language Pre-training: Basics, Recent Advances, and Future Trends". Foundations and Trends in Computer Graphics and Vision.

---

**End of Research Proposal**
