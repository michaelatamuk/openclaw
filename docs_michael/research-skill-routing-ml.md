# Research Proposal: Learning to Route: Efficient Skill Selection for Autonomous Agents using Classical Machine Learning

**Author**: Technical Research Proposal
**Date**: 2026-04-28
**Domain**: Agent Systems, Classical Machine Learning, Information Retrieval
**Code Analysis**: OpenClaw codebase (`openclaw/openclaw`)

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

Modern autonomous agent systems (e.g., OpenClaw, AutoGPT, LangChain) rely on **skills** — specialized instruction sets that guide large language models (LLMs) in domain-specific tasks. However, current skill selection mechanisms are **naive**: skills are sorted alphabetically and truncated by hard budget limits, with no consideration for task relevance. This leads to:

- **Budget waste** on irrelevant skills
- **Dropped relevant skills** when limits are exceeded
- **Zero adaptation** to user patterns or task characteristics

We propose **Learning to Route (L2R)**, a classical machine learning approach to skill selection that:

1. Predicts skill relevance from task characteristics using lightweight, interpretable models
2. Ranks skills by predicted utility before budget truncation
3. Learns from trajectory logs without requiring explicit labels
4. Operates with <1ms latency overhead for real-time agent systems

Our approach uses **gradient boosted decision trees** (XGBoost/LightGBM) trained on features extracted from:

- Skill descriptions and metadata (TF-IDF, categorical features)
- User prompt embeddings (reusing existing infrastructure)
- Historical invocation patterns (from trajectory logs)
- Environmental context (OS, available tools, channel type)

**Key contribution**: A practical, production-ready skill routing system that improves agent efficiency without requiring labeled datasets or deep learning infrastructure.

---

## Problem Statement

### Current Skill Selection in OpenClaw

**Location**: `src/agents/skills/workspace.ts:101-122`, `workspace.ts:676-717`

OpenClaw's skill selection follows this algorithm:

```typescript
// 1. Load all skills from directories (workspace, bundled, managed, personal, extra)
const allSkills = loadSkillsFromDirs(dirs);

// 2. Filter by allow/deny lists, OS/binary requirements
const eligible = allSkills.filter(shouldIncludeSkill);

// 3. Sort alphabetically (!)
eligible.sort((a, b) => a.name.localeCompare(b.name, "en"));

// 4. Hard truncation by budget
const byCount = eligible.slice(0, 150); // DEFAULT_MAX_SKILLS_IN_PROMPT
const byChars = fitToBudget(byCount, 18_000); // DEFAULT_MAX_SKILLS_PROMPT_CHARS

// 5. Send to model as catalog (model decides which to read)
return formatSkillsForPrompt(byChars);
```

### Critical Flaws

1. **Alphabetical ordering is task-agnostic**:
   - Skill named `"aaa-irrelevant"` ranks higher than `"zzz-critical"`
   - No consideration of prompt content, user intent, or historical patterns

2. **Budget limits drop relevant skills**:
   - Default: Max 150 skills, max 18,000 characters
   - If you have 500 skills, the last 350 are **completely invisible** to the model
   - Dropped skills cannot be invoked, regardless of relevance

3. **No learning or adaptation**:
   - System doesn't learn which skills are useful for which tasks
   - Repeated similar tasks reload identical skill lists
   - User-specific preferences ignored

4. **Inefficient budget utilization**:
   - Irrelevant skills waste prompt space
   - Relevant skills may be truncated
   - No prioritization mechanism

### Concrete Example

**User prompt**: `"Create a GitHub pull request with my recent commits"`

**Relevant skills** (should be prioritized):

- `github-pr` (rank: ?, alphabetical position unknown)
- `git-commit` (rank: ?, alphabetical position unknown)
- `code-review` (rank: ?, alphabetical position unknown)

**Irrelevant skills** (may rank higher alphabetically):

- `android-build` (rank: 1 if starts with 'a')
- `audio-processing` (rank: 2 if starts with 'a')
- `aws-deploy` (rank: 3 if starts with 'a')

If the user has 200 skills and the top 150 alphabetically include many irrelevant ones, `github-pr` might be dropped simply due to its name starting with 'g'.

**Impact**:

