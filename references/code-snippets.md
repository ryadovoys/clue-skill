# Code Snippets for Clue

QuickJS-compatible code for `mcp__plugin_figma_figma__use_figma`.

## Scan — scoped to a node or page, with ancestor chain

Replace `ROOT_ID` with a node/page ID to scope, or `""` to scan all pages.

```js
var ROOT_ID = "";
var notes = [];
function scanTarget(target, pageName) {
  target.findAll(function(n) {
    try {
      var raw = n.getSharedPluginData("clue", "note");
      if (raw && raw !== "") {
        var data;
        try { data = JSON.parse(raw); } catch (e) { data = { text: raw, createdAt: "" }; }
        if (data.text) {
          var p = n.parent;
          var ancestors = [];
          while (p && p.type !== "PAGE" && p.type !== "DOCUMENT") {
            ancestors.push({ id: p.id, name: p.name, type: p.type });
            p = p.parent;
          }
          notes.push({
            id: n.id, name: n.name, type: n.type,
            page: pageName,
            text: data.text,
            createdAt: data.createdAt || "",
            ancestors: ancestors
          });
        }
      }
    } catch (e) {}
    return false;
  });
}
if (ROOT_ID) {
  var root = await figma.getNodeByIdAsync(ROOT_ID);
  if (root) {
    var pg = root;
    while (pg && pg.type !== "PAGE") pg = pg.parent;
    var pageName = pg ? pg.name : "";
    if (root.type === "PAGE") {
      scanTarget(root, root.name);
    } else {
      scanTarget(root, pageName);
    }
  }
} else {
  for (var i = 0; i < figma.root.children.length; i++) {
    var page = figma.root.children[i];
    scanTarget(page, page.name);
  }
}
notes.sort(function(a, b) { return (a.createdAt || "").localeCompare(b.createdAt || ""); });
return notes;
```

The `ancestors` array is ordered inner → outer (immediate parent first, top-level frame last).

**Scoping rules:**
- `ROOT_ID = ""` → scans all pages (use when no node-id in URL)
- `ROOT_ID = page ID` → scans that page only
- `ROOT_ID = frame ID` → scans only descendants of that frame

## Inspect text nodes (safe traversal)

```js
var node = await figma.getNodeByIdAsync("NODE_ID");
var texts = [];
function walk(n) {
  if (n.type === "TEXT") {
    texts.push({ id: n.id, name: n.name, text: n.characters });
  }
  if ("children" in n && n.children) {
    for (var i = 0; i < n.children.length; i++) { walk(n.children[i]); }
  }
}
walk(node);
return texts;
```

Always check `"children" in n` before recursing — leaf nodes like `ELLIPSE` throw otherwise.

## Detach instance before editing

Use this when the user asked for an edit to an instance and the change would modify structure or content beyond a simple override. Detaching protects the shared master component from accidental propagation.

```js
var node = await figma.getNodeByIdAsync("NODE_ID");
if (node.type === "INSTANCE") {
  var frame = node.detachInstance();
  // `frame` is a brand-new FrameNode — use it for all further edits
  frame.name = "New name";
}
```

**`detachInstance()` is destructive.** It deletes the original instance and returns a *new* `FrameNode` with a *new* ID. Get this wrong and every subsequent lookup silently returns `null`:

```js
// WRONG — the old ID is dead after detach
var id = inst.id;
inst.detachInstance();
var node = await figma.getNodeByIdAsync(id); // returns null!

// CORRECT — use the return value directly
var frame = inst.detachInstance();
frame.name = "New name"; // works
```

The rules, with the reasoning:

- **Capture the return value** of `detachInstance()`. It's the only valid reference to the post-detach node.
- **Never re-fetch by the old ID.** The old ID belongs to a deleted node; the lookup returns `null`.
- **Don't cache a reference to the instance before detaching.** The in-memory reference is invalidated along with the node itself.
- **Detach and modify in the same scope.** Don't detach in one loop and modify in another — if the reference doesn't survive the loop boundary, the second loop has nothing to work with.

## QuickJS rules

- `function(){}` not arrow functions
- `catch (e)` not `catch`
- No optional chaining (`?.`)
- `var` over `const`/`let`
- `getNodeByIdAsync` (async) not `getNodeById`
- Load fonts before text edits: `await figma.loadFontAsync(node.getRangeFontName(0, 1))`
