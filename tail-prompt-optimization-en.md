# Cache Tree and Tail Prompt Optimization

## 1. Cache Tree (Cache Tree)

Multiple turns share a common Context Cache prefix, forming a tree structure:

```
                    [system prompt + conversation history]    ← trunk (shared cache)
                   /                                        \
    [branch A: question A]    [branch B: question B]    [branch C: question C]   ← branches (new computation)
```

Core properties:
- **Shared trunk**: All branches reuse the parent's Context Cache, no redundant computation
- **Independent branches**: Each branch's new tokens are computed independently
- **Cache reuse**: When switching branches, the common prefix cache remains available (within TTL)

Typical scenario: IM platform thread mechanisms. Each thread is a branch sharing the main thread's prefix cache. Usually, base information is pre-filled into cache first (e.g., loading project code/docs in coding scenarios), then different branches execute different tasks — each branch inherits the cached project context and only needs to compute its own task tokens.

```
Main thread (trunk)
  ├── thread_A: fix bug      → inherits trunk cache + bug description
  ├── thread_B: add feature  → inherits trunk cache + requirement description
  └── thread_C: write tests  → inherits trunk cache + test scope
```

Each thread's cost ≈ task description tokens (project context uses cache).

---

## 2. Tail Prompt Optimization

*A temporary instruction injected at the end of a prompt, leveraging the Context Cache bypass branch mechanism to perform specific tasks.*

Tail prompts are a variant of cache trees — **Cache Vine**. One main trunk grows continuously, with leaves sprouting periodically; after leaves fall, the trunk continues growing.

```
[system prompt + conversation history] ─── trunk (accumulates continuously)
       │
       ├── [tail prompt A + question]  ← leaf (one turn, falls off)
       │
    trunk continues growing...
       │
       ├── [tail prompt B + question]  ← leaf (one turn, falls off)
       │
    trunk continues growing...
       │
       └── [tail prompt C + question]  ← leaf (one turn, falls off)
```

Differences from cache trees:

| Dimension | Cache Tree | Tail Prompt (Vine) |
|:--|:--|:--|
| Trunk | Shared trunk | Same, but trunk accumulates continuously |
| Branch | Persistent thread | Temporary leaf (one turn) |
| Lifecycle | Cross-turn, user-driven | Single turn, system-injected |
| Controller | User selects branch | System decides when to sprout leaves |
| Typical use | Multi-topic parallelism | Background tasks (compression, extraction, audit) |

### Context Cache Utilization

```
Context Cache (unchanged): [conversation history]        ← trunk
main branch (critical path):   [history + user question]        → normal answer
bypass branch (new computation): [history + tail prompt]   ← leaf (fired in parallel)
  → the leaf is its own request: dedicated attention, pays only its delta
    (prefix KV shared with the main branch)
  → after the request ends, the leaf falls off (tail prompt discarded),
    its result lands in external state, the trunk keeps growing
```

Analogy to TCO (Tail Call Optimization): tail calls reuse the current stack frame; tail prompts reuse the current Context Cache. Both are "tail" operations that reuse existing state. (The old form prepended the tail prompt to the user question and completed answer + task in one turn — the compression scenario now uses the parallel side branch, see below; external-state sedimentation tasks generally belong on the bypass, not in the main branch's attention.)

### Core Properties

- **Cache utilization**: History portion uses Context Cache; only the tail prompt is newly computed
- **Bypass branch**: Does not modify the main trunk (conversation history), only appends temporary instructions at the end
- **One-shot**: Removed from prompt after turn ends, not written to session
- **Control**: Injector decides when and what to inject; Agent only executes

### Compression Scenario: Parallel Side Branch

Tail prompts use the vine pattern — one main trunk sprouts leaves, then continues growing. The compression scenario's special case — when the leaf falls, the preceding trunk is also replaced (checkpoint replaces old history).

Form revision (2026-09-28): compression no longer mixes into the answer turn (the old "answer first, then call tools" shape did two jobs in one turn) — it runs as a **side branch parallel to the main branch**: the main branch answers the user as normal, the compression instruction is forked from the same immutable prefix in parallel; prefix KV is shared, the branch pays only its delta, and the summary lands before the next user request.

```
Immutable prefix [H] = [ckpt_0] + [msg_101..msg_150]   ← in cache, shared by both branches

Main branch (critical path): [H] + [user message]        → normal answer
Compression side branch:     [H] + [compression instr.]  → tool_call: memory_checkpoint / memory_store
```

After the requests end, session stores clean history:

```
Stored in session:
  [checkpoint_1] + ["Why is Fluxora's component set closed?"] + ["Fluxora's component set is closed because..."]

NOT:
  [checkpoint_1] + [tail prompt + "Why is Fluxora's component set closed?"] + ["Fluxora's component set is closed because..."]
```

Both requests discard their tail prompts; no residue. Full mechanism: the "compression is a parallel side branch" section in [stateless-agent-architecture.md](stateless-agent-architecture.md).

### Comparison with Traditional Approaches

| Dimension | Traditional (independent call) | Mixed into turn (old form) | Parallel side branch (current) |
|:--|:--|:--|:--|
| LLM Call | Independent API call, prefix-unrelated | Reuses answer turn | Independent branch, shares prefix with main |
| Context Cache | From scratch (0% hit) | History cached (~99%) | History cached (~99%), one KV for both branches |
| Attention | Fully on compression | Split (answer + compress in one turn) | Dedicated (branch does only compression) |
| User Latency | Serial when model is busy | Preempts conversation rhythm | Zero (hidden in the typing gap) |
| Control | Compression module controls | Injector controls | Injector controls |

### Cost Analysis

The traditional compression argument ("auxiliary model is cheaper than cache") disappears when amortized per call:

- Auxiliary model price ≈ 1/60 of main model, cache price ≈ 1/10 of main model
- Compress once every 100 turns: savings = (1/10 - 1/60) = 8.3% of input cost
- Amortized per call: 8.3% / 100 = **0.083%**
- Trade-off: compression quality loss (information dropped, compressor lacks Agent context)

Side-branch compression uses the main model plus a compression instruction: the history portion hits cache (~99%), and the instruction owns the entire request's attention — higher quality than the mixed-turn form (which split attention between answering and compressing) and far cheaper than a standalone auxiliary call (whose prefix hit is 0%).

The auxiliary model, while fast per-token, has no Context Cache (0% hit rate) and must recompute the full input.

## References

- [agent-memory.md](agent-memory.md) — Tail prompt application in Prefix Checkpoint
- [graph-memory.md](graph-memory.md) — Extraction prompts in graph-structured memory
- [LLM Caching Destruction Patterns](llm-caching-destruction-patterns.md) — Cache branch patterns (thread) vs destruction patterns