- Model cannot invoke `github-pr` skill (doesn't know it exists)
- User task fails or requires manual intervention
- Wasted prompt budget on `android-build`, `audio-processing`, etc.

---

## Research Objectives

### Primary Objective

**Develop a classical ML-based skill routing system that maximizes task success rate while respecting hard budget constraints.**

### Specific Research Questions

1. **RQ1: Predictability**
   Can we predict which skills will be invoked by the model for a given task using features available at routing time?
   - **Hypothesis**: Skill invocation is predictable from (task, skill, context) features
   - **Null hypothesis**: Invocation is random/unpredictable

2. **RQ2: Feature Importance**
   Which features are most predictive of skill relevance?
   - Textual similarity (prompt ↔ skill description)?
   - Historical usage patterns?
   - Environmental context (OS, tools)?
   - Skill metadata (requirements, source)?

3. **RQ3: Model Selection**
   What classical ML model achieves the best precision/recall tradeoff for skill ranking?
   - Logistic regression (baseline)
   - Gradient boosted trees (XGBoost, LightGBM)
   - Random forests
   - k-Nearest Neighbors (task similarity)

4. **RQ4: Generalization**
   Does the model generalize across:
   - Different users?
   - Different agent configurations?
   - New/unseen skills (cold start)?
   - Different task domains?

5. **RQ5: Efficiency**
   Can inference run in <1ms per routing decision to avoid adding latency to agent systems?
   - **Requirement**: Real-time agents need fast skill loading
   - **Constraint**: Inference must be faster than alphabetical sorting

6. **RQ6: Online Improvement**
   Can the model improve over time via incremental learning from new trajectory logs?
   - **Goal**: Adapt to user-specific patterns
   - **Constraint**: No manual labeling required

---

## Background & Related Work

### Agent Systems & Tool/Skill Selection

**OpenClaw Architecture** (this codebase):

- Skills = Markdown instruction sets (SKILL.md)
- Model reads skills via `Read` tool when relevant
- Current selection: alphabetical + hard budget limits
- **Gap**: No learned relevance

**AutoGPT, LangChain, ReAct**:

- Focus on tool selection (APIs, functions) rather than instruction skills
- Often use LLM-based selection ("which tool should I use?")
- **Limitation**: Expensive (extra LLM call per decision)

**Toolformer (Meta AI, 2023)**:

- Teaches LLMs when to use external tools via fine-tuning
- Requires large-scale training data
- **Not applicable**: We can't fine-tune closed models (Claude, GPT-4)

### Classical ML for Information Retrieval

**Learned Sparse Retrieval**:

- BM25 (Robertson & Zaragoza, 2009): TF-IDF-based ranking
- **Applicable**: Skill descriptions as "documents", prompts as "queries"

**Learning to Rank (LTR)**:

- NDCG optimization (Burges et al., 2005)
- Gradient boosted trees for ranking (XGBoost, LightGBM)
- **Applicable**: Rank skills by relevance for each task

**Feature Engineering for Text**:

- TF-IDF, n-grams, word embeddings
- **Applicable**: Extract features from skill descriptions + prompts

**Cold Start in Recommender Systems**:

- Content-based filtering (Pazzani & Billsus, 2007)
- Hybrid approaches (collaborative + content)
- **Applicable**: New skills with no usage history

### Trajectory Learning for Agents

**Behavioral Cloning**:

- Learn from expert demonstrations (Pomerleau, 1988)
- **Applicable**: Treat model's skill selections as "expert" labels

**Inverse Reinforcement Learning**:

- Infer reward function from observed behavior (Abbeel & Ng, 2004)
- **Overkill**: We just need skill ranking, not full policy learning

---

## Proposed Approach

### Overview

We frame skill selection as a **Learning to Rank (LTR)** problem:

- **Input**: User task/prompt `T`, set of eligible skills `S = {s₁, s₂, ..., sₙ}`
- **Output**: Ranked list of skills `[s'₁, s'₂, ..., s'ₙ]` where `relevance(s'ᵢ, T) ≥ relevance(s'ⱼ, T)` for `i < j`
- **Constraint**: Select top-k skills (k ≤ 150) that fit character budget (≤ 18,000 chars)

### Training Data Construction

**Location**: Trajectory logs at `~/.openclaw/trajectories/*.jsonl`

**Schema** (from `src/trajectory/types.ts`, `export.ts:613-662`):

```typescript
{
  traceId: string,
  sessionId: string,
  metadata: {
    skills: {
      entries: [
        {
          id: string,
          name: string,
          description: string,
          filePath: string,
          source: "workspace" | "bundled" | ...,
          invoked: boolean,  // ← LABEL!
          invocationDetectedBy?: "tool-call-file-path"
        }
      ]
    }
  },
  events: [
    {
      type: "user.message",
      data: { message: { content: string } }  // ← TASK PROMPT
    },
    {
      type: "tool.call",
      data: { name: string, arguments: unknown }  // Skill invocation
    }
  ]
}
```

**Label Extraction**:

1. Parse trajectory JSONL files
2. For each session:
   - Extract user prompt: First `user.message` event content
   - Extract available skills: `metadata.skills.entries[]`
   - Extract invoked skills: Skills with `invoked: true`
3. Generate training examples:
   - **Positive**: `(task, skill)` pairs where `skill.invoked == true`
   - **Negative**: `(task, skill)` pairs where `skill.invoked == false` (skill was available but not used)

**Dataset Statistics** (estimated):

- **Sessions**: 1,000-10,000 (depending on user activity)
- **Skills per session**: ~50-150 (post-filtering)
- **Invoked skills per session**: ~1-5 (sparse!)
- **Training examples**: ~50,000-1,500,000 (task, skill) pairs
- **Class imbalance**: ~97% negative (skills not invoked)

### Assumptions & Limitations

**Assumption 1**: Skills invoked by the model were relevant

- **Justification**: Model (Claude Sonnet 4.6) is intelligent; unlikely to read irrelevant skills
- **Risk**: Model may make mistakes or explore unnecessarily
- **Mitigation**: Filter by session success (`finalStatus == "success"`)

**Assumption 2**: Skills not invoked were irrelevant

- **Justification**: If truly relevant, model would have read them
- **Risk**: Relevant skills might be missing due to alphabetical ranking
- **Mitigation**: Use later sessions (after fixing alphabetical issue) for training

**Assumption 3**: No distribution shift

- **Justification**: User task patterns are relatively stable
- **Risk**: New task types may not match historical data
- **Mitigation**: Incremental learning, fallback to alphabetical for OOD tasks

---

## Data Sources & Availability

### 1. Trajectory Logs

**Location**: `~/.openclaw/trajectories/*.jsonl`
**Format**: JSONL (newline-delimited JSON)
**Retention**: Indefinite (until manual cleanup)
**Privacy**: Local-only (not uploaded)

**Key Fields**:

```json
{
  "traceSchema": "openclaw-trajectory",
  "schemaVersion": 1,
  "traceId": "uuid",
  "source": "export",
  "type": "session.metadata",
  "ts": "2026-04-28T10:30:00.000Z",
  "data": {
    "metadata": {
      "harness": { "os": "darwin", "version": "..." },
      "model": { "provider": "anthropic", "name": "claude-sonnet-4.6" },
      "skills": {
        "snapshotVersion": 2,
        "skillFilter": null,
        "entries": [
          {
            "id": "commit",
            "name": "commit",
            "description": "Create git commits following repository conventions",
            "filePath": "~/.agents/skills/commit/SKILL.md",
            "source": "managed",
            "invoked": true,
            "invocationDetectedBy": "tool-call-file-path"
          },
          {
            "id": "test-runner",
            "name": "test-runner",
            "description": "Run automated tests and fix failures",
            "filePath": "~/.agents/skills/test-runner/SKILL.md",
            "source": "bundled",
            "invoked": false
          }
        ]
      }
    }
  }
}
```

**User Message Events**:

```json
{
  "type": "user.message",
  "ts": "2026-04-28T10:30:05.000Z",
  "seq": 1,
  "data": {
    "message": {
      "role": "user",
      "content": "Create a git commit with my recent changes"
    }
  }
}
```

**Tool Call Events**:

```json
{
  "type": "tool.call",
  "ts": "2026-04-28T10:30:12.000Z",
  "seq": 5,
  "data": {
    "toolCallId": "call_abc123",
    "name": "Read",
    "arguments": {
      "file_path": "~/.agents/skills/commit/SKILL.md"
    }
  }
}
```

### 2. Skill Metadata

**Location**: Skill SKILL.md files + frontmatter
**Format**: Markdown with YAML frontmatter
**Schema** (from `src/agents/skills/types.ts`):

```yaml
---
always: false
primaryEnv: GITHUB_TOKEN
requires:
  bins: ["git", "gh"]
  env: ["GITHUB_TOKEN"]
os: ["darwin", "linux", "win32"]
exposure:
  includeInRuntimeRegistry: true
  includeInAvailableSkillsPrompt: true
  userInvocable: true
emoji: 🔧
homepage: https://github.com/openclaw/openclaw
---
# Skill Content (Markdown)
Instructions for the model...
```

**Extractable Fields**:

- `name`: Skill identifier
- `description`: Natural language description
- `requires.bins`: Required binaries (e.g., `["git", "npm"]`)
- `requires.env`: Required environment variables
- `os`: Supported operating systems
- `source`: `"workspace"` | `"bundled"` | `"managed"` | `"personal"` | `"extra"`
- `always`: Boolean flag for always-include skills

### 3. Embedding Infrastructure (Reusable)

**Location**: `src/memory-host-sdk/host/embeddings.ts`, `extensions/memory-*/`
**Purpose**: Currently used for memory/RAG, not skill selection
**Providers**: OpenAI, Voyage, Google, Mistral, Amazon Bedrock

**Available Models**:

- `text-embedding-3-small` (OpenAI, 1536 dims)
- `text-embedding-3-large` (OpenAI, 3072 dims)
- `voyage-2` (Voyage AI, 1024 dims)
- `embedding-001` (Google, 768 dims)

**Reuse Potential**:

- Generate embeddings for skill descriptions offline
- Generate embeddings for user prompts at routing time
- Compute cosine similarity as feature

**Performance**:

- Embedding generation: ~10-50ms per text (API call)
- **Optimization**: Pre-compute skill embeddings offline, cache in memory
- Cosine similarity: <1ms for 1536-dim vectors

### 4. Session Transcripts

**Location**: `~/.openclaw/sessions/*.jsonl`
**Format**: JSONL with `SessionEntry` records
**Purpose**: Full conversation logs

**Extractable Context**:

- Conversation history (multi-turn tasks)
- Previous tool calls in session
- Previously invoked skills (co-occurrence patterns)

### 5. Configuration Files

**Location**: `~/.openclaw/config.json`, workspace `.openclaw/config.json`
**Schema** (from `src/config/types.openclaw.ts`):

```typescript
{
  skills: {
    allow?: string[],
    deny?: string[],
    limits: {
      maxSkillsInPrompt: number,
      maxSkillsPromptChars: number
    }
  },
  agents: {
    [agentId]: {
      skillFilter?: string[],
      runtime?: string
    }
  }
}
```

**Extractable Features**:

- User-specific skill preferences (allow/deny lists)
- Agent configuration (runtime, tools)

---

## Feature Engineering

### Feature Categories

#### 1. **Skill-Task Textual Similarity**

**Hypothesis**: Skills with descriptions semantically similar to the user prompt are more likely to be invoked.

**Features**:

| Feature               | Description                                                                | Computation                                                       | Expected Range |
| --------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------- | -------------- |
| `cos_sim_desc_prompt` | Cosine similarity between skill description embedding and prompt embedding | `dot(embed(skill.desc), embed(prompt)) / (norm * norm)`           | [0, 1]         |
| `tfidf_overlap`       | TF-IDF weighted overlap between skill description and prompt               | `sum(tfidf(token) for token in intersection)`                     | [0, ∞)         |
| `jaccard_bigrams`     | Jaccard similarity of character bigrams                                    | `len(intersection) / len(union)`                                  | [0, 1]         |
| `keyword_match_count` | Number of skill name tokens appearing in prompt                            | `len([token for token in skill.name.split() if token in prompt])` | [0, N]         |
| `levenshtein_dist`    | Edit distance between skill name and first prompt word                     | `editdist(skill.name, prompt.split()[0])`                         | [0, ∞)         |

**Implementation**:

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

# Offline: Pre-compute skill embeddings
skill_embeddings = {}
for skill in all_skills:
    skill_embeddings[skill.name] = embed_text(skill.description)

# Online: At routing time
def extract_similarity_features(task_prompt, skill):
    prompt_embedding = embed_text(task_prompt)
    skill_embedding = skill_embeddings[skill.name]

    features = {
        'cos_sim_desc_prompt': cosine_similarity(
            [prompt_embedding], [skill_embedding]
        )[0][0],
        'tfidf_overlap': compute_tfidf_overlap(
            task_prompt, skill.description
        ),
        'jaccard_bigrams': jaccard_similarity(
            bigrams(task_prompt), bigrams(skill.description)
        ),
        'keyword_match_count': sum(
            1 for token in skill.name.split()
            if token.lower() in task_prompt.lower()
        )
    }
    return features
```

#### 2. **Skill Metadata Features**

**Hypothesis**: Skill properties (source, requirements, always-flag) correlate with invocation likelihood.

**Features**:

| Feature                   | Type    | Description                                          | Values |
| ------------------------- | ------- | ---------------------------------------------------- | ------ |
| `source_priority`         | Ordinal | Skill source priority (workspace=6, bundled=2, etc.) | [1, 6] |
| `has_always_flag`         | Binary  | Whether skill has `always: true` in frontmatter      | {0, 1} |
| `requires_bins_satisfied` | Binary  | All required binaries available in environment       | {0, 1} |
| `requires_env_satisfied`  | Binary  | All required env vars present                        | {0, 1} |
| `os_compatible`           | Binary  | Skill supports current OS                            | {0, 1} |
| `num_required_bins`       | Count   | Number of required binaries                          | [0, N] |
| `num_required_env`        | Count   | Number of required env vars                          | [0, N] |
| `is_user_invocable`       | Binary  | Skill can be invoked by user (not just model)        | {0, 1} |
| `skill_name_length`       | Count   | Character length of skill name                       | [1, ∞) |

**Implementation**:

```python
def extract_metadata_features(skill, env_context):
    features = {
        'source_priority': SOURCE_PRIORITIES[skill.source],
        'has_always_flag': int(skill.metadata.get('always', False)),
        'requires_bins_satisfied': int(all(
            has_binary(b) for b in skill.requires.bins or []
        )),
        'requires_env_satisfied': int(all(
            has_env_var(e) for e in skill.requires.env or []
        )),
        'os_compatible': int(
            env_context.os in (skill.metadata.get('os', []))
        ),
        'num_required_bins': len(skill.requires.bins or []),
        'num_required_env': len(skill.requires.env or []),
        'is_user_invocable': int(skill.invocation.userInvocable),
        'skill_name_length': len(skill.name)
    }
    return features

SOURCE_PRIORITIES = {
    'workspace': 6,
    'project': 5,
    'personal': 4,
    'managed': 3,
    'bundled': 2,
    'extra': 1
}
```

#### 3. **Historical Usage Features**

**Hypothesis**: Frequently invoked skills and co-occurring skills are more likely to be relevant.

**Features**:

| Feature                  | Type  | Description                                                 | Computation                        |
| ------------------------ | ----- | ----------------------------------------------------------- | ---------------------------------- | ------------ |
| `global_invocation_rate` | Float | Fraction of sessions where skill was invoked                | `invocations / total_sessions`     |
| `user_invocation_rate`   | Float | User-specific invocation rate                               | `user_invocations / user_sessions` |
| `recency_days`           | Float | Days since last invocation                                  | `(now - last_invocation_ts).days`  |
| `co_occurrence_score`    | Float | Probability skill is invoked given another skill in session | `P(skill                           | prev_skill)` |
| `session_position_avg`   | Float | Average position in session when invoked (1st, 2nd, ...)    | `mean(positions)`                  |
| `success_rate`           | Float | Fraction of sessions with skill invocation that succeeded   | `successes / invocations`          |

**Precomputation** (from trajectory logs):

```python
from collections import defaultdict
import json

# Parse all trajectory files
skill_stats = defaultdict(lambda: {
    'invocation_count': 0,
    'total_sessions': 0,
    'last_invocation_ts': None,
    'success_count': 0,
    'co_occurrences': defaultdict(int)
})

for trajectory_file in glob("~/.openclaw/trajectories/*.jsonl"):
    for line in open(trajectory_file):
        event = json.loads(line)

        if event['type'] == 'session.metadata':
            session_id = event['sessionId']
            skills = event['data']['metadata']['skills']['entries']
            invoked_skills = [s['name'] for s in skills if s['invoked']]

            for skill in skills:
                skill_name = skill['name']
                skill_stats[skill_name]['total_sessions'] += 1

                if skill['invoked']:
                    skill_stats[skill_name]['invocation_count'] += 1
                    skill_stats[skill_name]['last_invocation_ts'] = event['ts']

                    # Co-occurrence tracking
                    for other in invoked_skills:
                        if other != skill_name:
                            skill_stats[skill_name]['co_occurrences'][other] += 1

        if event['type'] == 'session.final':
            if event['data']['finalStatus'] == 'success':
                for skill_name in invoked_skills:
                    skill_stats[skill_name]['success_count'] += 1

# Compute rates
for skill_name, stats in skill_stats.items():
    stats['global_invocation_rate'] = (
        stats['invocation_count'] / max(stats['total_sessions'], 1)
    )
    stats['success_rate'] = (
        stats['success_count'] / max(stats['invocation_count'], 1)
    )
```

**Online Feature Extraction**:

```python
def extract_historical_features(skill_name, session_context, skill_stats):
    stats = skill_stats.get(skill_name, {})

    # Check if any previously invoked skill in this session
    prev_skills = session_context.get('previously_invoked_skills', [])
    co_occurrence_score = max(
        [stats.get('co_occurrences', {}).get(prev, 0) for prev in prev_skills],
        default=0
    )

    features = {
        'global_invocation_rate': stats.get('global_invocation_rate', 0.0),
        'recency_days': (
            (datetime.now() - stats['last_invocation_ts']).days
            if stats.get('last_invocation_ts') else 999
        ),
        'co_occurrence_score': co_occurrence_score,
        'success_rate': stats.get('success_rate', 0.5)
    }
    return features
```

#### 4. **Contextual Features**

**Hypothesis**: Task context (channel, agent, time) affects skill relevance.

**Features**:

| Feature             | Type        | Description                                    | Values          |
| ------------------- | ----------- | ---------------------------------------------- | --------------- |
| `channel_type`      | Categorical | Message channel (cli, telegram, discord, etc.) | One-hot encoded |
| `agent_id`          | Categorical | Agent identifier                               | One-hot encoded |
| `is_cron_trigger`   | Binary      | Task triggered by cron (not user)              | {0, 1}          |
| `hour_of_day`       | Ordinal     | Hour when task started (0-23)                  | [0, 23]         |
| `day_of_week`       | Ordinal     | Day of week (0=Monday, 6=Sunday)               | [0, 6]          |
| `conversation_turn` | Count       | Turn number in multi-turn conversation         | [1, ∞)          |
| `prompt_length`     | Count       | Character count of user prompt                 | [1, ∞)          |
| `has_code_block`    | Binary      | Prompt contains code block (`...`)             | {0, 1}          |

**Implementation**:

````python
def extract_contextual_features(task, session_context):
    features = {
        'is_cron_trigger': int(task.trigger == 'cron'),
        'hour_of_day': task.timestamp.hour,
        'day_of_week': task.timestamp.weekday(),
        'conversation_turn': len(session_context.get('history', [])),
        'prompt_length': len(task.prompt),
        'has_code_block': int('```' in task.prompt)
    }

    # One-hot encoding for categorical features
    features.update(one_hot_encode('channel_type', task.channel))
    features.update(one_hot_encode('agent_id', task.agent_id))

    return features
