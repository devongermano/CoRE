# CoRE — Continuity Refactor Engine

*A practical architecture for global, document‑scale, semantically consistent refactoring of long narratives*

## What is it?

**TLDR.** Given a very long manuscript (hundreds of thousands of tokens) plus a high‑level edit (e.g., change a character’s nationality/traits), CoRE (a) **finds** every impacted span, (b) **plans** ripple effects across entities/events/timelines, (c) **proposes** minimal, style‑preserving rewrites under **global constraints**, and (d) **verifies** factual/temporal consistency and spoiler safety before commit.

**Core ideas (evidence):**

* **Hybrid long‑context reading** beats “just make the window bigger.” Empirical studies show long‑window models often degrade with length and struggle to use mid‑context info (“lost in the middle”), and broader long‑context benchmarks confirm performance drops as context grows—even for models advertising very large windows. CoRE streams the book and **selects** only what matters instead of dumping everything at once. ([arXiv][1])
* **Training‑free efficiency methods** now make massive context practical: **InfiniRetri** turns a transformer’s own attention into a retrieval signal over arbitrarily long inputs; **RetrievalAttention** uses ANN over KV to read only \~1–3% of keys per step; **Cascading KV Cache** and **DuoAttention** retain what matters without ballooning memory. CoRE builds on these to read/book‑keep at scale. ([arXiv][2])
* **World modeling** is essential: editing requires addressable facts, events, and timelines with links back to exact text spans. Recent work shows **event‑centric retrieval (EventRAG)** and **storyline graphs (Narrative Trails)** help reason across narrative‑rich documents, and **BOOKCOREF** finally makes book‑scale coreference measurable. CoRE formalizes a story “bible” as a graph. ([ACL Anthology][3], [arXiv][4])
* **Document‑level editing is a distinct, hard task.** New benchmarks (**DocMEdit**) show existing methods struggle when multiple facts and long contexts must be edited; **DeepEdit** demonstrates that **constraint‑aware decoding** improves faithfulness when integrating new knowledge. CoRE operationalizes edits as constrained generation with verification gates. ([arXiv][5])
* **Verification matters.** Lightweight NLI‑style inconsistency checking (**SummaC**) and discourse‑driven evaluation for long documents, plus contradiction checks for RAG inputs and spoiler detectors, provide practical safety rails. CoRE adds these as mandatory pre‑merge checks. ([ACL Anthology][6], [arXiv][7])

---

## System Responsibilities

1. **Understand:** Build and maintain a **World Model** (entities, attributes, relations; events with temporal order) from the evolving manuscript; keep span‑level provenance. ([ACL Anthology][3])
2. **Plan:** Given an **Edit Specification**, compute the **impact set** (direct assertions + implied ripples), scoped by acts/POVs, with spoiler guards.
3. **Edit:** Produce **minimal‑diff** patches per impacted span via **constraint‑aware decoding** that must satisfy global constraints (new facts, voice/tone, temporal consistency). ([arXiv][8])
4. **Verify:** Run factual/temporal **consistency checks**, cross‑doc contradiction checks (for multi‑source evidence), and **spoiler filters**; only then propose diffs to the user. ([ACL Anthology][6], [arXiv][9])
5. **Orchestrate:** Provide git‑style diffs, confidence scores, failure hotspots, and “reasons” tags for each change; maintain an audit trail.

---

## End‑to‑End Architecture

```
             ┌───────────────────────────────────────────────────────────┐
             │                         CoRE                              │
             ├───────────────┬─────────────────────────┬─────────────────┤
             │  Ingest &     │  Long-Reader Core       │  World Model    │
             │  Routing      │  (Training-free)        │  (Graph+Spans)  │
             ├───────────────┴─────────────────────────┴─────────────────┤
             │                 Refactor Planner (global)                  │
             ├───────────────────────────────────────────────────────────┤
             │               Rewrite Executor (local)                     │
             │    (constraint-aware decoding; minimal-diff patches)       │
             ├───────────────────────────────────────────────────────────┤
             │           Verifier (NLI, discourse, RAG conflicts,        │
             │                 spoiler detection, style)                  │
             ├───────────────────────────────────────────────────────────┤
             │                 Human-in-the-loop UI                       │
             └───────────────────────────────────────────────────────────┘
```

### Ingest & Routing

