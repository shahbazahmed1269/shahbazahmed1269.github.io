---
layout: post
title: "The 120-Second Swarm: How Claude Code Ate My 5-Hour Session Quota"
date: 2026-06-16 17:00:00 +0530
categories: [Engineering, AI]
tags: [Claude Code, Anthropic, LLMOps, Post-Mortem]
---

Software has always had its classic failure modes — infinite loops and recursive calls. We learn to spot them. But agentic development has introduced an entirely new kind of failure modes. I am calling this particular variant the **Parallel Agent Fan-Out Explosion**.

I was exploring AI agents in `claude-code` when it asked for permission to use the deep-research tool. Seemed reasonable. I approved it just for the current session and turned back to my other work. The session was dead before I looked up. My 5-hour Session Quota exhausted.

When I checked the usage metrics, it looked less like a focused research task and more like a distributed load test:

| Metric          | Value         |
|-----------------|---------------|
| Input Tokens    | 22.8 million  |
| Output Tokens   | 2.3 million   |
| Cache Reads     | 32.7 million  |
| Cache Writes    | 5.7 million   |
| Web Searches    | 1,504         |
| Web Fetches     | 1,640         |
| Cost Equivalent | ~$59.25       |

```text
┌────────────────────────────────────────────────────────────────────────┐
│  Haiku 4.5 Usage Breakdown (120 Seconds)                               │
├────────────────────────────────────────────────────────────────────────┤
│  Input: 22.8m tokens    │ Cache Reads:  32.7m tokens                   │
│  Output: 2.3m tokens    │ Cache Writes:  5.7m tokens                   │
│  WebSearch: 1,504       │ WebFetch:      1,640                         │
│  Est. Cost: ~$59.25     │                                              │
└────────────────────────────────────────────────────────────────────────┘
```

This got me curious about what was happening behind the scenes and what was the root cause of consuming millions of tokens within 2 minutes.

## Claude Code Session Internals

