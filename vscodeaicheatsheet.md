# Claude Code in VS Code — Cheat Sheet

Companion to [readme.md](readme.md) (concepts) and [readmecli.md](readmecli.md) (CLI).

---

## 1. What this is / how to read it

This guide shows you how to use Claude Code as a VS Code extension — not just as a terminal tool — so you can take advantage of everything the IDE gives you: clickable file references, inline diff previews, selection-as-context, and a chat panel that is aware of what you are looking at.

You do not need prior experience with AI coding tools. Every concept is explained before it is used.

**How the "Your setup" callouts work**

Throughout this guide you will see blockquotes titled **"🔧 Your setup"**. Each one takes a concept that was just explained generically and maps it to your real environment — your actual file paths, agent names, and installed tools. You can follow the guide without them; read them when you want to see exactly how the idea applies to you.

---

## 2. Claude Code inside VS Code — the basics

**What is the VS Code extension?** The Claude Code VS Code extension puts a dedicated chat panel inside the editor. It is the same Claude Code you use in the terminal, but it gains extra powers from living inside VS Code: it can see what file you have open, read whatever text you have highlighted, and show its proposed edits as inline diffs inside the editor rather than printing raw text to a terminal.

**How to open it:** Click the Claude Code icon in the Activity Bar (the vertical icon strip on the left side of VS Code), or use the Command Palette (`Ctrl+Shift+P`) and type `Claude Code`.

**What it can do for you:**
- Read files anywhere in your workspace and propose edits.
- Show edits as a side-by-side diff so you can accept or reject changes before they are written.
- Receive your currently highlighted code or text as automatic context — no copy-pasting needed.
- Reference files by clickable links inside the chat, so you can jump straight to the file it is talking about.

### Terminal CLI vs VS Code extension — when to use which

| Situation | Better choice | Why |
|---|---|---|
| Scripting, automation, headless runs | Terminal CLI (`claude -p "..."`) | No UI needed; output pipes to stdout |
| Working on code you can see in the editor | VS Code extension | Inline diffs, selection context, clickable file links |
| Multi-file refactors with visual review | VS Code extension | You can review each file's diff before accepting |
| Scheduled or overnight tasks | Terminal CLI (+ Windows Task Scheduler) | Extension is tied to a running VS Code window |
| Quick one-off questions | Either — use whichever is already open | No meaningful difference |
| Invoking subagents during active development | VS Code extension | You stay in one place while agents do their work |

> **🔧 Your setup**
>
> You have Claude Code installed as a native VS Code extension on Windows 11. Node.js 24.16.0 is installed and powers it. Python and Docker are NOT installed (only a Windows Store stub exists) — see Section 6 before installing Python or Docker-related extensions.

---

## 3. The VS Code views that make Claude powerful

Each panel in VS Code can feed Claude information or let you review what Claude did. This table maps each view to how you should use it with Claude.

| VS Code view | Where to find it | How it pairs with Claude |
|---|---|---|
| **Explorer** | Activity Bar → file icon | Browse to a file, right-click it, or drag its name into chat to give Claude a file reference without pasting the whole file |
| **Editor + Selection** | Center — the main editing area | Highlight any code or text; that selection is automatically read as context when you ask Claude something in the chat panel |
| **Source Control / diff view** | Activity Bar → branch icon | After Claude edits a file, the diff appears here — review exactly what changed line by line before accepting |
| **Problems panel** | Menu → View → Problems (`Ctrl+Shift+M`) | When ESLint or the TypeScript checker flags an error, copy the error text and paste it into Claude chat — or describe the error and Claude will read the file |
| **Integrated Terminal** | Menu → View → Terminal (`Ctrl+\``) | Run commands Claude suggests, see test output, or run the CLI directly if you need headless mode |
| **Search** (`Ctrl+Shift+F`) | Activity Bar → magnifier icon | Find a symbol or pattern, then reference the result in Claude chat with the file path and line number |
| **Claude chat panel** | Activity Bar → Claude icon | Your main interface — prompt Claude, review responses, invoke subagents, use slash commands |

### Selection as context — explained with an example

"Selection as context" means that when you highlight text in the editor and then type a question in the Claude chat panel, Claude receives the highlighted text as part of your question automatically. You do not need to paste it.

**Example — without selection:**
> You: "Why does the authentication check fail?"
> Claude has to guess which file and which function you mean. It may ask for clarification or read the wrong thing.

**Example — with selection:**
> You open `auth.js`. You highlight the `validateToken` function (10 lines). You type in Claude chat: "Why does this function return false when the token is valid?"
> Claude sees the exact 10 lines you highlighted. It answers with specific references to your code.

**The rule:** Before asking Claude about a specific piece of code, highlight it first. This scopes the question to exactly what you care about and saves tokens — Claude is not guessing or reading an entire file.

