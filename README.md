# appstore-screenshot-translator

A [Claude Code](https://claude.com/claude-code) plugin that localizes App Store / Play Store screenshots into multiple languages while preserving the original visual layout — fonts, colors, position, background photography, device frames, everything except the words.

It reads your project for context, walks you through a language picker and screenshot drop, shows you a translation table to confirm before spending money, then regenerates each image via the Gemini Nano Banana MCP and verifies the result.

---

## Install

### 1. Install the Gemini Nano Banana MCP server

```bash
npm install -g @houtini/gemini-mcp
```

Then register it with Claude Code, inserting your Gemini API key from [aistudio.google.com/apikey](https://aistudio.google.com/apikey):

```bash
claude mcp add --scope user gemini --env GEMINI_API_KEY=your_key_here -- gemini-mcp
```

Verify it's connected:

```bash
claude mcp list
```

### 2. Install the skill

```bash
git clone https://github.com/chriswakefield87/appstore-screenshot-translator.git \
  ~/.claude/skills/appstore-screenshot-translator
```

Restart Claude Code. `/appstore-screenshot-translator` is now available as a slash command.

---

## Use

From the root of your app's project:

```
/appstore-screenshot-translator
```

The skill walks you through:

1. **Pick languages** from a 20-locale menu (numbers, codes, names, or `all`).
2. **Drop screenshots** into the terminal (drag from Finder/Explorer).
3. **Confirm the translation table** — review every text change before any images are generated.
4. **Watch the ticks** — one line per output as Nano Banana finishes each image.

Outputs land in `<project_root>/marketing/<lang>/<image>.png`.

---

## Output layout

For three screenshots (`1.png`, `2.png`, `3.png`) translated to Spanish, French, and Japanese:

```
<project_root>/marketing/
├── es/
│   ├── 1.png
│   ├── 2.png
│   └── 3.png
├── fr/
│   ├── 1.png
│   ├── 2.png
│   └── 3.png
└── ja/
    ├── 1.png
    ├── 2.png
    └── 3.png
```

Just the localized images — no manifests, no JSON, no logs.

---

## Cost

One Gemini Nano Banana image-edit call per (screenshot × language). 10 screenshots × 5 languages = 50 calls. See [ai.google.dev/pricing](https://ai.google.dev/pricing) for current rates.

---

## Troubleshooting

**Slash command doesn't show up** — restart Claude Code. Confirm the skill lives at `~/.claude/skills/appstore-screenshot-translator/` and contains `SKILL.md`.

**"I need the Gemini Nano Banana MCP server configured"** — `claude mcp list` should show `gemini` as connected. Restart Claude Code after adding it.

**Translated text doesn't fit** — tell Claude which image and string and ask it to retry with a shorter phrasing.

**Brand name got translated** — mention the brand in your prompt before running, or after the fact ask Claude to redo the image.

---

## License

MIT — see [LICENSE](./LICENSE).
