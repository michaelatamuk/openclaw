Title: Multimodal Context Injection - Grounding Non-Text Inputs for Autonomous Agents

Column 1 Upper Part:
Background and problem statement:
Current OpenClaw agents (CLI, Telegram, Discord, mobile):

- Text-only processing - images, audio, video, files are stripped or ignored
- Users must re-type screenshots, describe voice messages, explain attachments
- All channels carry richer inputs that the pipeline discards
- No cross-modal signals for skill routing
- No multimodal memory or trajectory learning

Gap Impact:

- "help" + screenshot of Python error → agent sees only "help"
- Voice message describing Docker failure → completely stripped
- CSV data file → invisible to agent, user must paste manually
- Screen recording of UI crash → dropped entirely

Column 1 Middle Part:
Diagram Description for Visual Creation:

[LEFT TO RIGHT FLOW]

1. START: Four parallel input streams entering from left:
   - IMAGE icon (camera/photo symbol)
   - AUDIO icon (microphone/waveform symbol)
   - VIDEO icon (film/play button symbol)
   - FILE icon (document/spreadsheet symbol)

2. MODALITY ROUTER (central diamond decision node):
   - Routes each input to specialized branch
   - Shows MIME type inspection + magic bytes

3. FOUR PARALLEL PROCESSING BRANCHES (vertical lanes):

   **Visual Branch:**
   - Classifier box → "Type: terminal/code/UI/diagram/chart"
   - OCR + VLM box → "Extract: error_class, file_path, language"
   - Output: VisualMCR structure

   **Audio Branch:**
   - Whisper box → "Transcribe: text + timestamps"
   - NER box → "Extract: errors, paths, services"
   - Prosody box → "Analyze: urgency, speaking_rate"
   - Output: AudioMCR structure

   **Video Branch:**
   - Keyframe selector box → "Select top-5 frames"
   - Visual analysis per frame → "Detect errors in frames"
   - Audio track extraction → feeds to Audio Branch
   - Output: VideoMCR structure

   **Structured File Branch:**
   - Parser box → "CSV/JSON/YAML/Log"
   - Schema inference box → "Detect columns, types"
   - Anomaly detection box → "Outliers, nulls, errors"
   - Output: StructuredFileMCR structure

4. MCR CONSTRUCTION (merge point):
   - All branches converge into single MCR envelope
   - Box showing: "id, timestamp, payload, text_repr, routing_features, embedding"

5. DUAL OUTPUT PATHS (split right):

   **Top path - Task Context:**
   - MCR text_repr → Agent prompt (structured block)
   - routing_features → Skill ranker boost

   **Bottom path - Persistence:**
   - MCR → Memory system (embeddings + retrieval)
   - MCR → Trajectory logs (ML training data)

6. END: Agent with enhanced context (icon showing agent + multiple modality symbols)

[KEY VISUAL ELEMENTS]

- Use color coding: Blue=Visual, Green=Audio, Red=Video, Yellow=File
- Show latency budgets as small text annotations (<300ms, <600ms, etc.)
- Add example annotations in small text: "Error: ModuleNotFoundError", "Voice: Docker config missing", "CSV: 500 rows, NULL spike"
- Show feedback arrow from Trajectory back to future routing (learning loop)

Column 1 Bottom Part:
Related Works:

1. Whisper (OpenAI, 2022) - State-of-the-art ASR, local-deployable
2. TrOCR (Microsoft, 2021) - Transformer OCR for printed text
3. ScreenAI (Google, 2024) - UI screenshot understanding
4. DePlot (Google, 2022) - Chart-to-table conversion
5. CodeBERT (Microsoft, 2020) - Developer-domain NER
6. SeeAct (2024) - Vision-language for web agents
7. AudioPaLM (Google, 2023) - Joint audio-text models

Column 2 Upper Part:
Key Challenges:
• The Grounding Gap - Non-text inputs currently stripped or passed raw with no structured representation
• The Cross-Modal Routing Gap - Skill routing ignores image/audio/video/file signals (alphabetical sort only)
• The Memory Gap - Screenshots, voice notes, videos never stored, never retrievable across sessions
• The Trajectory Gap - ML training data captures only sparse text, masking true task drivers
• Latency Constraints - Must process within <300-800ms for real-time agent interaction
• Privacy Sensitivity - Voice biometrics, credentials in screenshots, proprietary data in files
• Audio Quality - Noisy mobile recordings degrade Whisper WER from 4% to 20%+
• Developer Entity Extraction - Error classes, file paths, service names in informal voice notes

