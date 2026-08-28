# AI Agent Advice — Collected Notes

A running collection of tips and practices shared by a colleague regarding the use of AI agents in development workflows.

---

## 1. Use `.gitignore` to preserve local working files across iterations

**Context:** Working with AI agents during development, across multiple iterations.

**Advice:** He uses `.gitignore` to keep certain folders and files **local only** (not committed to the repository), so they persist on disk and he can continue working with them in later sessions, even though they aren't part of the tracked codebase.

**Example use case:** At the end of a development iteration — which he refers to as an **"analysis"** — he collects the development status of tasks into local files. These aren't meant to be committed, but they stick around locally as a working log/reference he can pick back up later.

**Why it's useful:**

- Keeps scratch/status files out of version control (avoids clutter in commits, PRs, diffs).
- Preserves continuity between sessions/iterations without polluting the shared repo history.
- Effectively creates a "local memory" layer for agent-assisted work that survives context resets.
- **Reduces token usage:** since the status/analysis is already captured in these local files, the agent doesn't need to re-read other, longer source documents again to regain context — it can just reference the shorter local summary instead.

---