````

### Feature Vector Dimension

**Total Features**: ~50-100 dimensions (depending on categorical cardinality)

| Category   | Features    | Dimensions |
| ---------- | ----------- | ---------- |
| Similarity | 5           | 5          |
| Metadata   | 9           | 9          |
| Historical | 6           | 6          |
| Contextual | 6 + one-hot | 6 + 10-20  |
| **Total**  |             | **30-40**  |

**Sparsity**: Moderate (one-hot encodings are sparse)

---

## Model Architecture

### Candidate Models

#### 1. **Logistic Regression** (Baseline)

**Advantages**:

- Fast training and inference (<1ms)
- Interpretable coefficients
- No hyperparameter tuning needed

**Disadvantages**:

- Assumes linear decision boundary
- May underfit complex interactions

**Implementation**:

```python
from sklearn.linear_model import LogisticRegression

model = LogisticRegression(
    penalty='l2',
    C=1.0,
    solver='lbfgs',
    max_iter=1000,
    class_weight='balanced'  # Handle class imbalance
)
model.fit(X_train, y_train)
```

#### 2. **Gradient Boosted Trees** (Primary Proposal)

**Model**: XGBoost or LightGBM

**Advantages**:

- Handles non-linear interactions
- Robust to feature scaling
- Feature importance via SHAP values
- State-of-the-art for tabular data
- Fast inference (<1ms for 100 trees)

