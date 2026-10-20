# Comprendo

**Stop AI from writing code you don't understand.**

Comprendo is a Claude Code plugin that forces a review step between "plan" and "implement." When you type `/comprendo <task>`, Claude reads your codebase, builds a structured plan with architecture diagrams and code previews, then opens a browser UI where you review everything before a single file is touched.

No more coming back to a 12-file diff you have to reverse-engineer. You see exactly what's about to change, why, and how — then you decide.

## The problem

AI coding assistants are fast. Sometimes too fast. You describe a change, and 30 seconds later you're staring at a diff across 8 files wondering:

- What was the reasoning behind this approach?
- Which components were affected and how do they connect?
- Did it change something I didn't expect?
- Do I actually understand what just happened to my codebase?

Most developers approve changes they don't fully understand because the alternative — reading every diff line and mentally reconstructing the architecture — takes longer than the AI saved them.

## What Comprendo does

Comprendo adds a gate between planning and execution. When you use `/comprendo` instead of asking Claude directly:

1. **Claude reads your code** — explores the relevant files and understands the current state
2. **Claude builds a plan** — structured JSON with a summary, explanation, before/after architecture diagrams, code snippets, and affected files
3. **A browser UI opens** — you see everything laid out visually at `localhost:2345`
4. **You decide** — click Approve to let Claude implement, or Reject to go back and adjust

No files are written until you explicitly approve.

## The review UI

The browser shows a single-page dark-themed review with:

- **Side-by-side architecture diagrams** — current state vs proposed state. New components are highlighted in purple. Rendered as SVG directly from the plan data — no fragile text-based syntax.
- **Plain English explanation** — 2-3 paragraphs describing what exists now, what's changing, and why this approach was chosen. Written for a developer who hasn't looked at this code in 3 weeks.
- **Code snippets** — before/after toggle with syntax highlighting showing the key changes
- **Files affected** — listed as pills so you know the blast radius
- **Approve / Reject** — sends the decision back to Claude Code

## Install

### As a Claude Code plugin (recommended)

```
/plugin marketplace add nikhilenmudi/comprendo
/plugin install comprendo
/reload-plugins
```

That's it. No npm install, no build step, no dependencies.

### Manual install

```bash
git clone https://github.com/nikhilenmudi/comprendo.git
cd comprendo
npm link

mkdir -p ~/.claude/commands
cp skills/comprendo/SKILL.md ~/.claude/commands/comprendo.md
```

## Usage

```
/comprendo add retry logic to the API client
/comprendo refactor the auth middleware into a service class
/comprendo add a /health endpoint that checks MongoDB connection status
```

## How it works

```
Developer types /comprendo <task>
        |
        v
Claude reads relevant files, builds structured JSON plan
        |
        v
CLI starts local HTTP server, injects plan into review UI
        |
        v
Browser opens at localhost:2345
        |
        v
Developer reviews diagrams, explanation, code, files
        |
   ┌────┴────┐
   v         v
Approve    Reject
(exit 0)   (exit 1)
   |         |
   v         v
Claude     Claude stops,
implements  asks what to
as planned  change
```

**Under the hood:**
- `skills/comprendo/SKILL.md` — instructs Claude to plan first, output structured JSON, then pipe it to the CLI
- `cli.js` — starts a local HTTP server, injects the plan data into the UI, waits for the developer's decision via POST endpoint
- `ui.html` — single-page review UI with custom SVG diagram renderer and highlight.js for code

**Zero dependencies.** The CLI uses Node.js built-in `http` module. Diagrams are rendered as SVG directly from JSON — no Mermaid, no D3, no chart library. The only CDN dependency is highlight.js for syntax coloring.

## Design decisions

- **Browser UI, not terminal** — architecture diagrams and code diffs need space. A terminal can't render side-by-side diagrams with color-coded nodes.
- **Custom SVG renderer, not Mermaid** — Mermaid uses a text-based DSL that breaks on labels containing `()`, `/`, `{}`, and other common characters in code. Our renderer takes JSON directly. No syntax to break.
- **5-minute timeout auto-rejects** — if you walk away, nothing gets written. The safe default is to do nothing.
- **Port fallback** — tries 2345, then 2346-2349. Won't crash if another instance is running.

## Roadmap

- [ ] Quiz mode — test your understanding of the proposed change before approving
- [ ] Audit trail — log all approved/rejected plans for team review
- [ ] Significance classifier — auto-detect high-risk changes and require extra scrutiny

## License

MIT
