Companion to [readme.md](readme.md), [readmecli.md](readmecli.md), and [vscodeaicheatsheet.md](vscodeaicheatsheet.md).

---

# Token Monitoring for Claude Code

This guide explains how to see exactly how many tokens your Claude Code sessions are using, what those numbers mean, and how to act on them. All example numbers in this guide are illustrative placeholders — your actual figures will vary.

---

## 1. What this is / how to read it

Claude Code stores a local log of every session on your machine. Those logs contain token counts for every AI response. This guide shows you:

- what the four token fields mean and which ones cost the most,
- how to check usage in real time while a session is running,
- three ready-to-run scripts to report on all your past sessions,
- a third-party dashboard tool called `ccusage`,
- when and how to graduate to team-scale or API-level telemetry,
- and industry best practices for keeping costs under control.

No personal usage numbers appear in this guide. Any numbers you see — like `input=12000 output=3400` — are made-up examples to illustrate the format.

---

## 2. Token basics for monitoring

**What is a token?** A token is a small chunk of text — roughly 3–4 characters, or about 3/4 of a word in English. Every word you type in a prompt, and every word the model writes back, costs tokens. APIs charge per token.

**Input vs. output tokens** — Input tokens are what you send to the model (your prompt, file contents, conversation history). Output tokens are what the model generates in reply. Output tokens typically cost more per token than input tokens, so a response that produces a lot of text is more expensive than a response that produces a little.

**Cache read vs. cache creation** — Claude uses a prompt cache to avoid re-processing the same context prefix over and over.

- **Cache creation** (`cache_creation_input_tokens`): the first time your context is processed and stored in the cache. You pay a one-time cost to write it.
- **Cache read** (`cache_read_input_tokens`): on subsequent turns, Claude reads from the warm cache instead of re-processing. This is much cheaper than fresh input tokens.

**The ~5-minute cache window (TTL)** — TTL stands for Time To Live: the window during which the cache stays warm. If you keep working without a long pause, each new turn reads from the warm cache (cheap). If you go idle for more than ~5 minutes, the cache expires and the next turn re-processes everything as fresh input (more expensive). A 1-hour cache tier also exists. The cache TTL is the single biggest cost lever in typical Claude Code use — keep your pace up during multi-step tasks.

### The four fields you will see in logs

| Field name | What it counts | Cost intuition |
|---|---|---|
| `input_tokens` | Fresh (uncached) tokens sent to the model | Mid-cost per token |
| `output_tokens` | Tokens the model generated in reply | Highest cost per token |
| `cache_creation_input_tokens` | Tokens written into the prompt cache | Paid once; cheaper than output |
| `cache_read_input_tokens` | Tokens read from a warm cache | Cheapest input option |

> **Reality-check:** These four fields come from the `message.usage` object on assistant message lines in Claude Code's local JSONL logs. Other fields may appear in the raw JSON — the scripts in this guide skip everything except these four.

---

## 3. Real-time, in-session monitoring

While a session is running, Claude Code gives you built-in signals you can check at any time by typing a slash command in the REPL.

| Signal / command | What it shows | What to do |
|---|---|---|
| `/cost` | Token count and estimated cost for the current session | Check after a heavy task to see where tokens went |
| `/context` | A visual meter showing how full the context window is | If it is near full, use `/clear` or wrap up the current task |
| `/status` | Current session status, model, and connection info | Quick sanity check if something seems off |
| `/statusline` | Customize what appears in the persistent status line | Add token or cost info to your status line so you can see it continuously |

**What "context near full" means in practice** — The context window is the model's working memory for one session. It has a fixed size. As the session grows (more messages, more file reads), it fills up. When it is nearly full, the model starts losing access to earlier parts of the conversation. The `/context` meter shows you where you are. When it is getting high, finish the current thought and either use `/clear` to start fresh or open a new session.

> **Reality-check:** `/cost`, `/context`, and `/status` are in-session commands — they only work while the Claude Code REPL is open. They do not persist between sessions. For historical reporting across all your past sessions, use the scripts in the next section.

---

## 4. Analysis scripts you can run

Claude Code writes every session to a local JSONL log file (one JSON object per line). These files live here:

