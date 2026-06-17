# Claude Code CLI Cheat Sheet

See [readme.md](readme.md) for the concepts behind these commands.

---

## 1. CLI Basics

"Interactive mode" means Claude Code opens a chat-style REPL (read-eval-print loop — a live back-and-forth session in your terminal). "Headless" or "one-shot" means Claude runs a single prompt and exits, useful for scripts.

| Action | Command | Notes |
|---|---|---|
| Launch interactive session | `claude` | Opens the REPL in your current directory |
| Run one prompt and exit (headless) | `claude -p "your prompt here"` | No interactive session; output goes to stdout |
| Continue the most recent session | `claude --continue` | Picks up where you left off |
| Resume a specific past session | `claude --resume` | Shows a list of past sessions to choose from |
| Exit the REPL | `/exit` or `Ctrl+C` | Either works inside an active session |
| MCP subcommands | `claude mcp ...` | Manage Model Context Protocol servers — see Section 4 |

> **Reality-check:** `--continue` and `--resume` reload context from disk, but very long conversations can still hit context-window limits. If a resumed session feels confused, start fresh with `/clear`.

---

## 2. Slash-Command Cheat Sheet

Slash commands are typed directly inside the Claude Code REPL. They are not shell commands — they only work once Claude is running.

### Setup & Configuration

| Command | What it does | When to use |
|---|---|---|
| `/config` | Opens Claude Code settings | Change model, theme, API key, output behavior |
| `/update-config` | Edit permissions and hooks | When you want to grant/restrict file access or add automation hooks |
| `/keybindings-help` | Lists keyboard shortcuts | When you forget how to navigate the REPL |
| `/init` | Scaffolds a `CLAUDE.md` in the current project | First time setting up a new project so Claude knows its context |
| `/memory` | View and edit Claude's persistent memory | Add facts you want Claude to remember across sessions |

### Review & Security

| Command | What it does | When to use |
|---|---|---|
| `/code-review` or `/review` | Runs a general code review on specified files | Before committing; general quality check |
| `/security-review` | Runs a security-focused review | Before pushing code that handles auth, secrets, or user input |
| `/verify` | Confirms a piece of code or logic does what you think | Sanity-checking before a big change |
| `/simplify` | Asks Claude to simplify the current code or explanation | When output is too complex or verbose |

### Automation & Flow

| Command | What it does | When to use |
|---|---|---|
| `/loop` | Runs a task in a feedback loop until it passes a condition | Iterative tasks like "fix until tests pass" |
| `/schedule` | Creates a scheduled/cron task inside the session | See the cron caveat in Section 5 before relying on this |
| `/run` | Executes a named task or script within Claude's context | Running predefined workflows |
| `/fast` | Enables fast output mode | When you want quicker responses on Opus — does NOT downgrade the model |

> **Note on `/fast`:** This mode is available on Opus 4.8, 4.7, and 4.6. It produces output faster but still uses the full Opus model — you are not switching to a cheaper or weaker model.

### Agents & Session

| Command | What it does | When to use |
|---|---|---|
| `/agents` | List, inspect, and manage your subagents | Before invoking an agent, or to check what agents exist |
| `/clear` | Clears the current conversation context | Between unrelated tasks to avoid context bleed |
| `/fewer-permission-prompts` | Reduces how often Claude asks for approval | When repetitive prompts are slowing you down (use with care) |

---

## 3. Subagent Invocation from Chat

A subagent is a specialized Claude Code instance defined by a Markdown file. Each agent has its own instructions, persona, and scope. They start "cold" — meaning they have no memory of your current session — so you must give them full context when you invoke them.

**The invocation grammar:**

```
Use the `<agent-name>` subagent to <describe the task in full detail>.
```

**Examples:**

```
Use the `security-reviewer` subagent to check C:\Users\<username>\Projects\tokenbestpractices for hardcoded secrets or insecure token handling.
```

```
Use the `doc-writer` subagent to write inline comments for every function in C:\Users\<username>\Projects\tokenbestpractices\utils.js.
```

**Tips:**
- Include file paths, goal, and any relevant constraints in the same message — the agent starts with no context from your chat.
- Use `/agents` inside the REPL to browse available agents and see their descriptions before invoking.
- Agents run at user scope, so they can access any project under your user profile unless further restricted.

> **Your setup:** You have 16 subagents at user scope, stored in `C:\Users\<username>\.claude\agents\*.md`. They break down as 6 security agents, 7 dev/SDLC agents, and 3 docs agents. Manage them with `/agents` in the REPL.

---

## 4. MCP Management

MCP stands for Model Context Protocol — a standard that lets Claude connect to external tools and data sources (servers). You manage MCP servers from the terminal, outside the REPL.

