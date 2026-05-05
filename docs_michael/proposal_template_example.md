Title: MemTier - Persistent, Learning Memory for AI Agents

Column 1 Upper Part:
Background and problem statement: 
Current agents (e.g., OpenClaw): 
- Flat memory (MEMORY.md, capped, no structure) 
- No prioritization or attribution 
- Forget over time → performance drops (~14% in 72h)

Column 1 Middle Part:
Diagram:
Episodic Store → Retrieval Engine → Attribution Loop → Semantic Layer → PPO Optimization
👉 Memory becomes a learning system, not storage
 Agents improve by reinforcing useful experiences and suppressing harmful ones
+0.128 accuracy / +0.269 F1 with semantic memory 
0 → 0.38 accuracy vs no-memory baseline 
High recall (>0.7) outperforming standard RAG

Column 1 Bottom Part:
Related Works:
1. <paper name> + link
2. <paper name> + link
3. <paper name> + link

Column 2 Upper Part:
Key Challenges:
• Noisy data vs useful facts
• Token budget constraints
• Multi-agent isolation & sharing
• Real-time learning without labels

Column 2 Middle and Bottom Part:
Key Technologies/Proposal: 
1. Episodic Memory (Structured, Append-Only)
   • JSONL per session (agent-private)
   • Stores raw experiences with metadata
   • No overwrites → full traceability 

2. Intelligent Retrieval (5-Signal Scoring)
   • Combines: BM25 + time decay + learned weights + tier boost
   • Two-stage retrieval → high precision under token budget

3. Attribution Loop (Self-Learning)
   • Memory entries updated from real outcomes
   • Success ↑ weight, failure ↓ weight
   • No labels needed → outcome = training signal 

4. Semantic Consolidation
   • Background process extracts structured facts
   • Shared knowledge across agents (project-level)
   • Clean, high-signal memory replaces noisy logs 

6. Online Optimization (PPO)
   • Retrieval weights learned continuously
   • Hot-reloaded → no downtime