```
~/.claude/projects/<encoded-project-dir>/<session-id>.jsonl
```

On Windows, `~` means `C:\Users\<your-username>\`. The `<encoded-project-dir>` part is a URL-encoded version of your project path.

**Subagents have separate transcript files.** When Claude Code launches a subagent, that agent gets its own JSONL file nested under the session directory:

```
~/.claude/projects/<encoded-project-dir>/<session-id>/subagents/agent-*.jsonl
```

Because the scripts below use recursive file scanning, they automatically pick up both top-level session files and subagent files in one pass. This means subagent token costs are included in the totals.

Only assistant message lines carry a `message.usage` object. User messages, tool results, and system lines do not — the scripts skip them automatically.

---

### Script 1 — PowerShell totals report (primary option)

**What it does:** Reads all session logs under `~/.claude/projects/`, totals up the four token fields, calculates a cache hit rate, and breaks output down by model. Read-only: it never modifies any file.

**How to run:**

1. Save the script below as `token-report.ps1` anywhere convenient (for example, `C:\Users\<you>\scripts\token-report.ps1`).
2. Open PowerShell and run: `powershell -ExecutionPolicy Bypass -File "C:\path\to\token-report.ps1"`

```powershell
# token-report.ps1 — summarize Claude Code token usage from local session logs.
# Read-only: it only reads the JSONL transcripts, never changes them.
$root = Join-Path $env:USERPROFILE ".claude\projects"

$totals  = [ordered]@{ input = 0; output = 0; cacheCreate = 0; cacheRead = 0 }
$byModel = @{}

Get-ChildItem -Path $root -Recurse -Filter *.jsonl -File | ForEach-Object {
    foreach ($line in [System.IO.File]::ReadLines($_.FullName)) {
        if ([string]::IsNullOrWhiteSpace($line)) { continue }
        try { $o = $line | ConvertFrom-Json } catch { continue }   # skip non-JSON lines
        $u = $o.message.usage
        if ($null -eq $u) { continue }                              # only assistant lines have usage

        $totals.input       += [int64]$u.input_tokens
        $totals.output      += [int64]$u.output_tokens
        $totals.cacheCreate += [int64]$u.cache_creation_input_tokens
        $totals.cacheRead   += [int64]$u.cache_read_input_tokens

        $model = if ($o.message.model) { $o.message.model } else { "unknown" }
        if (-not $byModel.ContainsKey($model)) {
            $byModel[$model] = [ordered]@{ input = 0; output = 0; cacheRead = 0 }
        }
        $byModel[$model].input    += [int64]$u.input_tokens
        $byModel[$model].output   += [int64]$u.output_tokens
        $byModel[$model].cacheRead += [int64]$u.cache_read_input_tokens
    }
}

# Cache hit rate = cached input / all input. Higher = cheaper.
$allInput = $totals.input + $totals.cacheRead
$hitRate  = if ($allInput -gt 0) { [math]::Round(100 * $totals.cacheRead / $allInput, 1) } else { 0 }

"== Totals (all sessions) =="
$totals | Format-Table -AutoSize
"Cache hit rate: $hitRate%"
""
"== Output tokens by model =="
$byModel.GetEnumerator() | Sort-Object { $_.Value.output } -Descending |
    ForEach-Object { "{0,-28} output={1}  input={2}  cacheRead={3}" -f $_.Key, $_.Value.output, $_.Value.input, $_.Value.cacheRead }
```

**Example output (illustrative — your numbers will differ):**

```
== Totals (all sessions) ==

Name        Value
----        -----
input       120000
output      42000
cacheCreate 80000
cacheRead   310000

Cache hit rate: 72.1%