Column 2 Middle and Bottom Part:
Key Technologies/Proposal:

1. Multimodal Context Records (MCRs)
   • Typed, schema-validated intermediate representation per modality
   • Persistent across sessions (not raw bytes)
   • Structured fields: entities, text_repr, confidence, routing_features
   • Content-hash deduplication + embedding for memory retrieval

2. Per-Modality Processing Pipelines

   **Visual (p50 ≤150ms):**
   • CLIP/EfficientNet-B0 classifier → 7 types (terminal, code, UI, diagram, chart, handwritten, other)
   • TrOCR/PaddleOCR → Extract error_class, file_path, language from terminal screenshots
   • ScreenAI → UI element detection for app screenshots
   • DePlot → Chart-to-data conversion with anomaly detection

   **Audio (p50 ≤400ms for ≤30s clips):**
   • Whisper turbo (809M params) → Transcription with word timestamps
   • Developer NER (regex + fine-tuned CodeBERT) → Extract error classes, file paths, services, versions
   • Prosody analysis → Speaking rate, pause count, urgency score
   • Confidence gating → Request re-recording if WER high

   **Video (p50 ≤500ms for ≤60s clips):**
   • Uniform frame sampling (1 fps) + significance scoring (frame diff + entropy)
   • Top-K keyframe selection (K=5 default)
   • Reuse visual branch for per-frame analysis
   • Audio track → Parallel audio branch processing
   • Temporal aggregation → State transitions, error frame timestamps

   **Structured Files (p50 ≤100ms for ≤10MB):**
   • CSV: Pandas schema inference + anomaly detection (outliers, null spikes, type inconsistency)
   • JSON/YAML: Key traversal + validation error detection + sensitive key flagging
   • Log files: Error pattern extraction (regex + frequency analysis), last 10k lines only
   • Size limits: >10MB gets header-only MCR to respect latency budget

3. Cross-Modal Skill Routing Integration
   • MCR routing_features extend text-only skill ranker
   • Modality-to-Skill Affinity Matrix for cold-start (visual terminal + Python → python-dev, debug, package-management)
   • Multi-attachment handling via element-wise max (preserve strongest signal)
   • Backwards compatible: zero features when no attachments

4. Memory & Trajectory Persistence
   • MCR text_repr → Embeddings for cross-session retrieval
   • Extended SessionMetadata schema with attachments field
   • ML training flywheel: MCRs + skill invocations + task success → future model training
   • Privacy: All processing local by default, cloud extraction opt-in per modality

5. Channel-Agnostic Adapter Layer
   • Unified RawAttachment interface across Telegram, Discord, CLI, mobile
   • Lazy download (only when needed for processing)
   • MIME type + magic byte detection (no ML overhead)

6. Developer-Context Benchmark Dataset (New Contribution)
   • ~70k images (terminal, code, UI, diagrams)
   • ~5k audio clips (synthetic TTS + volunteer recordings)
   • ~1k video clips (scripted screen recordings + real debugging sessions)
   • ~10k structured files (synthetic anomalies + real logs)
   • Ground-truth labels: modality type, extracted entities, associated skills

Expected Outcomes:
• Task success rate: +3-5% (visual), +5-8% (audio), +4-7% (video), +6-10% (files)
• Extraction fidelity: Visual error F1 ≥0.90, Audio entity F1 ≥0.80, CSV anomaly precision ≥0.85
• Latency: Image p50 ≤150ms, Audio p50 ≤400ms, Video p50 ≤500ms, File p50 ≤100ms
• Cross-modal NDCG@10 improvement: +5 pts (image), +7 pts (audio), +8 pts (files)

Implementation Phases:
Phase 1: Channel adapter extension (attachments field)
Phase 2: MCI core pipeline (all four branches)
Phase 3: Command handler integration (MCR injection)
Phase 4: Trajectory & memory integration
Phase 5: ML routing integration (feature vector extension)
Phase 6: Benchmark dataset release
Phase 7: Online A/B test (10% sessions with attachments)
