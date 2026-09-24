# Lab 2: Investigating BOTSv3 — BYO Agent or Louie

## Overview

You investigate a real incident from the BOTSv3 dataset — **Incident B2, AWS Key
Compromise** — and you can run it **two ways**:

- **Path A — BYO agent.** Point your own coding agent (Claude Code, OpenCode, Codex) at
  Splunk over **Model Context Protocol (MCP)** and drive the investigation yourself. The
  raw harness experience.
- **Path B — Running with Louie.** Let Louie's `SplunkAgent` write the SPL for you — a few
  lines of `louie-py`, or by typing into Louie desktop (no code).

Either way you answer the same six questions (**Q1–Q6**) and score yourself. Pick the
path that fits you, or try both and compare.

**Time:** ~45 minutes • **Notebook:** `lab2_splunk_mcp.ipynb`

> **Prerequisite: finish Lab 1 first.** Lab 1 installs `splunk-mcp-server2` and sets your
> Splunk credentials (Path A), and connects Louie (Path B). This lab assumes that's done.

By the end of this lab you will:
- Understand how an agent reaches Splunk — over MCP (Path A) or via Louie's connector (Path B)
- Investigate **Incident B2: AWS Key Compromise** — six questions (Q1–Q6) following a real attack chain
- Observe how the agent writes SPL, where it succeeds, and where it struggles

---

## Part 1: What is MCP?

**Model Context Protocol (MCP)** is an open standard that lets AI agents call external tools. Think of it as a universal adapter between an LLM and the systems it needs to query.

### Key concepts

| Concept | Analogy for SOC Analysts |
|---------|--------------------------|
| **MCP Server** | Like a SOAR connector — it wraps an external system (Splunk) and exposes actions |
| **MCP Tool** | A specific action the server exposes (e.g., `search_oneshot` to run an SPL query) |
| **Transport** | How the agent talks to the server: `stdio` (local process) or `SSE` (network) |
| **MCP Client** | The AI agent itself (Claude Code) — it discovers and calls tools |

### How it works in practice

```
You (natural language) → AI Agent → MCP Protocol → Splunk MCP Server → Splunk API → Results
                         ↑                                                           |
                         └───────────────── formatted answer ←──────────────────────┘
```

The agent:
1. Receives your question in plain English
2. Decides it needs Splunk data to answer
3. Generates an SPL query
4. Calls the MCP tool `search_oneshot(query=...)`
5. Reads the returned data
6. Formulates a human-readable answer

---

## Part 2: Setup — done in Lab 1

The Splunk MCP server, credentials, and `mcp_config.json` are all installed and
verified in **Lab 1**. If you skipped it, do it now — Lab 1 walks through cloning
`splunk-mcp-server2`, exporting the `SPLUNK_*` variables, and confirming the config
parses. The server exposes `validate_spl`, `search_oneshot`, `search_export`,
`get_indexes`, `get_saved_searches`, `run_saved_search`, and `get_config`, with
built-in SPL validation and sensitive-data sanitization.

---

## Part 3: The Investigation — BOTSv3 Incident B2

### The scenario

**Incident B2: AWS Key Compromise**

An employee named Bud (username `bstoll`) accidentally commits AWS access keys to a public GitHub repository. AWS detects the exposure and notifies him, but an adversary has already grabbed the keys. The adversary uses the compromised key `AKIAJOGCDXJ5NW5PXUPA` to:
- Enumerate IAM resources
- Attempt to create new access keys
- Try to launch EC2 instances across multiple regions

You'll investigate this incident through **six questions** that follow the attack chain (all in `botsv3_b2_tests.json`): two easy lookups (Q1, Q2), three medium multi-step questions (Q3, Q4 — the lethal-trifecta GitHub fetch — and Q5), and the hard one the dataset can't answer alone (Q6). Budget roughly 5–8 minutes per question. **If you fall behind, prioritise Q1, Q4 and Q6** — they span the range — and come back to the rest. The same six are reused by the eval and contamination labs, so the numbering is shared across labs.

### Launch Claude Code with MCP

`mcp_config.json` lives in `lab1/` (set up in Lab 1). Point `--mcp-config` at it — you
can launch from this `lab2/` folder; the flag takes a path, so you don't need to `cd`:

```bash
claude --mcp-config ../lab1/mcp_config.json
```

Claude Code will start and discover the Splunk MCP tools automatically.

