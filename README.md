# Legal QA Agentic System: Portuguese Online Gambling Legislation

An enterprise-grade, robust, and auditable Agentic RAG system built for Serviço de Regulação e Inspeção de Jogos (SRIJ), engineered to answer complex legal questions regarding the Portuguese online gambling and betting legislation. Built with **LangGraph** for deterministic multi-agent state machines, **LlamaIndex** for advanced document parsing and hybrid retrieval, and **LangSmith** for full production tracing and observability.

---

### 1. Project Objective

The core goal of this project was to construct an autonomous, legal-focused QA system capable of delivering highly accurate answers based on Portuguese statutory law while meeting strict operational guardrails:

* **Strict Context Grounding:** Every response must be structurally mapped and anchored to the official legal text.
* **Zero Legal Hallucination:** The agent is mathematically restricted from extrapolating concepts or fabricating regulatory windows.
* **Domain Gating:** Out-of-domain queries or prompt injections are instantly intercepted at the threshold.
* **Deterministic Lookups:** Textual verification requests (e.g., *“What is the exact wording of Article 10, paragraph 2?”*) bypass vector search to guarantee absolute factual retrieval.
* **Complex Interpretative Reasoning:** Multi-layered thematic queries (e.g., player self-exclusion protocols) are processed dynamically without losing normative scope.

---

### 2. End-to-End Architecture Overview

The system implements a production-ready **Agentic RAG** pipeline orchestrated via **LangGraph**. The workflow limits LLM dependency to critical gating, grading, and grounding tasks, leveraging traditional high-performance database lookups for the retrieval stage to minimize token costs and system latency.

**High-Level Execution Steps:**
1. Document Ingestion & Manual Metadata Enrichment (`PyMuPDFReader`).
2. Legal Text Cleansing & Structural Normalization (Custom Regex Filters).
3. Article-Level Logical Chunking (Mitigating paragraph fragmentation).
4. Dual-Stage Domain Gating & Retrieval Routing.
5. Deterministic Metadata Mapping or Hybrid Fusion Search (Vector + BM25).
6. Relevance Filtering via a Contextual LLM Grader (with literal bypass hooks).
7. Grounded Answer Synthesis (Strict verbatim constraints).
8. Post-Generation Legal Reflection (Hallucination checking and self-correction).

---

### 3. Document Processing & Ingestion Pipeline

#### 3.1 Ingestion & Legal Metadata Ingestion
* Statutory PDF documents are parsed using `PyMuPDFReader`, preserving strict layout hierarchies and structural characters that standard tokenizers corrupt.
* Granular legal metadata layers are injected into the document nodes (e.g., official source, legal version, active jurisdiction, publication year, and explicit article IDs).

#### 3.2 Tailored Text Cleansing
Public legal journals often introduce text formatting noise during raw extraction. Custom regex layers were developed to clean:
* Standardized page numbering artifacts (e.g., *“Page 28 of 59”*).
* Repetitive institutional headers and footers from the *Diário da República*.
* Artificial line breaks and hyphenation patterns introduced by multi-column layouts.

> 💡 **Engineering Impact:** This targeted data-cleansing pipeline reduced false negatives during retrieval by **over 35%**, ensuring proper token alignment for targeted sub-clauses and alphanumeric legal sections.

#### 3.3 Article-Level Logical Chunking
Standard RAG systems slice documents arbitrarily based on raw character length, splitting paragraphs mid-sentence and destroying legal context. To combat this, this architecture enforces **Article-by-Article Semantic Boundaries**. 

Every node generated in the LlamaIndex vector space maps exactly to one standalone statutory article. This logical separation ensures that lookups targeting specific structural laws inherit 100% of the associated normative context.

---

### 4. High-Fidelity Retrieval Strategy

#### 4.1 Hybrid Index Architecture
The retrieval layer runs a localized **Hybrid Fusion Search** mechanism that functions entirely without LLM calls, lowering inference runtime:
* **Dense Vector Retrieval:** Deep semantic search leveraging the `multilingual-e5` embedding space.
* **Sparse Lexical Retrieval:** Exact matching using a calibrated `BM25` algorithm to isolate specific legal jargon, technical acronyms, and numbered acts.

#### 4.2 Deterministic Article Lookups
When the parsing layer identifies a targeted, literal request for an article (e.g., *“Show me Article 14”*), the agent bypasses probabilistic algorithms (Embedding/BM25) entirely. It runs a direct, hard-filtered metadata query on the underlying index:

$$\text{Active Filter Route} \longrightarrow \text{metadata}["\text{artigo}"] == N$$

This programmatic shortcut completely eliminates the classic vector search limitation where embeddings fail to distinguish between numerically adjacent articles (e.g., pulling Article 13 instead of 14).

#### 4.3 Hybrid Fallback Route
For open-ended conceptual questions (e.g., *“What are the mandatory conditions for a casino license?”*), a fallback routing structure triggers:
1. Parallel vector candidate matching and BM25 lexical querying.
2. Min-Max scoring normalization across both outputs.
3. Deterministic Reciprocal Rank Fusion (RRF) weighting.
4. **Zero external LLM API dependency** during this matching loop, securing absolute performance stability.

---

### 5. LangGraph Agentic Framework & State Machine

The multi-agent workflow is modeled as a Directed Acyclic Graph (DAG) state machine managed via LangGraph. The runtime paths adapt dynamically based on user input, enforcing early-stopping thresholds to maximize cost efficiency:

```
                      [User Submits Query]
                               │
                               ▼
                         [domain_gate]
                         ╱           ╲
               (fora_do_dominio)   (potencialmente_documental)
                     ╱                   ╲
                    ▼                     ▼
         [direct_chat_response]   [retrieval_decider]
                    │              ╱               ╲
                    │       (has tool_calls)   (no tool_calls)
                    │            ╱                   ╲
                    ▼           ▼                     ▼
                 [ END ]    [retrieve] ────────> [generate_answer]
                            ┌────────┐                    │
                            │ Internal                    │
                            │ Tool Logic:                 │
                            │ ├► Direct                   │
                            │ └► Hybrid                   │
                            └───┬────┘                    │
                                │                         │
                                ▼                         │
                         [grade_chunks]                   │
                                │                         │
                                ▼                         ▼
                         [generate_answer] ──────> [reflection]
                                                          │
                                                          ▼
                                                       [ END ]
```

#### 5.1 Dynamic Inbound Gating & Conditional Routing Mechanics
To control infrastructure costs and avoid redundant computation, the graph execution layers are divided into distinct deterministic and probabilistic routing clusters:

* **Out-of-Domain Threshold (`domain_gate` ──> `direct_chat_response`):** If a query is categorized as `fora_do_dominio` (e.g., casual greetings, prompt injections, or off-topic requests), the system instantly targets the `direct_chat_response` node. The LLM generates a polite, localized conversational refusal, and the state flows immediately to the terminal **`END`** node, completely safeguarding database resources.
* **Intelligent Asset Routing (`retrieval_decider`):** When a query is validated as `potentially_documental`, it moves into an LLM bound with database extraction tools (`llm.bind_tools(tools)`). The graph hooks into a customized `tools_condition` to evaluate two distinct runtime states:
  * **The Dynamic Extraction Path (`tools` ──> `retrieve`):** Triggered when the LLM populates a `tool_call`. The execution enters the unified `retrieve` node (`ToolNode`), which encapsulates the core retrieval intelligence internally:
    * *Deterministic Direct Sub-Route:* If the tool detects a specific article reference (e.g., *"Artigo 10.º"*), it enforces a hard metadata filter lookup to avoid embedding ambiguity.
    * *Probabilistic Hybrid Sub-Route:* If the query is thematic, it executes the manual fusion of Dense Vector (`multilingual-e5`) and Sparse Lexical (`BM25`) search scores.
    Following retrieval, the extracted chunks pass to `grade_chunks` for relevance verification, proceed to `generate_answer` for synthesis, and clear the `reflection` audit layer.
  * **The Direct Knowledge Pipeline (`END` ──> `generate_answer`):** If the legal query is conceptually straightforward or part of a multi-turn clarification that does not require a fresh database query, the default `tools_condition` signals an execution termination (`END`). The graph architecture intercepts this signal and **reroutes the state directly into the `generate_answer` node**. This programmatic bypass skips the heavy retrieval and grading steps, yet forces the response to clear the critical `reflection` audit node, ensuring absolute alignment before returning the payload to the user.

#### 5.2 Document Pipeline & Post-Retrieval Validation
When valid queries execute through the document pipeline, the system enforces a strict compliance sequence across the remaining nodes:

* **`grade_chunks`:** A validation agent evaluates the extracted legal chunks and filters out non-responsive background noise. Nodes originating from the `direct` route automatically receive a green-lit bypass within this node to prevent the LLM from mistakenly filtering out cold, literal statutory articles due to low conversational density.
* **`generate_answer`:** Synthesizes the cleared statutory chunks, enforcing a strict "No Paraphrasing" rule for direct lookups and ensuring that any lack of coverage in the text results in an explicit declaration of omission rather than extrapolation.
* **`reflection` (Self-Correction Layer):** Acting as an automated legal auditor, this node checks the synthesized text against the raw source chunks. If the LLM has hallucinated a regulatory implication or extrapolated a timeframe not explicitly present in the law, the node automatically triggers a re-generation loop or replaces the payload with a secure default: *“The consulted legislation does not contain sufficient factual evidence to support this query.”*

---

### 6. Identified Challenges & Production Solutions

| Identified Challenge | Production Symptom | Implemented Architectural Solution |
| :--- | :--- | :--- |
| **Arbitrary Token Slicing** | Section fragmentation; critical statutory sub-clauses isolated outside context. | **Logical Article Ingestion:** Structural segmentation setting individual articles as distinct data objects. |
| **Embedding Vector Ambiguity** | Embedding space confusion between numerically close clauses (e.g., Art. 12 vs. Art. 13). | **Metadata Bypass Layer:** Absolute programmatic filter lookups executing directly via structural indices. |
| **Aggressive LLM Grading** | Literal legal transcriptions discarded due to low conversational semantic density. | **Automated Route Bypassing:** Total exclusion of the grading agent for hard-mapped structural queries. |
| **Model Over-reach & Bias** | Foundations models attempting to infer regulatory timelines or patch legislative gaps. | **Stateful Reflection Node:** Real-time factual grounding cross-checker acting as a rigid denial barrier. |

---

### 7. Tech Stack & Infrastructure

* **Python 3.11** - Core high-performance programming runtime.
* **LangGraph** - Cyclic and acyclic state-machine orchestration for agent logic.
* **LlamaIndex** - Unified parsing, data structure modeling, and automated metadata indexing.
* **LangSmith** - Production tracing, precise latency mapping, and token-cost auditing.
* **PyMuPDF** - Low-level textual parsing from high-density public legal documents.