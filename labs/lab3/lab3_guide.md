# Lab 3: Skills & Timelining

## Overview

In Lab 2 you investigated the B2 incident by asking questions one at a time. Some queries worked on the first try; others took several wrong turns. In this lab you'll turn that experience into **skills** — reusable automation that makes future investigations faster and more reliable.

**Goals:**
1. Practice the flow: **use something → automate it into a Skill → prove the skill's value**
2. Build a timelining skill for the B2 incident
3. (Optional, Python users) Automate the flow via the louie-py API

**Running example:** BOTSv3 Incident B2 — AWS Key Compromise (same incident from Lab 2)

**Time:** ~60 minutes — Task 1 ~20m, Task 2 ~40m; Task 3 is optional if you finish early, or take-home.
**Notebooks:** `lab3_skills_louie.ipynb` (Tasks 1–2, code track), `lab3_api.ipynb` (Task 3).
No-code track: follow the tasks below in Louie desktop.

---

## Task 1: Skills Hello World

### The problem

In Lab 2, you may have noticed the agent:
- Doesn't know to use `index=botsv3`
- Tries wrong sourcetypes before finding the right one
- Doesn't know which fields matter in CloudTrail events
- Takes 3-5 queries to orient itself every time you start a new thread

### The fix: create a BOTS Q skill

A skill encodes what you already know so the agent starts smart.

### Step 1: Baseline (without skill)

Open a **new dthread** in louie and run one of the easier B2 questions:

```
Bud accidentally commits AWS access keys to an external code repository.
Shortly after, he receives a notification from AWS that the account had been
compromised. What is the support case ID that Amazon opens on his behalf?
```

**Observe:**
- How many queries did it take?
- Did it find the right sourcetype on the first try?
- How long before you got the answer?

### Step 2: Create the skill

In the same or new dthread, ask louie to create a skill:

```
Create a new skill called "BOTS Q" that preloads the following context for
BOTSv3 investigations:
- Always use index=botsv3
- Key sourcetypes: aws:cloudtrail for API/IAM activity, stream:smtp and
  ms:o365:reporting:messagetrace for emails, WinEventLog:Security for auth
  events (4624=success, 4625=failure), osquery:results for endpoint telemetry
- The dataset covers August 2018, use earliest=0
- For CloudTrail events, check errorCode to distinguish success from failure
- Start every investigation by scoping the activity, then drill in
```

> **Note:** See `skills/bots_q_template.lui.md` for a reference implementation of what this skill should look like.

### Step 3: Test in a new dthread