**Disadvantages**:

- Hyperparameter tuning required
- Less interpretable than logistic regression
- Risk of overfitting on small datasets

**Implementation (XGBoost)**:

```python
import xgboost as xgb

model = xgb.XGBRanker(
    objective='rank:pairwise',
    eval_metric='ndcg@10',
    n_estimators=100,
    max_depth=6,
    learning_rate=0.1,
    subsample=0.8,
    colsample_bytree=0.8,
    random_state=42
)

# Create ranking groups (skills per task)
group_sizes = [len(skills_for_task) for task in tasks]
model.fit(X_train, y_train, group=group_sizes)
```

**Implementation (LightGBM)**:

```python
import lightgbm as lgb

model = lgb.LGBMRanker(
    objective='lambdarank',
    metric='ndcg',
    n_estimators=100,
    learning_rate=0.1,
    num_leaves=31,
    min_child_samples=20,
    random_state=42
)

model.fit(
    X_train, y_train,
    group=group_sizes,
    eval_set=[(X_val, y_val)],
    eval_group=[val_group_sizes]
)
```

#### 3. **Random Forest** (Alternative)

**Advantages**:

- Robust to overfitting
- Handles non-linear interactions
- Feature importance

**Disadvantages**:

- Slower inference than GBT
- Less accurate than GBT for ranking

**Implementation**:

```python
from sklearn.ensemble import RandomForestClassifier

model = RandomForestClassifier(
    n_estimators=100,
    max_depth=10,
    min_samples_split=20,
    class_weight='balanced',
    random_state=42
)
model.fit(X_train, y_train)
```

#### 4. **k-Nearest Neighbors** (Similarity-based)

**Approach**: Find k most similar historical tasks, rank skills by their invocation frequency in those tasks

**Advantages**:

- No training required (instance-based)
- Handles cold start (new skills) via task similarity

**Disadvantages**:

- Slow inference (need to search all historical tasks)
- Requires good task embedding

**Implementation**:

```python
from sklearn.neighbors import NearestNeighbors

# Index historical tasks by embedding
task_index = NearestNeighbors(n_neighbors=10, metric='cosine')
task_index.fit(historical_task_embeddings)

# At routing time
def rank_skills_knn(task_prompt):
    task_emb = embed_text(task_prompt)
    distances, indices = task_index.kneighbors([task_emb])

    # Aggregate skill invocations from k-nearest tasks
    skill_scores = defaultdict(float)
    for idx in indices[0]:
        for skill in historical_tasks[idx]['invoked_skills']:
            skill_scores[skill] += 1.0 / (distances[0][idx] + 1e-6)

    return sorted(skill_scores.items(), key=lambda x: x[1], reverse=True)
```

### Recommended Approach

**Primary**: **LightGBM Ranker**
**Rationale**:

- Best accuracy/speed tradeoff for ranking tasks
- Handles class imbalance well
- Fast inference (<1ms with 100 trees, 40 features)
- Feature importance via SHAP
- Production-ready (used in Bing, Yandex search ranking)

**Baseline**: **Logistic Regression**
**Purpose**: Sanity check, interpretability analysis

**Fallback**: **BM25 (TF-IDF-based)**
**Purpose**: No training required, works out-of-the-box

---

## Evaluation Methodology

### Offline Evaluation

**Dataset Split**:

- **Training**: 70% of sessions (chronologically early)
- **Validation**: 15% of sessions (middle)
- **Test**: 15% of sessions (chronologically late)
- **Rationale**: Time-based split simulates deployment (train on past, test on future)

**Metrics**:

#### 1. **Ranking Metrics**

**Precision@K**: Fraction of top-K predictions that were actually invoked

```
P@K = (|{predicted top-K} ∩ {actually invoked}|) / K
```

**Recall@K**: Fraction of actually invoked skills in top-K predictions

```
R@K = (|{predicted top-K} ∩ {actually invoked}|) / |{actually invoked}|
```

**Mean Reciprocal Rank (MRR)**: Average inverse rank of first relevant skill

```
MRR = (1/|Q|) * Σ (1 / rank_i)
```

**Normalized Discounted Cumulative Gain (NDCG@K)**:

```
NDCG@K = DCG@K / IDCG@K
DCG@K = Σ(i=1 to K) (2^rel_i - 1) / log2(i + 1)
```

**Target Metrics**:

- **Precision@10** ≥ 0.6 (60% of top-10 are relevant)
- **Recall@150** ≥ 0.95 (95% of relevant skills in top-150)
- **NDCG@10** ≥ 0.7

#### 2. **Classification Metrics** (Binary: Invoked vs. Not Invoked)

**ROC-AUC**: Area under ROC curve (threshold-independent)
**Target**: ≥ 0.85

**PR-AUC**: Area under Precision-Recall curve (handles class imbalance)
**Target**: ≥ 0.4 (given 97% negative class)

**F1 Score**: Harmonic mean of precision and recall
**Target**: ≥ 0.5

#### 3. **Efficiency Metrics**

**Inference Latency**: Time to rank all skills for one task
**Target**: <1ms (median), <5ms (p99)

**Memory Footprint**: Model size + feature cache
**Target**: <100 MB

### Online Evaluation (A/B Test)

**Setup**:

- **Control**: Alphabetical skill ranking (current system)
- **Treatment**: ML-based skill ranking

**Assignment**: Per-user (50/50 split)

**Primary Metric**: **Task Success Rate**

- **Definition**: Fraction of sessions with `finalStatus == "success"`
- **Hypothesis**: Treatment > Control
- **Statistical Test**: Two-proportion z-test (α = 0.05)

**Secondary Metrics**:

- **Average skills invoked per session**: Lower is better (efficiency)
- **Average session duration**: Shorter is better (faster completion)
- **Skill cache hit rate**: Higher is better (fewer re-loads)
- **User satisfaction** (if survey available)

**Sample Size Calculation**:

- **Baseline success rate**: 80% (assumed)
- **Minimum detectable effect**: 2% (absolute improvement to 82%)
- **Power**: 80%
- **Significance**: α = 0.05
- **Required sample**: ~10,000 sessions per group

**Duration**: 2-4 weeks (until statistical significance or max duration)

### Ablation Studies

**Question**: Which feature categories contribute most to performance?

**Method**: Train models with feature subsets:

1. **All features** (full model)
2. **Similarity only** (textual features)
3. **Metadata only** (skill properties)
4. **Historical only** (usage patterns)
5. **Contextual only** (task context)
6. **Similarity + Historical** (no metadata, no context)
7. **Random baseline** (random skill ranking)

**Metric**: NDCG@10 on test set

**Expected Ranking**:

1. All features (best)
2. Similarity + Historical
3. Similarity only
4. Historical only
5. Metadata only
6. Contextual only
7. Random baseline (worst)

### Error Analysis

**False Positives** (predicted invoked, actually not):

- Manual inspection: Were they truly irrelevant?
- Or were they relevant but model chose alternative skill?

**False Negatives** (predicted not invoked, actually invoked):

- Why was the skill ranked low?
- Missing features? Poor embedding quality?

**Case Studies**:

- Collect 50 FP and 50 FN examples
- Analyze feature distributions
- Identify systematic failure modes

---

## Implementation Strategy

### Phase 1: Data Collection & Preprocessing (Week 1-2)

**Tasks**:

1. **Export existing trajectory logs**:

   ```bash
   find ~/.openclaw/trajectories -name "*.jsonl" -type f
   ```

2. **Parse JSONL files**:

   ```python
   import json
   from pathlib import Path

   trajectory_dir = Path("~/.openclaw/trajectories").expanduser()

   for file in trajectory_dir.glob("*.jsonl"):
       for line in open(file):
           event = json.loads(line)
           # Process event
   ```

3. **Extract training examples**:
   - Iterate through sessions
   - For each session:
     - Extract user prompt (first `user.message`)
     - Extract available skills (`metadata.skills.entries`)
     - Label each skill: `invoked == true` → 1, else → 0
   - Save to CSV: `task_id, skill_id, label, prompt_text, skill_name, skill_desc, ...`

4. **Compute historical statistics**:
   - Aggregate skill invocation counts, success rates, co-occurrences
   - Save to JSON: `skill_stats.json`

5. **Pre-compute skill embeddings**:

   ```python
   # Reuse OpenClaw's embedding infrastructure
   from memory_host_sdk.host.embeddings import create_embedding_provider

   provider = create_embedding_provider(
       provider_type="openai",
       model="text-embedding-3-small"
   )

   skill_embeddings = {}
   for skill in all_skills:
       skill_embeddings[skill.name] = provider.embed(skill.description)

   # Cache to disk
   import pickle
   with open("skill_embeddings.pkl", "wb") as f:
       pickle.dump(skill_embeddings, f)
   ```

**Deliverable**: `training_data.csv` (50k-1.5M rows), `skill_stats.json`, `skill_embeddings.pkl`

### Phase 2: Feature Engineering (Week 3)

**Tasks**:

1. **Implement feature extractors**:
   - `extract_similarity_features()`
   - `extract_metadata_features()`
   - `extract_historical_features()`
   - `extract_contextual_features()`

2. **Generate feature matrix**:

   ```python
   import pandas as pd

   df = pd.read_csv("training_data.csv")

   features = []
   for idx, row in df.iterrows():
       feature_vec = {
           **extract_similarity_features(row['prompt_text'], row['skill_name']),
           **extract_metadata_features(skills[row['skill_id']], env_context),
           **extract_historical_features(row['skill_name'], skill_stats),
           **extract_contextual_features(row, session_context)
       }
       features.append(feature_vec)

   X = pd.DataFrame(features)
   y = df['label'].values
   ```

3. **Handle missing values**:
   - Fill missing embeddings with zero vector
   - Impute missing historical stats with global mean

4. **Feature normalization** (if needed for logistic regression):

   ```python
   from sklearn.preprocessing import StandardScaler

   scaler = StandardScaler()
   X_scaled = scaler.fit_transform(X)
   ```

**Deliverable**: `X_train.npy`, `y_train.npy`, `X_val.npy`, `y_val.npy`, `X_test.npy`, `y_test.npy`

### Phase 3: Model Training (Week 4)

**Tasks**:

1. **Train baseline (Logistic Regression)**:

   ```python
   from sklearn.linear_model import LogisticRegression

   lr_model = LogisticRegression(class_weight='balanced')
   lr_model.fit(X_train, y_train)

   # Evaluate
   from sklearn.metrics import roc_auc_score, precision_recall_curve
   y_pred_proba = lr_model.predict_proba(X_val)[:, 1]
   print(f"ROC-AUC: {roc_auc_score(y_val, y_pred_proba)}")
   ```

2. **Train LightGBM ranker**:

   ```python
   import lightgbm as lgb

   # Prepare ranking format
   train_data = lgb.Dataset(
       X_train, label=y_train,
       group=train_group_sizes  # Skills per task
   )
   val_data = lgb.Dataset(
       X_val, label=y_val,
       group=val_group_sizes,
       reference=train_data
   )

   params = {
       'objective': 'lambdarank',
       'metric': 'ndcg',
       'ndcg_eval_at': [10, 20, 50],
       'num_leaves': 31,
       'learning_rate': 0.05,
       'feature_fraction': 0.8
   }

   model = lgb.train(
       params,
       train_data,
       num_boost_round=200,
       valid_sets=[val_data],
       early_stopping_rounds=20
   )

   # Save model
   model.save_model('skill_ranker.txt')
   ```

3. **Hyperparameter tuning**:

   ```python
   from sklearn.model_selection import GridSearchCV

   param_grid = {
       'num_leaves': [15, 31, 63],
       'learning_rate': [0.01, 0.05, 0.1],
       'n_estimators': [100, 200, 300]
   }

   # Use validation set for early stopping
   # Select best params by NDCG@10
   ```

4. **Feature importance analysis**:

   ```python
   import shap

   explainer = shap.TreeExplainer(model)
   shap_values = explainer.shap_values(X_val)

   shap.summary_plot(shap_values, X_val, feature_names=feature_names)
   ```

**Deliverable**: `skill_ranker.txt` (trained model), `feature_importance.png`, `shap_summary.png`

### Phase 4: Evaluation (Week 5)

**Tasks**:

1. **Offline evaluation on test set**:

   ```python
   # Predict on test set
   y_pred = model.predict(X_test)

   # Compute ranking metrics
   from sklearn.metrics import ndcg_score

   # Group predictions by task
   for task_id in test_task_ids:
       mask = (test_data['task_id'] == task_id)
       y_true_task = y_test[mask]
       y_pred_task = y_pred[mask]

       ndcg_10 = ndcg_score([y_true_task], [y_pred_task], k=10)
       print(f"Task {task_id}: NDCG@10 = {ndcg_10}")
   ```

2. **Precision@K, Recall@K**:

   ```python
   def precision_at_k(y_true, y_pred, k):
       top_k_indices = np.argsort(y_pred)[-k:]
       return y_true[top_k_indices].sum() / k

   def recall_at_k(y_true, y_pred, k):
       top_k_indices = np.argsort(y_pred)[-k:]
       return y_true[top_k_indices].sum() / y_true.sum()
   ```

3. **Inference latency benchmark**:

   ```python
   import time

   # Warm-up
   for _ in range(100):
       model.predict(X_test[:1])

   # Benchmark
   latencies = []
   for i in range(1000):
       start = time.perf_counter()
       model.predict(X_test[i:i+1])
       latencies.append(time.perf_counter() - start)

   print(f"Median latency: {np.median(latencies) * 1000:.2f} ms")
   print(f"P99 latency: {np.percentile(latencies, 99) * 1000:.2f} ms")
   ```

4. **Error analysis**:
   - Collect false positives and false negatives
   - Analyze feature distributions
   - Identify failure modes

**Deliverable**: `evaluation_report.md`, `offline_metrics.json`

### Phase 5: Integration into OpenClaw (Week 6-7)

**Integration Points**:

**Location**: `src/agents/skills/workspace.ts:101-122`

**Current Code**:

```typescript
function filterSkillEntries(
  entries: SkillEntry[],
  config?: OpenClawConfig,
  skillFilter?: string[],
  eligibility?: SkillEligibilityContext,
): SkillEntry[] {
  let filtered = entries;

  // ... apply filters ...

  // CURRENT: Alphabetical sorting
  filtered.sort((a, b) => a.skill.name.localeCompare(b.skill.name, "en"));

  return filtered;
}
```

**Modified Code**:

