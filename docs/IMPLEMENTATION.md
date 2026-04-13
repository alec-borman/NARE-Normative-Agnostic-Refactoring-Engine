# NARE Implementation Outline: Hierarchical Context & Automated Retrieval

This document outlines a reference architecture for a fully automated NARE pipeline that supports arbitrarily large codebases through semantic chunking, embedding‑based retrieval, and deterministic actuation. It extends the v1.2.0 specification to address the "Context Window Cliff" identified in the validation critique.

## 1. System Overview

The implemented NARE system consists of five primary components:

| Component | Responsibility |
| :--- | :--- |
| **Snapshot Indexer** | Chunks the codebase into semantically meaningful units, computes embeddings, and stores them in a vector database. |
| **Oracle (Planner)** | Accepts an Intent, queries the vector store for relevant chunks, and generates a sequence of YAML Tela Tasks. |
| **Orchestrator** | Manages the task queue, feeds tasks to the Actuator, handles revision loops, and maintains state. |
| **Actuator** | Sandboxed executor that applies deterministic mutation scripts and captures verification output. |
| **Compliance Dashboard** | Displays the final Compliance Report for Master Builder review and commit approval. |

All components are **language‑agnostic** and **model‑agnostic**, adhering to the NARE protocol's core constraints.

## 2. Component Details

### 2.1 Snapshot Indexer

**Purpose:** Transform a large repository into a queryable, immutable knowledge base without exceeding context limits.

**Input:** A local path to the repository, plus an optional `.nareignore` file to exclude build artifacts and vendor directories.

**Processing Steps:**
1. **Traverse** the file tree, respecting ignore rules.
2. **Chunk** each source file using a syntax‑aware splitter (e.g., `tree-sitter` for the target language) that preserves function/class boundaries.
   - *Rationale:* Semantic chunking ensures that retrieved context is self‑contained and meaningful to the Planner.
3. **Compute Embeddings** for each chunk using a lightweight embedding model (e.g., `all-MiniLM-L6-v2` or a cloud‑provided embeddings API).
4. **Store** chunks in a vector database (e.g., Chroma, LanceDB, or a simple SQLite + FAISS hybrid) with metadata:
   - `file_path`
   - `chunk_start_line`, `chunk_end_line`
   - `hash_before` (SHA‑256 of the original chunk content)
   - `dependencies` (list of imported modules / symbols extracted via static analysis)

**Output:** A populated vector index and a manifest file (`snapshot_manifest.json`) recording the repository state (commit hash, timestamp, chunk count).

**Idempotency:** The Indexer is run once per sprint (or on‑demand) and produces an immutable snapshot view. All subsequent operations reference this snapshot.

### 2.2 Oracle (Planner) with Retrieval Augmentation

**Purpose:** Decompose an Intent into verifiable Tela Tasks using *only* the context necessary to perform the surgery.

**Input:**
- Plain‑English `Intent`.
- Access to the vector store (read‑only).
- The NARE Oracle System Instructions (`Oracle-System-Instructions.md`).

**Workflow:**
1. **Generate Query Embeddings:** Convert the Intent into an embedding vector.
2. **Retrieve Relevant Chunks:** Query the vector store for the top‑K most similar chunks (K configurable, default 20).
3. **Dependency Expansion (Optional):** For each retrieved chunk, follow its `dependencies` metadata to pull in additional chunks that are statically linked, ensuring the Planner sees a complete subgraph.
4. **Assemble Context Window:** Compose a single prompt containing:
   - The Oracle system instructions.
   - The Intent.
   - The retrieved chunks (each prefixed with its `file_path` and `hash_before`).
5. **Generate Tela Task Sequence:** The LLM outputs a sequence of YAML TDBs strictly conforming to the NARE Task Schema.

**Fallback:** If the Planner determines that the retrieved context is insufficient (e.g., a required file is missing), it may request additional chunks by file path. The Orchestrator will fetch those chunks directly from the vector store and re‑prompt the Oracle.

### 2.3 Orchestrator

**Purpose:** The central nervous system that coordinates all phases and enforces the NARE revision loop.

**Responsibilities:**
- Maintain a task queue (`pending`, `running`, `done`, `blocked`).
- Invoke the Oracle to generate initial TDBs.
- For each Tela Task:
  - Spin up a fresh Actuator sandbox (ephemeral container or temporary directory).
  - Provide the sandbox with the exact file contents referenced in the TDB (fetched from the vector store using the `hash_before`).
  - Feed the TDB to the Actuator and await `<END_OF_IMPLEMENTATION>`.
  - Parse the Verification Block; if all checks pass, persist the mutated files to a staging area and mark the task `done`.
  - If verification fails, feed the failure log back to the Oracle for revision (up to 3 attempts).
- After all tasks are `done`, trigger the Compliance Report generation.
- **Optimistic Rebase:** Before committing, check if the live repository `HEAD` has diverged from the snapshot. If so, attempt to apply the mutation scripts to the new `HEAD` and re‑verify. If successful, continue; if not, alert the Master Builder.

### 2.4 Actuator

**Purpose:** Execute a single Tela Task in a deterministic, sandboxed environment.

