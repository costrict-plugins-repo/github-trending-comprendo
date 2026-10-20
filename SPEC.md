# Comprendo Phase 1 — Corrected Spec

Build Comprendo Phase 1 — a Claude Code plugin that opens a local browser UI showing before/after architecture diagrams and an approve/reject flow before any code is written.

## What you're building

When a developer types `/comprendo <task>`, Claude reads the relevant files, generates a structured plan including diagram data, then the Comprendo CLI opens a browser tab at localhost:2345 showing a visual review UI. The developer reads the before/after diagrams, reviews the explanation and code snippets, then clicks Approve or Reject. The browser response is sent back to the CLI which exits 0 or 1 to Claude Code.

No quiz in Phase 1. Just the visual review experience.

## Deliverables

### 1. `~/.claude/commands/comprendo.md`

Slash command that instructs Claude to:

- Read all files relevant to the requested change
- Output a JSON block wrapped in `<comprendo_data>` tags with this exact structure:

```json
{
  "summary": "One sentence: what this change does",
  "explanation": "2-3 paragraphs in plain English. What exists now, why it's a problem or limitation, what the change does and why this approach. Write for a developer who hasn't looked at this code in 3 weeks.",
  "current_diagram": {
    "nodes": [
      { "id": "middleware", "label": "Auth Middleware", "type": "component" },
      { "id": "routes", "label": "Route Handlers", "type": "component" }
    ],
    "edges": [
      { "from": "middleware", "to": "routes", "label": "passes req" }
    ]
  },
  "updated_diagram": {
    "nodes": [
      { "id": "middleware", "label": "Auth Middleware", "type": "component" },
      { "id": "authservice", "label": "AuthService", "type": "service" },
      { "id": "routes", "label": "Route Handlers", "type": "component" }
    ],
    "edges": [
      { "from": "middleware", "to": "authservice", "label": "delegates to" },
      { "from": "authservice", "to": "routes", "label": "passes req" }
    ]
  },
  "snippets": [
    {
      "label": "Before — middleware.ts",
      "language": "typescript",
      "code": "// key before snippet, max 20 lines"
    },
    {
      "label": "After — middleware.ts",
      "language": "typescript",
      "code": "// key after snippet, max 20 lines"
    }
  ],
  "files_affected": ["src/middleware.ts", "src/services/AuthService.ts"]
}
```

- After outputting the JSON block, pipe it to `comprendo review` using the Bash tool:

```bash
echo '<comprendo_data>
{ ... the full JSON ... }
</comprendo_data>' | comprendo review
```

- Wait for the command to exit
- Exit code 0: implement exactly as planned
- Exit code 1: stop, tell the developer it was rejected, ask what they want differently
- **Never write files before `comprendo review` exits with code 0**

### 2. `package.json`

```json
{
  "name": "comprendo",
  "version": "0.2.0",
  "bin": { "comprendo": "./cli.js" },
  "files": ["cli.js", "ui.html"],
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

Note: No `open` package. Browser opening is handled via `child_process.exec` with platform detection (`open` on macOS, `xdg-open` on Linux) to avoid the ESM-only issue with `open` v9+.

### 3. `cli.js`

Node.js CLI that handles `comprendo review`:

**Step 1 — Parse the plan**
Read stdin, extract the JSON from inside `<comprendo_data>` tags, parse it.

**Step 2 — Start local server**
Spin up an Express server on port 2345. Serve the review UI as a single HTML page. Inject the parsed plan data as a JS variable into the page.

Port fallback: if 2345 is in use, try 2346, 2347, up to 5 attempts then error.

**Step 3 — Open browser**
Use `child_process.exec` with platform detection:
- macOS: `open http://localhost:<port>`
- Linux: `xdg-open http://localhost:<port>`

**Step 4 — Wait for decision**
Expose a POST endpoint at `/decision` that accepts `{ "approved": true/false }`. When called, store the decision, shut down the server, and exit with code 0 (approved) or 1 (rejected).

**Step 5 — Timeout and tab-close safety**
If no decision is received within 5 minutes, exit with code 1 (reject) and log a warning to stderr: "Timed out waiting for decision. Rejecting to be safe."

This ensures no code is ever written without explicit approval.

### 4. `ui.html`

A single self-contained HTML file that cli.js serves and injects data into. Build this with care — this is the face of the product.

**Layout (single page, scrollable):**