```typescript
import { rankSkillsML } from "./ml-ranking.js";

function filterSkillEntries(
  entries: SkillEntry[],
  config?: OpenClawConfig,
  skillFilter?: string[],
  eligibility?: SkillEligibilityContext,
  taskPrompt?: string, // NEW PARAMETER
): SkillEntry[] {
  let filtered = entries;

  // ... apply existing filters ...

  // NEW: ML-based ranking (if enabled and prompt provided)
  if (config?.skills?.mlRanking?.enabled && taskPrompt) {
    filtered = rankSkillsML(filtered, taskPrompt, config, eligibility);
  } else {
    // Fallback to alphabetical
    filtered.sort((a, b) => a.skill.name.localeCompare(b.skill.name, "en"));
  }

  return filtered;
}
```

**New Module**: `src/agents/skills/ml-ranking.ts`

```typescript
import * as lightgbm from "lightgbm-node"; // Node.js bindings
import { computeFeatures } from "./ml-features.js";
import { loadSkillEmbeddings, loadSkillStats } from "./ml-cache.js";

let mlModel: lightgbm.Booster | null = null;
let skillEmbeddings: Map<string, Float32Array> | null = null;
let skillStats: Map<string, SkillStats> | null = null;

export function initializeMLRanking(config: OpenClawConfig): void {
  const modelPath =
    config.skills?.mlRanking?.modelPath ?? path.join(CONFIG_DIR, "skills", "skill_ranker.txt");

  if (!fs.existsSync(modelPath)) {
    logger.warn("ML ranking model not found, falling back to alphabetical");
    return;
  }

  mlModel = lightgbm.Booster.loadFromFile(modelPath);
  skillEmbeddings = loadSkillEmbeddings();
  skillStats = loadSkillStats();

  logger.info("ML skill ranking initialized");
}

export function rankSkillsML(
  skills: SkillEntry[],
  taskPrompt: string,
  config: OpenClawConfig,
  eligibility: SkillEligibilityContext,
): SkillEntry[] {
  if (!mlModel || !skillEmbeddings || !skillStats) {
    // Fallback to alphabetical
    return skills.sort((a, b) => a.skill.name.localeCompare(b.skill.name, "en"));
  }

  // Extract features for each (task, skill) pair
  const features = skills.map((skill) =>
    computeFeatures(taskPrompt, skill, eligibility, skillStats),
  );

  // Predict relevance scores
  const scores = mlModel.predict(features);

  // Rank by score (descending)
  const rankedSkills = skills
    .map((skill, idx) => ({ skill, score: scores[idx] }))
    .sort((a, b) => b.score - a.score)
    .map((item) => item.skill);

  return rankedSkills;
}
```

**Feature Extraction**: `src/agents/skills/ml-features.ts`

```typescript
import { computeEmbedding } from "../../memory-host-sdk/host/embeddings.js";
import { cosineSimilarity, tfidfOverlap } from "./ml-utils.js";

export function computeFeatures(
  taskPrompt: string,
  skill: SkillEntry,
  eligibility: SkillEligibilityContext,
  skillStats: Map<string, SkillStats>,
): Float32Array {
  const features: number[] = [];

  // 1. Similarity features
  const promptEmbedding = computeEmbedding(taskPrompt);
  const skillEmbedding = skillEmbeddings.get(skill.skill.name);

  if (skillEmbedding) {
    features.push(cosineSimilarity(promptEmbedding, skillEmbedding));
  } else {
    features.push(0.0); // Missing embedding
  }

  features.push(tfidfOverlap(taskPrompt, skill.skill.description));
  // ... more similarity features ...

  // 2. Metadata features
  features.push(SOURCE_PRIORITIES[skill.skill.source] / 6.0); // Normalize
  features.push(skill.metadata?.always ? 1.0 : 0.0);
  // ... more metadata features ...

  // 3. Historical features
  const stats = skillStats.get(skill.skill.name);
  features.push(stats?.globalInvocationRate ?? 0.0);
  features.push(stats?.successRate ?? 0.5);
  // ... more historical features ...

  // 4. Contextual features
  features.push(new Date().getHours() / 24.0); // Normalize hour
  // ... more contextual features ...

  return new Float32Array(features);
}
```

**Configuration Schema**: `src/config/types.openclaw.ts`

```typescript
export type OpenClawConfig = {
  skills?: {
    mlRanking?: {
      enabled: boolean;
      modelPath?: string;
      embeddingsPath?: string;
      statsPath?: string;
      fallbackToAlphabetical?: boolean;
    };
    // ... existing config ...
  };
  // ... rest of config ...
};
```

**Initialization**: `src/agents/skills/workspace.ts` (module init)

```typescript
import { initializeMLRanking } from "./ml-ranking.js";

// At module load time
export function initializeSkillSubsystem(config: OpenClawConfig): void {
  if (config.skills?.mlRanking?.enabled) {
    initializeMLRanking(config);
  }
}
```

### Phase 6: A/B Testing (Week 8-10)

**Setup**:

1. **Feature flag**:

   ```typescript
   const useMLRanking = config.skills?.mlRanking?.enabled && Math.random() < 0.5; // 50/50 split
   ```

2. **Logging**:

   ```typescript
   // Log which ranking method was used
   trajectoryLogger.log({
     type: "skill.ranking.method",
     data: {
       method: useMLRanking ? "ml" : "alphabetical",
       sessionId: session.id,
     },
   });
   ```

3. **Analysis** (after 2-4 weeks):

   ```python
   import pandas as pd
   from scipy.stats import ttest_ind

   # Load trajectory logs
   df = pd.read_json("trajectories.jsonl", lines=True)

   # Split by ranking method
   ml_group = df[df['ranking_method'] == 'ml']
   alpha_group = df[df['ranking_method'] == 'alphabetical']

   # Compute success rates
   ml_success_rate = (ml_group['finalStatus'] == 'success').mean()
   alpha_success_rate = (alpha_group['finalStatus'] == 'success').mean()

   # Statistical test
   t_stat, p_value = ttest_ind(
       ml_group['finalStatus'] == 'success',
       alpha_group['finalStatus'] == 'success'
   )

   print(f"ML success rate: {ml_success_rate:.2%}")
   print(f"Alphabetical success rate: {alpha_success_rate:.2%}")
   print(f"p-value: {p_value:.4f}")
   ```

**Deliverable**: `ab_test_results.md`, decision to roll out or roll back

---

## Expected Outcomes & Contributions

### Quantitative Outcomes

**Primary Metrics** (based on A/B test):

- **Task success rate**: +2-5% absolute improvement (80% → 82-85%)
- **Average skills invoked per session**: -10-20% reduction (more efficient)
- **Skill budget utilization**: -15-25% reduction in wasted prompt space

**Offline Metrics**:

- **Precision@10**: 0.6-0.8 (60-80% of top-10 skills are relevant)
- **Recall@150**: 0.95+ (95%+ of relevant skills in top-150)
- **NDCG@10**: 0.7-0.85
- **Inference latency**: <1ms median, <5ms p99

### Qualitative Outcomes

1. **Better user experience**:
   - Relevant skills appear first, regardless of name
   - Fewer "skill not found" errors due to dropped relevant skills

2. **Interpretability**:
   - Feature importance analysis shows which signals matter
   - SHAP values explain individual predictions
   - Users can debug why a skill was/wasn't selected

3. **Adaptability**:
   - System learns user-specific patterns over time
   - Incremental learning adapts to new skills and task types

### Research Contributions

1. **Novel problem formulation**:
   - First work (to our knowledge) on skill routing for LLM agents
   - Formalizes the task as Learning to Rank with budget constraints

2. **Practical ML system**:
   - Production-ready implementation (not just a research prototype)
   - <1ms latency (real-time constraint)
   - No labeled data required (learns from trajectory logs)

3. **Feature engineering insights**:
   - What signals predict skill relevance?
   - Relative importance of similarity vs. historical patterns

4. **Open-source dataset**:
   - Release anonymized trajectory logs for reproducibility
   - Benchmark dataset for future skill routing research

5. **Generalizable approach**:
   - Applicable to other agent systems (AutoGPT, LangChain, etc.)
   - Transferable to tool selection, plugin selection, API routing

---

## Challenges & Limitations

### 1. **Label Noise**