> **Feel free to skip this section** if you just want the post-mortem and jump to the [Summary](#summary) section at the end. It is background on how Claude Code stores session data — useful context for understanding how the numbers below were derived.

Claude Code writes a complete, machine-readable record of every session to your local disk. Knowing the structure lets you audit exactly what happened after any run.

**Directory layout**

```
~/.claude/
└── projects/
    └── <url-encoded-absolute-project-path>/
        └── sessions/
            ├── <session-uuid>.jsonl        ← main session transcript
            └── <session-uuid>/
                └── subagents/
                    ├── <subagent-uuid>.jsonl
                    ├── <subagent-uuid>.meta.json
                    └── ...                 ← one pair per spawned subagent
```

Each top-level session gets a UUID; if that session spawned subagents, they live in a subdirectory alongside the parent JSONL.

**Main session JSONL (`<session-uuid>.jsonl`)**

Each line is a JSON object representing one message turn. The key fields:

```json
{
  "type": "assistant",
  "message": {
    "content": [
      {
        "type": "tool_use",
        "name": "WebSearch",
        "input": { "query": "..." }
      }
    ]
  }
}
```

Tool calls appear as `content` blocks with `"type": "tool_use"`. The `name` field is the tool (`WebSearch`, `WebFetch`, `Task`, `Agent`, `Bash`, etc.) and `input` holds the exact parameters passed. Tool results appear in subsequent turns as `"type": "tool_result"` blocks.

**Subagent `meta.json`**

Each subagent gets a `meta.json` alongside its transcript. It stores the task description the orchestrator passed when spawning that subagent — effectively the "prompt" given to that worker instance. This is how you can see what research angle each subagent was assigned.

**Subagent JSONL files**

Structurally identical to the main session JSONL — same format, same tool call schema. Because each subagent has its own isolated context window and its own file, the parent session transcript only records the `Task`/`Agent` spawn event; all intermediate tool calls made *by* the subagent are in the subagent's own file.

This is why a tool call count against the parent JSONL alone returns zero for subagent activity — you have to `cat` across the entire `subagents/` directory to get the true picture.

---

## Root Cause: The Anatomy of a Subagent Explosion

My first instinct was a **context snowball** — the kind of bug where a growing chat history gets re-sent on every turn, ballooning the input token count with each step.

But when I ran `/context` immediately after the incident, the numbers told a different story:

```text
Context Usage
30.3k/200k tokens (15%)
  System prompt:  6.6k tokens  (3.3%)
  System tools:  14.6k tokens  (7.3%)
  Skills:         1.2k tokens  (0.6%)
  Messages:       8.1k tokens  (4.0%)
  Free space:   169.5k         (84.7%)
```

The chat history ("Messages") was only 8.1k tokens — 4% of the context window. The window was not even close to full.

The real culprit was a **horizontal fan-out**. When you trigger `/deep-research` in Claude Code, it switches from a simple back-and-forth into a multi-agent coordinator. It invokes the [`Task` tool](https://docs.anthropic.com/en/docs/claude-code/sdk/subagents), which spawns independent subagents in parallel — each one assigned a different angle of the research query, each running in its own isolated context window.

### What the session logs actually showed

Claude Code stores every session as structured JSONL files under `~/.claude/projects/`. After digging into the session directory, I found a `subagents/` folder containing **1,898 files** — one `.jsonl` transcript and one `meta.json` per subagent. Parsing them revealed the full tool call breakdown:

```text
1640  WebFetch
1504  WebSearch
 969  Agent
 678  Skill
 357  ToolSearch
 115  Bash
  28  Monitor
  12  Read
```

That `969 Agent` line is the part that matters most. This was not a single-level fan-out. **Subagents were spawning their own subagents.** The orchestrator spun up 949 top-level workers; those workers then spawned further agents to handle sub-tasks, each independently loading tools, executing searches, and fetching pages.

Haiku 4.5 was released in October 2025 and is [designed for exactly this role](https://www.anthropic.com/news/claude-haiku-4-5) — high-throughput, parallel sub-agent execution, running four to five times faster than Sonnet on typical workloads. When deployed as an unconstrained orchestrator *and* executor with no depth or breadth limits, that throughput becomes an accelerant. It does not bottleneck. It just keeps going.

Each subagent initialised its own system tools baseline (~14.6k tokens), then immediately started firing web queries. Multiply that across 949 instances, add recursive nesting, and 1,500+ searches in 120 seconds becomes straightforward arithmetic.

## The Cost Illusion of "Cheaper" Models

Here is the mental model most engineers use when picking a model for an automated task:

> Haiku is cheaper per token than Sonnet — so point the bulk work at Haiku.

This holds for tasks with a fixed, bounded scope. It breaks completely for open-ended agentic tasks, because the actual cost equation is:

> **Total Cost = Cost per Step × Number of Steps**

When the model controls how many steps to take, the per-step price is almost irrelevant. What matters is whether anything caps the step count.

There is also a planning quality factor. Sonnet is [more consistent on complex, multi-step instructions](https://haimaker.ai/blog/claude-haiku-4-5-vs-sonnet-4-6/) — making fewer false "I'm done" claims and following through more reliably across long agentic runs. Haiku is designed for [execution of individual sub-tasks, not orchestration of open-ended research](https://www.morphllm.com/sonnet-vs-haiku). A more capable planner is more likely to recognise a dead-end search path and stop — rather than spawning another agent to try a variation.

But that is a secondary effect. The primary fix is not model selection. It is putting a hard ceiling on what the orchestrator can authorise before it asks you.

## Guard Rails: Keeping the Swarm on a Leash

The solution is two independent layers: a **behavioral policy** that shapes the plan before execution begins, and a **runtime wall** that enforces hard limits regardless of what the model decides.

### Layer 1: The Behavioral Policy — `CLAUDE.md`

Your project's `CLAUDE.md` is read by the orchestrator before it builds its research plan. Adding explicit constraints here instructs it to scope the work conservatively from the start.

Add this section to your local or global `CLAUDE.md`:

```markdown
# Research Policy (IMPORTANT)

- Maximum 20 web searches per session.
- Maximum 25 tool calls without explicit approval.
- Present a research plan before beginning.
- Stop and ask if more sources are needed — explain why and request permission.
- Prefer repository inspection over internet research.
- Do not recursively expand search queries.
- Do not spawn nested subagents.
```

Note the last line. Given that subagents here were spawning their own subagents, explicitly prohibiting recursive nesting is worth adding.

### Layer 2: The Runtime Wall — `~/.claude/settings.json`

`CLAUDE.md` can be misinterpreted or overridden by the model's own planning. `settings.json` cannot — it is enforced by the binary itself and halts execution the moment a threshold is breached.

Create or edit `~/.claude/settings.json`:

```json
{
  "permissions": {
    "ask": [
      "WebSearch",
      "WebFetch"
    ]
  },
  "maxSearches": 20,
  "maxToolCalls": 50
}
```

With this in place, every web search and every page fetch triggers a confirmation prompt. The process cannot fire off 3,000+ combined search-and-fetch operations silently.

### How the two layers interact

| Layer | Mechanism | What It Controls |
|---|---|---|
| Behavioral | `CLAUDE.md` Research Policy | Shapes the orchestrator's plan before execution begins |
| Runtime | `settings.json` limits | Stops the process if the plan exceeds your thresholds |

`CLAUDE.md` is the operating procedure. `settings.json` is the circuit breaker. You want both — the policy prevents most runaway scenarios from starting; the circuit breaker catches the ones that slip through.

## Summary
<a id="summary"></a>

The failure here was not a bug and it was not a deficiency in Haiku 4.5. The `/deep-research` feature worked exactly as designed by spawning multiple sub-agents. What was missing was any limit on how much it was authorised to do.

949 subagents. 969 nested agent calls. 1,504 searches. 1,640 page fetches. Two minutes.

Set the limits before you grant the permissions.