Title: Learning to Route - ML-Based Skill Selection for Autonomous Agents

Column 1 Upper Part:
Background and problem statement:
Current OpenClaw skill selection (workspace.ts:101-122):

- Alphabetical sorting - skill "aaa-irrelevant" ranks higher than "zzz-critical"
- Hard budget limits - max 150 skills, max 18,000 chars
- Task-agnostic - no consideration of prompt content or user intent
- No learning - repeated similar tasks reload identical skill lists
- Budget waste - irrelevant skills consume prompt space, relevant skills dropped

Critical Failure Example:
User: "Create a GitHub pull request with my recent commits"
Relevant skills: github-pr, git-commit, code-review (may be ranked 47th, 68th, 93rd)
Alphabetically higher: android-build, audio-processing, aws-deploy (ranks 1st, 2nd, 3rd)
→ If 200 skills exist, top 150 alphabetical may exclude github-pr entirely
→ Model cannot invoke skill it doesn't know exists
→ Task fails, user intervention required

Performance Impact:

- ~15-25% wasted prompt budget on irrelevant skills
- ~10-20% relevant skills dropped beyond budget cutoff
- Zero adaptation to user patterns or task characteristics
- No improvement over time despite trajectory data available

Column 1 Middle Part:
Diagram Description for Visual Creation:

[TOP-DOWN FLOW WITH THREE PARALLEL STREAMS]

1. START - Three input sources (top):
   **Left stream - User Task:**
   - User prompt box: "Create a GitHub pull request..."
   - Arrows pointing down labeled: "text", "channel", "timestamp"

   **Center stream - Skill Inventory:**
   - Grid of skill boxes (5x6 grid showing ~30 skills)
   - Each box labeled: "skill name", "description", "metadata"
   - Color-coded by source: workspace (blue), bundled (green), managed (yellow)

   **Right stream - Historical Context:**
   - Database cylinder icon
   - Labels: "Trajectory logs", "Usage stats", "Co-occurrences"

2. FEATURE EXTRACTION LAYER (middle section):
   Four parallel processing boxes feeding into feature vector:

   **Box 1 - Textual Similarity:**
   - Compute prompt embedding (small neural network icon)
   - Pre-computed skill embeddings (cache icon)
   - Cosine similarity calculation
   - Output features: cos_sim, tfidf_overlap, jaccard_bigrams, keyword_match

   **Box 2 - Skill Metadata:**
   - Parse frontmatter (YAML icon)
   - Check requirements (bins, env vars)
   - Source priority scoring
   - Output features: source_priority, has_always_flag, os_compatible, num_required_bins

   **Box 3 - Historical Patterns:**
   - Query trajectory database
   - Compute invocation rates
   - Find co-occurrences
   - Output features: global_invocation_rate, recency_days, co_occurrence_score, success_rate

   **Box 4 - Contextual Signals:**
   - Extract session context
   - Time features
   - Channel type
   - Output features: channel_type, hour_of_day, conversation_turn, has_code_block

   [FEATURE MERGE]
   All four boxes converge into single "Feature Vector" box:
   - Show dimensions: ~40-50 features
   - Visual: horizontal bar divided into colored segments matching the 4 categories

3. ML RANKING MODEL (center):
   Large box labeled "LightGBM Ranker"
   - Input: Feature matrix (N skills × 40 features)
   - Internal: Tree ensemble icon (3-4 decision trees shown)
   - Output: Relevance scores [0.92, 0.87, 0.15, 0.08, ...]
   - Small annotation: "<1ms inference"

4. RANKING & BUDGET ENFORCEMENT (bottom middle):
   **Sort by Score:**
   - Skills reordered by relevance descending
   - Show top-5 skills with high scores in green
   - Show bottom skills with low scores in red

   **Budget Truncation:**
   - Horizontal bar showing budget limits
   - "Top 150 skills" marker
   - "18,000 chars" marker
   - Scissors icon cutting at budget limit

