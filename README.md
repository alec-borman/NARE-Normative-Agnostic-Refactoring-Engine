# NARE: Normative Agnostic Refactoring Engine

[![Specification](https://img.shields.io/badge/Spec-v1.2.0-blue)](./SPEC.md)
[![Status](https://img.shields.io/badge/Status-Design%20%2F%20Discussion-yellow)](./DISCUSSION.md)
[![License](https://img.shields.io/badge/License-Apache%202.0-green)](./LICENSE)

**A deterministic, specification‑first protocol for autonomous code refactoring in legacy environments.**

---

## 📖 Overview

NARE (Normative Agnostic Refactoring Engine) is an open protocol that establishes a verifiable boundary between *intent* and *execution* when using Large Language Models (LLMs) to modify software codebases.

Traditional AI coding assistants operate probabilistically—they generate code patches directly, which can lead to semantic drift, hallucinated dependencies, and unreproducible mutations. NARE replaces this stochastic workflow with a **compiled, deterministic pipeline**:

1. **Plan**: Human intent is decomposed into machine‑readable YAML task definitions.
2. **Actuate**: A sandboxed executor applies mutations via deterministic scripts, not LLM‑generated diffs.
3. **Verify**: Cryptographic hashes, AST checks, and compiler validation guarantee that only intended changes are applied.

The protocol is **model‑agnostic** and **language‑agnostic**; it defines *how* autonomous refactoring should be orchestrated, not which AI provider or toolchain to use.

---

## 🧠 Core Principles

| Principle | Description |
| :--- | :--- |
| **Air‑Gap** | Strict separation between the planning phase (LLM) and execution phase (sandboxed actuator). |
| **Canonical Snapshot** | All tasks start from an immutable, version‑controlled view of the codebase. |
| **Deterministic Scripts** | Mutations are expressed as executable scripts, not probabilistic text patches. |
| **Tela Tasks** | Refactoring intent is atomized into verifiable, self‑contained units of work. |
| **Master Builder Oath** | Humans remain the final arbiter; the protocol supports evaluation, not blind automation. |

---

## 📁 Repository Contents

This repository currently hosts the **normative specification and technical validation** of the NARE protocol. It does **not** contain a reference implementation (yet).

- **[`SPEC.md`](./SPEC.md)** – The formal NARE v1.2.0 protocol definition, including YAML schemas, phase workflows, and security requirements.
- **[`ANALYSIS.md`](./ANALYSIS.md)** – A comprehensive critique covering theoretical foundations, long‑context behavior, verification mechanics, and comparative agent analysis.
- **[`DISCUSSION.md`](./DISCUSSION.md)** – Open issues, limitations, and proposed enhancements for v1.3.0.

Future milestones may include:
- Reference implementation of the Actuator (Rust / Python)
- Planner prompt templates for popular LLMs
- CI/CD integration examples

---

## 🚀 Why NARE?

- **Reproducibility**: Every mutation is fully auditable and repeatable, down to the exact file hash.
- **Security**: Sandboxed script execution eliminates direct prompt injection risks and prevents "vibe‑coding" vulnerabilities.
- **Scale‑Ready**: The atomic Tela Task model mitigates context window degradation, allowing operation on codebases up to ~2M tokens without architectural changes.
- **Vendor Neutral**: Works with any foundation model and any static analysis toolchain.

---

## 📚 Quick Start (Reading the Spec)

If you are new to NARE, we recommend starting with the **Overview** section of [`SPEC.md`](./SPEC.md) and then reviewing the **Phase‑by‑Phase Workflow** (§4).

For a deeper understanding of the protocol's rationale and performance characteristics, see [`ANALYSIS.md`](./ANALYSIS.md).

---

## 🤝 Contributing & Discussion

The NARE protocol is an open specification under active discussion. We welcome feedback, use‑case reports, and proposals for future versions.

- **Discussions**: Please open a [GitHub Discussion](https://github.com/your-org/nare/discussions) for questions, ideas, or experience reports.
- **Spec Amendments**: For substantive changes to the protocol, please open an Issue first to discuss the motivation and impact.

This repository follows a **specification‑first** development model. Code contributions will be accepted once a reference implementation roadmap is established.

---

## 📄 License

The NARE specification and accompanying documents are licensed under the [Apache License 2.0](./LICENSE).

---

*“The machines handle the mechanics, but the human remains the final arbiter of intent.”*