**Problem**: Invoked skills are assumed to be relevant, but:

- Model may read irrelevant skills out of curiosity
- Model may fail to read relevant skills due to poor current ranking

**Mitigation**:

- Filter training data to successful sessions only (`finalStatus == "success"`)
- Use multiple skill invocations as stronger signal (if invoked in 5+ sessions, likely relevant)
- Manual labeling of subset for validation

### 2. **Class Imbalance**

**Problem**: ~97% of (task, skill) pairs are negative (skill not invoked)

**Mitigation**:

- Use `class_weight='balanced'` in logistic regression
- Use ranking loss (lambdarank) instead of binary classification
- Oversample positive examples via SMOTE (Synthetic Minority Oversampling)

### 3. **Cold Start Problem**

**Problem**: New skills have no historical usage data

**Mitigation**:

- Rely on content-based features (similarity, metadata)
- Fallback to alphabetical for skills with zero history
- Active exploration: Occasionally promote low-history skills to gather data

### 4. **Concept Drift**

**Problem**: User task patterns change over time

**Mitigation**:

- Incremental learning: Retrain model monthly on new trajectory logs
- Time-based weighting: Give more weight to recent sessions
- Detect drift: Monitor model performance over time, trigger retraining if degrades

### 5. **Embedding Cost**

**Problem**: Computing prompt embeddings at every routing decision may add latency

**Mitigation**:

- **Async embedding**: Compute in parallel with other features
- **Caching**: Cache prompt embeddings for repeated identical prompts
- **Approximate embeddings**: Use faster local models (distilled BERT) instead of API calls
- **Precomputation**: If prompt is known ahead of time (cron tasks), precompute

**Benchmark**:

- OpenAI API: ~20-50ms per embedding (network latency)
- Local model (sentence-transformers): ~5-10ms per embedding (CPU)
- **Goal**: Keep total routing time <50ms

### 6. **Feature Distribution Shift**

**Problem**: Training data may not represent deployment distribution

**Example**:

- Training data from CLI usage
- Deployment includes Telegram/Discord (different prompt styles)

**Mitigation**:

- Stratified sampling: Ensure training data includes all channels, agents, task types
- Domain adaptation: Train separate models per channel if distribution differs significantly
- Robust features: Use features that generalize (TF-IDF, embeddings) rather than brittle patterns

### 7. **Model Staleness**

**Problem**: Model trained on old data may not reflect current skill inventory

**Mitigation**:

- **Scheduled retraining**: Retrain weekly/monthly on fresh trajectory logs
- **Fallback**: If skill not seen during training, use content-based features only
- **Version tracking**: Log model version in trajectories for debugging

### 8. **Overfitting to Alphabetical Bias**

**Problem**: Training data was generated under alphabetical ranking, so labels may reflect alphabetical bias

**Example**:

- Skill "aaa-test" was invoked frequently simply because it appeared first
- Model may learn "skills starting with 'a' are relevant" (spurious correlation)

**Mitigation**:

- Use later sessions (after fixing alphabetical issue) for training
- Exclude features that correlate with alphabetical position (skill name)
- Sanity check: Does model assign high scores to "zzz-git" for git tasks?

---

## Future Work

### 1. **Deep Learning Extensions**

**Current**: Classical ML (XGBoost, logistic regression)

**Future**: Neural ranking models

- **BERT-based ranker**: Fine-tune BERT on (prompt, skill description) pairs
- **Cross-encoders**: Jointly encode task + skill for relevance prediction
- **Neural embeddings**: Learn task-specific embeddings (not generic sentence embeddings)

**Challenges**:

- Inference latency (BERT is slow)
- Requires GPU for fast inference
- More data hungry (need 100k+ labeled pairs)

### 2. **Multi-Task Learning**

**Current**: Single task (skill ranking)

**Future**: Joint learning of:

- Skill ranking
- Tool ranking
- Argument prediction (which arguments to pass to tools)
- Invocation sequence (which skill to read first, second, ...)

**Benefits**:

- Shared representations across tasks
- Better generalization with limited data

### 3. **Active Learning**

**Current**: Passive learning from trajectory logs

**Future**: Active exploration

- **Epsilon-greedy**: Occasionally rank skills randomly to explore
- **Thompson sampling**: Sample skill rankings from posterior distribution
- **Contextual bandits**: Learn user-specific ranking policies

**Benefits**:

- Faster adaptation to new skills/users
- Better handling of cold start

### 4. **Personalization**

**Current**: Global model (same for all users)

**Future**: Per-user models

- **User embeddings**: Learn user-specific preferences
- **Hierarchical models**: Global model + user-specific offsets
- **Federated learning**: Train on user data without centralizing

**Benefits**:

- Adapt to individual coding styles, task patterns
- Better user experience

### 5. **Skill Recommendation**

**Current**: Rank existing skills

**Future**: Suggest new skills to create

- **Skill gap detection**: Identify tasks that fail due to missing skills
- **Skill template generation**: Use LLM to draft new SKILL.md for common task patterns
- **Community skill marketplace**: Share skills across users

### 6. **Explainability Interface**

**Current**: SHAP values for developers

**Future**: User-facing explanations

- **"Why was this skill selected?"**: Show top-3 reasons (similarity, past usage, etc.)
- **"Why was this skill NOT selected?"**: Debug missing skills
- **"Suggest skills I might need"**: Proactive recommendations

### 7. **Cross-Agent Transfer Learning**

**Current**: One agent system (OpenClaw)

**Future**: Transfer learning across agent systems

- Train on OpenClaw data, apply to AutoGPT
- Learn general skill→task patterns
- Meta-learning: Quickly adapt to new agent systems with few examples

### 8. **Skill Clustering**

**Current**: Flat skill list

**Future**: Hierarchical skill taxonomy

- **Cluster skills by topic**: Git skills, testing skills, deployment skills
- **Topic-aware routing**: "This is a git task" → prioritize git cluster
- **Dynamic cluster formation**: Discover clusters from co-occurrence patterns

---

## References & Code Locations

### Key Files in OpenClaw Codebase

| Component                  | File Path                                |
| -------------------------- | ---------------------------------------- |
| Skill loading              | `src/agents/skills/workspace.ts`         |
| Skill filtering            | `src/agents/skills/workspace.ts:101-122` |
| Budget enforcement         | `src/agents/skills/workspace.ts:676-717` |
| Trajectory export          | `src/trajectory/export.ts`               |
| Skill invocation detection | `src/trajectory/export.ts:613-662`       |
| Trajectory types           | `src/trajectory/types.ts`                |
| Skill frontmatter          | `src/agents/skills/frontmatter.ts`       |
| Skill types                | `src/agents/skills/types.ts`             |
| Embedding infrastructure   | `src/memory-host-sdk/host/embeddings.ts` |
| Config schema              | `src/config/types.openclaw.ts`           |

### External Libraries

| Library               | Purpose                               | Link                                               |
| --------------------- | ------------------------------------- | -------------------------------------------------- |
| LightGBM              | Gradient boosting (ranking)           | https://lightgbm.readthedocs.io/                   |
| XGBoost               | Gradient boosting (ranking)           | https://xgboost.readthedocs.io/                    |
| scikit-learn          | ML utilities (metrics, preprocessing) | https://scikit-learn.org/                          |
| SHAP                  | Feature importance / explainability   | https://shap.readthedocs.io/                       |
| sentence-transformers | Text embeddings (local)               | https://www.sbert.net/                             |
| OpenAI API            | Text embeddings (cloud)               | https://platform.openai.com/docs/guides/embeddings |

### Academic References

1. **Learning to Rank**:
   - Burges, C. (2010). "From RankNet to LambdaRank to LambdaMART: An Overview". Microsoft Research Technical Report.
   - Liu, T. Y. (2009). "Learning to Rank for Information Retrieval". Foundations and Trends in Information Retrieval.

2. **Gradient Boosting**:
   - Chen, T., & Guestrin, C. (2016). "XGBoost: A Scalable Tree Boosting System". KDD 2016.
   - Ke, G., et al. (2017). "LightGBM: A Highly Efficient Gradient Boosting Decision Tree". NIPS 2017.

