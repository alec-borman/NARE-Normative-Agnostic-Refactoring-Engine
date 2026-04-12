# NARE: Normative Agnostic Refactoring Engine

**Version:** 1.2.0 — Agnostic Normative Specification  
**Status:** Final  
**Classification:** Autonomous CI/CD Protocol for Deterministic LLM-Based Code Actuation  
**Date:** 2026-04-12  
**Author:** Alec Borman, Master Builder

---

## Abstract

NARE (Normative Agnostic Refactoring Engine) is a deterministic protocol for transforming high‑level human intent into verified, production‑ready code mutations within arbitrarily large legacy codebases. It resolves the fundamental failure modes of Large Language Model (LLM) code generation—context drift, hallucination, lazy stubbing, and syntactic corruption—by establishing a mathematically verifiable boundary between *intent definition* and *mechanical actuation*. This specification defines the roles, artifacts, workflows, and verification requirements in a manner that is **agnostic to specific LLM providers, agent implementations, and tooling choices**. Any long‑context reasoning engine capable of processing a structured snapshot and generating YAML Test‑Driven Boundaries may serve as the Planner. Any agent capable of executing terminal commands in a sandboxed environment may serve as the Actuator. The protocol reduces the human operator's active involvement to articulating intent, providing a codebase snapshot, and reviewing a final compliance report.

---

## 1. Terminology and Core Concepts

### 1.1 Master Builder
The human operator responsible for defining the high‑level *Intent* and providing the initial *Canonical Snapshot*. The Master Builder writes no implementation code, debugs no compiler errors, and never manually edits source files within the scope of an active NARE sprint. Their sole active duties are intent articulation and final compliance sign‑off.

### 1.2 Intent
A plain‑English statement describing a desired architectural change, feature addition, or refactoring goal. The Intent must be scoped such that it can be decomposed into a finite sequence of verifiable, atomic tasks.

**Example:**
> *"Refactor the WebAudio graph builder to use a connection pool pattern. Maintain all existing public APIs and ensure the unit test suite passes with zero regressions."*

### 1.3 Canonical Snapshot
A single, structured file containing the complete, unmodified source tree of the target codebase at the start of a sprint. The snapshot format **must** be parseable by the Planner and reconstructable by the Actuator. The recommended format is a machine‑readable structured document (e.g., XML, JSON) that wraps each source file's path and content in an unambiguous hierarchy. The snapshot **must** exclude build artifacts, dependency directories, version control metadata, and other non‑source files. It serves as the immutable ground truth against which all mutations are measured.

### 1.4 Planner
A long‑context reasoning engine (LLM) that receives the *Intent* and the *Canonical Snapshot*. The Planner **must** be capable of:
- Ingesting the entire snapshot (or a semantically compressed representation thereof) within a single context window.
- Analyzing the codebase to understand relevant architecture and dependencies.
- Decomposing the Intent into an ordered sequence of atomic, verifiable *Tela Tasks*.
- Generating, for each task, a strict YAML *Test‑Driven Boundary (TDB)* conforming to the NARE Task Schema (§3).
- Outputting the sequence as discrete, machine‑readable YAML blocks.

The Planner **must not** have write access to the live repository or the ability to execute arbitrary code. Its output is limited to structured specifications.

### 1.5 Tela Task
An atomic unit of work defined by a YAML TDB. A Tela Task specifies exactly:
- The target file(s) to mutate.
- Positive and negative boundary constraints.
- The required verification commands (AST parse, unit tests, compiler check).
- The method of mutation (deterministic script or patch).

