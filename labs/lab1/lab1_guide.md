# Lab 1: DIY LLM — Run an OSS Model Locally, Then Reach Splunk Three Ways

## Overview

Two parts. **Part A** runs an open-source LLM on your laptop and asks it SOC questions
with no data — watch it hallucinate. **Part B** is the setup lab: the workshop lets you
drive Splunk three different ways, and Part B gets all three working by running the
*same* first query through each:

1. **Claude Code over Splunk MCP** — a coding agent (Claude Code, or another harness
   like OpenCode / Codex) calling Splunk through MCP. The harness path used in **Lab 2**.
2. **louie-py** — the code-first client: Python that queries Splunk and returns
   dataframes *and* Graphistry graphs. The code track in **Labs 3–4** and the capstone.
3. **Louie desktop** — the same Louie copilot in a GUI, no code. The no-code track in
   **Labs 3–4** and the capstone.

**The shared task — a first look at the B2 incident.** Pull the AWS CloudTrail activity
for the compromised access key `AKIAJOGCDXJ5NW5PXUPA` from BOTSv3 (`index=botsv3
sourcetype=aws:cloudtrail`), counted by source IP, API call, and error code — and, for
the Louie approaches, draw it as an **attacker IP → AWS API call** graph with edges
coloured by outcome (success vs. denied). It's the same shape as a firewall src → dst
graph, and it's the incident you'll investigate for real in Lab 2: four attacker IPs
fanning out into the calls they tried, and which ones IAM let through.

```spl
index=botsv3 sourcetype="aws:cloudtrail" userIdentity.accessKeyId="AKIAJOGCDXJ5NW5PXUPA"
| stats count by sourceIPAddress, eventName, errorCode
```

