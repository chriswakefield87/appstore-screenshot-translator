# appstore-screenshot-translator

A [Claude Code](https://claude.com/claude-code) slash command (`/appstore-screenshot-translator`) that **localizes App Store / Play Store screenshots into multiple languages** while preserving the original visual layout — fonts, colors, position, background photography, device frames, everything except the words.

Under the hood it does four things, all driven by Claude itself with one MCP call per image:

1. **Reads your project for context.** Claude scans your CLAUDE.md, README, package manifest, and a couple of source files to learn what your app does, who it's for, what tone to use, and which feature names to preserve.
2. **Walks you through a language picker** with the 20 highest-value App Store / Play Store locales. Pick any combination.
3. **Asks you to drop your screenshots into the terminal** (drag-and-drop), then OCRs each one and translates the text in-conversation.
4. **Regenerates each image via Gemini Nano Banana** using the original screenshot + the translated text replacements. Nano Banana edits the image in place, keeping the layout byte-identical to the source.

**One API key. No Python. No pip install.** All you need is the Gemini MCP.

---

## Install

### 1. Drop the skill into your Claude skills directory

```bash
git clone https://github.com/<your-username>/appstore-screenshot-translator.git \
  ~/.claude/skills/appstore-screenshot-translator
```

Claude Code picks up skills under `~/.claude/skills/` automatically. Restart Claude Code and `/appstore-screenshot-translator` will be available as a slash command.

### 2. Configure the Gemini Nano Banana MCP server

This skill is wired to use [`@houtini/gemini-mcp`](https://www.npmjs.com/package/@houtini/gemini-mcp), which exposes an `edit_image` tool backed by Nano Banana Pro.

Grab a Gemini API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey), then register the MCP with Claude Code at user scope so it's available in every project:

```bash
claude mcp add --scope user gemini \
  --env GEMINI_API_KEY=your_key_here \
  -- npx -y @houtini/gemini-mcp
```

This uses `npx`, so there's no global install to manage. The package is fetched on first use and cached.

Verify it's connected:

```bash
claude mcp list
```

You should see `gemini` listed and reporting connected. Restart Claude Code if it doesn't show up.

**Using a different Gemini image MCP?** Any server that exposes a tool taking a reference image plus an edit prompt will work, but you'll need to adjust the call shape in `SKILL.md` (search for `edit_image`). The default assumes `@houtini/gemini-mcp`'s parameters: `prompt`, `images: [{filePath}]`, `outputPath`.

That's it. Two commands, one API key.

---

## Use

**Run Claude Code from the root of your app's project.** Then:

```
/appstore-screenshot-translator
```

The skill walks you through four steps:

### 1. Language picker

You'll see a numbered menu of the 20 highest-value App Store / Play Store locales:

```
 1. Spanish              (es)        — Spain & Latin America
 2. French               (fr)        — France & Canada
 3. German               (de)
 4. Italian              (it)
 5. Portuguese (Brazil)  (pt-BR)
 6. Portuguese (Portugal)(pt-PT)
 7. Dutch                (nl)
 8. Russian              (ru)
 9. Polish               (pl)
10. Turkish              (tr)
11. Arabic               (ar)        — right-to-left
12. Hebrew               (he)        — right-to-left
13. Hindi                (hi)
14. Japanese             (ja)
15. Korean               (ko)
16. Chinese (Simplified) (zh-Hans)
17. Chinese (Traditional)(zh-Hant)
18. Indonesian           (id)
19. Thai                 (th)
20. Vietnamese           (vi)
```

Reply with any combination — numbers (`1, 3, 14`), codes (`es, fr, ja`), names (`Spanish, French, Japanese`), or just `all`.

### 2. Drop your screenshots into the terminal

Drag the image files from Finder/Explorer onto the Claude Code window. The paths show up in your prompt — submit, and the skill picks them up. PNG or JPG.

You can also pass a directory and the skill will pick up every image inside it.

### 3. OCR + translate

Claude quickly reads your project (CLAUDE.md, README, manifest, a few source files) to internalize what the app does, the tone, and which feature names to preserve — no file is written, no playback. Then it OCRs each screenshot and translates the text into every selected language in-conversation, applying the context as it goes.

### 4. Image regeneration

For each (screenshot × language) pair, Claude calls the Gemini Nano Banana MCP with the source image + the translated text replacements. Each output lands in `<project_root>/marketing/<lang_code>/<image>.png`.

---

## Output layout

For two input screenshots (`hero.png`, `home.png`) translated to Spanish, French, and Japanese:

```
<project_root>/marketing/
├── es/
│   ├── hero.png
│   └── home.png
├── fr/
│   ├── hero.png
│   └── home.png
└── ja/
    ├── hero.png
    └── home.png
```

That's it — just the localized images, one folder per language. No manifest files, no intermediate JSON, no clutter. If a translation needs a tweak, tell Claude what to change and ask it to regenerate that specific image (e.g. "redo `marketing/fr/hero.png` — translate 'Get Started' as 'Commencer' instead of 'Démarrer'").

---

## Cost notes

- **Translation** runs inside your Claude Code conversation — no separate bill.
- **Gemini Nano Banana**: one image-edit call per (image × language). Pricing depends on Google's current rates — check [ai.google.dev/pricing](https://ai.google.dev/pricing). 10 screenshots × 5 languages = 50 calls.

---

## Troubleshooting

**Slash command doesn't show up** — restart Claude Code. Skills are loaded at session start. Confirm the directory is at `~/.claude/skills/appstore-screenshot-translator/` and contains `SKILL.md`.

**"I need the Gemini Nano Banana MCP server configured"** — Claude couldn't find the `edit_image` tool. Run `claude mcp list` and confirm the `gemini` server shows as connected. Restart Claude Code after adding it.

**"Run this skill from the root of the project"** — `cd` into your app's project directory first. The skill needs a project root (`.git/`, `package.json`, etc.) so it can read code for translation context. If you're truly translating one-off marketing assets with no project, reply `skip context` and the skill will run without it (translations will be more generic).

**Translated text doesn't fit / gets cut off** — tell Claude which image and which string, and ask it to re-translate that one element more tightly (e.g. "redo `marketing/de/hero.png` — the headline is too long, try a shorter German phrasing").

**Brand name got translated** — tell Claude in your prompt before running ("the brand is MyApp, never translate it"), or after the fact ask it to redo the offending image with the correct brand name.

**Nano Banana changed parts of the image other than text** — Claude retries once automatically. If it still drifts on a particular image, the source may have non-trivial typography (handwritten, heavily stylized) that's outside Nano Banana's edit reliability. Fall back to Figma / vector workflow for those.

**RTL languages (Arabic, Hebrew) look wrong** — the skill adds an RTL hint to the Nano Banana prompt for `ar` and `he`, but RTL text in screenshots is harder to get right than LTR. Spot-check the output carefully and re-run with explicit corrections if needed.

---

## How it differs from other localization tools

- **No template required.** Most app-store-screenshot tools want you to author screenshots in Figma / a YAML template first, then re-render per language. This skill works on already-rendered PNGs.
- **No font management.** Nano Banana matches the original font from the pixels — you don't need to ship .ttf files or know what font the designer used.
- **Context-aware.** Translations are informed by your actual codebase, not just the source strings, so feature names and tone survive the trip.
- **One dependency.** Gemini MCP. That's it.

---

## License

MIT — see [LICENSE](./LICENSE).
