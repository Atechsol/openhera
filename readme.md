# OpenHera

**Open-source, local-first infographic video generator.**
Describe what you want. OpenHera plans the scenes, has an LLM write the HTML and CSS, and renders a crisp MP4 or WebM on your own machine.

> **Status:** planning. This file marks the start of the project. Nothing is built yet.

---

## What it is

- You bring your own LLM and API key. No account, no cloud.
- An LLM writes HTML and CSS for each scene.
- Playwright steps every animation to an exact time, captures frames, and ffmpeg encodes them.
- You can add logos, images, brand colours and fonts.
- The interface is a terminal-style web app with an optional 3D look.

## How it works

```
Brief + options + brand assets
        |
   Planner (LLM)  -->  storyboard (scenes, durations, styles)
        |
   Scene writer (LLM, one call per scene)  -->  HTML + CSS
        |
   Renderer (Playwright frame capture)  -->  frames per scene
        |
   ffmpeg  -->  scene clips  -->  final MP4 / WebM
```

Only the LLM call leaves your machine.

## User workflow

1. Open OpenHera (`npx openhera`). On first run, set provider, model and API key in Settings.
2. Pick a **video type**.
3. Choose the basics: aspect ratio, total length, style pack.
4. Add the brief: a topic, pasted data or an uploaded file.
5. Optional: upload a logo, set brand colours and fonts, add images.
6. Generate the **storyboard**.
7. Review and edit it: reorder, delete or add scenes, change durations and text.
8. Generate scenes one by one, with a live preview.
9. Refine by prompt, for example "make scene 3 bolder". Only that scene is regenerated.
10. Export MP4 or WebM, with platform presets.
11. Optional: "Make all formats" regenerates the same storyboard for other aspect ratios.

## Options