* **Corpus**: Markdown chapters/notes; structure‑aware chunking by headings/scene markers.
* **Dual index**:

  * **Vector** (FAISS) for coarse retrieval of relevant chapters/scenes.
  * **Lexical** (BM25) for high‑precision keyword hits (e.g., passport/visa/brand mentions).
* **Why**: Route the long‑reader only to likely‑relevant files/scenes; don’t stream the whole corpus every time. This aligns with evidence that long‑context alone is brittle and cost‑heavy. ([arXiv][1])

### 2.2 Long‑Reader Core (training‑free, scalable)

* **Primary mode — InfiniRetri loop.** For each routed chapter, iterate through chunks (e.g., 1–2K tokens). After a forward pass, **extract final‑layer attention**, select **top‑K salient sentences** (smoothing over tokens), and **carry them forward** as a compact memory. This yields an “infinite” effective read without quadratic attention, demonstrated on million‑token needle tests and long‑doc QA. ([arXiv][2])
* **Acceleration knobs (optional):**

  * **RetrievalAttention:** index past KV on CPU; fetch only \~1–3% most relevant keys per step with near‑full accuracy, enabling very long contexts on commodity GPUs. ([arXiv][10])
  * **Cascading KV Cache:** multi‑tier caches that retain important tokens; linear prefill scaling and strong retrieval at \~1M effective tokens (ICLR’25). ([arXiv][11], [OpenReview][12])
  * **DuoAttention:** keep **full** KV only for “retrieval heads,” constant‑length cache for “streaming heads,” cutting memory/latency while preserving long‑context ability. ([arXiv][13])

> **Why it matters:** Long‑window models often falter as length grows (RULER) and especially on mid‑context info (Lost in the Middle). Streaming + selective retention is more robust and cost‑effective in practice. ([arXiv][14])

### World Model (Story “Bible” with Span Provenance)

* **Entities & Coreference:** Run document/book‑scale coref; unify all mentions of characters/props/places with links back to every asserting span. **BOOKCOREF** provides the first book‑scale benchmark (avg >200k tokens/book) and shows models improve markedly when trained/evaluated at this scale (+20 CoNLL‑F1)—use its pipeline/insights to guide modeling and evaluation. ([ACL Anthology][15])
* **Event & Timeline Graph:** Extract events (subject‑verb‑object with time/place), causal/temporal links (before/after, act/scene indices), and attach provenance (spans). Recent **EventRAG** results show event‑centric representations improve multi‑document, temporal reasoning in narrative‑rich settings; CoRE adopts an Event‑KG schema. ([ACL Anthology][3])
* **Storyline Structure (optional):** Build **Narrative Trails** (coherence graph → maximum‑capacity paths) to surface arcs/threads when planning ripple effects. ([arXiv][4])

---

## Edit Specification (CoRE‑DSL)

A declarative DSL captures the “what” and the guardrails:

```yaml
edit:
  type: CharacterProfileUpdate
  target: "Marisol Vega"
  changes:
    nationality: "Peruvian"         # was "French"
    add_traits: ["compulsively punctual"]
    remove_traits: ["chain-smoker"]
  scope:
    acts: [1, 2]                    # do not touch Act 3 (twist)
    viewpoints: ["narrator","Tomas"]
  invariants:
    - "keeps heirloom locket"
    - "parents met in Lima"
  policy:
    rewrite: "minimal-diff"
    voice: "preserve-local-style"
```

* **Semantics:** “Canonicalize” the new facts in the World Model → compute an **impact set** of spans where old facts are asserted or **entailed** (e.g., accents/cuisine/visas/family origin if nationality changes). Event‑ and timeline‑links propagate ripples safely. ([ACL Anthology][3])

---

## Refactor Planner (Global)

1. **Apply DSL to the graph:** Update entity attributes and event preconditions.
2. **Compute impact set:**

   * **Direct assertions** (e.g., “French passport,” “lights a Gauloises”).
   * **Implicit cues** (idioms, cuisine, locale stereotypes) via KG relations and lexical patterns.
   * **Temporal/ripple paths** (e.g., border crossing, school nationality) along event edges. ([ACL Anthology][3])
3. **Scope & spoiler gates:** Exclude scenes outside scope; block edits that would reveal downstream twists (spoiler classifier). ([arXiv][16])

---

## Rewrite Executor (Local, Many Sites)

For each impacted site:

* **Context package** = local scene (+ few preceding/following paragraphs) + **InfiniRetri memory** for this site + **character card** + **constraints** derived from DSL.
* **Constrained decoding (DeepEdit‑style):** Generate **minimal‑diff** patches that must satisfy **local + global constraints** (entail new facts, preserve voice, obey scope). This **guided‑decoding** approach improved coherence and faithfulness when integrating new knowledge in multi‑step reasoning tasks, without altering model weights. ([arXiv][8])
* **Multi‑candidate & ranking:** Sample a small beam under constraints; rank by constraint satisfaction + NLI consistency + style metrics.

---

## Verifier (Pre‑Merge Checks)

1. **Factual consistency (local & doc‑level):** NLI‑style inconsistency detection (e.g., **SummaC**) compares the candidate patch to its supporting spans; proven lightweight and effective on multi‑dataset summary consistency, and remains a solid baseline checker. ([ACL Anthology][6])
2. **Discourse‑aware long‑doc checks:** Weight contradictions by discourse importance and segment structure; recent work shows better detection for long summaries and complex sentences with a **discourse‑driven** approach. ([arXiv][7])
3. **RAG conflict check (if multi‑source evidence was retrieved):** Run a **context‑validator** pass to flag contradictions among retrieved documents before final synthesis; 2025 evaluations show this is still challenging and varies with prompt/model choice—so treat it as a blocking gate. ([arXiv][9])
4. **Spoiler detection:** Classify candidate diffs for spoiler risk (genre‑aware mixture‑of‑experts methods achieve SOTA on spoiler benchmarks); block or require explicit user override. ([arXiv][16])

---

## Orchestration & UI

* **Diff bundles** grouped by *reason* (e.g., “nationality ripple → passport,” “habit removal → ashtray mentions”), with per‑patch **confidence**, **contradiction**, and **spoiler** signals.
* **Traceability:** Every KG node and patch retains **span provenance** and **evidence links**; clicking a node shows all asserting spans and the proposed edits.
* **Approval workflow:** Approve per patch, per bundle, or per chapter; regenerate with stricter/looser constraints.

---

## Data & Model Choices (initial, practical stack)

* **Long‑reader:** Any HF transformer that exposes attentions; apply **InfiniRetri** externally; add **RetrievalAttention** or **Cascading KV** for speed when needed. ([arXiv][2])
* **Indexes:** FAISS (vector), BM25 (lexical).
* **Coreference:** Start with the best available long‑doc/coref model; target **BOOKCOREF** style behavior; use the released pipeline/data for evaluation. ([ACL Anthology][15])
* **Event extraction:** Off‑the‑shelf OpenIE/SRL/Temporal taggers + LLM prompts; map to **EventRAG**‑like schema. ([ACL Anthology][3])
* **Constrained decoding:** Implement **DeepEdit** constraints as a decoding/reranking layer; black‑box friendly. ([arXiv][8])
* **Verifiers:** NLI model for SummaC‑style checks; discourse segmenter; contradiction checker; spoiler classifier. ([ACL Anthology][6], [arXiv][7])

---

## Why This Design (Evidence Synthesis)

* **Robustness over raw window size.** Long‑window models often **under‑use** long context and degrade as length grows; “lost‑in‑the‑middle” is persistent. CoRE’s streaming + salience retention aligns with empirical findings and avoids quadratic attention. ([arXiv][1])
* **Training‑free scalability.** **InfiniRetri** (attention‑driven retrieval), **RetrievalAttention** (ANN over KV), **Cascading KV**, and **DuoAttention** show strong wins without fine‑tuning—ideal for immediate deployment on massive texts. ([arXiv][2])
* **Edit as a first‑class task.** **DocMEdit** demonstrates that multi‑fact, document‑level updates are hard for existing methods; **DeepEdit** shows constraint‑aware decoding improves faithfulness—hence CoRE’s edit executor. ([arXiv][5])
* **Narrative‑specific reasoning.** **EventRAG** (event‑centric, temporal‑aware retrieval) and **Narrative Trails** (coherence‑optimized storyline paths) provide principled ways to reason across narrative‑rich corpora—mirrored in CoRE’s World Model. ([ACL Anthology][3], [arXiv][4])
* **Measurable continuity.** **BOOKCOREF** elevates evaluation to full books (avg >200k tokens) and reveals gaps; CoRE’s coref‑centric bible and tests close that loop. ([ACL Anthology][15])

---

## Evaluation Plan (make it measurable)