5. OUTPUT TO AGENT (bottom):
   - Ranked skill list → Agent prompt
   - Show example: [github-pr (0.92), git-commit (0.87), code-review (0.81), ...]
   - Annotation: "Relevant skills prioritized, irrelevant skills dropped"

6. FEEDBACK LOOP (right side, flowing back up):
   - Agent invokes skills → Trajectory logger
   - Skill invocations recorded
   - Arrow feeding back to Historical Context (top right)
   - Label: "Continuous learning from usage"

[KEY VISUAL ELEMENTS]

- Use grayscale gradient for skill scores (dark=high relevance, light=low relevance)
- Show actual numbers: "Before: 25% budget waste → After: <5% budget waste"
- Highlight the alphabetical vs. ML comparison with side-by-side mini-lists
- Add small "NDCG@10: 0.75" metric badge on model output
- Show inference latency as speedometer icon: "<1ms"

Column 1 Bottom Part:
Related Works:

1. Learning to Rank (Burges, 2010) - LambdaRank/LambdaMART for NDCG optimization
2. LightGBM (Microsoft, 2017) - Efficient gradient boosting for ranking tasks
3. XGBoost (Chen & Guestrin, 2016) - Scalable tree boosting system
4. BM25 (Robertson & Zaragoza, 2009) - Classical TF-IDF ranking
5. Cold Start in Recommender Systems (Schein et al., 2002) - Content-based filtering
6. SHAP (Lundberg & Lee, 2017) - Model explainability for tree ensembles
7. Toolformer (Meta, 2023) - Teaching LLMs to use tools (requires fine-tuning)

Column 2 Upper Part:
Key Challenges:
• Label Noise - Invoked skills assumed relevant, but model may explore or miss skills due to poor alphabetical ranking
• Class Imbalance - ~97% of (task, skill) pairs are negative (skill not invoked in session)
• Cold Start Problem - New skills have zero historical usage data
• Concept Drift - User task patterns change over time, model becomes stale
• Embedding Cost - Computing prompt embeddings may add latency (API calls ~20-50ms)
• Feature Distribution Shift - Training data from CLI may not represent Telegram/Discord usage patterns
• Model Staleness - Training on old data may not reflect current skill inventory
• Overfitting to Alphabetical Bias - Training data generated under alphabetical ranking may encode spurious correlations

Column 2 Middle and Bottom Part:
Key Technologies/Proposal:

1. Learning to Rank Framework
   • Frame as ranking problem: predict relevance score per (task, skill) pair
   • Objective: Maximize NDCG@10 (Normalized Discounted Cumulative Gain)
   • Constraint: Select top-k skills fitting budget (≤150 skills, ≤18,000 chars)
   • Training signal: Trajectory logs with skill invocation labels (no manual annotation)

2. Training Data Construction from Trajectories
   • Parse ~/.openclaw/trajectories/\*.jsonl files
   • Extract per session: user prompt + available skills + invoked skills
   • Positive examples: (task, skill) where skill.invoked == true
   • Negative examples: (task, skill) where skill.invoked == false
   • Filter: Use only successful sessions (finalStatus == "success") to reduce label noise
   • Dataset size: 50k-1.5M examples from 1k-10k sessions

