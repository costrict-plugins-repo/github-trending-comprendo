---
name: comprendo
description: Use when the developer wants to review and approve a planned code change before implementation. Shows a visual diff with architecture diagrams in the browser.
allowed-tools: Read Glob Grep Bash
argument-hint: <describe the change you want to make>
---

You are tasked with implementing a change requested by the developer. Before writing any code, you MUST follow the Comprendo approval workflow.

The developer's request is: $ARGUMENTS

## Step 1: Understand the change

Read all files relevant to the requested change. Explore the codebase thoroughly to understand the current state of the code that will be affected.

## Step 2: Output a structured plan

Before making ANY file changes, output a JSON block wrapped in `<comprendo_data>` tags with this exact structure:

```
<comprendo_data>
{
  "summary": "One sentence: what this change does",
  "explanation": "2-3 paragraphs in plain English. What exists now, why it's a problem or limitation, what the change does and why this approach. Write for a developer who hasn't looked at this code in 3 weeks.",
  "current_diagram": {
    "nodes": [
      { "id": "unique_id", "label": "Human Readable Name", "type": "component" }
    ],
    "edges": [
      { "from": "source_id", "to": "target_id", "label": "relationship" }
    ]
  },
  "updated_diagram": {
    "nodes": [
      { "id": "unique_id", "label": "Human Readable Name", "type": "component" },
      { "id": "new_thing", "label": "New Component", "type": "service" }
    ],
    "edges": [
      { "from": "source_id", "to": "new_thing", "label": "relationship" },
      { "from": "new_thing", "to": "target_id", "label": "relationship" }
    ]
  },
  "snippets": [
    {
      "label": "Before — filename.ts",
      "language": "typescript",
      "code": "// key before snippet showing current implementation, max 20 lines"
    },
    {
      "label": "After — filename.ts",
      "language": "typescript",
      "code": "// key after snippet showing new implementation, max 20 lines"
    }
  ],
  "files_affected": ["relative/path/to/file1.ts", "relative/path/to/file2.ts"]
}
</comprendo_data>
```

### Diagram guidelines:
- Node `type` can be: `component`, `service`, `database`, `api`, `middleware`, `util`
- Keep diagrams focused — show only the components relevant to this change (3-8 nodes)
- Nodes in `updated_diagram` that have IDs not present in `current_diagram` are treated as NEW and highlighted in the UI
- Include meaningful edge labels that describe the relationship

### Snippet guidelines:
- Show the most important before/after code change
- Max 20 lines per snippet
- Always include at least one before/after pair
- Use the actual code, not pseudocode

## Step 3: Wait for developer approval

After outputting the plan, you MUST pipe it to `comprendo review` using the Bash tool. Use a heredoc to avoid shell escaping issues with quotes in the JSON. Copy the entire `<comprendo_data>...</comprendo_data>` block (including tags) and pipe it:

```bash
cat <<'COMPRENDO_EOF' | comprendo review
<comprendo_data>
{ ... your full JSON from Step 2 ... }
</comprendo_data>
COMPRENDO_EOF
```

IMPORTANT: Use the single-quoted heredoc form `<<'COMPRENDO_EOF'` (with quotes around the delimiter) to prevent shell variable expansion and preserve backslashes in code snippets.

Wait for the command to exit. Do NOT proceed until it exits.

## Step 4: Act on the result

- **If exit code is 0**: The developer approved. Proceed with implementing the change exactly as described in your plan. Write the files now.
- **If exit code is 1**: The developer rejected the plan. Do NOT write any files. Tell the developer the change was rejected and ask what they would like to do differently.

## CRITICAL RULES

- NEVER write, edit, or create any files before `comprendo review` exits with code 0.
- NEVER skip the approval step.
- Always output the full JSON plan first, then pipe it to the CLI command.
- If the comprendo CLI is not installed, tell the developer to install it with: `/plugin marketplace add nikhilenmudi/comprendo` then `/plugin install comprendo`
- Make sure the JSON is valid — escape any quotes or special characters in code snippets properly.
- When piping to comprendo review, ensure single quotes in the JSON are properly handled (use the heredoc form if needed).