1. **Reading at scale:** RULER / OneRULER + needle‑in‑haystack variants at your target lengths; track accuracy vs. context growth to show stability. ([arXiv][14])
2. **Entity tracking:** BOOKCOREF metrics (CoNLL‑F1) on samples of your manuscript or similar texts. ([ACL Anthology][15])
3. **Edit quality:** DocMEdit‑style evaluation—multi‑fact, document‑level updates; minimal‑diff ratio; human blind ratings for voice/continuity. ([arXiv][5])
4. **Consistency & safety:** SummaC (NLI) scores pre/post; discourse‑driven inconsistency; contradiction‑in‑RAG stress tests; spoiler detection precision/recall on twist‑heavy chapters. ([ACL Anthology][6], [arXiv][7])

---

## Implementation Roadmap

**Phase 1 — Prototype (4–6 weeks)**

* **Routing + Long‑reader**: FAISS + BM25 → **InfiniRetri** per chapter; store salient‑sentence memory with provenance. ([arXiv][2])
* **World Model v0**: long‑doc coref; entity cards; basic event tuples with timestamps/scene indices.
* **Edit‑DSL & Planner v0**: YAML spec → collect direct assertion spans + nearest cues.
* **Executor v0**: constrained decoding with minimal‑diff prompt; 1‑candidate drafts. **Verifier v0**: SummaC check. ([ACL Anthology][6])

**Phase 2 — Scale & Reliability (6–10 weeks)**

* Add **RetrievalAttention**/**Cascading KV**/**DuoAttention** for speed/scale; expand events → Event‑KG; add **discourse‑aware** verification and **spoiler** filter; multi‑candidate ranking. ([arXiv][10])

**Phase 3 — Narrative Mastery (10–16 weeks)**

* **Ripple analysis** over Event‑KG; **Narrative Trails** for arc‑aware planning; **RAG contradiction** gate; full human‑in‑the‑loop UI with reason tags and confidence. ([arXiv][4])

---

## Risk & Mitigations

* **Coref drift across long books.** Use book‑scale models/datasets; verify with BOOKCOREF; boost with manual entity seeds (character lists). ([ACL Anthology][15])
* **Over‑editing voice.** Enforce “minimal‑diff” and style similarity in constraints; prefer local paraphrase with preserved rhythm and punctuation.
* **False‑positive spoilers.** Allow user override; calibrate classifier per genre; support scope masks by act/POV. ([arXiv][16])
* **Latency on very long chapters.** Use RetrievalAttention/KV‑caching and parallelize chunk scoring; cache per‑chapter salience maps. ([arXiv][10])

---

## Appendix A — Minimal Data Schemas

**Entity (Character/Prop/Place)**

* `id`, `aliases`, `attributes` (e.g., nationality, habits), `first_seen`, `last_seen`, `spans[]` (doc\_id, start, end, evidence\_type).

**Event**

* `id`, `participants{role→entity_id}`, `predicate`, `time`, `place`, `causal_links[]`, `spans[]`. (Event‑RAG compatible). ([ACL Anthology][3])

**Constraint Pack (per edit)**

* `must_include[]` (facts/terms), `must_exclude[]`, `style_guidelines`, `scope_masks`, `invariants[]`.

---

## Appendix B — Edit‑DSL Examples

**(1) Habit removal + nationality change, scoped to Acts 1–2)**

```yaml
edit:
  type: CharacterProfileUpdate
  target: "Marisol Vega"
  changes:
    nationality: "Peruvian"
    remove_traits: ["chain-smoker"]
  scope: { acts: [1,2] }
  invariants: ["keeps heirloom locket"]
  policy: { rewrite: "minimal-diff", voice: "preserve-local-style" }
```

**(2) Prop relocation across scenes)**

```yaml
edit:
  type: PropRelocation
  target: "heirloom locket"
  changes:
    location: "Tomas's desk drawer"
  scope: { chapters: [4,5,6] }
  invariants: ["locket is not mentioned on Marisol in ch5 until reveal"]
```

---

## References (key sources cited)

* **Long‑context efficiency & behavior**:
  **InfiniRetri** (attention‑based, training‑free infinite retrieval). ([arXiv][2])
  **RetrievalAttention** (ANN over KV; \~1–3% keys). ([arXiv][10])
  **Cascading KV Cache** (ICLR’25, linear prefill scaling). ([arXiv][11], [OpenReview][12])
  **DuoAttention** (keep full cache for retrieval heads only). ([arXiv][13])
  **Lost in the Middle** (mid‑context weakness). ([arXiv][1])
  **RULER / OneRULER** (comprehensive long‑context evaluation). ([arXiv][14])

