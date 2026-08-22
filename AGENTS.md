# Agent instructions (Codex / any AGENTS.md-reading agent)

This repository is an agent skill for testing the Apache HugeGraph Helm
chart and landing reviewed chart fixes.

When a task involves testing, validating, hardening, or fixing the HugeGraph
Helm chart (or diagnosing a HugeGraph deployment on Kubernetes/Kind —
including PD raft, IpAuthHandler/allowlist behavior, install failures,
upgrade testing, or chart review):

1. Read `SKILL.md` in this directory and follow it as your operating
   procedure. It is the entry point; do not improvise a test plan when the
   skill defines one.
2. Start with its **Intake** questions to the user and create the **state
   file** it prescribes before running anything.
3. Open files under `references/` exactly when SKILL.md points you to them
   (`test-suite.md`, `ipauth-bug-family.md`, `chart-engineering.md`,
   `fix-loop.md`, `pitfalls.md`, `evidence.md`) — they are the detailed
   playbooks; `pitfalls.md` is worth skimming before writing any test
   script.
4. Honor the hard rules in SKILL.md: never modify the chart to make a test
   pass, never push or mutate anything remote without the user's explicit
   per-session approval, tear down only session-labelled resources, and
   never report a result that has no backing file in the session's
   `findings/`.