3. Feature Engineering (40-50 dimensions total)

   **Textual Similarity (5 features):**
   • cos_sim_desc_prompt: Cosine similarity of embeddings (reuse OpenClaw embedding infrastructure)
   • tfidf_overlap: TF-IDF weighted token intersection
   • jaccard_bigrams: Character bigram Jaccard similarity
   • keyword_match_count: Skill name tokens appearing in prompt
   • levenshtein_dist: Edit distance between skill name and first prompt word

   **Skill Metadata (9 features):**
   • source_priority: Workspace=6, project=5, personal=4, managed=3, bundled=2, extra=1
   • has_always_flag: Binary (always: true in frontmatter)
   • requires_bins_satisfied: All required binaries available
   • requires_env_satisfied: All required env vars present
   • os_compatible: Skill supports current OS
   • num_required_bins: Count of required binaries
   • num_required_env: Count of required env vars
   • is_user_invocable: userInvocable flag
   • skill_name_length: Character count of name

   **Historical Patterns (6 features):**
   • global_invocation_rate: Fraction of sessions where skill invoked
   • user_invocation_rate: Per-user invocation rate
   • recency_days: Days since last invocation
   • co_occurrence_score: P(skill | previously invoked skill in session)
   • session_position_avg: Average turn position when invoked
   • success_rate: Fraction of invocations leading to session success

   **Contextual Signals (6+ features):**
   • channel_type: One-hot (cli, telegram, discord, mobile)
   • agent_id: One-hot (agent identifier)
   • is_cron_trigger: Binary (automated vs. user-initiated)
   • hour_of_day: 0-23 ordinal
   • conversation_turn: Count in multi-turn conversation
   • prompt_length: Character count
   • has_code_block: Binary (``` present in prompt)

4. Model Architecture: LightGBM Ranker
   • Objective: lambdarank (pairwise ranking loss)
   • Metric: NDCG@10 (optimized during training)
   • Hyperparameters: 100-200 trees, depth=6, learning_rate=0.05
   • Handles class imbalance via ranking loss (no class_weight needed)
   • Fast inference: <1ms for 150 skills × 40 features
   • Interpretable: SHAP feature importance analysis

5. Offline Evaluation Metrics
   • Precision@K: Fraction of top-K that are relevant (Target: P@10 ≥0.6)
   • Recall@K: Fraction of relevant skills in top-K (Target: R@150 ≥0.95)
   • NDCG@10: Ranking quality metric (Target: ≥0.7)
   • ROC-AUC: Binary classification metric (Target: ≥0.85)
   • Inference latency: p50 <1ms, p99 <5ms

6. Online A/B Test Design
   • Control: Alphabetical skill ranking (current system)
   • Treatment: ML-based ranking
   • Assignment: Per-user 50/50 split
   • Primary metric: Task success rate (Hypothesis: +2-5% improvement)
   • Secondary metrics: Avg skills invoked per session, session duration, skill cache hit rate
   • Sample size: ~10k sessions per group (2-4 weeks)

7. Cold Start & Robustness
   • Content-based fallback: New skills use only similarity + metadata features (no historical)
   • Alphabetical fallback: If model fails to load, graceful degradation
   • Incremental learning: Retrain monthly on fresh trajectory logs
   • Confidence thresholding: Use alphabetical for out-of-distribution tasks (low feature confidence)

8. Integration into OpenClaw
   • Location: src/agents/skills/workspace.ts:101-122 (filterSkillEntries)
   • New module: src/agents/skills/ml-ranking.ts (rankSkillsML function)
   • Config schema: config.skills.mlRanking.enabled (feature flag)
   • Model artifacts: ~/.openclaw/skills/skill_ranker.txt (LightGBM model file)
   • Pre-computed caches: skill_embeddings.pkl, skill_stats.json
   • Initialization: Load model + caches at startup (<100ms)

Expected Outcomes:
• Task success rate: +2-5% absolute (80% → 82-85%)
• Skill budget utilization: -15-25% wasted prompt space
• Precision@10: 0.6-0.8 (60-80% of top-10 relevant)
• Recall@150: ≥0.95 (95%+ relevant skills in budget)
• NDCG@10: 0.7-0.85
• Inference latency: <1ms median, <5ms p99

Implementation Phases:
Phase 1: Data collection (parse trajectories, extract examples)
Phase 2: Feature engineering (implement extractors)
Phase 3: Model training (LightGBM ranker + hyperparameter tuning)
Phase 4: Offline evaluation (test set metrics + SHAP analysis)
Phase 5: Integration (workspace.ts modification + ml-ranking.ts module)
Phase 6: A/B testing (50/50 split, 2-4 weeks)
Phase 7: Rollout decision (deploy or rollback based on results)