---

## 4. Intelligent prompting (cheat sheet)

Good prompts give Claude a clear goal, the right constraints, and a way to know when it is done. Vague prompts produce generic answers that need follow-up, which wastes time and tokens.

### Prompting patterns that work well in VS Code

**1. Give a goal + constraints + acceptance criteria**

Tell Claude what you want, what it should not do, and how you will know it worked.

```
Refactor the `parseConfig` function in C:\Users\pstib\Projects\myapp\config.js
to handle missing keys gracefully. Do not change the function signature.
The function should return a default value instead of throwing. Add a comment
explaining each new guard clause.
```

**2. Reference files and lines instead of pasting**

Claude can read files directly. Give it a path and a line number rather than copying and pasting large blocks of code into chat.

```
Look at C:\Users\pstib\Projects\myapp\utils.js lines 45–78.
The `formatDate` function has an off-by-one error on leap years. Fix it.
```

**3. Highlight to scope a question**

Highlight a function in the editor, then ask a focused question in chat. Claude sees the selection without you typing it out. (See Section 3 for a worked example.)

**4. Ask for a plan first on anything non-trivial (plan mode)**

Before Claude edits multiple files, ask it to describe the changes it intends to make. Review the plan, then tell it to proceed. This is "plan mode" — Claude researches and writes a plan, you approve, then it executes.

```
Before making any changes: describe in plain language what files you will edit,
what you will change in each, and why. Wait for my approval before starting.
```

**5. Name the right subagent for the job**

If a task matches a specialist, say so explicitly. The agent starts cold — give it full context.

```
Use the `code-reviewer` subagent to review
C:\Users\pstib\Projects\myapp\auth.js for security issues.
Focus on token validation and session handling.
```

**6. Iterate in small steps**

Ask Claude to make one change, review the diff in Source Control, confirm it looks right, then ask for the next change. Small steps are easier to review and easier to revert if something goes wrong.

**7. Tell Claude how to verify**

If there is a way to confirm the change worked, say so.

```
After updating the function, show me a test call that proves the edge case
is handled correctly.
```

### Weak prompt → stronger prompt

| Weak prompt | Stronger prompt | What improved |
|---|---|---|
| "Fix the bug in my code." | "The `calculateTotal` function in `cart.js` line 34 returns NaN when `quantity` is undefined. Fix it so it treats undefined as 0." | Specific file, line, and symptom |
| "Make this better." | "Highlight the `getUserById` function → Simplify this function: reduce nesting, keep the same return type, do not change the SQL query." | Scoped via selection; clear goal and constraints |
| "Write tests." | "Write Jest unit tests for the three functions in `validators.js`. Cover the happy path and at least one edge case per function. Put the tests in `validators.test.js`." | Named test framework, scope, output location |

---

## 5. Best practices for building with Claude in VS Code

A consistent workflow prevents Claude from making changes you did not intend, keeps your git history clean, and makes it easy to review and roll back.

### The practical workflow

**Start every project with `/init` or a `CLAUDE.md`**

Run `/init` in the Claude chat panel when you begin a new project. Claude will ask a few questions and scaffold a `CLAUDE.md` file in the project root. This file tells Claude about your project every time it opens — your stack, conventions, and what it should and should not do. A project without `CLAUDE.md` makes Claude guess context every session. (See Section 7 for a ready-to-copy `CLAUDE.md` template.)

**Use plan mode for anything that touches more than one file**

Before Claude edits multiple files at once, ask for a plan first (see the prompting pattern in Section 4). Read the plan carefully. If something looks wrong, correct it before a single file is touched.

**Let `tech-lead` plan multi-step work, then run specialists**

For larger tasks — building a new feature, restructuring a module, setting up a workflow — ask `tech-lead` to produce a delegation plan first. It will output a step-by-step table with the right specialist for each step and which steps can run in parallel. Then you fire the specialists one by one from the main chat.

> **🔧 Your setup**
>
> Your `tech-lead` agent runs on `claude-opus-4-8` (deepest reasoning) and is strictly read-only — it plans but never edits files. You have 15 other agents waiting: 6 security specialists, 6 dev/SDLC doers, and 3 docs agents. For a multi-step job, the sequence is: ask `tech-lead` to plan → review the plan → fire each specialist in order. Invoke any agent with: *"Use the `<name>` subagent to `<self-contained task with full context>`."* Agents start cold — always include the file path(s) and goal in the same message.

**Review every diff in Source Control before accepting**

After Claude edits a file, open the Source Control panel (`Ctrl+Shift+G`). VS Code shows you a line-by-line diff of everything that changed. Read it. You are responsible for what goes into your codebase — Claude makes mistakes. Do not accept changes you have not read.

