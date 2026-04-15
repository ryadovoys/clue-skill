---
name: clue
description: "Read and understand design notes ('clues') left in Figma via the Clue plugin. Use this skill whenever the user says 'address clues', 'check my clues', 'look at my figma notes', 'review figma notes', 'what do my clues say', 'check figma comments', 'address comments', 'look at figma notes', 'summarize my clues', or calls /clue. Also trigger when the user shares a Figma URL and mentions clues, notes, or feedback on frames. The skill scans the file (scoped to the page or frame from the URL), groups clues by surrounding context, interprets each clue's intent (edit request, build-time hint, change log, brainstorm prompt, or hidden context), and reports findings in chat — acting on them (editing Figma, generating code, etc.) only when the user explicitly asks."
---

# Clue — Design notes for agents

Scan a Figma file for clues left via the **Clue** Figma plugin, understand each one in its surrounding context, and report back to the user in chat. Clues can mean a lot of different things; the skill's job is to figure out *what each clue is asking for* and respond helpfully — not to force every clue into the same action.

Code snippets are in `references/code-snippets.md` — read it before running any `use_figma` calls.

## What clues can be

A clue is a short note a designer pinned to a node. What it *means* depends on context. Common patterns:

- **Edit request** — "use DS/Button/Primary, not the custom one." The designer wants this changed in Figma.
- **Build-time hint** — "lazy-load this image," "this list is virtualized in code." Guidance for when an agent is generating or modifying code from this frame. The Figma file itself may not need changing.
- **Change log / annotation** — "refactored to use the new tokens last week." Historical context. Usually nothing to do; just note it.
- **Open question / brainstorm** — "what if we dropped this on mobile?" A prompt for discussion, not an order.
- **Hidden context** — "this is the shared master — do not clone," "colors here come from the v2 tokens." Warnings, invariants, or rules to respect in future work.

One clue can straddle two categories. Read the clue text *together with* the frame it's attached to and infer what the designer actually wants. Don't assume every clue is an edit request — that's the single most common way this skill gets misused.

## Data format

Clues live as `sharedPluginData` on Figma nodes in the `clue` namespace. Each clue is stored on the `note` key as JSON: `{ "text": "...", "createdAt": "ISO" }`. Legacy clues may be plain strings — handle both.

The skill only *reads* this data. It never writes to clue nodes. Everything the user sees happens in chat.

## Workflow

### Step 1 — File key & scope

Use a Figma URL from the conversation. Extract `fileKey` and `nodeId` from `figma.com/design/:fileKey/...?node-id=:nodeId`. Convert `-` to `:` in `nodeId`.

**Scope:**
- URL has `node-id` → use that as `ROOT_ID` in the scan. Covers only that node's descendants (if a frame) or that page (if a page node).
- URL has no `node-id` → `ROOT_ID = ""` scans every page.

If no URL is provided, ask.

### Step 2 — Get clues

**Fast path (prompt already lists clues):** When the user pasted the Clue plugin's "Copy" output (clue text + node IDs + frame names), skip the scan — you already have the text. Run one lightweight `use_figma` call to fetch ancestor chains for all listed IDs, because Step 3 needs them:

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

**Full scan (no IDs in the prompt):** Run the scan code from `references/code-snippets.md` with the appropriate `ROOT_ID`. The scan returns clues with an `ancestors` array (parent chain from immediate parent up to the top-level frame).

If there are no clues, tell the user and stop.

### Step 3 — Context analysis

**This step matters a lot.** Clue text alone is rarely enough. The surrounding frame, sibling elements, and visual layout usually carry most of the meaning. Skipping context is the single most common way agents misread clues.

**Group by context.** Clues sharing a common ancestor belong to the same group. Find the most useful common ancestor for each group — usually the screen-level frame that represents one complete UI state. Frame names often help (e.g., "Global context / Empty" suggests a screen variant).

**Screenshot strategy.** Take **one screenshot per group** at the common ancestor level, not per individual clue. That shows the full screen and reveals things the annotated node alone would miss — titles above, sibling components, surrounding layout. For completely separate screens, screenshot each. For a single isolated clue, screenshot its parent frame (one level up) for surrounding context.