| Option | Choices |
| --- | --- |
| Video type | Defined by template folders (list to come from Creeto's current types) |
| Aspect ratio | 16:9, 9:16, 1:1, 4:5, or custom width x height |
| Length | Total target, split across scenes by the planner |
| Style pack | clean-light, bold-dark, corporate, playful, editorial (planned) |
| Frame rate | 30 fps default, 60 fps optional |
| Resolution | 1080p default |
| Export | MP4 (H.264), WebM (VP9) |

Changing the aspect ratio later regenerates the scenes, because layouts differ between ratios.

## Scene model

A project is an ordered list of scenes. Each scene is validated with Zod.

```json
{
  "id": "s03",
  "title": "Revenue by region",
  "durationMs": 6000,
  "intent": "Show APAC growing fastest",
  "data": { "values": [] },
  "style": "bold-dark",
  "assets": ["logo"],
  "transitionOut": { "type": "fade", "durationMs": 400 },
  "html": "",
  "css": ""
}
```

- Default scene length is 4 to 8 seconds, with hard limits of 2 and 15 seconds.
- All animation must run on the browser animation timeline (CSS animations or the Web Animations API) and finish inside `durationMs`.
- No free-running timers, no `requestAnimationFrame` loops, no `Date.now()` or random-driven motion. The renderer must be able to seek to any time.

## Rendering

Export uses **frame capture**, not screen recording.

1. Load the scene in a clean page at the project resolution.
2. Wait for fonts and assets.
3. Pause all animations. For each frame, set the timeline to time `t` and take a screenshot.
4. Pipe frames to ffmpeg to make the scene clip.
5. Check for console errors, overflow outside the frame, and animation that is not seekable or ends late.
6. Join the clips with transitions and write the final file.

Why: exact timing, true 30 or 60 fps, and identical output on every run. The cost is that export is slower than real time, so scenes render in parallel pages. Previews play live in an iframe and are not recorded.

## LLM layer and prompt caching

- One `LLMProvider` interface, one adapter per provider: Anthropic, OpenAI, Gemini, and OpenAI-compatible endpoints (OpenRouter, Ollama, LM Studio, vLLM).
- Settings page: provider, model, API key, optional base URL, and separate model slots for **planner** and **scene writer**.
- Structured output validated by Zod. One automatic repair retry on failure.

Prompts are built from most stable to least stable so the prefix caches well:

1. System prompt
2. Style pack rules and tokens
3. Component library and animation helpers
4. Few-shot example scenes
5. Brand kit and asset list
6. Storyboard so far
7. The one scene to write now

Steps 1 to 5 are the shared cached prefix. Never put timestamps, random IDs or reordered keys in it. The UI shows cached reads, cache writes and fresh input tokens.

## Prompts

Prompts are plain Markdown files, not code.

```
prompts/
  system/
    planner.md
    scene-writer.md
    repair.md
  components/
  styles/
    clean-light/  style.md  tokens.css  examples/
    bold-dark/
  types/
    <video-type>/  type.md  recipes/  defaults.json
```

A **video type** is a folder: planner rules, scene recipes, defaults for length, ratio and style, and the brief form fields. Adding a type is adding a folder.

## Brand assets

- Upload logos (SVG preferred), images, colours and fonts per project.
- The LLM sees only a short asset manifest, for example `logo-light (SVG, 4:1)`.
- Scenes reference files as `asset://logo-light`. The renderer maps these to local files, which are never sent to the model.
- Render pages block network access except the local asset server. Uploaded SVGs are sanitised.

## Tech stack

| Layer | Choice |
| --- | --- |
| Language / runtime | TypeScript, Node 20+ |
| Monorepo | pnpm workspaces + Turborepo |
| Server | Fastify |
| UI | React + Vite + Tailwind, terminal style |
| UI effects | CSS 3D transforms first, Three.js only for background or intro, behind an FX toggle |
| Schemas | Zod |
| Rendering | Playwright (Chromium) |
| Encoding | ffmpeg (ffmpeg-static) |
| Storage | SQLite + project folders on disk |
| Secrets | Local config, optional OS keychain |
| Packaging | npm (`npx openhera`) and Docker |
| Tests | Vitest + Playwright test |

A headless CLI (`openhera render brief.md`) uses the same core engine.

## Planned repo layout

```
openhera/
  apps/
    web/          React + Vite UI
    server/       Fastify API, job queue
  packages/
    core/         scene schema, planner, scene pipeline, prompt builder
    providers/    LLM adapters
    renderer/     Playwright frame capture, ffmpeg
    styles/       built-in style packs
  prompts/
  examples/
  docs/
  docker/
```

## Roadmap

Four phases. Each one ends with a gate. The phases show order, not dates.

| Phase | Scope | Gate |
| --- | --- | --- |
| 1. Foundation | Monorepo, CI, settings page, one LLM adapter, one scene rendered by frame capture, plain terminal-style UI | One scene renders from a brief |
| 2. Storyboard | Planner, multi-scene projects, durations, clip joins, 3 style packs, Creeto prompts ported, video type templates | A 10-scene video exports end to end |
| 3. Brand and cost | Logo and asset kit, prompt caching, all four provider types, token usage view, aspect ratio presets | Cache hits shown in the UI |
| 4. Launch | 6 style packs, parallel export, 3D UI layer, Docker image, docs, examples | Release v1.0 on GitHub and npm |

## Risks and fixes

- **Frame capture is slow.** Render scenes in parallel pages, use JPEG frames for drafts, play previews live.
- **LLM HTML quality varies by model.** Strict schema, one repair pass, an eval set of 20 to 30 briefs, a recommended-models list.
- **Caching differs per provider.** Each adapter owns its rules, with a prefix-stability test.
- **Chromium and ffmpeg installs.** Bundle ffmpeg, install Chromium on first run, offer Docker.
- **Generated code safety.** Block network in the render page, sanitise SVGs, keep LLM output inside the browser sandbox.
- **Prompt ownership.** Confirm the Creeto prompts can be released under the chosen licence before the first public commit.

## Open decisions

- [ ] Licence: Apache-2.0 or MIT.
- [ ] Confirm the OpenHera name, domain and npm package.
- [ ] List of video types ported from Creeto.
- [ ] Which Creeto prompts can be published.

## Contributing

Not open yet. When it is, the easiest first contributions will be new style packs, new video types, new provider adapters and prompt improvements.

## License

Apache 2.0