**Keep changes small and verifiable**

One change at a time is easier to review, easier to test, and easier to revert. If Claude tries to rewrite three files in one pass on a non-trivial task, push back: "Make only the change to `auth.js` first. I will review before you touch anything else."

**Use `/clear` between unrelated tasks**

When you finish one task and move to something different, type `/clear` in the chat panel. This resets the conversation context. Carrying an old conversation into a new task wastes tokens and can cause Claude to make assumptions from the previous task that do not apply to the new one.

**Commit often**

Every time Claude completes a discrete change that works and you have reviewed it, commit it. This gives you a clean restore point. Waiting until the end of a long session means a single bad change can be hard to isolate.

> **🔧 Your setup**
>
> Your filesystem MCP is scoped to `C:\Users\pstib\Projects`. Claude can read and write any file under that folder. Keeping your MCP scope narrow means Claude reads less irrelevant content per request — this is already configured correctly in `~\.claude.json`. Do not broaden it unless you have a specific reason.

---

## 6. Recommended VS Code extensions

These extensions pair well with AI-assisted development and this repo's markdown-heavy workflow. Install them from the Extensions panel (`Ctrl+Shift+X`) by searching the Marketplace ID.

| Extension | Marketplace ID | Why it helps | Required runtime |
|---|---|---|---|
| GitLens | `eamodio.gitlens` | Inline blame, rich diff history — makes reviewing Claude's git changes easier | None (built on VS Code APIs) |
| Markdown All in One | `yzhang.markdown-all-in-one` | Table formatting, TOC generation, keyboard shortcuts for markdown — useful since this repo is mostly `.md` files | None |
| Markdown Preview Mermaid Support | `bierner.markdown-mermaid` | Renders Mermaid diagrams (flowcharts, sequence diagrams) in the VS Code preview — `readme.md` uses Mermaid | None |
| Code Spell Checker | `streetsidesoftware.code-spell-checker` | Catches typos in code and docs — catches things Claude occasionally misses | None |
| Error Lens | `usernamehw.errorlens` | Shows ESLint/TypeScript errors inline on the line they occur — makes it easier to copy an error into Claude chat | None |
| EditorConfig | `editorconfig.editorconfig` | Reads `.editorconfig` files to enforce consistent indentation and line endings across contributors and AI edits | None |
| Prettier | `esbenp.prettier-vscode` | Auto-formats JavaScript, TypeScript, JSON, and Markdown on save — keeps Claude's output consistently formatted | Node.js (already installed) |
| ESLint | `dbaeumer.vscode-eslint` | Flags JavaScript/TypeScript code quality issues inline — feeds the Problems panel that you can then describe to Claude | Node.js (already installed) |
| Python | `ms-python.python` | Python language support, linting, debugging | **Optional — only install after installing Python. A Windows Store stub is present but will not work with this extension.** |
| Docker | `ms-azuretools.vscode-docker` | Manage Docker containers and images from VS Code | **Optional — only install after installing Docker Desktop. Docker is not installed on this machine.** |

> **Reality-check:** The Python and Docker extensions will show errors or fail silently if the corresponding runtime is not installed. Do not install them yet — they will add noise to the Problems panel without benefit. Install them only after you install the actual runtime.

---

## 7. A starter template you can drop into a fresh project

Copy these three files into any new project to get consistent settings, recommended extensions, and Claude context in place from the start.

### How to use this template (3 steps)

1. Create the files below in your project folder (create a `.vscode/` folder if it does not exist, and place `extensions.json` and `settings.json` inside it; place `CLAUDE.md` in the project root).
2. Close and reopen VS Code in that folder. VS Code will detect `.vscode/extensions.json` and prompt you to install the recommended extensions — click "Install All".
3. Open the Claude chat panel and run `/init`, or fill in the placeholder sections in `CLAUDE.md` manually so Claude knows your project from the first session.

---

### `.vscode/extensions.json`

> Note: this is valid JSON. Comments are not allowed in `.json` files — if you want to add comments, rename this file to `extensions.jsonc` and change the reference in any tooling accordingly. VS Code reads both formats.

```json
{
  "recommendations": [
    "eamodio.gitlens",
    "yzhang.markdown-all-in-one",
    "bierner.markdown-mermaid",
    "streetsidesoftware.code-spell-checker",
    "usernamehw.errorlens",
    "editorconfig.editorconfig",
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint"
  ]
}
```

Python (`ms-python.python`) and Docker (`ms-azuretools.vscode-docker`) are intentionally excluded — add them only after installing the respective runtime.

---

### `.vscode/settings.json`