**Read relationships.** Same screen → related clues about one UI state. Different states of the same flow → check frame names for cues ("Empty," "Filled," "Error"). Clues that reference each other → handle as a coordinated set.

**Look for what you're NOT seeing.** The annotated node may be a subset of a larger component. Before concluding "X is missing," check the parent frame — X might be there, just outside the annotated bounds. Titles, labels, and surrounding context are usually load-bearing.

### Step 4 — Interpret each clue

Read each clue together with its screenshot and decide which of the five categories it falls into (from "What clues can be" above). This is the core of the skill — take it seriously. A "what if we..." clue is not the same thing as "change copy to X," and treating them identically wastes effort on one and misses the point of the other.

**Conflicting clues:** prefer the newer one (by `createdAt`). If two clues in the same group conflict and the newer one doesn't clearly win, flag it in the report instead of picking silently.

### Step 5 — Report findings in chat

Present the clues as a numbered list, grouped by context. For each clue, tell the user:

1. **Where it is** — screen / frame name (include the page if it helps).
2. **What it says** — the clue text, quoted verbatim.
3. **How you're reading it** — the category, and what you think the designer actually wants.
4. **Context they might miss** — anything the clue depends on that isn't obvious from the text alone (the surrounding elements, what state this screen is in, how it relates to a neighboring clue).

Keep each entry short. The goal is scannable findings, not an essay. If a clue is genuinely ambiguous, say so and offer your best guess plus the alternative.

**Default mode is "report, then let the user drive."** Don't edit the Figma file or generate code unless the user explicitly asked for that. Many clues are notes, context, or brainstorm prompts — acting on them without being asked either wastes effort or actively breaks things the designer wanted left alone.

End the report with a short prompt back, something like: *"Want me to apply any of these to Figma, build code from them, or dig into a specific one?"* — unless the user's original ask already made next steps explicit.

### Step 6 — Act, if the user asked

Switch into action mode only when the user's prompt explicitly asks for it ("apply these," "implement the changes," "edit the Figma file," "build this screen using these notes," "fix the copy"). What that looks like depends on which mode they asked for:

#### Editing Figma

Use `use_figma` to make changes. The tricky part is components.

- **Instance + structural change** → detach first. `var frame = node.detachInstance();`. Detaching protects the shared master component from accidental edits and lets you freely modify the result. Clues often represent iterations or exploration — detach is the safe default.
- **Master component** → if the clue explicitly targets the master, or the node IS the master (`type === "COMPONENT"`), edit it directly without detaching.
- **Simple override** → text content or fill changes that don't break component structure can be applied to an instance without detaching.

**`detachInstance()` is destructive and replaces the node.** It has a specific footgun that will silently null out references if you get it wrong — the full warning and correct pattern live in `references/code-snippets.md` under "Detach instance before editing." Read it before detaching.

Make one change at a time and report what you did after each. Don't write anything back into the clue node — clues are input; chat is the output.

#### Building code from clues

Treat each clue as a line in the spec you're implementing:

- Edit requests → reflected in the generated code (e.g. "use DS/Button/Primary" → import the real DS component instead of hand-rolling one).
- Build-time hints → implementation requirements (e.g. "lazy-load" → `<img loading="lazy">`).
- Hidden context → constraints to respect while coding (e.g. "colors come from v2 tokens" → don't hardcode hex values).

When the same clue has both an edit-request and a build-time reading, ask which one the user wants — they usually know.

#### Brainstorming

Engage conversationally. Offer options, tradeoffs, and questions back. Clues in this category are the start of a discussion, not a task. It's fine if the whole "action" is just a thoughtful reply in chat.

#### Ask or act?

Default to acting when a clue is unambiguous and there's one reasonable interpretation ("change this button's color to blue"). Ask when:
- The intent is open-ended ("make this nicer," "rethink this layout")
- Multiple plausible interpretations exist
- Required info is missing (target component, exact value, scope)
- The change is destructive or hard to undo
- Two clues in the same group conflict

When asking, use the **AskUserQuestion** tool with 2–3 concrete options. Describe what you see in the screenshot so the user has the same context you do.

### Step 7 — Wrap up

A short summary of what you found, plus what you changed (if the user asked for edits). Nothing more.