== Output tokens by model ==
claude-sonnet-4-6            output=28000  input=85000  cacheRead=210000
claude-opus-4-8              output=14000  input=35000  cacheRead=100000
```

**How to read the cache hit rate** — `72.1%` means 72% of all input was served from the cache rather than re-processed fresh. A higher hit rate is cheaper. If your hit rate is very low (under 30%), you may be starting too many short sessions or waiting too long between turns.

---

### Script 2 — PowerShell live watcher (one session)

**What it does:** Watches a single active session file and prints a running token tally as new lines appear. Useful for tracking a long-running task in real time. Stops when you press Ctrl+C.

**How to run:**

1. Save as `token-watch.ps1`.
2. Find your current session file — it is the most recently modified `.jsonl` under `~/.claude/projects/`. In PowerShell: `Get-ChildItem -Path "$env:USERPROFILE\.claude\projects" -Recurse -Filter *.jsonl | Sort-Object LastWriteTime -Descending | Select-Object -First 1 -ExpandProperty FullName`
3. Run: `powershell -ExecutionPolicy Bypass -File token-watch.ps1 "C:\Users\<you>\.claude\projects\<dir>\<session-id>.jsonl"`

```powershell
# token-watch.ps1 <path-to-session.jsonl> — live running token tally for one session.
param([Parameter(Mandatory)][string]$SessionFile)
$in=0; $out=0
Get-Content -Path $SessionFile -Wait | ForEach-Object {
    if ([string]::IsNullOrWhiteSpace($_)) { return }
    try { $u = ($_ | ConvertFrom-Json).message.usage } catch { return }
    if ($null -eq $u) { return }
    $in  += [int64]$u.input_tokens
    $out += [int64]$u.output_tokens
    Write-Host ("`rinput={0}  output={1}" -f $in, $out) -NoNewline
}
```

`Get-Content -Wait` keeps the file open and watches for new lines as they are written — the PowerShell equivalent of `tail -f` on Linux/macOS. The tally updates on the same line each time (the `\r` moves the cursor back to the start of the line). Press Ctrl+C to stop.

> **Reality-check:** This script only watches a single session file. It does not include subagent files that appear in the `subagents/` subfolder. Use Script 1 after a session finishes to get the complete picture including subagents.

---

### Script 3 — Node alternative (if you have Node installed)

**What it does:** The same all-sessions total as Script 1, but written in JavaScript. Use this only if you have Node.js installed (`node --version` in a terminal to check). PowerShell ships with Windows and requires no install — prefer Script 1 if you are unsure.

**How to run:** Save as `token-report.js` and run `node token-report.js` from any terminal.

```javascript
// token-report.js — same idea as the PowerShell report, for Node users.
// Run: node token-report.js
const fs = require("fs"), path = require("path"), os = require("os");
const root = path.join(os.homedir(), ".claude", "projects");
const t = { input: 0, output: 0, cacheCreate: 0, cacheRead: 0 };

function walk(dir) {
  for (const e of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, e.name);
    if (e.isDirectory()) walk(p);
    else if (e.name.endsWith(".jsonl")) scan(p);
  }
}
function scan(file) {
  for (const line of fs.readFileSync(file, "utf8").split("\n")) {
    if (!line.trim()) continue;
    let u; try { u = JSON.parse(line).message?.usage; } catch { continue; }
    if (!u) continue;
    t.input       += u.input_tokens || 0;
    t.output      += u.output_tokens || 0;
    t.cacheCreate += u.cache_creation_input_tokens || 0;
    t.cacheRead   += u.cache_read_input_tokens || 0;
  }
}
walk(root);
console.log(t);
```

---

### ccusage — third-party dashboard tool

`ccusage` is a popular **community-built, third-party** npm package that reads the same local JSONL logs and produces daily, monthly, and session-level reports. It is not made by Anthropic. Run it without installing anything permanently using `npx`:

```powershell
# Daily and monthly usage summary
npx ccusage@latest