> Note: this is plain JSON — comments are not allowed. If you want to add comments to explain settings, rename it `settings.jsonc`. VS Code reads both.

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.trimAutoWhitespace": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "editor.wordWrap": "on",
  "[markdown]": {
    "editor.wordWrapColumn": 100,
    "editor.quickSuggestions": {
      "other": false,
      "comments": false,
      "strings": false
    }
  },
  "editor.rulers": [100],
  "editor.tabSize": 2,
  "editor.insertSpaces": true
}
```

---

### `CLAUDE.md`

> What is `CLAUDE.md`? It is a plain Markdown file that Claude reads automatically at the start of every session in this project folder. Use it to tell Claude about your project once, so you do not have to repeat yourself every time. You can generate a first draft with `/init` in the Claude chat panel — it will ask you questions and scaffold the file. The template below shows what to include.

```markdown
# CLAUDE.md — Project Instructions

<!-- This file is read by Claude Code at the start of every session.
     Fill in each section. The more specific you are, the better Claude's output.
     Run `/init` in the Claude chat panel to auto-generate a version of this file. -->

## Project overview

<!-- What does this project do? Who uses it? One short paragraph. -->

## How to run

<!-- List the commands needed to install dependencies, start the app, and run tests.
     Be specific — include the actual commands, not just descriptions. -->

```powershell
# Example — replace with your real commands
npm install
npm run dev
npm test
```

## Conventions

<!-- List the coding style rules, naming conventions, and patterns this project follows.
     Examples:
     - Use camelCase for variables, PascalCase for components
     - All async functions must handle errors with try/catch
     - No hardcoded strings — use the constants file
     - Prettier + ESLint rules are in .prettierrc and .eslintrc -->

## Subagents to use

<!-- List which subagents from your bench are relevant to this project and when to use them.
     Examples: -->

- `tech-lead` — use first for any task that touches more than one file; ask it to produce a delegation plan
- `implementer` — for writing and editing code
- `code-reviewer` — read-only review before committing
- `tech-writer` — for updating docs, comments, and ADRs
- `debugger` — when something is broken and you need focused diagnosis

## Definition of done

<!-- What must be true before a change is considered complete?
     Examples:
     - Code passes ESLint with no errors
     - All existing tests still pass
     - New behavior has at least one test
     - CLAUDE.md and any affected docs are updated
     - Change is committed with a clear commit message -->
```

---

## 8. Token-efficient habits specific to VS Code

The VS Code extension gives you tools that make token efficiency easier than in a raw terminal session — use them.

- **Highlight, don't paste.** Selecting code in the editor and asking a question in chat is cheaper and more precise than copying code into your message. The selection is sent automatically.
- **Reference files by path.** Type or paste a file path in chat (`C:\Users\pstib\Projects\myapp\auth.js`) — Claude reads the file directly. Do not paste the entire file's contents unless Claude specifically needs content it cannot read from disk.
- **Review diffs instead of asking Claude to re-explain.** After Claude edits a file, read the diff in Source Control. Do not ask Claude "what did you change?" — that costs tokens to answer something you can see for free.
- **`/clear` between unrelated tasks.** Switching topics without clearing carries old context into the new task — wasted tokens and potential confusion.
- **Mind the 5-minute prompt-cache window.** If you are running a multi-step task, keep the pace up. Pausing for more than 5 minutes causes the cache to expire, and the next message re-reads all the context from scratch (slower, more expensive). If you know you need a long break, finish the current step cleanly and `/clear` before you stop.
- **`/fast` for speed on Opus.** When you want faster responses during back-and-forth exploration, `/fast` speeds up output without downgrading the model. It still runs the full Opus model — it just returns tokens faster.
- **Batch related questions.** "Review functions A, B, and C for the same issue" in one message is more efficient than three separate messages.

---

## 9. Quick-reference card

| What you want to do | The move |
|---|---|
| Scope a question to specific code | Highlight the code in the editor first, then ask in chat |
| Avoid pasting large files | Reference by file path — Claude reads it directly |
| Make a non-trivial change safely | Ask for a plan first ("describe what you'll change before changing it") |
| Multi-file or multi-step work | Ask `tech-lead` to produce a delegation plan, then run specialists |
| Review what Claude changed | Open Source Control (`Ctrl+Shift+G`) and read the diff |
| Feed an error to Claude | Copy the text from the Problems panel into chat, or describe it with the file path |
| Switch to a different task | `/clear` to reset context before starting |
| Keep responses fast on Opus | `/fast` |
| Set up a new project | Run `/init` in chat — it creates `CLAUDE.md` |
| Check what agents you have | `/agents` in the chat panel |
| Invoke a specialist | `Use the <name> subagent to <task with full context>` |
| Commit a good change | Do it immediately — don't wait until the end of a long session |

---

*See [readme.md](readme.md) for token-efficiency concepts and agent anatomy. See [readmecli.md](readmecli.md) for CLI commands, MCP management, and headless automation.*
