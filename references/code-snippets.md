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

## Complete — mark note done with resolution

```js
var node = await figma.getNodeByIdAsync("NODE_ID");
var raw = node.getSharedPluginData("clue", "note");
var noteData;
try { noteData = JSON.parse(raw); } catch(e) { noteData = { text: raw, createdAt: "" }; }
var doneData = JSON.stringify({
  text: noteData.text,
  resolution: "SHORT DESCRIPTION OF WHAT WAS DONE",
  createdAt: noteData.createdAt || "",
  completedAt: new Date().toISOString()
});
node.setSharedPluginData("clue", "done", doneData);
node.setSharedPluginData("clue", "note", "");
```

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

Note: always check `"children" in n` before accessing — leaf nodes like ELLIPSE throw otherwise.

## Detach instance before editing

```js
var node = await figma.getNodeByIdAsync("NODE_ID");
if (node.type === "INSTANCE") {
  node.detachInstance();
}
```

Note: after `detachInstance()`, the node becomes a regular `FRAME`. Proceed with edits on it directly.

## QuickJS rules

- `function(){}` not arrow functions
- `catch (e)` not `catch`
- No optional chaining (`?.`)
- `var` over `const`/`let`
- `getNodeByIdAsync` (async) not `getNodeById`
- Load fonts before text edits: `await figma.loadFontAsync(node.getRangeFontName(0, 1))`