* **Document‑level editing**:
  **DocMEdit** (document‑level, multi‑fact edits; Findings of ACL 2025). ([arXiv][5], [ACL Anthology][17])
  **DeepEdit** (decoding with constraints; black‑box friendly). ([arXiv][8])

* **Narrative world modeling**:
  **EventRAG** (event‑centric KG for generation across narrative‑rich docs). ([ACL Anthology][3])
  **Narrative Trails** (coherence‑optimized storyline extraction). ([arXiv][4])
  **BOOKCOREF** (book‑scale coreference; avg >200k tokens; +20 CoNLL‑F1 gains). ([ACL Anthology][15])

* **Consistency & spoiler safety**:
  **SummaC** (NLI‑based summary inconsistency detection; strong lightweight baseline). ([ACL Anthology][6])
  **Discourse‑driven long‑doc inconsistency** (segment by discourse structure). ([arXiv][7])
  **Contradiction detection in RAG contexts** (context validators are still hard). ([arXiv][9])
  **Spoiler detection (MMoE, genre‑aware)**. ([arXiv][16])

---

### Closing Note

This design is intentionally **engine‑first** and **training‑free** where possible: you can implement it today with open‑weight models and standard libraries. It piggybacks on strong, recent evidence that selective, attention‑guided reading plus explicit world modeling—and edits executed under constraints with rigorous verification—yields the most reliable path for **global, narrative‑scale refactoring**.

[1]: https://arxiv.org/abs/2307.03172?utm_source=chatgpt.com "Lost in the Middle: How Language Models Use Long Contexts"
[2]: https://arxiv.org/abs/2502.12962?utm_source=chatgpt.com "Infinite Retrieval: Attention Enhanced LLMs in Long ..."
[3]: https://aclanthology.org/2025.acl-long.830.pdf?utm_source=chatgpt.com "Enhancing LLM Generation with Event Knowledge Graphs"
[4]: https://arxiv.org/abs/2503.15681?utm_source=chatgpt.com "Narrative Trails: A Method for Coherent Storyline Extraction via Maximum Capacity Path Optimization"
[5]: https://arxiv.org/abs/2505.19572?utm_source=chatgpt.com "DocMEdit: Towards Document-Level Model Editing"
[6]: https://aclanthology.org/2022.tacl-1.10/?utm_source=chatgpt.com "SummaC: Re-Visiting NLI-based Models for Inconsistency ..."
[7]: https://arxiv.org/abs/2502.06185?utm_source=chatgpt.com "Unveiling Factual Inconsistency in Long Document ..."
[8]: https://arxiv.org/abs/2401.10471?utm_source=chatgpt.com "DeepEdit: Knowledge Editing as Decoding with Constraints"
[9]: https://arxiv.org/abs/2504.00180?utm_source=chatgpt.com "Contradiction Detection in RAG Systems: Evaluating LLMs as Context Validators for Improved Information Consistency"
[10]: https://arxiv.org/abs/2409.10516?utm_source=chatgpt.com "RetrievalAttention: Accelerating Long-Context LLM ..."
[11]: https://arxiv.org/abs/2406.17808?utm_source=chatgpt.com "[2406.17808] Training-Free Exponential Context Extension ..."
[12]: https://openreview.net/forum?id=dSneEp59yX&noteId=cXYZDTrz2p&utm_source=chatgpt.com "Training Free Exponential Context Extension via ..."
[13]: https://arxiv.org/abs/2410.10819?utm_source=chatgpt.com "DuoAttention: Efficient Long-Context LLM Inference with ..."
[14]: https://arxiv.org/abs/2404.06654?utm_source=chatgpt.com "RULER: What's the Real Context Size of Your Long-Context Language Models?"
[15]: https://aclanthology.org/2025.acl-long.1197/?utm_source=chatgpt.com "BOOKCOREF: Coreference Resolution at Book Scale"
[16]: https://arxiv.org/abs/2403.05265?utm_source=chatgpt.com "[2403.05265] MMoE: Robust Spoiler Detection with Multi- ..."
[17]: https://aclanthology.org/2025.findings-acl.1012/?utm_source=chatgpt.com "DocMEdit: Towards Document-Level Model Editing"