# Live updating dashboard (refreshes as new session lines appear)
npx ccusage@latest blocks --live
```

> **Reality-check:** `npx` requires Node.js. `ccusage` reads your local `~/.claude/projects/` logs the same way the scripts above do — it does not contact any external service. Because it is a third-party tool, check its npm page for updates, changelog notes, and any breaking changes before relying on it in a workflow. Always pin a version with `npx ccusage@<version>` in automated scripts rather than using `@latest`, so you are not surprised by changes.

---

## 5. Team / industry-grade telemetry

The scripts above are great for a single developer. When you are working on a team — or when you need to track usage trends over time in a shared dashboard — Claude Code supports exporting telemetry data to standard observability platforms.

**The environment variable to enable it:** Set `CLAUDE_CODE_ENABLE_TELEMETRY=1` before launching Claude Code. Claude Code then emits metrics (token usage, estimated cost, session counts, and related signals) in the OpenTelemetry (OTLP) format.

**OpenTelemetry (OTLP)** is an open standard for sending metrics and traces from any application to any compatible backend. You configure an OTLP exporter to point at your collector — which then feeds data into Prometheus, Grafana, Datadog, or any other platform that speaks OTLP.

**When to graduate to this approach:**

- You have more than one developer and want a shared cost dashboard.
- You want alerts when usage crosses a budget threshold.
- You want to correlate token spend with specific projects, branches, or team members over weeks or months.
- Your organization requires centralized audit trails.

For a solo developer running occasional projects, the local scripts in Section 4 are enough. Telemetry is for when you need visibility at scale.

> See the official Claude Code telemetry documentation for the exact environment variables, metric names, and exporter configuration. The available options may expand over time — check the docs rather than relying on any hardcoded list here.

---

## 6. API / SDK-level measurement (for apps you build yourself)

Everything above is about monitoring Claude Code — the CLI tool. This section is different: it is about measuring token use when you are writing your own application that calls the Claude API directly.

If you are building a chatbot, a workflow tool, or any other app that calls Claude programmatically, the API gives you token data on every response.

**Usage object on API responses** — Every response from the API includes a `usage` field with the same four counts you have seen throughout this guide: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens`. Read this object after every API call to log or accumulate your own totals.

**Token-counting endpoint** — Before making a call, you can send your prompt to a token-counting endpoint to estimate how many tokens it will use. This is useful for pre-flight checks: "is this prompt going to blow the context window?" or "is this going to cost more than I want to spend?" Check the API reference for the exact endpoint path and request shape.

**Streaming usage deltas** — If you use streaming (receiving the response incrementally as it is generated), the API can emit token counts as the stream progresses rather than only at the end. This lets you display a live counter in your UI or cut off a response early if it is growing unexpectedly large.

**Prompt-cache metrics in API responses** — The same `cache_creation_input_tokens` and `cache_read_input_tokens` fields appear in API responses. Track these in your own telemetry to understand your effective cache hit rate and optimize your prompt structure accordingly.

**Usage and cost reporting** — The Anthropic console provides usage reports across your API key. For more granular attribution (by user, by feature, by project), implement your own logging on top of the per-response `usage` object.

> **Scope note:** This section applies only to code you write that calls the Claude API. Claude Code the CLI tool is separate — its token data comes from the local JSONL logs described in Section 4, not from an API you instrument yourself.

---

## 7. Industry best practices for measuring and optimizing

### Measure the right things

- **Track input and output separately.** They have different per-token prices, and the ratio tells you a lot. A task with very high output relative to input might benefit from a more concise system prompt or a tighter response format instruction.
- **Track your cache hit rate.** The PowerShell report in Section 4 calculates this for you. A high hit rate (above 60–70%) means you are keeping sessions focused and your context prefix is stable — both good habits. A low hit rate is a signal to investigate: are you starting too many new sessions? Going idle too long?
- **Attribute cost by model.** The report script breaks totals down by model. If you see a lot of Opus tokens on tasks that do not need deep reasoning, those could probably move to Sonnet. See the model routing section of [readme.md](readme.md) for guidance on which task types belong on which model.

### Right-size your context

- **Reference files instead of pasting them.** Claude can read files directly from disk. Giving a file path uses far fewer tokens than pasting the file's contents into your message.
- **Use `/clear` between unrelated tasks.** Starting a new topic in a long existing session carries the full weight of everything that came before it. Clear the context and start fresh.
- **Scope subagents tightly.** Each subagent gets only the context you give it at invocation time. A well-scoped invocation message is cheaper than a vague one that causes the agent to read many extra files to figure out what you want.
- **Plan before building.** A brief planning exchange with a planner agent costs relatively few tokens. Catching a misunderstanding at planning time is far cheaper than re-doing an implementation.

### Set budgets and watch trends

- Run the PowerShell report weekly to establish a baseline. Watch for spikes — a sudden jump in output tokens often means a session ran longer or produced more verbose responses than necessary.
- Set a mental (or a literal) budget per project. If a feature costs significantly more in tokens than you expected, that is a signal to revisit your workflow, not just to accept the bill.
- Do not over-optimize. Chasing the lowest possible token count by making prompts cryptically terse, switching to weaker models for tasks that need quality, or interrupting agents mid-task to "save tokens" will cost you more in re-runs and mistakes than you save. Optimize the workflow, not the word count.