### The questions

Work through these **in order** — they follow the attack chain from detection through adversary activity.

---

**Q1 (Easy) — The Notification**

```
Bud accidentally commits AWS access keys to an external code repository.
Shortly after, he receives a notification from AWS that the account had been
compromised. What is the support case ID that Amazon opens on his behalf?
```

Watch for: Does the agent find email data in Splunk? Does it know which sourcetype to search?

---

**Q2 (Easy) — The Adversary's Tool**

```
Using a leaked AWS key AKIAJOGCDXJ5NW5PXUPA, the adversary makes an unauthorized
attempt to describe an account. What is the full user agent string of the
application that originated the request?
```

Watch for: Does the agent go straight to `aws:cloudtrail`? Does it extract the `userAgent` field?

---

**Q3 (Medium) — IAM Enumeration**

```
What IAM user access key generates the most distinct errors when attempting
to access IAM resources?
```

Watch for: Does the agent filter by `eventSource=iam.amazonaws.com`? Does it use `dc()` (distinct count) in SPL correctly?

---

**Q4 (Medium) — The Leaked Secret**

```
AWS access keys consist of two parts: an access key ID (e.g., AKIAIOSFODNN7EXAMPLE)
and a secret access key (e.g., wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY).
What is the secret access key of the key that was leaked to the external code repository?
```

Watch for: This requires finding the notification email, extracting a GitHub link from its content, and following it. Multi-step reasoning.

---

**Q5 (Medium) — Unauthorized Key Creation**

```
Using a leaked AWS key AKIAJOGCDXJ5NW5PXUPA, the adversary makes an unauthorized
attempt to create a key for a specific resource. What is the name of that resource?
```

Watch for: Does the agent find `CreateAccessKey` events in CloudTrail and extract the target from `requestParameters`?

---

**Q6 (Hard) — EC2 Launch Attempt**

```
The adversary attempts to launch an Ubuntu cloud image as the compromised IAM user.
What is the codename for that operating system version in the first attempt?
Answer guidance: Two words.
```

Watch for: This is the hardest question. The agent must find `RunInstances` events, extract the AMI ID, and look up what Ubuntu version it corresponds to. This requires external knowledge beyond what's in Splunk.

---

### What to observe

For each question, pay attention to:
- **SPL quality**: Does the agent write correct, efficient SPL? Would you have written it differently?
- **Tool calls**: How many calls to `search_oneshot` does it make? Does it iterate or get it in one shot?
- **Wrong paths**: Does it go down dead ends (wrong sourcetype, wrong field, wrong filter)?
- **Interpretation**: Does it explain the security significance of what it finds?

---

## Part 4: Check Your Answers

Expected answers are below, and the notebook's scoring cell (`lab2_splunk_mcp.ipynb`)
checks them for you. The **approaches are withheld on purpose** — working out *why* a
query failed is the Lab 4 exercise, so printing the method here would spoil it.
Instructors have them in `botsv3_b2_hints.json` (gitignored).

| Q | Expected Answer |
|---|----------------|
| Q1 | `5244329601` |
| Q2 | `ElasticWolf/5.1.6` |
| Q3 | `AKIAJOGCDXJ5NW5PXUPA` |
| Q4 | `Bx8/gTsYC98T0oWiFhpmdROqhELPtXJSR9vFPNGk` |
| Q5 | `nullweb_admin` |
| Q6 | `Xenial Xerus` |

---

## Part 5: Reflection

After completing the investigation, consider:

1. **What made questions easy or hard?** Q1 and Q2 are straightforward lookups. Q6 requires external knowledge. Where's the boundary of what the agent can do with Splunk alone?

2. **Where did the agent struggle?** For any wrong answers, classify the failure:
   - **Incorrect answer** — wrong SPL, hallucinated data, misinterpreted the question
   - **Slow reasoning** — went down the wrong path, made too many queries, timed out

3. **What would have helped?** Think about what would make these investigations faster and more reliable:
   - A system prompt that knows BOTSv3 sourcetypes and fields?
   - A tool that auto-enriches AWS access keys?
   - A structured investigation template?

   These are the **skills** you'll build in Lab 3.

---

## Next Lab

In **Lab 3**, you'll take the pain points from this lab and turn them into skills — automation that makes the agent faster and more reliable. You'll also compare the Claude Code experience with louie-py's data-oriented approach.
