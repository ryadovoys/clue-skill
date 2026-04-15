---
name: clue
description: "Address Figma design notes left via the Clue plugin. Use this skill when the user says 'address clues', 'address notes', 'address figma notes', 'check my notes', 'look at figma notes', 'address figma comments', 'check figma comments', 'address comments', or calls /clue. Scans for notes (scoped to the page or frame from the URL), analyzes context and relationships between them, addresses each one, and marks them complete with a resolution comment."
---

# Clue — Notes for agents

Scan a Figma file for notes left via the **Clue** Figma plugin, understand their context and relationships, address each one, and mark done with a resolution.

Code snippets are in `references/code-snippets.md` — read it before running any `use_figma` calls.

## Data format

Notes stored as `sharedPluginData` on nodes (namespace: `clue`):
- **`note`** — JSON: `{ "text": "...", "createdAt": "ISO" }` (or legacy plain string)
- **`done`** — JSON: `{ "text": "...", "resolution": "...", "createdAt": "ISO", "completedAt": "ISO" }`

## Workflow

### Step 1 — File key & scope

Use a Figma URL from the conversation. Extract `fileKey` and `nodeId` from `figma.com/design/:fileKey/...?node-id=:nodeId`. Convert `-` to `:` in nodeId.

**Determine scan scope:**
- URL has `node-id` → use that node ID as `ROOT_ID` in the scan. The scan will cover only that node's descendants (if it's a frame) or that page (if it's a page node).
- URL has no `node-id` → set `ROOT_ID = ""` to scan all pages.

If no URL provided, ask.

### Step 2 — Get notes

**Fast path (prompt includes note IDs):** When the prompt already lists notes with IDs, text, types, and pages (as produced by the Clue plugin's "Copy" action), skip the scan entirely. You already have the notes. Instead, run a single lightweight `use_figma` call to fetch ancestor chains for all listed node IDs at once — this is needed for context grouping in Step 3:

```js
var ids = ["NODE_ID_1", "NODE_ID_2"]; // from the prompt
var results = [];
for (var i = 0; i < ids.length; i++) {
  var n = await figma.getNodeByIdAsync(ids[i]);
  if (!n) continue;
  var ancestors = [];
  var p = n.parent;
  while (p && p.type !== "PAGE" && p.type !== "DOCUMENT") {
    ancestors.push({ id: p.id, name: p.name, type: p.type });
    p = p.parent;
  }
  results.push({ id: n.id, ancestors: ancestors });
}
return results;
```

**Full scan (no IDs in prompt):** Run the scan code from `references/code-snippets.md` with the appropriate `ROOT_ID`. The scan returns notes with an `ancestors` array (parent chain from immediate parent up to the top-level frame).

If no notes found, tell the user and stop.

### Step 3 — Context analysis

**This step is critical.** Before addressing anything, understand how notes relate to each other and their surroundings.

**Group by context:**
- Look at each note's `ancestors` array. Notes sharing a common ancestor are in the same group.
- Find the **most useful common ancestor** for each group — typically the screen-level frame that represents a complete UI state. Use frame names as clues (e.g., "Global context / Empty" suggests a screen variant).

**Screenshot strategy:**
- Take **one screenshot per group** at the common ancestor level, not per individual note. This shows the full screen context and reveals elements the annotated node alone would miss (titles above, sibling components, surrounding layout).
- If notes are on completely different screens with no shared ancestor below page level, screenshot each screen separately.
- For a single isolated note, screenshot its parent frame (one level up) to get surrounding context.

**Analyze relationships:**
- Same screen? → These are related changes to the same UI state
- Different states of the same flow? → Look at frame names for state indicators (e.g., "Empty", "Filled", "Error")
- Independent screens? → Treat as separate notes
- Do notes reference each other? → Address them as a coordinated change

**Look for what you're NOT seeing:**
- The annotated node may be a subset of a larger component. Before concluding something is "missing," check the parent frame — it might be there, just outside the annotated node's bounds.
- Titles, headers, labels, and other contextual elements above or beside the annotated frame provide crucial context.

### Step 4 — Process notes

Present all notes as a numbered list with their group context. Then process them by group.

**Ask or act? Decide per note.**

For every note, judge whether the request gives you enough context to act confidently:

- **Act when** the note is unambiguous, the target is clear, and there's only one reasonable interpretation. Examples: "make this button blue," "change copy to X," "remove this section," "fix the typo." Just do it and report what changed.
- **Ask when** any of the following are true:
  - The intent is open-ended ("make this nicer", "improve the layout", "rethink this")
  - There are multiple plausible interpretations and picking the wrong one would waste a round-trip
  - Required information is missing (target component, exact value, scope)
  - The change is destructive or hard to undo (deleting nodes, restructuring a screen)
  - The note conflicts with another note in the same group

When asking, use the **AskUserQuestion** tool with clear options. Mention what you see in the screenshot for context. Prefer offering 2–3 concrete options over open-ended questions.

Default toward acting on small, clear changes — don't ask permission for things you'd safely do anyway. Default toward asking on anything that could go in multiple directions.

**Conflicting notes:** Prefer the newer one (by `createdAt`). If unsure, ask.

**Component safety rule:**
If the note targets a component instance and the change would modify the component's structure or content beyond simple overrides:
1. **Detach the instance first** — use `use_figma` to call `.detachInstance()` on the node before making changes. This prevents edits from propagating to the master component.
2. **Exception — master component:** If the note explicitly asks to edit the master/main component, or the annotated node IS the master component (`type === "COMPONENT"`), edit it directly without detaching.
3. **Exception — safe overrides:** Simple text or fill overrides that don't break the component structure can be applied without detaching.

Notes often represent iterations and explorations — detaching protects the design system while allowing free edits.

**Critical: `detachInstance()` replaces the node.**
`detachInstance()` is destructive — it deletes the original instance and returns a **new FrameNode with a new ID**. You MUST:
- **Capture the return value:** `const frame = instance.detachInstance();`
- **Use the returned reference** for all subsequent modifications.
- **Never re-fetch by old ID** — `figma.getNodeByIdAsync(oldId)` will return `null`.
- **Never store references before detaching** — the in-memory reference is invalidated too.
- **Detach and modify in the same scope** — do not detach in one loop and modify in another.

```js
// WRONG — old ID is dead after detach
const id = inst.id;
inst.detachInstance();
const node = await figma.getNodeByIdAsync(id); // null!

// CORRECT — use the return value directly
const frame = inst.detachInstance();
frame.name = "New name"; // works
```

**For each note:**
1. Check if the node is a component instance (`type === "INSTANCE"`) and whether the change requires detaching (see rule above)
2. Make the change using `use_figma`
3. Mark complete with a resolution using the complete code from `references/code-snippets.md`
4. Briefly report what changed

**Resolution comments** must be short and specific:
- Good: "Rewrote AI message from report-style to conversational tone, shortened user reply"
- Bad: "Done" / "Made the requested changes"

### Step 5 — Summary

Short summary of all changes. The user can see the full history with resolutions in the Clue plugin's Completed section.