**Implementation Options:**
- **Lightweight:** A Python subprocess that runs inside a temporary directory, using `subprocess.run()` to execute the mutation script and verification commands.
- **Containerized:** A Docker container with the target language toolchain pre‑installed, offering stronger isolation.

**Input:** A Tela Task YAML file and the exact file contents (by `hash_before`).

**Execution Loop:**
1. Write the target files to the sandbox.
2. Verify `hash_before` matches.
3. Write the mutation script (from `surgery.script_content`).
4. Execute the script.
5. Run all `verification` commands.
6. Capture `stdout`/`stderr` into a Verification Block.
7. Output `<END_OF_IMPLEMENTATION>`.

**Security Gate:** Before executing the mutation script, the Actuator runs a static analysis scanner (e.g., Bandit for Python, Semgrep for multiple languages) on the script itself. If any `BLOCKER` severity issues are found, the task is aborted and reported.

### 2.5 Compliance Dashboard / CLI

**Purpose:** Provide the Master Builder with a human‑reviewable summary before commit.

**Features:**
- List of all tasks with pass/fail status.
- Inline diff view of each file change (computed from `hash_before` and `hash_after`).
- Full Verification Block logs (collapsible).
- One‑click "Approve & Commit" button (or CLI command) that stages and commits the changes with a generated message linking to the Tela Task IDs.

## 3. Data Flow Diagram (Text)

```
[Repository] → Snapshot Indexer → Vector Store (chunks + embeddings)
                                           ↑
                                           | query
[Intent] → Oracle (Planner) ←──────────────┘
                ↓
         Tela Tasks (YAML)
                ↓
         Orchestrator (Queue)
                ↓
      ┌─────────┴─────────┐
      ↓                   ↓
  Actuator Sandbox    Actuator Sandbox
  (Task 1)            (Task N)
      ↓                   ↓
  Verification Block  Verification Block
      └─────────┬─────────┘
                ↓
         Compliance Report
                ↓
         Master Builder Review → Git Commit
```

## 4. Adherence to NARE Protocol Guarantees

| Guarantee | How It's Preserved |
| :--- | :--- |
| **Air‑Gap** | Planner never writes code; Actuator starts fresh from snapshot chunks each time. |
| **Determinism** | All mutations via scripts; `hash_before` verification ensures correct starting state. |
| **Auditability** | Every TDB, chunk hash, and verification log is stored and linked to the final commit. |
| **Language Agnostic** | Indexer and Actuator use language‑specific parsers (tree‑sitter) but the overall pipeline is generic. |
| **Model Agnostic** | Oracle can be swapped with any LLM that supports function calling / YAML output. |

## 5. Handling Repository Scale

- **Small repos (< 2M tokens):** The entire snapshot can still be fed directly to the Oracle (bypassing retrieval) if preferred.
- **Large repos (> 2M tokens):** Retrieval mode is mandatory. The vector store scales horizontally; chunks are loaded on‑demand.
- **Massive repos (Linux kernel scale):** The Snapshot Indexer may be run per‑subsystem (e.g., `drivers/`, `kernel/`) to keep the vector store manageable. The Oracle can query across multiple collections.

## 6. Example Workflow (End‑to‑End)

1. **Setup:**
   ```bash
   nare index --repo /path/to/monorepo --output ./snapshots/2026-04-13
   ```
2. **Intent Submission:**
   ```bash
   nare plan --intent "Refactor authentication module to use Argon2 instead of bcrypt" \
             --snapshot ./snapshots/2026-04-13
   ```
   (Oracle queries vector store, generates `task.001.tela.yaml`, `task.002.tela.yaml`, ...)
3. **Execution:**
   ```bash
   nare execute --tasks ./tela/
   ```
   (Orchestrator processes tasks sequentially, outputs verification logs to `./proofs/`)
4. **Review & Commit:**
   ```bash
   nare report --open
   # Master Builder reviews diffs and verification blocks
   nare commit --approve
   ```

## 7. Next Steps (MVP Scope)

To move from specification to a working prototype, the following are recommended:

1. **Snapshot Indexer MVP:** Python script that chunks Python files using `ast` and stores embeddings in Chroma.
2. **Actuator MVP:** Python sandbox that accepts a single Tela Task and executes it locally (no container).
3. **Oracle Integration:** A simple script that calls the Gemini API with the Oracle system prompt and a few retrieved chunks.
4. **End‑to‑End Demo:** Refactor a single function in a 10‑file Python project using the full NARE loop.

## 8. Future Extensions (Post‑MVP)

- **Multi‑language support** via tree‑sitter grammars.
- **Remote Actuator** for executing tasks on cloud‑based sandboxes (e.g., Fly.io Machines).
- **Continuous NARE:** A GitHub Action that runs the NARE loop on every PR labeled `nare/refactor`.
- **Human‑in‑the‑loop fallback:** If a task fails after 3 revisions, the Orchestrator can open a draft PR with the current state and request manual guidance.

---

*This outline represents a practical path toward a fully autonomous, specification‑first refactoring system that scales to any repository size. It preserves the rigorous guarantees of the NARE protocol while removing the context‑window bottleneck identified in the validation critique.*
```
If you'd like, I can drill down into any specific component (e.g., the chunking strategy, the retrieval prompt template, or the Orchestrator state machine) with more detailed pseudo‑code.
