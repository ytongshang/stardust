---
title: "Harness Engineering for Coding Agent Users"
author: Birgitta Böckeler (Thoughtworks), published on martinfowler.com
source_url: https://martinfowler.com/articles/harness-engineering.html
fetched: 2026-04-19
tags: [agents, harness, martin-fowler, guides-sensors, thoughtworks]
---

# Harness Engineering for Coding Agent Users

## Core Concept

"Agent = Model + Harness" — the harness encompasses everything surrounding the model itself.

## Central Framework: Guides and Sensors

**Guides (Feedforward Controls)**
- Anticipate agent behavior before it acts
- Steer the coding agent toward good results on the first attempt
- Examples: architectural documentation, coding conventions, bootstrap scripts, code modification tools

**Sensors (Feedback Controls)**
- Observe after the agent acts
- Enable self-correction through feedback loops
- Examples: static analysis tools, test suites, code review agents, structural tests

Critical insight: systems relying only on feedback repeat mistakes; those with only feedforward guides never validate effectiveness.

## Computational vs. Inferential Execution

**Computational Controls**
- Deterministic, CPU-based
- Fast (milliseconds to seconds)
- Examples: linters, type checkers, structural tests
- Reliable but limited to rule-based analysis

**Inferential Controls**
- Semantic analysis using AI models
- Slower and more expensive (GPU/NPU-dependent)
- Examples: AI code review, LLM-based judgment
- Non-deterministic but semantically richer

## The Steering Loop

Humans continuously improve the harness by iterating on both feedforward and feedback mechanisms. When issues recur, controls should evolve to prevent future occurrences. Coding agents themselves can help build custom controls — writing structural tests, generating rules, scaffolding linters, or creating documentation guides.

## Three Regulation Categories

### 1. Maintainability Harness

Regulates internal code quality:
- Computational sensors: duplicate code, cyclomatic complexity, test coverage, style violations
- Inferential sensors: semantic duplication, redundant tests, overengineering

**Limitation**: neither type reliably catches misdiagnosis, feature scope creep, or instruction misunderstandings.

### 2. Architecture Fitness Harness

Enforces architectural characteristics:
- Performance requirements feeding forward; performance tests feeding back
- Observability standards (logging conventions) with feedback mechanisms
- Based on fitness function principles

### 3. Behaviour Harness

**Currently the weakest area** — functional correctness verification remains problematic:
- Feedforward: functional specifications (varying detail levels)
- Feedback: AI-generated test suites with coverage metrics
- Limitation: over-reliance on agent-generated tests

The article acknowledges "approved fixtures" pattern shows promise but isn't universally applicable.

## Timing: Keep Quality Left

Controls distribute across the development lifecycle based on cost and speed:

**Pre-integration (before commit)**
- Fast checks: linters, basic test suites, initial code review agents
- Language servers, architecture documentation, how-to guides

**Post-integration (CI/CD pipeline)**
- More expensive: mutation testing, comprehensive code reviews
- Architecture reviews considering broader system context

**Continuous Monitoring**
- Drift detection: dead code analysis, coverage quality, dependency scanning
- Runtime feedback: SLO degradation, response quality sampling, anomaly detection

## Key Concepts

**Harnessability**: The degree to which a codebase supports control implementation. Strongly-typed languages, clear module boundaries, and opinionated frameworks improve harnessability.

**Ambient Affordances**: Structural properties of the agent environment making it "legible, navigable, and tractable" — termed by colleague Ned Letcher. Technology choices, architectural patterns, and framework selection that implicitly increase agent success probability.

**Ashby's Law of Requisite Variety**: A regulator requires at least as much variety as the system it governs. Committing to defined service topologies reduces the variety a harness must handle.

## Harness Templates

Evolving service templates (common architectural topologies) into **harness templates** — bundled guides and sensors specific to architecture patterns:
- Business services exposing data via APIs
- Event processing services
- Data dashboards

## Real-World Examples Referenced

- **OpenAI's harness**: Layered architecture enforced by custom linters, structural tests, and recurring "garbage collection" scans for drift
- **Stripe's minions**: Pre-push hooks running heuristic-selected linters, emphasis on shifting feedback left, blueprint-integrated feedback sensors
- **Thoughtworks teams**: Tackling architecture drift through mixed computational/inferential sensors; API quality improvements using agents with custom linters; "janitor army" code quality initiatives

## Outstanding Challenges

1. **Behavioral verification** — How to gain sufficient confidence in functional correctness?
2. **Harness coherence** — Maintaining synchronization among growing guide and sensor systems
3. **Trade-off management** — How agents handle conflicting instructions and signals
4. **Coverage quality** — Evaluating harness adequacy (similar to mutation testing for test suites)
5. **Systemic tooling** — Unified approaches for configuring and reasoning about scattered controls
6. **Entropic drift** — Keeping harnesses coherent as they grow and evolve

## Conclusion

Harness engineering is an emerging engineering practice rather than a one-time configuration task. Success requires strategic system-level thinking about control mechanisms. The vision: building confidence in agent output sufficient to reduce supervision while directing human expertise toward irreplaceable judgment calls about business alignment.
