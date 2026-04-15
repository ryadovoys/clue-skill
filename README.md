# Clue

![Clue — sticky notes for AI agents in Figma](cover.png)

**Sticky notes for AI agents in Figma. Because without them, they have no clue.**

Clue is a two-part system for bridging Figma and AI coding agents:

1. **[Clue Figma plugin](https://www.figma.com/community/plugin/TODO)** — designers leave inline sticky notes on any frame or section.
2. **Clue Claude Code skill** — this repo. Lets your agent find those notes, act on them, and mark each one done with a resolution.

## Why Clue exists

Your AI coding agent opens your Figma file and has no clue what's going on. It's staring down 400 frames, 12 component variants, and a page called `wip-final-FINAL-v3`. It guesses. It copies the wrong spacing. It misses the "empty state" frame entirely. It rebuilds the button from scratch because it couldn't see the note you left on the master component.

Clue fixes that. You leave a note on the frame ("use DS/Button/Primary, not the custom one"), your agent reads it, makes the change, and marks the note done with a short resolution. Everybody wins. Nobody re-explains the same thing four times.

## Requirements

You need all three of these installed and working together:

- A coding agent with tool access — [Claude Code](https://claude.com/claude-code), Codex CLI, Cursor, or any CLI/IDE agent that can call MCP servers and read skill folders.
- The [Figma MCP server](https://help.figma.com/hc/en-us/articles/32132100833559) — so your agent can query your Figma file, read nodes, and export assets.
- The [Clue Figma plugin](https://www.figma.com/community/plugin/TODO) — installed in your Figma workspace so you can leave notes in the first place.

Without the plugin, there are no notes. Without Figma MCP, your agent can't see your file. Without this skill, your agent won't know Clue exists. Install all three.

## Install

Clone this repo directly into your Claude Code skills folder:

```bash
git clone https://github.com/ryadovoys/clue-skill.git ~/.claude/skills/clue
```

That's it. Claude Code picks up the skill automatically the next time you start a session.

To update later:

```bash
cd ~/.claude/skills/clue && git pull
```

## How to use

1. In Figma, open the **Clue** plugin and leave notes on any frame or section you want your agent to pay attention to.
2. Copy the Figma file URL (or a link to a specific section/frame to scope the scan).
3. In your Claude Code session, say any of:

   ```
   /clue https://figma.com/design/...
   address my clues in https://figma.com/design/...
   check figma notes in https://figma.com/design/...
   look at my figma notes
   ```

4. The agent scans the file, groups clues by screen, takes contextual screenshots, addresses each clue, and marks them done with a short resolution note.

The scope follows the URL:
- URL with a `node-id` → scans only that node's descendants.
- URL without a `node-id` → scans every page in the file.

## How notes are stored

Clue notes live as `sharedPluginData` on Figma nodes under the `clue` namespace. Each open note is a JSON blob on the `note` key:

```json
{ "text": "...", "createdAt": "ISO timestamp" }
```

When a note is resolved, the payload moves to the `done` key with a short resolution string and a completion timestamp:

```json
{
  "text": "original note",
  "resolution": "what the agent did",
  "createdAt": "ISO",
  "completedAt": "ISO"
}
```

You can review the full history of resolved clues in the Clue plugin's **Completed** section.

## License

MIT. Use freely, fork gladly, no warranty.

---

Built by [Sergey Ryadovoy](https://www.ryadovoy.com/). Enjoying Clue? [Support the project ☕](https://buymeacoffee.com/ryadovoys)