### Privacy and security

The JSONL files under `~/.claude/projects/` contain the full text of your prompts, the files you asked Claude to read, and Claude's responses. This means they may contain:

- source code and configuration files from your projects,
- API keys or secrets if you accidentally pasted them into a prompt,
- any other sensitive content you discussed in a session.

**Treat these log files as sensitive data:**

- Do not commit the `~/.claude/projects/` directory to version control.
- Do not share raw log files without reviewing and scrubbing them first.
- If you are working on a machine shared with others, be aware that these files sit under your user profile and are readable by any process running as you.
- If you need to share logs for debugging (for example, with a support team), open the file first and manually remove any credentials, personal data, or proprietary code before sharing.

---

## 8. Quick-reference card

| Goal | Command or script | How often |
|---|---|---|
| Check cost for the current session | `/cost` in the Claude Code REPL | During any active session |
| See how full the context window is | `/context` in the REPL | When a session feels sluggish or confused |
| Check session status and model | `/status` in the REPL | As needed |
| Report totals across all past sessions | `powershell -ExecutionPolicy Bypass -File token-report.ps1` | Weekly, or after a heavy work period |
| Watch a single session live | `powershell -ExecutionPolicy Bypass -File token-watch.ps1 "<path-to-session.jsonl>"` | During a long-running task |
| Node-based totals report | `node token-report.js` | Same as PowerShell report — use whichever runtime you have |
| Third-party live dashboard | `npx ccusage@latest blocks --live` | During active development if you want a richer UI |
| Historical summary (third-party) | `npx ccusage@latest` | Weekly |
| Find your current session file | `Get-ChildItem "$env:USERPROFILE\.claude\projects" -Recurse -Filter *.jsonl \| Sort-Object LastWriteTime -Descending \| Select-Object -First 1` | When you need to pass the path to the watcher script |

---

## 9. Glossary

**Input tokens** — Tokens you send to the model: your prompt text, conversation history, file contents, and system instructions. Charged per token at the input rate.

**Output tokens** — Tokens the model generates in its reply. Typically the most expensive per token. Long, verbose responses cost more than short, focused ones.

**Cache read (`cache_read_input_tokens`)** — Input tokens that were served from a warm prompt cache rather than re-processed from scratch. Much cheaper than fresh input tokens. A high cache read count is a good sign.

**Cache creation (`cache_creation_input_tokens`)** — Tokens written into the prompt cache for the first time. You pay a one-time cost to create the cache entry; subsequent turns that hit the cache pay the cheaper cache-read rate instead.

**Context window** — The model's working memory for one conversation. It has a fixed maximum size. When it fills up, the oldest content is no longer visible to the model. The `/context` command shows you how full it is.

**TTL (Time To Live)** — The duration a prompt cache entry stays warm. Claude Code's default cache TTL is approximately 5 minutes; a 1-hour tier also exists. Once the TTL expires, the next request re-processes context as fresh input.

**OTLP / OpenTelemetry** — OpenTelemetry is an open standard for collecting and exporting metrics, traces, and logs from applications. OTLP is the wire protocol it uses. When `CLAUDE_CODE_ENABLE_TELEMETRY=1` is set, Claude Code emits token and cost metrics in this format, which can be routed to Prometheus, Grafana, Datadog, or any compatible backend.

**Service tier** — A field (`service_tier`) that appears alongside `message.usage` in Claude Code's JSONL logs. It indicates which processing tier the request used. The scripts in this guide read it from the log but do not currently use it in calculations — it is available if you want to extend the scripts.

**ccusage** — A third-party, community-built npm tool that reads Claude Code's local JSONL session logs and produces usage reports. Run with `npx ccusage@latest`. Not made by Anthropic; check its npm page for current documentation and version notes.

**JSONL** — JSON Lines: a file format where each line is a self-contained, valid JSON object. Claude Code writes one JSON object per message turn to its session transcript files. The scripts in this guide parse these files line by line.