### 1.6 Actuator
A local or remote agent that executes a Tela Task autonomously. The Actuator **must** be capable of:
- Parsing the Canonical Snapshot to reconstruct the exact file tree in a temporary, sandboxed workspace.
- Reading a YAML TDB and verifying the `hash_before` of all target files against the snapshot content.
- Generating and executing a deterministic mutation script (in a language of the project's ecosystem) to transform the target files.
- Executing the verification commands defined in the TDB within the sandboxed workspace.
- Capturing the complete `stdout`/`stderr` of the verification run.
- Outputting a structured *Verification Block* and a termination marker.

The Actuator **must not** use probabilistic file‑editing tools (e.g., LLM‑generated patches applied directly). All mutations **must** be performed via the execution of a deterministic script.

### 1.7 Verification Block
A fenced block (`<verification>...</verification>`) containing the raw output of all verification commands executed by the Actuator. The Verification Block serves as cryptographic proof that the mutation satisfied the TDB constraints. It is the only artifact reviewed by the Master Builder prior to commit.

### 1.8 Compliance Report
A final summary generated by the Planner after all Tela Tasks in a sequence have been executed and verified. It contains:
- A list of all tasks and their status (PASS/FAIL).
- The full Verification Block for each task.
- A diff summary of all changes made.
- A final assertion that the Intent has been satisfied.

---

## 2. Architectural Philosophy

### 2.1 The Autoregressive Degradation Problem
LLMs are autoregressive next‑token predictors. When allowed to read their own output as input (the "context loop"), small errors compound into catastrophic drift—hallucinated syntax, invented APIs, and silent logic corruption. Standard AI‑assisted development treats the LLM as a co‑pilot operating directly on the source tree, creating an unbounded contamination surface.

### 2.2 The Air‑Gap Principle
NARE eliminates the contamination loop by enforcing a strict air‑gap:
- The Planner **never writes code**. It writes *specifications* (YAML TDBs).
- The Actuator **never sees its own previous output**. It starts each task from the immutable Canonical Snapshot.
- The Actuator **never edits files directly**. It writes and executes a deterministic mutation script whose output is a *new* file state, not a probabilistic diff.

### 2.3 Deterministic Verification
Every mutation is validated by two independent, non‑LLM judges:
1. **AST Parser** — guarantees syntactic well‑formedness.
2. **Strict Compiler / Type Checker** — guarantees semantic coherence.

A Tela Task is considered *proven* only when both judges return exit code `0`. Behavioral correctness is additionally validated by the project's test suite, which **should** be invoked as part of verification.

### 2.4 The Master Builder's Contract
The Master Builder agrees to:
- Never manually edit files touched by an active Tela Task.
- Never intervene in the autonomous execution loop.
- Trust the Verification Block over any conversational prose from the Actuator.
- Commit changes only after reviewing the final Compliance Report.

In exchange, the system guarantees that all committed mutations are syntactically valid, compiler‑clean, and verifiably derived from the stated Intent.

---

## 3. NARE Task Schema (YAML TDB v4.1.0)

Every Tela Task generated by the Planner **must** strictly conform to the following YAML schema. No free text, no markdown formatting outside of explicit fields.

```yaml
# task.<task_id>.tela.yaml
version: "4.1.0"
task_id: string                    # Unique identifier, e.g., "refactor-001"
created: datetime                  # ISO 8601, UTC
master_builder: string             # Identifier of the human operator
intent_summary: string             # One-line summary of parent Intent

target:
  repository: string               # Name of the codebase
  files:
    - path: string                 # Relative path from repo root
      hash_before: string          # SHA-256 of file BEFORE mutation (populated by Actuator)

boundary:
  must:
    - string                       # Positive constraint
  must_not:
    - string                       # Negative constraint

verification:
  ast_check:
    command: string                # e.g., "tree-sitter parse --quiet {file}"
    expected_exit: 0
  compiler_check:
    command: string                # e.g., "cargo check --message-format=short"
    expected_exit: 0
  behavioral_checks:               # Optional but strongly recommended
    - name: string
      command: string              # e.g., "pytest tests/test_graph.py"
      expected_exit: 0
  additional_checks:               # Optional
    - name: string
      command: string
      expected_exit: integer

surgery:
  method: "deterministic_script" | "patch_file"
  script_language: string          # e.g., "python", "rust", "typescript", "bash"
  script_path: string              # Path where Actuator will write the mutation script
  patch_content: string            # Unified diff (only if method = patch_file and diff < 50 lines)

expected_output:
  files_changed:
    - path: string
      hash_after: string           # Expected SHA-256 AFTER mutation (optional)
```

### 3.1 Schema Constraints
- **`hash_before`** must be populated by the Actuator from the Canonical Snapshot before any mutation. If the computed hash does not match, the Actuator aborts and reports divergence.
- **`ast_check.command`** must include the `{file}` placeholder, which the Actuator expands to each target file path.
- **`surgery.method`** must be `deterministic_script` for any mutation affecting more than 50 lines or involving complex AST restructuring. `patch_file` is permitted only for trivial, single‑line changes.
- **`behavioral_checks`** should be included whenever the project has an existing test suite. This is the primary defense against silent logic errors.

---

## 4. The Autonomous Execution Pipeline

### 4.1 Phase 0: Snapshot and Intent Capture (Human)
The Master Builder generates a Canonical Snapshot of the codebase using a tool that produces a structured, machine‑readable document (e.g., XML, JSON). The snapshot is saved to a known location within the repository (e.g., `./tela/snapshots/`).

The Master Builder then invokes the Planner with:
- The Canonical Snapshot (uploaded or referenced).
- The plain‑English Intent.
- Optionally, a system prompt that instructs the Planner to act as the NARE Planner and adhere to the Task Schema.

### 4.2 Phase 1: Planning (Planner)
The Planner analyzes the snapshot and Intent, then outputs an ordered sequence of YAML TDBs. Each TDB **must** be enclosed in a ` ```yaml ` code fence or delivered as a separate file.

### 4.3 Phase 2: Task Ingestion
The YAML TDBs are placed in a designated directory (e.g., `./tela/intent/`). A daemon or orchestration script monitors this directory and processes tasks sequentially.

### 4.4 Phase 3: Autonomous Actuation
For each Tela Task, the Actuator:
1. Reconstructs the file tree from the Canonical Snapshot into a temporary workspace.
2. Verifies `hash_before` for all target files.
3. Writes a deterministic mutation script to the specified `script_path`.
4. Executes the script within the temporary workspace, mutating the target files.
5. Runs all verification commands defined in the TDB.
6. Captures the complete verification output and places it inside `<verification>...</verification>`.
7. Emits exactly `<END_OF_IMPLEMENTATION>` as the final transmission.

### 4.5 Phase 4: Proof Capture and Feedback
The orchestration layer captures the Actuator's output and writes it to `./tela/proofs/<task_id>.proof.log`. It parses the log for:
- Presence of `<END_OF_IMPLEMENTATION>`.
- All verification commands returning `exit: 0`.

If all checks pass, the mutated files are applied to the working copy and the task is marked `DONE`. If any verification fails, the failure log is fed back to the Planner with a request to revise the TDB. The revision loop is capped at a maximum of 3 attempts per task.

### 4.6 Phase 5: Compliance and Commit
After all tasks are `DONE`, the Master Builder reviews the Compliance Report generated by the Planner. The report includes a summary of changes and the Verification Blocks. Upon approval, the Master Builder commits the mutated source tree to version control.

---

## 5. Implementation Requirements (Agnostic)

### 5.1 Planner Requirements
- **Context Capacity:** Must accommodate the entire Canonical Snapshot (or a semantically compressed equivalent) within a single context window.
- **Output Format:** Must produce YAML TDBs conforming to the NARE Task Schema.
- **Interface:** Must be invokable programmatically (API, CLI, or file‑based triggering) to enable integration with the orchestration layer.

### 5.2 Actuator Requirements
- **Sandboxing:** Must execute mutations in an isolated, temporary workspace that does not affect the live repository until verification passes.
- **Tool Restrictions:** Must be configurable to disable direct file‑editing tools (e.g., Edit, Write) and rely solely on terminal command execution.
- **Deterministic Scripting:** Must be capable of generating and executing mutation scripts in the target project's ecosystem.

### 5.3 Snapshot Tool Requirements
- **Output Format:** Must produce a structured, machine‑readable document that preserves file paths and content without ambiguity.
- **Exclusion:** Must respect ignore patterns to exclude build artifacts, dependencies, and version control metadata.

### 5.4 Verification Tool Requirements
- **AST Parser:** A parser for the target language(s) that can validate syntactic correctness (e.g., `tree-sitter`).
- **Compiler/Type Checker:** The native strict compiler or type checker for the project.
- **Test Runner:** The project's test suite runner (e.g., `pytest`, `cargo test`, `jest`).

---

## 6. Safety, Limitations, and Failure Modes

### 6.1 Protocol Guarantees
- **Syntactic Correctness:** Every mutated file is guaranteed to parse without syntax errors.
- **Compilation Success:** Every mutated file is guaranteed to pass the project's strict compiler/type checker.
- **Auditability:** Every mutation is accompanied by a cryptographically verifiable hash and a full verification log.

### 6.2 Protocol Limitations
- **Logical Correctness:** NARE guarantees structural integrity, not business logic correctness. Comprehensive behavioral test coverage is a prerequisite for semantic safety.
- **Intent Ambiguity:** Vague Intents may result in TDBs that diverge from the Master Builder's true goals. The Compliance Report review is the final human checkpoint.
- **Snapshot Staleness:** The Canonical Snapshot is a point‑in‑time artifact. If the live repository diverges significantly during a sprint, `hash_before` mismatches will cause tasks to abort.

### 6.3 Failure Recovery
- **Single Task Failure:** The orchestration layer halts the sequence and requests Planner revision. The Master Builder never manually fixes code.
- **Catastrophic Drift (Planner Hallucination):** If the Planner generates a TDB targeting a non‑existent file, the `hash_before` check fails before any mutation occurs.
- **Infinite Loop Prevention:** Maximum 3 revision attempts per task. After 3 failures, the task is marked `BLOCKED` and the Master Builder is notified.

---

## 7. The Master Builder's Oath

> *I will not write implementation logic within the scope of an active NARE sprint.*  
> *I will generate a clean Canonical Snapshot and provide a clear Intent.*  
> *I will trust the Verification Block, not the Actuator's prose.*  
> *I will feed failure logs back to the Planner, not to the code editor.*  
> *I will review the Compliance Report and lock the build only when behavioral tests pass.*  
> *I am the Master Builder. The machines handle the rest.*

---

**End of Agnostic Normative Specification**

*NARE v1.2.0 — A protocol for those who build systems that build systems.*
