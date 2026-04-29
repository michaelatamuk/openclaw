# Research Proposal: Learning to Route — Efficient Skill Selection for Autonomous Agents using Classical Machine Learning

**Author**: Technical Research Proposal
**Date**: 2026-04-28
**Domain**: Agent Systems, Classical Machine Learning, Information Retrieval
**Codebase**: OpenClaw (`openclaw/openclaw`)

---

## Table of Contents

1. [Abstract](#abstract)
2. [Problem Statement](#problem-statement)
3. [Research Objectives](#research-objectives)
4. [Background & Related Work](#background--related-work)
5. [Proposed Approach](#proposed-approach)
6. [Data Sources & Availability](#data-sources--availability)
7. [Feature Engineering](#feature-engineering)
8. [Model Architecture](#model-architecture)
9. [Evaluation Methodology](#evaluation-methodology)
10. [Implementation Strategy](#implementation-strategy)
11. [Expected Outcomes & Contributions](#expected-outcomes--contributions)
12. [Challenges & Limitations](#challenges--limitations)
13. [Future Work](#future-work)
14. [References & Code Locations](#references--code-locations)

---

## Abstract

Modern autonomous agent systems — OpenClaw, AutoGPT, LangChain, and others — rely on **skills**: specialized instruction sets (markdown files) that guide large language models in domain-specific tasks. When a user sends a prompt, the agent presents the model with a catalog of available skills, and the model decides which ones to read and follow.

The critical decision of _which skills to include in that catalog_ is currently made naively: skills are sorted alphabetically and then cut off by hard budget limits, with no consideration for whether those skills are actually relevant to the task at hand. This creates three compounding problems. First, irrelevant skills consume prompt budget that could go to relevant ones. Second, relevant skills with names that sort late in the alphabet may be dropped entirely simply because they weren't alphabetically first. Third, the system never learns — it makes the same blind choices on every prompt regardless of what has worked before.

We propose **Learning to Route (L2R)**, a classical machine learning approach that replaces alphabetical sorting with learned relevance ranking. The model predicts, for any incoming task, which skills are most likely to be useful — and places those at the front of the catalog before any budget truncation occurs. This means the 150 skills the model sees are actually the 150 most relevant ones, rather than just the 150 that happen to come first alphabetically.

The approach uses gradient boosted decision trees (XGBoost or LightGBM) trained on signals extracted from four sources: skill descriptions and metadata, user prompt content, historical invocation patterns from trajectory logs, and contextual signals like channel type or time of day. All training data is already being collected automatically — no manual labeling is required. Inference runs in under one millisecond, adding no perceptible latency to the agent pipeline.

The key contribution is a production-ready skill routing system that improves agent efficiency without requiring labeled datasets, deep learning infrastructure, or GPU hardware.

---

## Problem Statement

### How Skill Selection Works Today

When a user sends a prompt to OpenClaw, the agent must decide which skills to include in the model's system prompt. The current algorithm works in five steps. First, all skills are loaded from the configured source directories — workspace, bundled skills, managed skills, personal dotfiles, and any extras. Second, skills that fail eligibility checks are filtered out: skills whose required binaries aren't installed, whose environment variables are absent, or whose operating system requirements don't match the current host are excluded. Third, the remaining eligible skills are sorted alphabetically by name. Fourth, the sorted list is truncated to a hard limit of 150 skills. Fifth, if those 150 skills still exceed the character budget of 18,000 characters, the list is trimmed further — first by switching to a compact format that omits descriptions, and then, if still too large, by binary-searching for the largest subset that fits.

The alphabetical sort is the core problem. It is entirely task-agnostic — it knows nothing about what the user actually asked for.

### Critical Flaws

**Alphabetical ordering ignores task relevance entirely.** A skill named `aaa-irrelevant` will always rank above a skill named `zzz-critical`, regardless of what the user asked. Skill names were chosen by their authors to be descriptive, not to optimize for alphabetical position, so there is no reason to expect that alphabetical order correlates with relevance to any particular task.

**Budget limits drop relevant skills with no recourse.** The default limit is 150 skills. If a user has 500 skills installed, the last 350 alphabetically are completely invisible to the model — it has no idea they exist and cannot use them. A skill named `github-pr` is dropped not because it is irrelevant, but because 150 skills starting with letters before 'g' happened to be installed. The model cannot invoke what it doesn't know about, so the user's task fails or degrades.

**The system never learns.** Every prompt gets the same alphabetically-ordered skill list, regardless of what has worked before. A user who always works with Git will always see the same 150 skills, including whichever irrelevant ones happen to alphabetically precede their Git skills.

**Prompt budget is wasted.** The 18,000-character budget is a scarce resource. Every character spent on an irrelevant skill is a character that could have conveyed a useful one. Alphabetical ordering makes no attempt to use this budget wisely.

### Concrete Example

Consider a user who sends: _"Create a GitHub pull request with my recent commits."_

The skills that should be prioritized are `github-pr`, `git-commit`, and `code-review`. However, if the user also has skills named `android-build`, `audio-processing`, and `aws-deploy`, those three alphabetically precede all three relevant skills and take their slots in the prompt. If the user is near the 150-skill limit, the relevant skills may not appear in the prompt at all. The model does not see `github-pr`, does not know it exists, and cannot use it. The user's task degrades to whatever the model can accomplish without that guidance.

---

## Research Objectives

### Primary Objective

Develop a classical ML-based skill routing system that maximizes task success rate while respecting hard budget constraints — choosing the most relevant skills, not the alphabetically earliest ones.

### Specific Research Questions

**RQ1: Predictability.** Can we predict which skills the model will invoke for a given task, using only information available at routing time — before the model runs? Our hypothesis is that skill invocation is predictable from the combination of task text, skill metadata, and historical patterns. The null hypothesis is that invocation is essentially random from the system's perspective.

**RQ2: Feature Importance.** Which signals are most predictive of skill relevance? Is it primarily the semantic similarity between the user's prompt and the skill's description? Or do historical patterns dominate — the fact that this skill has been useful on similar tasks before? Or does environmental context (the channel, the agent, the time of day) play a meaningful role? Understanding this ranking matters both for the model's accuracy and for its interpretability.

**RQ3: Model Selection.** Among classical ML approaches — logistic regression, gradient boosted trees, random forests, and k-nearest neighbors — which achieves the best precision/recall tradeoff for the skill ranking task? Each has different characteristics around interpretability, inference speed, and accuracy on tabular data.

**RQ4: Generalization.** Does a trained model generalize across different users, different agent configurations, new skills it has never seen before (the cold-start problem), and different task domains? A system that works only for one user's patterns is not production-ready.

**RQ5: Efficiency.** Can inference complete in under one millisecond per routing decision? The current alphabetical sort is essentially free, so any ML-based approach must add negligible latency to avoid a regression in responsiveness.

**RQ6: Online Improvement.** Can the model improve over time as new trajectory logs accumulate, without requiring manual re-labeling? Users' patterns evolve, new skills are added, and the system should adapt automatically.

---

## Background & Related Work

### Agent Systems and Tool/Skill Selection

**OpenClaw** is the system under study. It treats skills as markdown instruction files (SKILL.md) that the model reads using its built-in file-reading tool when it judges a skill to be relevant. The current selection mechanism is alphabetical plus budget limits, with no learned relevance. This is the gap we address.

**AutoGPT, LangChain, and ReAct** focus on tool selection — deciding which APIs or functions to call — rather than instruction skill selection. These systems often solve the selection problem by asking the LLM itself which tool to use, which costs an additional model call and its associated latency and token cost. Our approach avoids this by making the routing decision with a lightweight ML model before the LLM is ever invoked.

**Toolformer** (Meta AI, 2023) takes a different approach: it fine-tunes the LLM itself to recognize when and how to use external tools, embedding tool selection into the model weights. This is inapplicable to our setting because OpenClaw uses closed frontier models — Claude, GPT-4, and others — which cannot be fine-tuned.

### Classical ML for Information Retrieval

**BM25** (Robertson & Zaragoza, 2009) is the canonical TF-IDF-based ranking function used in search engines. It treats documents as bags of words and ranks them by term frequency weighted against inverse document frequency. In our setting, skill descriptions are the documents and user prompts are the queries. BM25 provides a strong no-training-required baseline.

**Learning to Rank (LTR)** is a family of ML techniques specifically designed for ranked retrieval (Burges et al., 2005; Liu, 2009). Rather than classifying documents as relevant or not, LTR models learn to order a set of candidates by relevance for a given query. Gradient boosted trees trained with ranking losses like LambdaRank (which directly optimizes NDCG) are the dominant LTR approach in production systems — they power ranking in Bing, Yandex, and most large-scale search and recommendation systems. This is our primary technical approach.

**Cold Start in Recommender Systems** is the problem of ranking items with no usage history. When a new skill is installed, it has zero invocations and no historical signal. Content-based filtering (Pazzani & Billsus, 2007) — ranking by item attributes and description similarity rather than usage history — is the standard solution. Our feature set includes both content-based and history-based signals so the system degrades gracefully for new skills.

### Trajectory Learning for Agents

**Behavioral cloning** (Pomerleau, 1988) learns a policy by imitating demonstrated behavior. In our setting, we treat the model's own skill selections as expert demonstrations — the model is intelligent and generally reads skills when they are useful, so its historical choices are informative labels. This is the conceptual basis for our label construction.

**Inverse Reinforcement Learning** (Abbeel & Ng, 2004) infers a reward function from observed behavior, enabling more general policy learning. This is more powerful than behavioral cloning but also far more complex. For our narrowly-scoped ranking problem, behavioral cloning from trajectory logs is sufficient; IRL would be overkill.

---

## Proposed Approach

### Problem Formulation

We formulate skill selection as a **Learning to Rank** problem. The input is a user task (a prompt text) and a set of eligible skills that have passed the existing eligibility filters. The output is those skills ranked by predicted relevance — highest relevance first. The budget constraint (150 skills, 18,000 characters) is then applied to the ranked list rather than the alphabetically sorted list. The effect is that the top of the list contains the skills most likely to be useful, rather than the skills whose names happen to come earliest alphabetically.

Formally: given a task T and eligible skills S = {s₁, s₂, ..., sₙ}, produce a ranking where skills with higher predicted relevance to T appear first, then select the top-k that fit the character budget.

### Training Data Construction

The training data already exists. OpenClaw automatically records every agent session as a trajectory log, stored as JSONL files under `~/.openclaw/trajectories/`. Each trajectory records the user's prompt, the full list of skills that were available in that session, and — critically — which skills the model actually read. A skill is marked as invoked when the model issues a file-read call targeting that skill's SKILL.md path.

From each session, we extract a set of (task, skill, label) triples. For every skill that was available in the session, we create one training example. The label is positive (1) if the model read that skill during the session, and negative (0) if the skill was available but not read. The training prompt is the content of the user's first message in the session.

Estimated dataset sizes given typical usage: between one thousand and ten thousand sessions translate to between fifty thousand and one and a half million training triples. The class distribution is heavily skewed — roughly 97% of examples are negative, since the model reads only a handful of skills per session out of the 50-150 that are available.

### Key Assumptions

**Assumption 1: Invoked skills were relevant.** We assume that when the model reads a skill file, it does so because the skill is genuinely useful for the task. This is reasonable — frontier models like Claude Sonnet 4.6 are capable enough that they rarely read files unnecessarily. The main risk is that the model explores skills out of uncertainty. Mitigation: restrict training data to sessions that completed successfully (`finalStatus == "success"`), where the model's skill selections were clearly productive.

**Assumption 2: Non-invoked skills were not relevant.** We assume that if a skill was available but not read, it was not needed. The main risk here is that the current alphabetical ranking system causes relevant skills to be dropped before they appear in the prompt — meaning a skill might be labeled "not invoked" not because it was irrelevant but because it was never shown to the model. Mitigation: use trajectory data collected after the alphabetical issue is acknowledged, when skill sets are relatively small and truncation is less common.

**Assumption 3: Stable distribution.** We assume user task patterns are sufficiently stable that a model trained on past sessions predicts future sessions. The main risk is concept drift — users' workflows evolve over time. Mitigation: incremental retraining on fresh logs, plus time-based weighting that gives more influence to recent sessions.

---

## Data Sources & Availability

### Trajectory Logs

The primary training data source is the trajectory logs OpenClaw already writes automatically. These are stored as newline-delimited JSON files at `~/.openclaw/trajectories/`. Each line is a structured event with a type, timestamp, session ID, and event-specific data payload.

The events most relevant to training are:

- **Session metadata events**, which record the full list of skills that were available in the session — including, for each skill, its name, description, file path, source directory, and whether it was ultimately invoked. This is where the labels come from.
- **User message events**, which record the content of each message the user sent. The first user message in a session is the task prompt.
- **Tool call events**, which record every tool invocation the model made. File-read tool calls targeting skill paths are how the system detects skill invocation — the metadata event's `invoked` flag is set based on these.

The trajectory format has a defined schema version (`schemaVersion: 1`) and is designed for long-term retention. Privacy is entirely local — these files are never uploaded.

### Skill Metadata

Each skill is defined by a markdown file with a YAML frontmatter header. The frontmatter records the skill's name, a natural language description of what it does, which operating systems it supports, which binary tools it requires (like `git` or `docker`), which environment variables it needs (like API keys), whether it should always be included regardless of budget, and whether users or only the model can invoke it. All of these are available as features at routing time.

### Existing Embedding Infrastructure

OpenClaw already has an embedding pipeline built for its memory and RAG features. This infrastructure — which connects to OpenAI, Voyage, Google, Mistral, and Amazon Bedrock embedding APIs — can be reused directly for skill routing without building anything new.

The key optimization is to pre-compute embeddings for all skill descriptions offline, once, and cache them to disk. At routing time, only the user's prompt needs to be embedded (a ~20-50ms API call), and then cosine similarities against the cached skill embeddings can be computed in under a millisecond. This makes the semantic similarity feature cheap at inference time even though the embedding model itself is expensive.

Available embedding models include OpenAI's `text-embedding-3-small` (1536 dimensions) and `text-embedding-3-large` (3072 dimensions), Voyage AI's `voyage-2` (1024 dimensions), and Google's `embedding-001` (768 dimensions).

### Session Transcripts

Full conversation logs are stored separately at `~/.openclaw/sessions/`. These enable extracting richer context for multi-turn conversations — specifically, which skills were invoked earlier in the same session (useful for computing co-occurrence features) and the full conversation history leading up to each turn.

### Configuration Files

User and workspace configuration files at `~/.openclaw/config.json` and `<workspace>/.openclaw/config.json` record each user's skill allow/deny preferences, agent-specific overrides, and runtime settings. These provide user-specific context signals that can be used as features.

---

## Feature Engineering

The feature vector for each (task, skill) pair is assembled from four categories. The total dimensionality is approximately 30-40 features, depending on how many channels and agents are one-hot encoded.

### Category 1: Skill-Task Textual Similarity

**Hypothesis**: Skills whose descriptions are semantically similar to the user's prompt are more likely to be invoked.

These features capture different facets of text similarity between the user's prompt and the skill's name/description:

| Feature               | Description                                                                                                | Range  |
| --------------------- | ---------------------------------------------------------------------------------------------------------- | ------ |
| `cos_sim_desc_prompt` | Cosine similarity between the embedding of the skill description and the embedding of the user prompt      | [0, 1] |
| `tfidf_overlap`       | Sum of TF-IDF weights for terms that appear in both the prompt and the skill description                   | [0, ∞) |
| `jaccard_bigrams`     | Jaccard similarity of character bigrams between prompt and description — captures morphological similarity | [0, 1] |
| `keyword_match_count` | Number of tokens in the skill name that appear in the prompt text                                          | [0, N] |
| `levenshtein_dist`    | Normalized edit distance between the skill name and the first significant word of the prompt               | [0, ∞) |

The cosine similarity feature is the most expensive to compute because it requires embedding both texts. The optimization is to pre-compute skill embeddings offline and cache them — then at routing time, only one embedding (the prompt) needs to be fetched. Cosine similarity computation itself on a cached 1536-dimensional vector takes under a millisecond.

### Category 2: Skill Metadata Features

**Hypothesis**: Structural properties of the skill — where it came from, what it requires, whether it's flagged as always-relevant — correlate with invocation likelihood independent of the prompt content.

| Feature                   | Type    | Description                                                                                                       | Values |
| ------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------- | ------ |
| `source_priority`         | Ordinal | Source directory priority: workspace skills are most specific to the project, bundled skills are generic defaults | 1–6    |
| `has_always_flag`         | Binary  | Whether the skill's frontmatter sets `always: true`, marking it as universally applicable                         | 0 or 1 |
| `requires_bins_satisfied` | Binary  | Whether all binary tools the skill requires (e.g., `git`, `docker`) are currently available                       | 0 or 1 |
| `requires_env_satisfied`  | Binary  | Whether all environment variables the skill needs (e.g., API keys) are present                                    | 0 or 1 |
| `os_compatible`           | Binary  | Whether the skill's OS requirements include the current platform                                                  | 0 or 1 |
| `num_required_bins`       | Count   | How many binaries the skill depends on — a proxy for its complexity                                               | [0, N] |
| `num_required_env`        | Count   | How many environment variables the skill needs                                                                    | [0, N] |
| `is_user_invocable`       | Binary  | Whether the skill can be invoked by a user command (vs. model-only)                                               | 0 or 1 |
| `skill_name_length`       | Count   | Character length of the skill name                                                                                | [1, ∞) |

Source priority is encoded numerically in the order: workspace (6) → project .agents (5) → personal .agents (4) → managed (3) → bundled (2) → extra dirs (1). A workspace skill is more likely to be relevant because it was placed there explicitly for the current project.

### Category 3: Historical Usage Features

**Hypothesis**: Skills that have been useful in the past — globally across all sessions, or specifically alongside other skills already invoked in the current session — are more likely to be useful again.

These features are precomputed by scanning all trajectory logs once and then updated incrementally as new sessions complete. At routing time, they are simply looked up from an in-memory table.

| Feature                  | Type  | Description                                                                                                                        | Computation                                                          |
| ------------------------ | ----- | ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `global_invocation_rate` | Float | Fraction of all sessions where this skill was invoked                                                                              | total invocations ÷ total sessions                                   |
| `user_invocation_rate`   | Float | The same rate, restricted to the current user's sessions                                                                           | user invocations ÷ user sessions                                     |
| `recency_days`           | Float | Days since the skill was last invoked — more recent = more relevant                                                                | (current time − last invocation timestamp) in days                   |
| `co_occurrence_score`    | Float | Whether this skill tends to be invoked alongside skills already used in the current session                                        | max co-occurrence count with any previously-invoked skill in session |
| `session_position_avg`   | Float | Average ordinal position when invoked (first skill read = position 1, etc.) — skills read first tend to be more broadly applicable | mean(invocation positions)                                           |
| `success_rate`           | Float | Fraction of sessions where this skill was invoked that completed successfully                                                      | successes ÷ invocations                                              |

The co-occurrence score is particularly valuable for multi-step tasks. If the user has already read the `git-commit` skill earlier in their session, the `github-pr` skill has a high co-occurrence score with `git-commit` — making it more likely to rank high for the next prompt in that session.

### Category 4: Contextual Features

**Hypothesis**: The channel, agent configuration, time of day, and structural properties of the current prompt affect which skills are appropriate.

| Feature             | Type        | Description                                                                  | Values          |
| ------------------- | ----------- | ---------------------------------------------------------------------------- | --------------- |
| `channel_type`      | Categorical | The channel through which the message arrived — CLI, Telegram, Discord, etc. | One-hot encoded |
| `agent_id`          | Categorical | Which agent configuration is handling the session                            | One-hot encoded |
| `is_cron_trigger`   | Binary      | Whether the task was triggered by a cron job rather than a user              | 0 or 1          |
| `hour_of_day`       | Ordinal     | Hour of day when the session started (0–23)                                  | [0, 23]         |
| `day_of_week`       | Ordinal     | Day of week (0 = Monday, 6 = Sunday)                                         | [0, 6]          |
| `conversation_turn` | Count       | Which turn in the conversation this prompt represents                        | [1, ∞)          |
| `prompt_length`     | Count       | Character length of the user's prompt                                        | [1, ∞)          |
| `has_code_block`    | Binary      | Whether the prompt contains a fenced code block                              | 0 or 1          |

Channel type matters because CLI prompts tend to be shorter and more command-like, while Telegram/Discord prompts tend to be more conversational. An agent that handles both will benefit from knowing which channel is active. Time features capture patterns like "this user always works on deployment tasks on Friday afternoons" — which the model cannot know without temporal context.

### Feature Vector Summary

| Category   | Number of Features    | Dimensions |
| ---------- | --------------------- | ---------- |
| Similarity | 5                     | 5          |
| Metadata   | 9                     | 9          |
| Historical | 6                     | 6          |
| Contextual | 6 + one-hot encodings | 6–26       |
| **Total**  |                       | **~30–40** |

Sparsity is moderate — the one-hot encodings for channel type and agent ID introduce sparse dimensions, but the numerical features are dense.

---

## Model Architecture

### Candidate Models

**Logistic Regression** is the baseline. It learns a linear combination of features that predicts the probability a skill will be invoked. Its advantages are fast training, fast inference (sub-millisecond), and fully interpretable coefficients — you can directly read off which features are most predictive. Its disadvantage is that it assumes the relationship between features and relevance is linear, which is almost certainly false: a skill is highly relevant when _both_ the description matches _and_ it has been recently used, not just when either condition holds independently. Logistic regression cannot capture such interactions without explicit feature engineering.

**Gradient Boosted Trees** (XGBoost or LightGBM) is the primary proposal. These models learn ensembles of decision trees, where each tree corrects the errors of the previous ones. They naturally capture non-linear interactions and feature dependencies without any manual feature crossing. They are robust to feature scaling (no normalization required), can output calibrated probability scores, and provide feature importance via SHAP values. Critically for our setting, they have state-of-the-art performance on tabular ranking tasks — the same family of models powers search ranking at Bing, Yandex, and most large recommendation systems. Inference on a 100-tree ensemble with 40 features takes well under one millisecond on CPU. The disadvantage relative to logistic regression is that hyperparameter tuning is required and the model is less directly interpretable (though SHAP values make it explainable post-hoc).

**Random Forests** are an alternative ensemble approach that builds trees independently rather than sequentially (as gradient boosting does). They are more robust to overfitting than gradient boosting but consistently less accurate on ranking tasks. Inference is also somewhat slower because random forests typically need more trees to match gradient boosting's accuracy.

**k-Nearest Neighbors (kNN)** takes a completely different approach: rather than learning a model, it finds the k most similar historical tasks to the current prompt (by embedding distance) and ranks skills by how often they were invoked in those similar tasks. The advantage is that it requires no training and handles new skills naturally — if a new skill description matches the query, it gets a high similarity score regardless of its invocation history. The disadvantage is slow inference: finding the k nearest neighbors requires a search over all historical task embeddings, which takes tens of milliseconds at scale rather than under a millisecond.

### Recommended Approach

**Primary model: LightGBM Ranker** with the LambdaRank objective.

LambdaRank directly optimizes NDCG (Normalized Discounted Cumulative Gain) — the standard ranking quality metric. Unlike a binary classifier trained on invocation labels, LambdaRank understands that the ordering of skills matters, not just which ones are classified as relevant. It penalizes errors at the top of the ranking more than errors lower down, which is exactly what we want: getting the first 10 skills right matters more than getting the 140th skill right.

The model is trained with the skills for each session grouped together — the loss function operates on rankings within a session, not across sessions. This is essential for a ranking model because relevance is always relative: a skill might be the best choice for one task and irrelevant for another.

**Baseline model: Logistic Regression** for sanity-checking and interpretability.

**Zero-training fallback: BM25** for when no trajectory data is available (new installations, fresh users).

---

## Evaluation Methodology

### Offline Evaluation

**Dataset split**: The trajectory logs are split chronologically into 70% training (earliest sessions), 15% validation (middle), and 15% test (latest sessions). This time-based split is critical — it simulates the real deployment scenario where we train on the past and predict the future. A random split would contaminate the test set with data from the same time period as training, producing overly optimistic results.

#### Ranking Metrics

**Precision at K (P@K)** measures what fraction of the top-K skills the model predicts are ones the model actually invoked. If we predict the top 10 most relevant skills and the model actually invoked 6 of them, P@10 = 0.6. Our target is P@10 ≥ 0.6.

**Recall at K (R@K)** measures what fraction of the skills the model actually invoked appear in our top-K predictions. If the model invoked 3 skills and 2 of them appear in our top-10 predictions, R@10 = 0.67. The most important recall target is R@150 ≥ 0.95 — we want to ensure that 95% of the skills the model would actually use appear somewhere in the 150 skills we show it.

**Mean Reciprocal Rank (MRR)** is the average of the inverse rank of the first actually-invoked skill across sessions. If the first relevant skill appears at rank 1, MRR = 1.0; if it appears at rank 5, MRR = 0.2. A higher MRR means relevant skills appear earlier in the ranking. Our target is MRR ≥ 0.5.

**Normalized Discounted Cumulative Gain at K (NDCG@K)** is the most comprehensive ranking metric. It rewards ranking relevant skills highly and penalizes ranking them low, with a logarithmic discount — a relevant skill at rank 1 is worth more than one at rank 5. It is normalized against the ideal ranking (what NDCG would be if we ranked perfectly). Our target is NDCG@10 ≥ 0.7.

#### Classification Metrics

Since each (task, skill) pair also has a binary label (invoked or not), we can also evaluate with standard classification metrics:

- **ROC-AUC** (area under the ROC curve): threshold-independent measure of discrimination ability. Target: ≥ 0.85.
- **PR-AUC** (area under the Precision-Recall curve): more appropriate than ROC-AUC for imbalanced classes (our 97% negative rate makes ROC-AUC overly optimistic). Target: ≥ 0.4.
- **F1 Score** at the optimal threshold: harmonic mean of precision and recall. Target: ≥ 0.5.

#### Efficiency Metrics

- **Inference latency**: median < 1ms, 99th percentile < 5ms for ranking 150 skills.
- **Memory footprint**: model file + cached embeddings + statistics < 100 MB total.

### Online Evaluation (A/B Test)

After offline validation passes, the system is tested in production with a randomized A/B experiment:

- **Control group** (50% of users): existing alphabetical ranking.
- **Treatment group** (50% of users): ML-based ranking.
- **Assignment**: per-user (stable assignment so users don't see inconsistent behavior across sessions).

**Primary metric**: Task success rate — the fraction of sessions where the agent reports `finalStatus == "success"`. This is already tracked in trajectory logs and requires no additional instrumentation.

**Statistical test**: Two-proportion z-test with α = 0.05 (95% confidence) and 80% power to detect a 2% absolute improvement in success rate (from 80% to 82%). This requires approximately 10,000 sessions per group.

**Secondary metrics**: Average number of skills invoked per session (lower = more efficient), average session duration (shorter = faster completion), skill cache hit rate (higher = less re-loading of the same skills).

**Duration**: 2-4 weeks, or until statistical significance is reached.

### Ablation Studies

To understand which feature categories are driving performance, we train seven model variants and compare their NDCG@10 on the test set:

1. All features (full model)
2. Similarity features only
3. Metadata features only
4. Historical features only
5. Contextual features only
6. Similarity + Historical (best expected subset)
7. Random baseline (no features — random ranking)

Our expectation is that all features together performs best, followed by the Similarity + Historical combination, which captures the two most informative signals: does the skill sound relevant, and has it been useful on similar tasks before?

### Error Analysis

After training, we collect 50 false positive examples (the model predicted high relevance but the skill was not invoked) and 50 false negative examples (the model ranked the skill low but it was actually invoked). For each group, we inspect which features drove the prediction and whether the error reflects a genuine model mistake or an artifact of our label construction. This analysis drives the next round of feature engineering.

---

## Implementation Strategy

### Phase 1: Data Collection and Preprocessing (Weeks 1–2)

The first task is parsing the existing trajectory logs from `~/.openclaw/trajectories/` and extracting structured training examples. For each session, we find the first user message (the task prompt), read the session metadata event for the skill list and invocation labels, and check the final-status event to know whether the session succeeded. We then emit one training row per (session, skill) pair with the prompt text, skill name, skill description, source, and all other metadata, plus the binary label.

Simultaneously, we scan all sessions to compute the historical statistics for each skill: total appearances across sessions, total invocations, success-rate-weighted invocations, co-occurrence counts with all other skills, and the timestamp of the most recent invocation. These statistics are serialized to a JSON file that is loaded into memory at inference time.

Finally, we pre-compute embeddings for all skill descriptions using the existing OpenClaw embedding infrastructure and serialize them to disk. This is a one-time offline cost — at inference time, we only need to embed the incoming prompt and then compute dot products against the cached skill embeddings.

**Deliverables**: a CSV training dataset (50k–1.5M rows), a skill statistics JSON file, and a serialized skill embedding cache.

### Phase 2: Feature Engineering (Week 3)

With training data in hand, we implement the four feature extractors — one per category — and use them to transform the raw training CSV into a numeric feature matrix. Each row in the matrix represents one (task, skill) pair and contains approximately 30–40 numerical values.

Missing values are handled conservatively: skills with no embedding get a zero vector (neutral similarity), and skills with no invocation history get the global average rate (a neutral prior). For logistic regression, features are normalized to zero mean and unit variance. For gradient boosting, no normalization is needed.

**Deliverables**: numeric feature matrices for train, validation, and test sets.

### Phase 3: Model Training (Week 4)

We first train the logistic regression baseline to establish a floor on performance and sanity-check the feature set. If logistic regression cannot beat random ranking by a meaningful margin, the features need revisiting.

We then train the LightGBM ranker. The dataset is structured so that examples from the same session are grouped together — the ranking loss operates within groups, comparing skills against each other for the same task, not across tasks. We tune the key hyperparameters — number of trees, learning rate, number of leaves per tree, and feature/row sampling rates — using the validation set with NDCG@10 as the objective. Early stopping prevents overfitting: training halts when validation NDCG@10 stops improving for 20 consecutive rounds.

After training, we compute SHAP values on the validation set to understand which features are driving predictions. SHAP (SHapley Additive exPlanations) decomposes each prediction into the contribution of each feature, enabling both global feature importance analysis and per-prediction explanation. This is critical for debugging and for deciding where to invest in additional features.

**Deliverables**: trained model file (~1–5 MB), feature importance visualization, SHAP summary plot.

### Phase 4: Evaluation (Week 5)

We evaluate the trained model on the held-out test set. For each session in the test set, we rank all available skills by their predicted scores and compute NDCG@10, P@10, R@150, and MRR. We also benchmark inference latency by running the model on 1,000 prediction requests (after a warm-up period) and recording the median and 99th-percentile latency.

We collect false positives and false negatives for error analysis as described above. If any systematic failure modes appear — for example, if the model consistently under-ranks skills whose names include uncommon technical terms — those findings drive targeted feature additions.

The go/no-go decision for integration is based on whether NDCG@10 ≥ 0.7 and median inference latency < 1ms.

**Deliverables**: evaluation report with all metrics, error analysis, go/no-go recommendation.

### Phase 5: Integration into OpenClaw (Weeks 6–7)

The integration point is the skill filtering function in `src/agents/skills/workspace.ts` (around line 101–122). Currently, after eligibility filtering, this function sorts skills alphabetically. We add an optional ML ranking step that replaces the alphabetical sort when the feature is enabled.

The integration adds two new source files. The first (`src/agents/skills/ml-ranking.ts`) handles model loading at startup and implements the ranking function: it extracts features for each (prompt, skill) pair, runs the LightGBM model, and returns skills sorted by predicted score. The second (`src/agents/skills/ml-features.ts`) implements the four feature extractors, loading cached embeddings and statistics from disk.

The feature is controlled by a new configuration key in `src/config/types.openclaw.ts`: `skills.mlRanking.enabled`. When this key is absent or false, the system falls back to alphabetical sorting exactly as before. The model, embedding cache, and statistics file paths are all configurable, with sensible defaults under `~/.openclaw/skills/`.

Startup initialization loads the model file and the embedding/statistics caches into memory once. At routing time, all lookups are in-memory — the only potentially expensive operation is embedding the user's prompt, which can be cached across turns in the same conversation.

**Deliverables**: two new TypeScript modules, updated config type definitions, unit tests, integration tests.

### Phase 6: A/B Testing (Weeks 8–10)

The A/B test is implemented as a feature flag that randomly assigns sessions to control or treatment at the start of each session. The assignment is logged as a trajectory event so it can be recovered during analysis. After 2–4 weeks, we read trajectory logs for both groups, compute session success rates, and run the two-proportion z-test. If the improvement is statistically significant and in the positive direction, we roll out ML ranking to all users. If not, we investigate whether the offline metrics were misleading and decide whether to iterate or revert.

**Deliverables**: A/B test results report, go/no-go rollout decision.

---

## Expected Outcomes & Contributions

### Quantitative Outcomes

**Online (A/B test)**:

- Task success rate: +2–5% absolute improvement (from ~80% baseline to 82–85%)
- Average skills invoked per session: −10–20% (the model needs fewer searches when the right skills appear first)
- Wasted prompt budget: −15–25% reduction in space consumed by never-invoked skills

**Offline**:

- Precision@10: 0.6–0.8
- Recall@150: ≥ 0.95
- NDCG@10: 0.7–0.85
- Inference latency: < 1ms median, < 5ms at 99th percentile

### Qualitative Outcomes

**Better user experience**: Users whose relevant skills had bad alphabetical luck — names starting with letters later in the alphabet — will see those skills reliably surfaced. Tasks that currently fail due to dropped skills will succeed. The connection between "what I asked for" and "which skills the agent knows about" will be made reliable rather than accidental.

**Interpretability**: Because we use gradient boosted trees with SHAP analysis, we can explain any individual routing decision. When a skill is ranked low, we can say exactly why — was it low semantic similarity? Low historical usage? Missing requirements? This makes the system debuggable in a way that deep learning approaches are not.

**Adaptability**: The model is retrained on fresh trajectory logs periodically. As user workflows evolve and new skills are added, the routing model updates to reflect the new patterns. A user who starts doing a lot of Kubernetes work will gradually see Kubernetes-related skills promoted without any manual reconfiguration.

### Research Contributions

**Novel problem formulation**: To our knowledge, this is the first work to formalize skill routing for LLM agents as a Learning to Rank problem with hard budget constraints. Prior work on tool selection in agent systems uses LLM-based selection (expensive) or fine-tuning (inapplicable to closed models). Our approach is neither.

**Practical ML system**: Most academic ML papers present research prototypes. This proposal targets a production-ready system with real latency requirements, no GPU dependency, no manual labeling, and a fallback path for when the model is absent or broken.

**Feature engineering insights**: By running ablation studies across all four feature categories, we will produce concrete findings about what signals are most predictive of skill relevance in LLM agent systems. These findings are generalizable beyond OpenClaw.

**Open-source dataset**: We intend to release an anonymized version of the trajectory logs as a public benchmark for future skill routing research.

**Generalizable approach**: The same architecture — ranked retrieval from a catalog using LTR trained on agent execution logs — is applicable to any agent system that uses skills, tools, plugins, or other retrievable instruction sets. AutoGPT, LangChain, and similar systems face the identical problem.

---

## Challenges & Limitations

### Label Noise

Our labels come from the model's own behavior — if the model read a skill, we label it as relevant. But the model is not a perfect oracle. It may read a skill out of exploratory curiosity, even if that skill does not help the task. It may also fail to read a relevant skill because it was ranked too low in the current alphabetical ordering and truncated before the model could see it.

The first type of noise (false positives in labels) is mitigated by restricting training to successful sessions — if a session completed successfully, the skills it invoked were probably genuinely useful. The second type of noise (false negatives in labels) is a more fundamental limitation of training on alphabetically-biased data. It is mitigated by using data from sessions where the skill catalog was small enough that truncation was unlikely, and by collecting new training data after the alphabetical bias is corrected.

### Class Imbalance

Approximately 97% of (task, skill) pairs are negative — the model invokes only a handful of skills out of the 50–150 available in each session. This severe imbalance means a model that always predicts "not relevant" would be 97% accurate, which is useless.

The primary mitigation is using a ranking loss (LambdaRank) rather than a binary classification loss. Ranking losses are inherently relative — they care about the order of skills within a session, not their absolute probabilities — so class imbalance matters less than it does for binary classification. For the logistic regression baseline, we use class weighting to counteract the imbalance.

### Cold Start for New Skills

When a new skill is installed, it has zero invocations and thus zero historical signal. All historical features (`global_invocation_rate`, `success_rate`, etc.) are undefined.

The mitigation is two-part. First, the similarity and metadata features work immediately for any skill, regardless of history — the model can still rank a new skill highly if its description matches the prompt. Second, we assign neutral priors for missing historical values: a new skill gets the global average invocation rate and a 0.5 success rate. This means it neither looks uniquely attractive nor uniquely unattractive from a historical perspective, letting the similarity features drive its ranking.

For active cold-start mitigation, we can implement occasional exploration: a small fraction of sessions (say, 5–10%) randomly promote a low-history skill to the top of the ranking to gather invocation data. This is the epsilon-greedy strategy from reinforcement learning applied to the skill selection problem.

### Concept Drift

Users' task patterns change over time. A developer who starts a new project using a different stack will find that the model's learned associations no longer match their current needs. A skill routing model trained six months ago on Python-heavy sessions may rank Python skills too highly for a user who has since switched to TypeScript.

The mitigation is periodic retraining — weekly or monthly — on fresh trajectory logs. We also weight recent sessions more heavily in training by scaling their contribution to the loss function by a time-decay factor. Finally, monitoring the model's held-out performance over time allows us to detect drift and trigger early retraining when performance degrades.

### Embedding Latency

Computing an embedding for the user's prompt requires either an API call (20–50ms over the network) or a local model inference (5–10ms on CPU). Either adds latency relative to the current zero-cost alphabetical sort.

The primary mitigation is caching: if the same prompt has been seen recently (common in iterative workflows where users refine the same request), we reuse the cached embedding. For cron-triggered tasks whose prompts are known in advance, we can precompute embeddings asynchronously. For novel prompts, we can compute the embedding in parallel with other initialization work so it does not add to the critical path. With these optimizations, the total routing overhead should stay well under 50ms — imperceptible in the context of a multi-second agent session.

As a fallback, if embedding is unavailable or too slow, the system can run with only the non-embedding features (metadata, historical, contextual), degrading gracefully to a weaker but still meaningful ranking.

### Feature Distribution Shift

The training data reflects the distribution of tasks and channels present in historical sessions. If the deployed system serves a different distribution — for example, a new Telegram integration brings in a different style of user prompt — the model may perform poorly on those new inputs.

The mitigation is stratified data collection: ensure training data spans all channels, agents, and task types rather than being dominated by the most common case. If a specific channel has a sufficiently different prompt distribution, training a channel-specific model may be warranted.

### Alphabetical Bias in Training Data

There is a subtle circularity in the training data: because the current system uses alphabetical ordering, skills that happen to have early-alphabet names are shown to the model more often and therefore have more invocation opportunities. A skill named `aaa-testing` accumulates more invocations than `zzz-testing` even if they are identical in content, simply because `aaa-testing` appears earlier in the catalog and is therefore seen first on every prompt.

A naive model trained on this data will learn that skills with short names or early-alphabet names are more relevant — a spurious correlation driven by the training system's bias rather than genuine task relevance.

The mitigation is to explicitly exclude any feature that correlates with alphabetical position (particularly the skill name as a string feature), to validate the model on sessions where we know the full skill catalog was small enough that no truncation occurred (eliminating the visibility bias), and to run a sanity check: does the model correctly rank `zzz-git` higher than `aaa-unrelated` for a git-related task? If not, alphabetical bias has leaked into the model.

### Model Staleness

A model trained on the current skill inventory may make poor predictions when skills are added, removed, or renamed. A skill the model has never seen gets no invocation history, and the model has no learned associations for its description.

Mitigation is automatic: skills the model has not seen fall back to content-based features (similarity and metadata), which work for any skill regardless of history. Scheduled retraining picks up the new skills as they accumulate invocations. Version tracking in trajectory logs allows us to associate each session with the model version that was active, enabling retrospective analysis when model updates cause performance changes.

---

## Future Work

### Deep Learning Extensions

The classical ML approach proposed here provides an excellent, practical first version. A natural extension is a neural ranking model — specifically, a BERT-based cross-encoder that jointly processes the task prompt and a skill description together, allowing the model to capture fine-grained interactions between them that the shallow TF-IDF and embedding similarity features cannot express.

The practical challenge is latency: BERT requires a separate forward pass for each (prompt, skill) pair, which means 150 forward passes per routing decision — easily 100–200ms in total even with GPU acceleration, and much slower on CPU. The practical resolution is a two-stage approach: use the fast LightGBM ranker to narrow the field to the top 20–30 candidates, then apply the BERT re-ranker only to those. Total latency would be approximately 8–10ms, still well within acceptable bounds.

### Multi-Task Learning

The current proposal focuses exclusively on skill ranking. But OpenClaw also needs to select which tools to make available, and there is likely signal sharing between the two tasks — a session that invokes the `github-pr` skill probably also benefits from the `gh` tool being available. A multi-task model that jointly ranks skills and tools would benefit from shared representations and could improve both tasks relative to training them independently.

### Active Learning

The current proposal uses passive learning — the model learns from whatever session data happens to accumulate. Active learning would have the system deliberately choose which skills to explore in order to gather the most informative labels. The epsilon-greedy strategy mentioned under cold start is the simplest version: with 5–10% probability, rank a randomly chosen low-history skill higher than the model otherwise would, observe whether the model invokes it, and use the outcome as a training signal. More sophisticated approaches include Thompson sampling (sample a score from the posterior distribution rather than always picking the MAP estimate) and contextual bandits (learn a policy that trades off exploration and exploitation as a function of context).

### Personalization

The current proposal trains a single global model. A natural extension is per-user personalization — learning not just that `github-pr` is generally useful for GitHub tasks, but that _this user_ specifically tends to use `github-pr` early in their workflow while another user uses `code-review` first. The simplest approach is a user embedding: a low-dimensional vector per user that is concatenated to the feature vector and learned jointly during training. A more sophisticated approach is a hierarchical model with shared global parameters and user-specific offsets. Either approach requires enough per-user session data to estimate user-specific behavior — a new user (cold start again) falls back to the global model.

### Skill Recommendation

The current proposal selects from existing skills. A complementary direction is identifying skills that _should exist but don't_. By analyzing sessions that fail or that invoke a long sequence of tools without using any skill, we can identify recurring task patterns that have no skill coverage. An LLM can then draft a candidate SKILL.md for the most common such patterns, which developers can review and publish. Over time, this closes the skill gap: the routing model helps users exploit existing skills, and the recommendation system helps developers create skills that are currently missing.

### Explainability Interface

The current proposal uses SHAP values for developer-facing explainability. A user-facing version would allow the agent to explain its skill selection choices directly in conversation: "I loaded the `github-pr` skill because your prompt mentioned creating a pull request and this skill has been helpful in 87% of similar sessions." This could also surface as a configuration interface: "Skills you might want to add based on your recent session patterns."

### Cross-Agent Transfer Learning

The skill routing problem is not specific to OpenClaw. AutoGPT, LangChain, and other agent frameworks face the identical challenge: a large catalog of tools or instructions, budget constraints on how many can be included in each prompt, and the need to select the most relevant ones. A model trained on OpenClaw trajectory data could be fine-tuned for other agent systems with minimal additional data — especially if the similarity and metadata features generalize across skill/tool schema differences. This is the standard transfer learning setup, and it would dramatically reduce the data requirements for deploying skill routing in new agent systems.

### Skill Clustering

The current proposal treats skills as a flat unordered set. A hierarchical taxonomy — grouping skills into topics like Git, testing, deployment, code review, documentation — would enable topic-aware routing: classify the task into a topic first, then rank within the relevant topic cluster. This two-stage approach trades some precision for a significant reduction in the ranking problem's complexity. It also enables a user to say "I'm working on a deployment task" and have the system immediately narrow to the deployment skill cluster.

---

## References & Code Locations

### Key Files in the OpenClaw Codebase

| Component                         | File Path                                |
| --------------------------------- | ---------------------------------------- |
| Skill loading & filtering         | `src/agents/skills/workspace.ts`         |
| Current filtering function        | `src/agents/skills/workspace.ts:101-122` |
| Budget enforcement logic          | `src/agents/skills/workspace.ts:676-717` |
| Trajectory export & parsing       | `src/trajectory/export.ts`               |
| Skill invocation detection        | `src/trajectory/export.ts:613-662`       |
| Trajectory event types            | `src/trajectory/types.ts`                |
| Skill frontmatter parsing         | `src/agents/skills/frontmatter.ts`       |
| Skill type definitions            | `src/agents/skills/types.ts`             |
| Existing embedding infrastructure | `src/memory-host-sdk/host/embeddings.ts` |
| Config type definitions           | `src/config/types.openclaw.ts`           |

### New Files to Be Created

| Component                 | Proposed File Path                 |
| ------------------------- | ---------------------------------- |
| ML ranking module         | `src/agents/skills/ml-ranking.ts`  |
| Feature extraction        | `src/agents/skills/ml-features.ts` |
| Model & cache loading     | `src/agents/skills/ml-cache.ts`    |
| Training pipeline         | `scripts/train-skill-ranker.py`    |
| Statistics precomputation | `scripts/compute-skill-stats.py`   |

### External Libraries Required

| Library               | Purpose                                                  | Notes                                                |
| --------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| LightGBM              | Gradient boosting — primary model                        | Needs Node.js bindings (`lightgbm-node`) for runtime |
| XGBoost               | Alternative gradient boosting — training                 | Python only (training is offline)                    |
| scikit-learn          | Evaluation metrics, logistic regression baseline         | Python only                                          |
| SHAP                  | Feature importance and prediction explanation            | Python only                                          |
| sentence-transformers | Local embedding model (optional, reduces API dependency) | Python/Node                                          |
| OpenAI Python SDK     | Cloud embedding API — pre-computation                    | Already available via existing infrastructure        |

### Academic References

1. **Learning to Rank**: Burges, C. (2010). "From RankNet to LambdaRank to LambdaMART: An Overview." Microsoft Research Technical Report. — Liu, T. Y. (2009). "Learning to Rank for Information Retrieval." Foundations and Trends in Information Retrieval.

2. **Gradient Boosting**: Chen, T., & Guestrin, C. (2016). "XGBoost: A Scalable Tree Boosting System." KDD 2016. — Ke, G., et al. (2017). "LightGBM: A Highly Efficient Gradient Boosting Decision Tree." NeurIPS 2017.

3. **Information Retrieval**: Robertson, S., & Zaragoza, H. (2009). "The Probabilistic Relevance Framework: BM25 and Beyond." Foundations and Trends in Information Retrieval.

4. **Cold Start**: Schein, A. I., et al. (2002). "Methods and Metrics for Cold-Start Recommendations." SIGIR 2002.

5. **Class Imbalance**: Chawla, N. V., et al. (2002). "SMOTE: Synthetic Minority Over-sampling Technique." JAIR.

6. **Behavioral Cloning**: Pomerleau, D. A. (1988). "ALVINN: An Autonomous Land Vehicle in a Neural Network." NeurIPS 1988.

7. **Inverse RL**: Abbeel, P., & Ng, A. Y. (2004). "Apprenticeship Learning via Inverse Reinforcement Learning." ICML 2004.

8. **Toolformer**: Schick, T., et al. (2023). "Toolformer: Language Models Can Teach Themselves to Use Tools." ArXiv.

9. **ReAct**: Yao, S., et al. (2022). "ReAct: Synergizing Reasoning and Acting in Language Models." ICLR 2023.

---

**End of Proposal**