| Action | Command | Notes |
|---|---|---|
| Add a server | `claude mcp add <name> <command-or-url>` | Registers the server; saved to `~/.claude.json` |
| List configured servers | `claude mcp list` | Shows all registered MCP servers |
| Get details on one server | `claude mcp get <name>` | Shows the stored config for a specific server |
| Remove a server | `claude mcp remove <name>` | Unregisters it; does not uninstall anything |

**Config file location:** `C:\Users\<username>\.claude.json` (the `~` shorthand points here on Windows too)

**Example — adding a filesystem server scoped to your Projects folder:**

```powershell
claude mcp add filesystem "npx @modelcontextprotocol/server-filesystem C:\Users\<username>\Projects"
```

> **Your setup:** You have three MCP servers already configured:
>
> | Server | What it does |
> |---|---|
> | `filesystem` | Scoped to `C:\Users\<username>\Projects` — Claude can read/write files there |
> | `memory` | Gives Claude a persistent key-value memory store across sessions |
> | `github` | Remote-hosted, OAuth authenticated — lets Claude read/write GitHub repos |
>
> You do not need to re-add these. Use `claude mcp list` to confirm they are still registered.

---

## 5. Headless & Automation

Headless mode lets you run Claude from a script or scheduled job without opening an interactive session. The output is plain text to stdout, which you can pipe, log, or email.

**Basic pattern:**

```powershell
claude -p "Summarize the file at C:\Users\<username>\Projects\tokenbestpractices\readme.md in three bullet points."
```

**Piping output to a file:**

```powershell
claude -p "Review C:\Users\<username>\Projects\myapp\app.js for security issues." > C:\Users\<username>\Projects\myapp\review.txt
```

---

### ⚠️ Cron Caveat — Session-Only Scheduling on This Machine

The built-in `/schedule` command and `CronCreate` tasks in Claude Code are **session-only** on your machine. This is because the feature flag `tengu_kairos_cron_durable` is set to `false` here.

**What that means in plain language:**
- Any task you schedule with `/schedule` will only fire while Claude Code is open and idle.
- The moment you close the app, the scheduled task disappears. It will not run overnight or survive a reboot.

**For tasks that must run unattended, use Windows Task Scheduler instead.** Here is a minimal example using PowerShell's `Register-ScheduledTask`:

```powershell
# Run a Claude headless prompt every morning at 8:00 AM
$action = New-ScheduledTaskAction -Execute "claude" -Argument '-p "Check C:\Users\<username>\Projects\tokenbestpractices for any TODO comments and log them." >> C:\Users\<username>\Projects\logs\daily-check.txt'

$trigger = New-ScheduledTaskTrigger -Daily -At "08:00AM"

Register-ScheduledTask -TaskName "ClaudeDailyCheck" -Action $action -Trigger $trigger -RunLevel Highest
```

> **Reality-check:** This assumes `claude` is on your system PATH. To confirm: open PowerShell and run `claude --version`. If you get an error, use the full path to the Claude executable instead (e.g., `C:\Users\<username>\AppData\...`). Also make sure your API key is available as an environment variable in the scheduled task's session — Task Scheduler does not always inherit your interactive shell's environment variables.

**Alternative:** A cloud-side routine (GitHub Actions, Azure Logic Apps, etc.) calling `claude -p` via a CI runner is more reliable for overnight or recurring work.

---

## 6. Token-Efficient CLI Habits

Tokens are the units Claude "reads and writes" — context has a limit, and sloppy habits burn through it fast.

- **Use `/clear` between unrelated tasks.** Each new topic should start with a clean slate. Carrying old context into a new task wastes tokens and can confuse Claude.
- **Plan before big changes.** Ask Claude to describe what it will do before it does it (e.g., "outline the changes you'd make to X, don't make them yet"). This catches misunderstandings cheaply.
- **Use `/fast` on Opus when you want speed.** It does not lower the model quality — same Opus, faster output. Good for exploratory back-and-forth.
- **Batch related work into one session.** "Review functions A, B, and C" in one prompt is more efficient than three separate prompts.
- **Scope your MCP filesystem access tightly.** Your filesystem MCP is already scoped to `C:\Users\<username>\Projects`. Avoid broadening it — a narrower scope means Claude reads less irrelevant content per request.
- **Avoid going idle inside a long session.** Claude caches your prompt context for roughly 5 minutes of inactivity. If you step away longer than that, the cache may expire and tokens are re-billed on your next message. Either `/clear` and start fresh, or keep the pace up.

---

## 7. Customization Pointer

| What you want to change | Command to use |
|---|---|
| Model, theme, output format, API key | `/config` |
| File/tool permissions, automation hooks | `/update-config` |
| Keyboard shortcuts inside the REPL | `/keybindings-help` |
| Project-level Claude instructions | `/init` (creates `CLAUDE.md` in your project root) |
| Persistent facts Claude remembers across sessions | `/memory` |

> **Tip:** `CLAUDE.md` is the best place to store project-specific context (tech stack, conventions, file layout). Claude reads it automatically at the start of each session in that directory, which saves you from re-explaining the same things every time.
