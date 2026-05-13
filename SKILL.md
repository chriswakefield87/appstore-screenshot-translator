---
name: appstore-screenshot-translator
description: Translate text in App Store / Play Store screenshots into multiple languages while preserving the original visual layout. Reads the user's project to learn what the app does, extracts visible text from each screenshot via vision, translates it in-context, then uses the Gemini Nano Banana MCP to regenerate each screenshot with the translated text. Outputs into <project>/marketing/<lang>/ for each selected language. Use when the user types /appstore-screenshot-translator, asks to localize app store screenshots, translate screenshot text, or generate multi-language versions of marketing screenshots.
user-invocable: true
---

# /appstore-screenshot-translator — App Store Screenshot Translator

This skill is **user-invocable**: it runs when the user types `/appstore-screenshot-translator`. Walk them through the phases below in order. Do not skip phases without explicit instruction from the user.

The skill orchestrates: **read the user's project for context → guided language picker → guided screenshot collection → OCR + translate in-conversation → confirm the translation plan with the user → regenerate each image via Gemini Nano Banana → text-diff verify each output → write outputs to `<project>/marketing/<lang>/`**.

---

## Phase 0 — Prerequisites

Before anything else, verify the environment. If a check fails, stop and tell the user what's missing.

1. **Project root.** The current working directory should look like a project root — at least one of: `.git/`, `package.json`, `pyproject.toml`, `setup.py`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `Gemfile`, or `CLAUDE.md` should be present. If none are, stop and say:

   > Run this skill from the root of the project whose screenshots you're translating — I read the codebase to learn what the app does, which feature names matter, and what tone to use. If you don't have a project (e.g. you're translating one-off marketing assets), reply "skip context" and I'll proceed without it (translations will be more generic).

   If the user replies "skip context," remember that and skip Phase 3 entirely.

2. **Gemini Nano Banana MCP.** Confirm the `edit_image` tool from `@houtini/gemini-mcp` is available (typical fully qualified name: `mcp__gemini__edit_image`). If it isn't exposed, stop and say:

   > I need the Gemini Nano Banana MCP server configured to regenerate the localized images. See the README in this skill for the install command (`claude mcp add --scope user gemini --env GEMINI_API_KEY=... -- npx -y @houtini/gemini-mcp`). Once `claude mcp list` shows it connected, restart Claude Code and re-run `/appstore-screenshot-translator`.

   Any other Gemini image-edit MCP works too as long as it accepts a reference image plus an edit prompt — adapt the call shape in Phase 6 if so.

---

## Phase 1 — Language picker

Present this menu to the user exactly as written, then ask them to reply with numbers, language codes, names, or `all`:

```
Which languages do you want to translate to? (Pick any combination — reply with
numbers like "1,3,7", codes like "es,fr,ja", names like "Spanish, French", or "all")

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

Parse the user's reply liberally. Accept mixed formats (`"1, French, ja"`). Resolve everything to BCP-47 codes (the parenthesized codes above are the canonical forms used everywhere downstream). If something is ambiguous (e.g. "Chinese") ask one quick clarifying question.

Echo back the resolved list as a comma-separated string of codes + names so the user can spot mistakes, then continue.

---

## Phase 2 — Screenshot collection

Tell the user, verbatim:

> **Drop your screenshots into the terminal now** — drag the image files from Finder/Explorer onto this window and they'll appear as paths. PNG or JPG. When you're done, hit enter on an empty line or just say "go".

Wait for their next message. It should contain one or more file paths. Resolve each to an absolute path, verify each file exists and is readable. If any path doesn't exist or isn't a PNG/JPG, list the bad ones back to the user and ask them to re-drop or correct.

If the user passes a directory instead of files, expand it: include every `*.png`, `*.jpg`, `*.jpeg`, `*.webp` directly inside (do not recurse). Show the user the expanded list before proceeding.

Confirm the resolved list of screenshots back to the user one last time before moving on.

---

## Phase 3 — Internalize app context (skip if user said "skip context")

Read the project just enough to translate fluently. **Do not write a context file. Do not play context back to the user.** Hold what you learn in conversation memory and apply it directly when translating in Phase 5.

Read in this order, stopping the moment you have a confident picture:

1. **`CLAUDE.md`** if present — usually nails "what is this app" in one read.
2. **`README.md`** — first 100–200 lines. Headline pitch, feature list, tone signals.
3. **Project manifest** — `package.json` (`name`, `description`, `keywords`, notable dependencies), or `pyproject.toml`, `Cargo.toml`, etc. Dependencies reveal domain (`expo` → mobile, `stripe` → payments, etc.).
4. **Up to 5 source files** only if 1–3 didn't suffice. Good targets: route/page files (`app/`, `pages/`, `src/screens/`), the main `App.*` entry point, or files named after core features. Read excerpts, not whole files. Three is usually enough; five is the cap.
5. **Glance at the screenshots themselves** for recurring feature names and brand voice signals.

When translating in Phase 5, keep these mental notes from what you read:

- **App name + brand vocabulary** — these never translate.
- **Domain** (fitness / finance / productivity / dev tool / game / social / ...) — informs word choice.
- **Tone** (playful & casual / direct & professional / technical & precise / aspirational & lifestyle / ...) — informs register.
- **Audience** (consumer / developer / SMB owner / enterprise / ...) — informs formality and assumed vocabulary.
- **Feature names from the codebase** — preserve verbatim unless the user said otherwise.

If the project is sparse and you can't read tone, translate neutrally — a wrong tone is worse than no tone bias. If you're uncertain about something specific (e.g. "is *Quick Sync* a feature name or just a description?"), ask the user one targeted question rather than guessing.

Do not delay translation for this phase — it's quick reading, not a deliverable. Move straight to Phase 4 when you're ready.

---

## Phase 4 — Create output dirs (silent)

Once languages and screenshots are confirmed and context is internalized, silently create the output directories — don't narrate this, and don't announce timing yet (announce happens after the user confirms the plan in Phase 5.5):

```bash
for lang in <each selected lang code>; do mkdir -p "marketing/$lang"; done
```

Use the BCP-47 codes (`es`, `fr`, `pt-BR`, `zh-Hans`, etc.) as directory names. The only files that should ever land in `marketing/` are the final localized PNGs:

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

**The only files that should ever appear under `marketing/` are the final localized PNGs.** No OCR manifests, no translated manifests, no per-language JSON, no `.txt` logs, no debug dumps — nothing but the images. OCR results, translations, and progress state all live in conversation memory only. If you catch yourself reaching for `Write` during this skill, stop: the only writer is the Gemini `edit_image` MCP call, which produces a PNG at `outputPath`.

---

## Phase 5 — OCR + translate (in memory, silently)

For each input screenshot, do this once. Hold the result in conversation memory only — do not write files, do not show the user what you extracted unless they ask.

Use vision (the `Read` tool on the image) to mentally catalogue every visible text element: headlines, sub-headlines, body, badges, ratings ("4.9 ★"), captions, footer text, button labels, everything. For each, note in your working memory:

- The exact source string.
- Its role (`headline | subheadline | body | caption | button | badge | rating | other`).
- Its position in the frame (top-left/center/right, middle, bottom-left/center/right + a more specific hint if useful — "upper third, ~80% width, centered").
- Style cues — font weight, relative size, color, casing, effects (drop shadow, stroke, gradient).
- What sits behind the text (solid color, photo, gradient, UI element) so Nano Banana preserves it.

If a piece of text is unreadable, ask the user briefly — don't invent.

For each target language, translate every captured element in-conversation. Hold these rules:

- **Honor the tone you internalized in Phase 3.** Playful app → playful target; enterprise tool → professional target. No context gathered → translate neutrally.
- **Translate naturally for marketing copy** — match the spirit, not the syntax.
- **Match source length where you can.** Headlines stay short. Buttons stay button-shaped.
- **Preserve casing style** (Title Case → Title Case; ALL CAPS → ALL CAPS where natural).
- **Keep source punctuation** (`!`, `?`, `…`) when present.
- **Pass through verbatim:** brand names, app names, hashtags, URLs, proper nouns, and feature names from the codebase.
- **Note overflow internally.** If a translation is >1.5× the source character count AND the role is `headline` or `button`, remember to add a wrap hint to the Nano Banana prompt — but don't surface this to the user as a warning.

Carry the translations into Phase 5.5 — do not write them to disk first.

---

## Phase 5.5 — Show the translation plan, ask to confirm

Before burning Nano Banana calls, show the user a compact preview of every text change so they can spot bad word choices before you generate. One round-trip prevents a batch of wrong images.

Display one block per screenshot. Group source strings on the left and each target language's translation in its own column (or slash-separated if there are only 2–3 languages). Append a short list of strings you're keeping verbatim (brand names, prices, dates, status-bar values) so the user sees the omissions too.

Example format for 2 languages:

```
Translation plan — 3 screenshots × 2 languages (es, fr):

[1.png]
  TRACK           → CONTROLA / SUIVEZ
  PAYMENTS        → PAGOS / PAIEMENTS
  PER WEEK        → POR SEMANA / PAR SEMAINE
  PER MONTH       → AL MES / PAR MOIS
  PER YEAR        → AL AÑO / PAR AN
  Due today       → Vence hoy / Dû aujourd'hui
  Due 14 May      → Vence 14 may / Dû le 14 mai
  Due 15 May      → Vence 15 may / Dû le 15 mai
  Mark paid       → Pagado / Payé
  Kept as-is: ChatGPT, Spotify, Netflix, Disney+, Crunchyroll, £9.69, £41.98, £503.76, £20.00, £11.99, £9.99, £4.99

[2.png]
  MANAGE          → GESTIONA / GÉREZ
  ...

Reply "go" to generate, or tell me what to change (e.g. "for French use 'GÉRER' instead of 'GÉREZ'", or "translate Streaming as well").
```

For 4+ languages, use a markdown table instead so columns stay readable.

Keep it terse — one line per source string. If a source string repeats across screenshots (e.g. "Due today"), list it on each screenshot anyway so the user sees it in context.

Wait for the user's reply. If they ask for changes, apply them and re-display only the affected rows. Only proceed to Phase 6 after explicit "go" / "yes" / "looks good" (or equivalent). Once confirmed, announce timing:

> Generating `<N>` images. This may take a few minutes — Nano Banana runs roughly 10–20 seconds per image.

If the total is under 5 images, drop the timing hint and say:

> Generating `<N>` images…

---

## Phase 6 — Regenerate images via Nano Banana

For each (image × language) pair, call the `edit_image` tool from `@houtini/gemini-mcp` using the translations confirmed in Phase 5.5:

```json
{
  "prompt": "<see template below>",
  "images": [
    { "filePath": "<absolute path to the source screenshot>" }
  ],
  "outputPath": "<absolute path: <project_root>/marketing/<lang_code>/<image_stem>.png>"
}
```

Always pass `filePath` (not base64) — Nano Banana edits work on the full-resolution image; base64 transport caps at 1 MB. Both paths must be absolute.

Build the `prompt` from the confirmed translations using this template:

```
Edit this app store screenshot to localize it into <Target Language Name>. Replace ONLY the text shown below — keep every other pixel (background, photo, UI chrome, icons, device frame, decorative elements) byte-identical to the input. Match the original font weight, casing style, color, alignment, and approximate size for each replacement. Do not translate brand names, app names, or proper nouns unless they have an official localized form.

Text replacements (preserve position and styling exactly):

1. [<role>, <position>, <style>] "<original>" → "<translated>"
2. [<role>, <position>, <style>] "<original>" → "<translated>"
...

Output a single image at the same resolution and aspect ratio as the input.
```

For any element you flagged in Phase 5 as >1.5× length expansion on a headline/button, append a final paragraph to the prompt: "Note: the translation for `<role>` text is longer than the original — wrap to multiple lines if needed rather than shrinking aggressively."

For right-to-left languages (Arabic `ar`, Hebrew `he`), add: "This is a right-to-left language — flip text alignment direction where appropriate (e.g. left-aligned text becomes right-aligned), but keep all non-text layout (icons, photos, device frame, decorative elements) unchanged."

The tool writes the edited image to `outputPath` and returns the saved path. Independent (image × language) calls can run in parallel.

### Clean up the sibling HTML preview

`@houtini/gemini-mcp` writes a sibling `.html` preview file next to every `outputPath` (e.g. for `marketing/fr/1.png` it also drops `marketing/fr/1.html`). The MCP has no flag to suppress it. After each successful `edit_image` call, delete the sibling `.html` — it's a side-effect artifact, not part of the deliverable. The contract is "PNGs only under `marketing/`".

```bash
rm -f "<outputPath without .png>.html"
```

### Progress messaging

Per generated image, emit **one** short tick line — nothing more:

> ✓ `marketing/fr/hero.png`

That's the user's only visible signal of progress. Don't paste the prompt, the translations, or any state beyond what Phase 5.5 already showed. Don't pause mid-batch — confirmation happened once, in Phase 5.5.

### Verify (text-diff, not just visual)

After each generation, Read the output image and use vision to enumerate every word actually rendered. Then run a deterministic checklist against the confirmed Phase 5.5 plan for this (image × language) pair:

1. **Every translated string from the plan must appear in the output** — character-for-character, allowing only whitespace and quote-style variations. If you translated "Due 14 May" → "Vence 14 may", the output must contain "Vence 14 may" — not "Vence 14 mayo", not "Vencehoy", not "suscoipción" when you wrote "suscripción".
2. **No source-language string from the plan should appear** — if the source said "DETAILS" and you translated it to "DÉTAILS", "DETAILS" must NOT be in the output.
3. **Strings marked "kept as-is" should appear unchanged** (brand names, prices, status-bar values).

If any check fails, retry **once** — name the failed strings explicitly in the retry prompt ("the previous output rendered 'suscoipción' but I need 'suscripción' — replace every instance"). Do this silently — don't tell the user about the retry unless it also fails.

If the retry still fails, surface the specific problem image with both source + bad output paths and which strings are wrong, then continue with the rest of the queue rather than blocking.

---

## Phase 7 — Final line

When the whole queue is done, emit a single closing line — nothing fancy:

> Done. `<N>` images generated under `marketing/`.

Only mention specifics if something needs the user's attention (a per-image retry that still failed, an unreadable text element that was skipped). Otherwise: one line, done.

---

## Things to avoid

- **Do not translate brand names, app names, hashtags, or proper nouns** unless the user explicitly says to.
- **Do not invent text that wasn't in the original.** Mark unreadable text `<UNREADABLE>` and ask.
- **Do not call Nano Banana without the source image attached.** Text-only prompts produce a from-scratch image, not a localized edit.
- **Do not loop on failure.** One retry per image, then surface.
- **Do not skip Phase 5.5.** The translation table preview is what stops bad word choices from turning into bad batches.
- **Do not commit user images or API keys to the skill repo.** The `.gitignore` covers this.
- **Do not start translating an image until you've fully catalogued its text in memory.** The OCR pass for an image must be complete before any of that image's translations begin — otherwise translations drift from the actual source strings.
- **Do not write outputs outside `<project_root>/marketing/`, and do not write anything other than the final localized PNGs.** No `.manifest.json`, no `.translated.json`, no logs, no per-image debug files — just the images. That's the contract with the user; respect it.