```
┌─────────────────────────────────────────────────────┐
│  COMPRENDO                          [Reject] [Approve]│
├────────────────┬────────────────────────────────────┤
│ CURRENT STATE  │ UPDATED STATE                       │
│                │                                     │
│  [diagram]     │  [diagram]                          │
│                │                                     │
├────────────────┴────────────────────────────────────┤
│ WHAT'S CHANGING                                      │
│ <explanation text>                                   │
├─────────────────────────────────────────────────────┤
│ KEY CHANGES    [Before] [After]  ← toggle           │
│ <code snippet with syntax highlighting>              │
├─────────────────────────────────────────────────────┤
│ Files: [pill] [pill] [pill]                          │
└─────────────────────────────────────────────────────┘
```

**Design requirements:**

- Dark theme. Background `#0d0d0d`, surface `#1a1a1a`, text `#e8e8e8`
- Monospace font for code: JetBrains Mono from Google Fonts
- Sans font for UI: Inter from Google Fonts
- Diagrams rendered using Mermaid.js loaded from CDN (`https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js`)
- Convert the nodes/edges JSON into Mermaid `graph LR` syntax in the browser before rendering
- Current state diagram on the left, updated state on the right, side by side
- Nodes that are NEW in the updated diagram highlighted with accent color (`#7c6aff`)
- Nodes that are UNCHANGED render in neutral (`#444`)
- Code snippets with syntax highlighting using highlight.js from CDN (load core + js, ts, python, css, json languages only)
- Before/After toggle on the snippets section switches between the two snippets
- Approve button: `#7c6aff` background, white text
- Reject button: transparent, `#666` border, `#999` text
- Files affected listed as small pills at the bottom
- When Approve or Reject is clicked: button shows a spinner, POST to `/decision`, then show a full-screen confirmation message ("Approved — Claude is implementing" or "Rejected — Claude will stop") before the tab can be closed

**Mermaid conversion function (corrected):**

```javascript
function toMermaid(diagram, newNodeIds) {
  const lines = ['graph LR'];

  // Declare nodes with labels and shapes
  diagram.nodes.forEach(node => {
    const shape = node.type === 'service' ? '([' + node.label + '])' : '[' + node.label + ']';
    lines.push(`  ${node.id}${shape}`);
  });

  // Declare edges
  diagram.edges.forEach(edge => {
    const label = edge.label ? `-->|${edge.label}|` : '-->';
    lines.push(`  ${edge.from} ${label} ${edge.to}`);
  });

  // Style classes for new vs existing nodes
  if (newNodeIds && newNodeIds.length > 0) {
    lines.push(`  classDef newNode fill:#7c6aff,stroke:#7c6aff,color:#fff`);
    lines.push(`  classDef existingNode fill:#444,stroke:#555,color:#e8e8e8`);
    lines.push(`  class ${newNodeIds.join(',')} newNode`);
    const existingIds = diagram.nodes
      .map(n => n.id)
      .filter(id => !newNodeIds.includes(id));
    if (existingIds.length > 0) {
      lines.push(`  class ${existingIds.join(',')} existingNode`);
    }
  }

  return lines.join('\n');
}
```

To determine which nodes are new: diff `current_diagram.nodes` IDs against `updated_diagram.nodes` IDs. Any ID present in updated but not in current is "new".

**Data injection:**
`cli.js` reads `ui.html`, replaces the placeholder string `__COMPRENDO_DATA__` with `JSON.stringify(plan)`, and serves the result.

### 5. `README.md`

```bash
# Install
git clone <repo>
cd comprendo
npm install
npm link

# Install slash command
mkdir -p ~/.claude/commands
cp comprendo.md ~/.claude/commands/comprendo.md

# Use
cd your-project
claude
/comprendo <describe your change>
```

## Technical constraints

- Node.js only, no TypeScript, no build step
- The server must shut down cleanly after receiving a decision — no zombie processes
- Port 2345, fallback to 2346-2349 if in use
- The UI must work in Chrome, Safari, and Firefox
- Mermaid and highlight.js loaded from CDN, no local assets
- All UI in the single `ui.html` file, no separate CSS or JS files
- `cli.js` injects data by replacing `__COMPRENDO_DATA__` in `ui.html` with `JSON.stringify(plan)` before serving
- Browser opened via `child_process.exec` (platform-detected), no `open` npm package
- If browser tab is closed without a decision, the 5-minute timeout catches it and exits 1

## What NOT to build in Phase 1

- No quiz
- No audit trail
- No significance classifier
- No automatic hook on every write
- No cloud sync
- No team features