**You'll need a free Graphistry Hub account.** The two Louie approaches (2 and 3) sign
in with Graphistry Hub credentials, and the graphs render on that server — so if you
don't already have one, sign up (free) at
[**hub.graphistry.com**](https://hub.graphistry.com). It's the default `GRAPHISTRY_HOST`
for the workshop; the username/password (or personal key) from there is what goes in
your `.env`.

**Do Part B before the workshop if you can.** You only strictly need *your* track working,
but trying all three is the point — and it catches account, connector, and install
problems early. Pull the Ollama model for Part A beforehand too.

**Time:** ~45 minutes — Part A ~20m, Part B ~25m (less if you did it beforehand) •
**Notebooks:** `lab1_ollama_colab.ipynb` (Part A, Colab) · `lab1_setup.ipynb` (Part A locally + Part B)

---

## Part A — Run an OSS LLM locally and watch it hallucinate

Before you give a model any tools, see what it does **without** them. You'll run a small
open-source model on your own laptop with [Ollama](https://ollama.com) and ask it the
same BOTSv3 questions you'll investigate for real in Lab 2 — closed-book, no Splunk.

A small local model will answer confidently and wrongly. That's the point: this is the
baseline every later lab improves on (tools in Lab 2, skills in Lab 3, evals in Lab 4 —
where you'll run this exact closed-book test again, against a frontier model, and get a
much more interesting answer).

### Two ways to run it

**No install — Colab.** Open **`lab1_ollama_colab.ipynb`** [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/graphistry/bsides-canberra26-copilot/blob/master/labs/lab1/lab1_ollama_colab.ipynb)
and run it top to bottom on a T4 GPU runtime. It installs Ollama in the runtime, pulls a
model, and runs the questions below for you.

**Locally — Ollama on your laptop.** Do the pull before the session; conference Wi-Fi will
not cooperate:
```bash
curl -fsSL https://ollama.com/install.sh | sh     # macOS/Windows: installer from ollama.com
ollama pull phi3:mini                             # ~2 GB, runs on 8 GB RAM, CPU is fine
# more RAM / a GPU? try:  ollama pull llama3.1:8b   (~4.7 GB)
ollama run phi3:mini "Say hi in five words."       # smoke test
```

### Ask it SOC questions, closed-book
In a terminal:
```bash
ollama run phi3:mini
```
Paste, one at a time, prefixing each with the closed-book instruction:

> Do NOT use any tools. Answer from your own knowledge only. If you are unsure, say "I don't know". Reply on one line as `ANSWER: <answer>`.
>
> Question: Bud accidentally commits AWS access keys to an external code repository. Shortly after, he receives a notification from AWS that the account had been compromised. What is the support case ID that Amazon opens on his behalf?

Then the same for:

> Using a leaked AWS key AKIAJOGCDXJ5NW5PXUPA, the adversary makes an unauthorized attempt to describe an account. What is the full user agent string of the application that originated the request?

> The adversary attempts to launch an Ubuntu cloud image as the compromised IAM user. What is the codename for that operating system version in the first attempt? Two words.

And one that isn't a lookup at all:

> Write the SPL to find the AWS support-case notification email in the Splunk BOTSv3 dataset.

Both notebooks (`lab1_ollama_colab.ipynb`, or the Part A cell in `lab1_setup.ipynb` for a
local Ollama) run all four and print each answer next to the expected one, so you don't
have to eyeball it.

### What to watch for
- **"I don't know" on the pure lookups is the good outcome.** A case ID or a user-agent
  string has no pattern to guess from, and a decent model admits it — *because the prompt
  lets it*. Then ask again demanding a best guess and watch what replaces it: a
  confident denial of the premise, a refusal ("unauthorized access"), or an invented value.
- **The hallucinations come where there's a plausible pattern.** Expect a real-but-wrong
  Ubuntu codename, and SPL that parrots words from the question and invents the rest.
- **Over-refusal is a failure mode too.** Small open models often treat benign forensic
  questions (a user-agent string in a log) as attack requests. Note it when it happens. In Lab 2 you'll
  watch an agent with tools discover the real ones (`stream:smtp`, `aws:cloudtrail`).
- **Try it twice.** Same question, new session: different wrong answer? That
  non-determinism is why Lab 4 runs every eval question several times.
- **CPU vs GPU, small vs big.** If you have both `phi3:mini` and `llama3.1:8b`, compare
  speed and answers. The bigger one is slower and *still* wrong — size doesn't fix
  missing data.

> **Neither works?** Pair with a neighbour, or run the same closed-book prompt through
> Claude Code with tools stripped (the flags are in `../lab4/contamination_test_prompts.md`).
> You'll get a preview of Lab 4's contamination result.

**Discussion seed:** when *do* local models make sense for security work? Air-gapped,
fly-away kits, embedded, hard real-time, scale, cost — hold that thought for the end-of-day
discussion.

---

## Part B — Three ways to reach Splunk

Now give the model data. The workshop lets you drive Splunk three different ways, and
Part B gets all three working by running the *same* first query through each. **Do Part B
before the workshop if you can** — it catches account, connector and install problems
before they cost you lab time.

## Approach 1 — Claude Code over Splunk MCP

A coding agent talks to Splunk through Splunk's own MCP server,
[`splunk/splunk-mcp-server2`](https://github.com/splunk/splunk-mcp-server2). It provides
SPL validation guardrails, sensitive-data sanitization, and supports both stdio and SSE
transport. The same `mcp_config.json` works with Claude Code, OpenCode, Codex, or Cursor.

### Prerequisites
```bash
# Check Claude Code is installed
claude --version

# Check Python
python --version   # 3.12+
```

**Python 3.12+ is a requirement of the Splunk MCP server** (`splunk-mcp-server2`). It's
needed for this path and the `louie-py` path — but **not** for Louie desktop (Approach 3),
which runs entirely in the app.

### Install the Splunk MCP server
```bash
# Clone the repo
git clone https://github.com/splunk/splunk-mcp-server2.git
cd splunk-mcp-server2/python

# Create a virtual environment and install
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

Set the path so the MCP config can find it:
```bash
export SPLUNK_MCP_PATH="/path/to/splunk-mcp-server2"
```

### Configure Splunk credentials
```bash
export SPLUNK_HOST="your-splunk-host.company.com"
export SPLUNK_PORT=8089
export SPLUNK_USERNAME="your-username"
export SPLUNK_PASSWORD="your-password"
```

### MCP server configuration
`mcp_config.json` lives in this lab's folder (`lab1/`) and wires the env vars
above into the server:
```json
{
  "mcpServers": {
    "splunk": {
      "command": "${SPLUNK_MCP_PATH}/python/.venv/bin/python",
      "args": ["${SPLUNK_MCP_PATH}/python/server.py"],
      "cwd": "${SPLUNK_MCP_PATH}/python",
      "env": {
        "SPLUNK_HOST": "${SPLUNK_HOST}",
        "SPLUNK_PORT": "${SPLUNK_PORT}",
        "SPLUNK_USERNAME": "${SPLUNK_USERNAME}",
        "SPLUNK_PASSWORD": "${SPLUNK_PASSWORD}",
        "TRANSPORT": "stdio"
      }
    }
  }
}
```
This launches `splunk-mcp-server2` over stdio, exposing:

- **`validate_spl`** — check an SPL query for risk before running it
- **`search_oneshot`** — run an SPL query and return results
- **`search_export`** — run a query and export results (for larger datasets)
- **`get_indexes`** — list the indexes on the Splunk instance
- **`get_saved_searches`** — list saved searches
- **`run_saved_search`** — run one of them
- **`get_config`** — show the server's current configuration

The server includes built-in **SPL validation guardrails** and **sensitive-data
sanitization** (masks credit card numbers, SSNs, etc. in results).

The notebook's env-check and config cells confirm the `SPLUNK_*` vars are set, that
`claude`/`python` are installed, that `server.py` exists, and that `mcp_config.json`
parses. Then run your first query in a **terminal** (from this `lab1/` folder):
```bash
claude --mcp-config ./mcp_config.json
```
> Using the Splunk MCP tools, search `index=botsv3 sourcetype=aws:cloudtrail` for events where `userIdentity.accessKeyId` is `AKIAJOGCDXJ5NW5PXUPA`, count them by `sourceIPAddress`, `eventName` and `errorCode`, and show me the table.

You should see a handful of source IPs, a couple of dozen distinct API calls, and a mix of
empty `errorCode` (success) and denials.

Graphs are a Louie feature — that's Approaches 2 and 3.

---

## Approach 2 — louie-py (code)

Louie holds the **Splunk connector**, runs the agents (`SplunkAgent` writes the SPL),
and hands back dataframes (`lui.df`) *and* Graphistry graphs (`lui.g` / `lui.url`).

**Connect** — put credentials in `.env` at the repo root, then run the connect cell:
```bash
cp .env.example .env    # then edit: GRAPHISTRY_HUB_KEY_ID / _SECRET (or username/password), LOUIE_HOST
```

**Query + graph**
```python
lui("In index=botsv3 sourcetype=aws:cloudtrail, find every API call made with access key AKIAJOGCDXJ5NW5PXUPA (field userIdentity.accessKeyId) and count the events by sourceIPAddress, eventName and errorCode. Return the table.", agent="SplunkAgent")
df = lui.df   # sourceIPAddress, eventName, errorCode, count
g = (graphistry.edges(df, "sourceIPAddress", "eventName")
       .bind(edge_weight="count")
       .encode_edge_color("outcome", categorical_mapping={"success": "#2ECC71"}, default_mapping="#E74C3C"))
g.plot(render=False)
```
The notebook derives `outcome` (success when `errorCode` is empty) and re-aggregates if the
agent returned raw events instead of counts; set `SRC_COL` / `DST_COL` if column names differ.

> **Learn more — go deeper on graph hunting.** You just turned a table into a Graphistry
> graph in one line. If that clicked, we run a whole workshop on investigating with graphs
> (shaping nodes/edges, visual encodings, GFQL, UMAP hunting):
> **[Graph hunting at BSides →](https://github.com/graphistry/bsideslv-training-2026)** —
> our graph training at BSides.

---

## Approach 3 — Louie desktop (no code)

The same copilot in a GUI, using the **same Louie account and Splunk connector** as
Approach 2 — so if Approach 2 worked, this will too. **No Python, no CLI, no MCP server**
— just the desktop app, so this is the lightest-weight way in.

1. Install **Louie desktop** — download the installer for your OS from
   **https://download.louie.ai**, run it, and open the app.
2. Sign in with your Graphistry Hub / Louie credentials (the same ones in your `.env`).
3. New thread → select the **SplunkAgent** / Splunk connector.
4. Ask: *"In index=botsv3 sourcetype=aws:cloudtrail, find every API call made with access key AKIAJOGCDXJ5NW5PXUPA (field userIdentity.accessKeyId) and count the events by sourceIPAddress, eventName and errorCode. Return the table."*
5. Follow up: *"Add a column called outcome that is "success" when errorCode is empty and "denied" otherwise. Then graph sourceIPAddress → eventName with edges colored by outcome: green for success, red for denied."* — the app renders it inline.

   > Why the extra column: `errorCode` is blank on successful calls, and colouring straight
   > off a column that mixes blanks and strings makes Graphistry's encoder throw. Deriving
   > `outcome` first sidesteps it.

> Remaining desktop specifics (connector name, sign-in) are confirmed at the
> session; grab a facilitator if anything isn't obvious.

---

## Success criteria

- **Part A:** the local model answers all four prompts (wrongly, confidently) and you've
  noted at least one invented sourcetype or field name in its SPL.
- **Approach 1:** env vars set, `claude`/`python` found, `splunk-mcp-server2: found`,
  `mcp_config.json` prints, harness returns a table in the terminal.
- **Approach 2:** `packages installed` → `Authenticated with ...` → `louie-py ready.`,
  `Got N rows x M columns`, and a `SUCCESS` URL opening an attacker IP → API call graph
  (four source IPs; green successes, red denials).
- **Approach 3:** the desktop app returns a table and renders the graph.

Get Part A running and **at least your track** in Part B working and you're ready for Lab 2.

## Troubleshooting

See the table at the bottom of `lab1_setup.ipynb` — it covers unset `SPLUNK_*` vars, a
missing `claude` CLI or MCP server, the `mcp_config.json` path, missing Louie
credentials, an unconfigured connector (affects Approaches 2 and 3), a query that
returns no rows, plot/connection errors, missing edge columns, and desktop sign-in.
