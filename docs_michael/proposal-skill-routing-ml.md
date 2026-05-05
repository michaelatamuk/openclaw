Title: Learning to Route - ML-Based Skill Selection for Autonomous Agents

Column 1 Upper Part:
Background and problem statement:
Current OpenClaw skill selection:

- Alphabetical sorting - "aaa-irrelevant" ranks higher than "zzz-critical"
- Hard budget limits - max 150 skills, max 18k chars
- Task-agnostic - no prompt content or user intent consideration
- No learning - repeated tasks reload identical lists

Performance Impact: ~15-25% wasted budget | ~10-20% relevant skills dropped | Zero adaptation

Column 1 Middle Part:
Diagram:
[Task + Skills + History] → Feature Extraction (Similarity/Metadata/Historical/Contextual) → LightGBM Ranker → Ranked Skills → Budget Truncation → Agent
👉 ML ranking replaces alphabetical - relevant skills prioritized, irrelevant dropped
NDCG@10: 0.7-0.85 | Precision@10: 0.6-0.8 | Task success: +2-5% | Inference: <1ms

Column 1 Bottom Part:
Related Works:

1. LambdaRank (Burges, 2010) - NDCG optimization
2. LightGBM (Microsoft, 2017) - Efficient gradient boosting
3. XGBoost (Chen & Guestrin, 2016) - Scalable tree boosting
4. BM25 (Robertson, 2009) - TF-IDF ranking
5. SHAP (Lundberg, 2017) - Model explainability

Column 2 Upper Part:
Key Challenges:
• Label Noise - Invoked skills assumed relevant, may be exploratory
• Class Imbalance - ~97% negative examples (skill not invoked)
• Cold Start - New skills have zero historical data
• Concept Drift - User patterns change, model becomes stale
• Embedding Cost - Prompt embeddings add latency (~20-50ms API)
• Alphabetical Bias - Training data reflects alphabetical ranking artifacts

Column 2 Middle and Bottom Part:
Key Technologies/Proposal:

1. Learning to Rank Framework
   • Predict relevance score per (task, skill) pair
   • Objective: Maximize NDCG@10
   • Training: Trajectory logs with skill invocation labels (no manual annotation)
   • Dataset: 50k-1.5M examples from 1k-10k sessions

2. Feature Engineering (40-50 dimensions)
   **Textual Similarity:** cos_sim (embeddings), tfidf_overlap, jaccard_bigrams, keyword_match
   **Skill Metadata:** source_priority, always_flag, os_compatible, required_bins/env
   **Historical Patterns:** invocation_rate, recency_days, co_occurrence, success_rate
   **Contextual Signals:** channel_type, hour_of_day, conversation_turn, has_code_block

3. LightGBM Ranker Model
   • Objective: lambdarank (pairwise ranking loss)
   • 100-200 trees, depth=6, learning_rate=0.05
   • Handles class imbalance via ranking loss
   • Fast inference: <1ms for 150 skills × 40 features
   • SHAP feature importance for interpretability

4. Cold Start & Robustness
   • Content-based fallback: New skills use similarity + metadata only
   • Alphabetical fallback: Graceful degradation if model fails
   • Incremental learning: Retrain monthly on fresh trajectories
   • Confidence thresholding: Use alphabetical for OOD tasks

5. Integration (workspace.ts:101-122)
   • New module: ml-ranking.ts (rankSkillsML function)
   • Config flag: config.skills.mlRanking.enabled
   • Pre-computed caches: skill_embeddings.pkl, skill_stats.json
   • A/B test: 50/50 split, 10k sessions per group, 2-4 weeks