3. **Information Retrieval**:
   - Robertson, S., & Zaragoza, H. (2009). "The Probabilistic Relevance Framework: BM25 and Beyond". Foundations and Trends in Information Retrieval.

4. **Cold Start in Recommender Systems**:
   - Schein, A. I., et al. (2002). "Methods and Metrics for Cold-Start Recommendations". SIGIR 2002.

5. **Class Imbalance**:
   - Chawla, N. V., et al. (2002). "SMOTE: Synthetic Minority Over-sampling Technique". JAIR.

### Related Agent Systems

- **AutoGPT**: https://github.com/Significant-Gravitas/AutoGPT
- **LangChain**: https://github.com/langchain-ai/langchain
- **ReAct**: Yao, S., et al. (2022). "ReAct: Synergizing Reasoning and Acting in Language Models". ICLR 2023.
- **Toolformer**: Schick, T., et al. (2023). "Toolformer: Language Models Can Teach Themselves to Use Tools". ArXiv.

---

## Appendix: Example Training Pipeline

```python
#!/usr/bin/env python3
"""
Training pipeline for OpenClaw skill routing model.

Usage:
  python train_skill_ranker.py --trajectories ~/.openclaw/trajectories --output skill_ranker.txt
"""

import argparse
import json
import pickle
from pathlib import Path
from collections import defaultdict
from datetime import datetime

import numpy as np
import pandas as pd
import lightgbm as lgb
from sklearn.model_selection import train_test_split
from sklearn.metrics import ndcg_score, roc_auc_score

# Feature extraction
from feature_extractors import (
    extract_similarity_features,
    extract_metadata_features,
    extract_historical_features,
    extract_contextual_features
)

def parse_trajectory_logs(trajectory_dir: Path) -> pd.DataFrame:
    """Parse trajectory JSONL files and extract training examples."""
    examples = []

    for trajectory_file in trajectory_dir.glob("*.jsonl"):
        session_events = []
        for line in open(trajectory_file):
            event = json.loads(line)
            session_events.append(event)

        # Group events by session
        sessions = defaultdict(list)
        for event in session_events:
            sessions[event['sessionId']].append(event)

        # Extract training examples from each session
        for session_id, events in sessions.items():
            # Find metadata event
            metadata_event = next(
                (e for e in events if e.get('type') == 'session.metadata'),
                None
            )
            if not metadata_event:
                continue

            # Find user message event
            user_message_event = next(
                (e for e in events if e.get('type') == 'user.message'),
                None
            )
            if not user_message_event:
                continue

            # Extract prompt
            prompt = user_message_event['data']['message']['content']

            # Extract skills
            skills = metadata_event['data']['metadata']['skills']['entries']

            # Extract final status
            final_event = next(
                (e for e in events if e.get('type') == 'session.final'),
                None
            )
            success = (
                final_event and
                final_event['data'].get('finalStatus') == 'success'
            )

            # Create training examples
            for skill in skills:
                examples.append({
                    'session_id': session_id,
                    'task_prompt': prompt,
                    'skill_name': skill['name'],
                    'skill_description': skill['description'],
                    'skill_source': skill['source'],
                    'skill_file_path': skill['filePath'],
                    'invoked': int(skill.get('invoked', False)),
                    'success': int(success)
                })

    return pd.DataFrame(examples)

def compute_historical_stats(df: pd.DataFrame) -> dict:
    """Compute historical skill statistics from training data."""
    skill_stats = defaultdict(lambda: {
        'invocation_count': 0,
        'total_sessions': 0,
        'success_count': 0
    })

    for _, row in df.iterrows():
        skill_name = row['skill_name']
        skill_stats[skill_name]['total_sessions'] += 1

        if row['invoked']:
            skill_stats[skill_name]['invocation_count'] += 1
            if row['success']:
                skill_stats[skill_name]['success_count'] += 1

    # Compute rates
    for skill_name, stats in skill_stats.items():
        stats['global_invocation_rate'] = (
            stats['invocation_count'] / max(stats['total_sessions'], 1)
        )
        stats['success_rate'] = (
            stats['success_count'] / max(stats['invocation_count'], 1)
        )

    return dict(skill_stats)

def extract_features(df: pd.DataFrame, skill_stats: dict) -> pd.DataFrame:
    """Extract features for each (task, skill) pair."""
    feature_rows = []

    for _, row in df.iterrows():
        features = {
            **extract_similarity_features(
                row['task_prompt'],
                row['skill_name'],
                row['skill_description']
            ),
            **extract_metadata_features(row),
            **extract_historical_features(row['skill_name'], skill_stats),
            **extract_contextual_features(row)
        }
        feature_rows.append(features)

    return pd.DataFrame(feature_rows)

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--trajectories', type=Path, required=True)
    parser.add_argument('--output', type=str, default='skill_ranker.txt')
    parser.add_argument('--stats-output', type=str, default='skill_stats.pkl')
    args = parser.parse_args()

    print("Parsing trajectory logs...")
    df = parse_trajectory_logs(args.trajectories)
    print(f"Loaded {len(df)} training examples from {df['session_id'].nunique()} sessions")

    # Compute historical statistics
    print("Computing historical statistics...")
    skill_stats = compute_historical_stats(df)

    # Extract features
    print("Extracting features...")
    X = extract_features(df, skill_stats)
    y = df['invoked'].values

    # Split by session (time-based)
    sessions = df['session_id'].unique()
    train_sessions, test_sessions = train_test_split(
        sessions, test_size=0.15, random_state=42
    )

    train_mask = df['session_id'].isin(train_sessions)
    test_mask = df['session_id'].isin(test_sessions)

    X_train, X_test = X[train_mask], X[test_mask]
    y_train, y_test = y[train_mask], y[test_mask]

    # Prepare ranking format
    train_groups = df[train_mask].groupby('session_id').size().values
    test_groups = df[test_mask].groupby('session_id').size().values

    print(f"Train: {len(X_train)} examples, {len(train_groups)} groups")
    print(f"Test: {len(X_test)} examples, {len(test_groups)} groups")

    # Train LightGBM ranker
    print("Training LightGBM ranker...")
    train_data = lgb.Dataset(X_train, label=y_train, group=train_groups)
    test_data = lgb.Dataset(X_test, label=y_test, group=test_groups, reference=train_data)

    params = {
        'objective': 'lambdarank',
        'metric': 'ndcg',
        'ndcg_eval_at': [10, 20, 50],
        'num_leaves': 31,
        'learning_rate': 0.05,
        'feature_fraction': 0.8,
        'bagging_fraction': 0.8,
        'bagging_freq': 5,
        'verbose': 1
    }

    model = lgb.train(
        params,
        train_data,
        num_boost_round=200,
        valid_sets=[test_data],
        valid_names=['test'],
        callbacks=[lgb.early_stopping(20), lgb.log_evaluation(10)]
    )

    # Evaluate
    print("\nEvaluating on test set...")
    y_pred = model.predict(X_test)

    # Compute NDCG per session
    ndcg_scores = []
    start_idx = 0
    for group_size in test_groups:
        end_idx = start_idx + group_size
        y_true_group = y_test[start_idx:end_idx]
        y_pred_group = y_pred[start_idx:end_idx]

        if y_true_group.sum() > 0:  # Only if there are relevant skills
            ndcg = ndcg_score([y_true_group], [y_pred_group], k=10)
            ndcg_scores.append(ndcg)

        start_idx = end_idx

    print(f"Mean NDCG@10: {np.mean(ndcg_scores):.4f}")
    print(f"Median NDCG@10: {np.median(ndcg_scores):.4f}")

    # ROC-AUC
    auc = roc_auc_score(y_test, y_pred)
    print(f"ROC-AUC: {auc:.4f}")

    # Save model
    print(f"\nSaving model to {args.output}...")
    model.save_model(args.output)

    # Save skill stats
    print(f"Saving skill stats to {args.stats_output}...")
    with open(args.stats_output, 'wb') as f:
        pickle.dump(skill_stats, f)

    print("Done!")

if __name__ == '__main__':
    main()
```

---

**End of Research Proposal**
