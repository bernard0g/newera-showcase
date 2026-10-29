<div align="center">

# NEW ERA

**An AI agent for software development and defensive security. The model decides; the server authorizes.**

**English** · [Português](README.pt-BR.md)

![Python](https://img.shields.io/badge/Python-FastAPI-009688?logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-SQLAlchemy%202-4169E1?logo=postgresql&logoColor=white)
![React](https://img.shields.io/badge/React%2019-TypeScript-61DAFB?logo=react&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Tools](https://img.shields.io/badge/agent%20tools-36-1F6FEB)
![Tests](https://img.shields.io/badge/tests-458%20passed-2EA043)

![NEW ERA Command Center](assets/command-center.png)

</div>

> **Showcase repository.** The source code is private. I'm happy to walk through it live or share access with a recruiter on request: [brnrdcontact@gmail.com](mailto:brnrdcontact@gmail.com).

You talk to NEW ERA in plain language. It plans, calls typed tools that the server validates, and answers with the evidence behind each result. Investigations keep a full trace of every observation and finding.

## Contents

[Two interfaces](#two-interfaces-one-session) · [What it can do](#what-the-agent-can-do) · [Design](#design-the-model-decides-the-server-authorizes) · [Security engineering](#security-engineering) · [Guardrail Lab](#guardrail-lab-ai-safety-evaluation) · [Architecture](#architecture) · [Validation](#validation)

## Two interfaces, one session

The web **Command Center** and the `newera` **CLI** share the same saved conversations: start in the terminal, continue in the browser, and the other side sees the same history, tool calls and results.

<p align="center">
  <img src="assets/cli.png" alt="NEW ERA terminal CLI" width="720">
</p>

## What the agent can do

The agent works through a single **server-side registry of 36 typed tools**. The model chooses a tool; the server checks every argument, permission and scope before anything runs.

| Area | Examples |
|---|---|
| **Code work** | Navigate and search a project, read `git status` / `git diff`, run the project's own tests in a bounded sandbox (scrubbed environment, resource limits, timeout) |
| **Research** | Web search and page reading through an SSRF-guarded fetcher; fetched content is treated as untrusted data |
| **Planning** | Explicit step/time budget per turn, `think` notes, skills loaded on demand, and `delegate` to a subagent with a clean context and its own budget |
| **Log analysis** | auth.log/syslog, Apache/Nginx, JSON, CSV and Windows event exports; 15 deterministic detections (brute force with time windows, password spraying, success after failures, web injection payloads, log clearing…) mapped to MITRE ATT&CK, with suggested Sigma rules and the triggering lines cited |
| **Vulnerability intelligence** | `requirements.txt`, `package.json`, lockfiles or CycloneDX SBOMs checked against OSV.dev, CISA KEV and NIST NVD, prioritized, with fixed versions; *potential* kept separate from *confirmed* |
| **Asset posture** | For domains you own and register: DNS, SPF/DMARC and RDAP passively; with an explicit active scope, a TLS check and a bounded website configuration review (security headers, cookie flags, CORS, mixed content, outdated JS libraries) |
| **File triage** | Hashes, real type from magic bytes, extension mismatch, entropy, PE and PDF active-content flags, extracted indicators. Files are analyzed statically and never executed |
| **Correlation and knowledge** | Indicators that show up in more than one artifact; answers from a curated ATT&CK / OWASP / NIST CSF knowledge base with sources |

Every investigation produces a persisted trace, evidence-linked findings, an ATT&CK table, a risk score, a coverage ledger (what was assessed versus what needs follow-up), explicit limitations and a JSON/Markdown report.

## Design: the model decides, the server authorizes

- **No permission lives in the LLM.** Tool arguments, scopes and limits are enforced by server code, so a confused or manipulated model can't escalate. There is no shell tool, and unknown tool names are rejected.
- **Deterministic where it must be right.** Detection, scoring and authorization are ordinary tested code. The model writes the narrative over findings that were already validated.
- **Provider-agnostic.** Claude, OpenAI-compatible APIs, Gemini, and local models (Ollama, LM Studio). Without a model, the deterministic workflows and CLI commands still work.
- **Errors teach instead of crash.** A recoverable tool failure goes back to the model as a structured hint, and the turn continues.
- **Authorization is explicit.** Passive lookups are the default. Any active check needs a registered asset with an attested, revocable scope, reloaded before every active request.

```
message ─▶ agent turn (budget) ─▶ tool call ─▶ server-side validation ─▶ tool ─▶ evidence ─▶ answer / report
                     └─▶ delegate ─▶ subagent (clean context, own budget)
```

## Security engineering

NEW ERA ingests untrusted data, talks to the internet and may hand data to language models, which makes the platform itself a target. It has a written **threat model (STRIDE + LLM-specific threats)** where each mitigation points to the tests that verify it. Highlights:

| Threat | Mitigation |
|---|---|
| Prompt injection through evidence | Evidence never reaches the LLM raw: the narrative step gets redacted summaries inside randomized fences, has no tools, and can't change plans, scopes or approvals. Instruction-like text in evidence is reported as a finding. |
| LLM hallucination | A post-generation guard removes sentences that cite indicators absent from the findings; the narrative is labeled as AI inference. |
| Data leaving to external LLMs | Per-investigation approval for external providers, secrets redacted first; local and offline modes never leave the host. |
| SSRF and DNS rebinding | Hostnames normalized, only public IPs allowed, resolve once and connect to the pinned IP. |
| Broken access control | Every investigation, asset, approval and evidence lookup checks ownership; foreign IDs return 404. |
| Credentials and sessions | scrypt password hashing with constant-time checks, login rate limit, session tokens stored as HMAC hashes, provider keys encrypted at rest. |
| Malicious files | Stored under random 128-bit keys, type from magic bytes, static parsers only, never executed or served back. |
| Web and supply chain | Strict CSP and `X-Frame-Options: DENY`; lockfiles, `pip-audit` and `npm audit` in CI; no LangChain/LiteLLM. |

## Guardrail Lab (AI safety evaluation)

A research workspace for measuring how robust LLM guardrails are. It records each experiment with the exact prompt strategy and model response, and recommends the next distinct experiment from what was observed. It never claims a bypass from a single answer: apparent results must be repeated in fresh sessions and reviewed by a person. The goal is reproducible evaluation from the defensive side.

## Architecture

| Layer | Stack |
|---|---|
| API | Python 3.12 · FastAPI · Pydantic v2 · SQLAlchemy 2 (async) · Alembic · httpx |
| AI | Anthropic, OpenAI-compatible (OpenAI, Gemini, LM Studio), Ollama; offline mode |
| Data | PostgreSQL 17 (SQLite in dev and tests); evidence on disk under random keys |
| Web | React 19 · TypeScript · Vite · Tailwind CSS 4 · live Server-Sent Events |
| Ops | Docker Compose · nginx with strict CSP · GitHub Actions (lint, tests, Postgres smoke test, audits, image builds) |

## Validation

Backend test suite (pytest): **458 passed** on 2026-10-27, Windows. The 6 tests that reach real external services are excluded by default (`-m 'not network'`).

## How I used AI

Built with Claude Code. I defined the scope, the authorization model, the threat model, the tool contracts and the workflows, and I review and test the results.

## Status and limitations

- Personal project in active development; not audited by a third party.
- Detections are rule-based: they catch known patterns and miss novel ones.
- Security features are for your own systems or with written permission.

---

<sub>Made by <a href="https://www.bernardogomes.dev.br">Bernardo Gomes</a> · Software Engineering @ FIAP</sub>