Open a **fresh dthread** (important — don't reuse the old one) and use the BOTS Q skill. Run the same question again.

**Compare:**
- Faster? Fewer wrong turns?
- Did it use `index=botsv3` immediately?
- Better first query?

### Step 4: Record your observations

| Metric | Without Skill | With Skill |
|--------|--------------|------------|
| Time to answer | | |
| Number of queries | | |
| Used correct index on first try? | | |
| Used correct sourcetype on first try? | | |
| Answer correct? | | |

This is a **vibes eval** — you're observing Correctness and Speed before doing anything formal.

---

## Task 2: Mini-Timelining Skill

### Background

Timelining is a key subflow of many investigation tasks: given a clue, go forwards and backwards in time to identify what happened. In this task you'll build a skill that does this for the B2 incident.

### The expected timeline

Open `b2_timeline_attack_phase.md` — this is the **attack phase** of the B2 incident (09:16:12 to 09:28:54). It contains 11 events across 4 attacker IPs.

Key events:
- 09:16:12 — Attacker validates stolen credentials (GetCallerIdentity)
- 09:16:12 — Attacker creates STS token for persistence
- 09:16:14 — Cryptomining campaign begins (RunInstances)
- 09:16:53 — AWS sends notification email (Case 5244329601)
- 09:27:06 — Different attacker uses ElasticWolf for manual exploration

### The clue

Your timelining skill will start from this clue:

> **"Access key AKIAJOGCDXJ5NW5PXUPA was used by IP 35.153.154.221 at 09:16:12"**

From this, the skill should search backwards and forwards to find related events.

### What to expect

With minimal prompt engineering, expect your skill to find **some** of these events but not all. For example:
- It should find the other CloudTrail events for the same key (easy — same filter)
- It may find the email notification (requires knowing to search stream:smtp)
- It may miss the ElasticWolf activity (different IP, 11 minutes later)

### Step 1: Build the skill

Create a new skill (or modify your BOTS Q skill) that adds timelining guidance:

```
Create a timelining skill for BOTSv3 investigations. Given a clue event
(timestamp, IP, key, or username), the skill should:
1. Search index=botsv3 for all events involving the same entities within +/- 30 minutes
2. Expand to related entities found in step 1 (e.g., if you find a new IP, search for that too)
3. Order events chronologically
4. For each event, note: timestamp, actor (IP/user), action, result (success/failure)
5. Identify attack phases: initial access, escalation, impact, detection, response
```

### Step 2: Test it

In a new dthread with the timelining skill, give it the clue:

```
Starting from this clue: Access key AKIAJOGCDXJ5NW5PXUPA was used by IP
35.153.154.221 at 2018-08-20 09:16:12. Build a timeline of the full attack
by searching forwards and backwards from this event.
```

### Step 3: Compare to the reference

Check your skill's output against `b2_timeline_attack_phase.md`:
- How many of the 11 events did it find?
- Did it identify all 4 attacker IPs?
- Did it find the AWS notification email?
- Did it correctly identify attack phases?

### Step 4: Iterate

Try improving your skill's prompt to catch the events it missed. Each iteration should get you closer to the reference timeline.

Stuck? `skills/bidirectional_timeframe.md` is a fuller reference skill for the same job —
backward/forward passes, pivoting on newly found entities, widening the window. Compare it
with what you wrote and borrow what's missing.

---

## Task 3 (optional): Automate the flow via the louie-py API

If you finish early and are comfortable with Python, reproduce Tasks 1–2 programmatically.
This is how investigations and evals get automated instead of done interactively — and
it's the machinery Lab 4 builds on. Otherwise treat it as take-home.

See `lab3_api.ipynb` for the full notebook. Key steps:

### Part 1: BOTS task with BOTS skill

```python
from louieai.notebook import lui

# Fresh thread
lui.new(name="B2 Automated - With Skill")

# Run a B2 question
lui("What is the support case ID that Amazon opens for Bud?", agent="SplunkAgent")
print(lui.text)

# Access the data
if lui.df is not None:
    display(lui.df)
```

Compare elapsed time and answer quality between threads with and without the BOTS Q skill.

### Part 2: Timelining via API

```python
lui.new(name="B2 Timeline - Automated")
lui("""Starting from this clue: Access key AKIAJOGCDXJ5NW5PXUPA was used by IP
35.153.154.221 at 2018-08-20 09:16:12. Build a timeline of the full attack.""",
    agent="SplunkAgent")

timeline_output = lui.text
print(timeline_output)
```

### Part 3: Score it against the reference via API

```python
# Load expected timeline
with open("b2_timeline_attack_phase.md") as f:
    expected = f.read()

lui.new(name="B2 Eval - Automated")
lui(f"""Evaluate this investigation timeline against the expected timeline.

Agent's output:
{timeline_output}

Expected timeline:
{expected}

For each expected event, mark it as: found, missed, or hallucinated.
Classify each error as an Accuracy error or Speed error.
Give an overall accuracy score.""")

print(lui.text)
```

This mirrors how a production eval pipeline works — run investigation, run eval, aggregate
results. Lab 4 turns this into a proper loop.

---

## Next Lab

In **Lab 4**, you will score this timelining skill with a real eval loop, error-analyze the failures, fix the skill, watch the score move — and then check whether the score is even real.
