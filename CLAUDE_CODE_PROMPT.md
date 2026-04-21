# Claude Code — Project Brief: Simrat Singh Gandhi Portfolio

Paste this entire file into Claude Code as the opening context. It describes the current state of the project, every design decision already made, and what remains.

---

## 1. What this project is

A product designer's portfolio website. The homepage is a **cinematic, scroll-driven 3D studio walkthrough**: the visitor lands inside a dark studio room, and as they scroll, the camera glides along a fixed path — pausing at each furniture piece on a V-formation of podiums. Hovering a piece reveals its name and collection; clicking "walks" the camera into the piece and transitions to that project's case-study page.

- **Owner:** Simrat Singh Gandhi — architect-turned-product-designer, based in Toronto, OCAD MDes in Inclusive Design.
- **Tagline:** *"Designing conversations first to create user informed products, furniture and decor."*
- **Vibe:** Dark, editorial, cinematic. Think high-end furniture brand meets gallery space. Not game-like, not playful — curated.
- **Current state:** A single working HTML file (`Draft26.html`) with the 3D homepage. The rest of the site (project case studies, about, playground, resume, contact) is **not yet built**.

## 2. Tech stack (locked in — do not swap)

- **Three.js r128** via CDN (`https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`)
- **GLTFLoader, RGBELoader, DRACOLoader, RoomEnvironment** from `https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/...`
- **Draco decoder** from `https://www.gstatic.com/draco/versioned/decoders/1.5.6/`
- **Fonts:** Google Fonts — `Outfit` (200/300/400/500) + `Cormorant Garamond` (400/500, italic variants)
- **Vanilla HTML/CSS/JS only.** No React, no bundler, no build step. Single-file architecture is intentional for now.
- **3D assets** live in a public GitHub repo: `https://github.com/Pachenko7/3D-Assets` — loaded via raw URLs. The combined studio is `Studio.glb` at `https://raw.githubusercontent.com/Pachenko7/3D-Assets/main/Studio.glb`. Individual furniture pieces are baked into this single GLB.

## 3. Design system

### Colors (CSS variables)
```
--bg:         #0E0D0B   /* near-black, warm */
--fg:         #E8E2D9   /* off-white, warm */
--muted:      #7A7368   /* warm grey */
--accent:     #C97941   /* burnt orange — the brand color */
--accent-dim: rgba(201,121,65,0.4)
```

### Typography
- **Body / UI:** `Outfit`, weights 200–500. Used for nav, buttons, labels, small caps text.
- **Display / editorial:** `Cormorant Garamond`, italic. Used for logo, piece names, transition text, mobile menu links.
- **Tracking:** Heavy letter-spacing (2–5px) on uppercase UI labels. Tight on display serif.

### Motifs (already implemented, keep consistent everywhere)
- **Custom cursor** — solid accent dot + spring-lagged ring that grows to 60px with `--fg` border on hover. Hidden on touch devices.
- **Film grain overlay** — SVG `feTurbulence` noise at 3.5% opacity, fixed over everything at `z-index: 200`.
- **Scroll progress bar** — 2px accent line fixed at top, `z-index: 500`.
- **Header** — transparent at rest, fades to `rgba(14,13,11,0.85)` + backdrop-blur once `scrollProgress > 0.02`.

## 4. Homepage scene — the canonical data

### The five interactive pieces
Order is **anticlockwise** around the room: Atelier (right podium) → Zephyr (left/center) → Bloom (far left).

| Name | Collection | Description | Object in Blender |
|---|---|---|---|
| **Easel** | The Atelier Collection | Designed for painters/artists with changing posture needs | `Easel` |
| **Canvas** | The Atelier Collection | Designed for artists with a 9 to 5 | `Canvas` |
| **Palette** | The Atelier Collection | Designed for den rooms with multiple table needs | `Palette` |
| **Zephyr** | *(standalone)* | Designed for gamers and their action figures | `Zephyr` |
| **Bloom** | *(standalone)* | Customizable potted planters | regex `/^Bloom\d+$/` (multiple objects) |

Each piece has a `closeup` object — a hand-tuned camera pose (pos, look, fov) used by the walk-in animation when the piece is clicked. These closeups are already dialed in; don't regenerate from heuristics.

### Camera path (scroll 0.00 → 1.00, loops after first full pass)

Seven keyframes with hold frames (two identical consecutive keyframes = camera pauses). Interpolation is smoothstep (`s = seg*seg*(3 - 2*seg)`). FOV is interpolated per-segment.

| t | Purpose |
|---|---|
| 0.00 | Entrance — wide establishing shot, camera at y≈2.6m, looking into the room |
| 0.06 | Hold (entrance) |
| 0.26 | Stop 1 — Atelier Collection (right podium), FOV 24 |
| 0.34 | Hold |
| 0.50 | Stop 2 — Zephyr, FOV 24 |
| 0.58 | Hold |
| 0.74 | Stop 3 — Bloom, FOV 28 |
| 0.82 | Hold |
| 1.00 | Back to entrance — seamless loop |

Exact coordinates are in `CAMERA_PATH` in `Draft26.html`. **Do not resample or regenerate these.** They were tuned using the in-scene debug camera (see §7).

### Zones — for the piece counter
Three scroll zones define which piece is "active" for the right-edge pip counter and mobile info card:
- Atelier: center `0.30`, range `±0.08`
- Zephyr: center `0.54`, range `±0.08`
- Bloom: center `0.78`, range `±0.08`

## 5. Scene setup (Three.js details)

- **Renderer:** WebGL, antialias, high-performance, `ACESFilmicToneMapping`, exposure 0.95, sRGB output, physically correct lights, shadow maps enabled (PCFSoft).
- **Environment:** `RoomEnvironment` through `PMREMGenerator` at intensity 0.04 — just enough for accurate PBR color, not for lighting.
- **Fog:** `FogExp2(0x0E0D0B, 0.04)` — bg color, exponential falloff.
- **Lighting:**
  - `AmbientLight` 0xE8E2D9 @ 0.25
  - `DirectionalLight` key 0xFFF1DC @ 2.2 from (-4, 8, 6), 2048² shadow map
  - `DirectionalLight` fill 0xC9B79A @ 0.35 from (3, 3, 2)
  - Per-piece `SpotLight` overhead — per-piece intensity/angle/height via `spotConfig` on the piece config.
- **GLB traversal overrides** on load:
  - `castShadow` + `receiveShadow` on all meshes
  - Textures forced to `sRGBEncoding`
  - `envMapIntensity = 0.5`
  - Any mesh whose name matches `/podium|wall|floor|ceiling|ground|backdrop|plinth|base/i` gets its map nulled and color overridden to `--bg` (so the podiums/walls render as deep black, letting the pieces pop)
  - Any mesh whose name starts with `text` gets a fresh `MeshStandardMaterial` in `--fg` color (for the "ZEPHYR" / "THE ATELIER COLLECTION" / "BLOOM" wall text)
- **Per-piece material cloning** — so hover emissive boosts don't bleed across shared materials. Each highlighted material stores `_origEmissive` + `_origEmissiveIntensity` in userData for lerping back.

## 6. UI modules (all present, all match the brand)

1. **Preloader** — centered "Entering Studio..." in Cormorant italic, 200px progress track, accent fill, live percent. Uses `THREE.LoadingManager` — real asset progress, not fake. Fades out in 0.8s when done.
2. **Custom cursor** — dot + ring, ring springs with `* 0.1` lerp, grows on hover.
3. **Film grain** — see §3.
4. **Scroll progress bar** — top, accent, 2px.
5. **Header** — logo (Cormorant, 17px) + nav (Work, Playground, About, Resume, Contact — Outfit 300, 10px, 3px tracking, uppercase, muted with accent underline on hover). Hamburger on mobile.
6. **Mobile menu** — full-screen overlay, Cormorant italic 28px links.
7. **Intro** — bottom-left, "PRODUCT DESIGNER" eyebrow + two-line practice description + animated scroll hint. Fades out once `scrollProgress > 0.04`.
8. **Piece info tooltip (desktop)** — follows cursor on hover, shows collection eyebrow + italic piece name + "Enter Project →" CTA. Only appears when raycaster hits an interactive mesh.
9. **Mobile piece info** — bottom card with active piece name + circular CTA arrow. Driven by scroll-zone detection, not hover.
10. **Piece counter** — right edge, vertical pip stack of 5 dots. Active pip stretches to a vertical accent bar.
11. **Walk-in transition** — 1.4s eased camera move to the piece's `closeup` pose, then a full-screen fade overlay displaying the piece name in Cormorant italic, then (currently) resets. The `window.location.href = piece.link` redirect is commented out — wiring that up is a future step.

## 7. Debug / authoring tools (leave intact — they're how poses get tuned)

A hidden fly-camera lives in the animate loop. The toggle key is **`J`** (not D — that's used for movement).

- `J` — toggle debug mode on/off, HUD shows in center-top
- `WASD` — move in the camera's local frame, `Shift` for fast
- `Space` / `Ctrl` — up / down
- Click-drag — look around (yaw/pitch)
- `[` / `]` — decrease / increase FOV
- `C` — copy current pose as a `CAMERA_PATH` line (`{ t, pos, look, fov },`) to clipboard
- `T` — prompt for the `t` value to stamp into the next copy
- HUD shows current `t`, scroll, FOV, focal length, and the pose line live

The debug mode must not run when user is scrolling normally. It fully overrides the scroll-driven camera while active.

## 8. Interaction rules

- **Scroll** is `wheel` + `touchmove`, accumulated into `targetScroll` with sensitivity `0.00035` (wheel) / `0.001` (touch). Smoothed into `scrollProgress` with `* 0.055` lerp.
- **Looping:** until the user reaches `t = 1` once, scroll is clamped `[0, 1]`. After that, `hasLooped = true` and scroll wraps seamlessly in both directions.
- **Hover:** raycaster runs every frame against `interactiveMeshes`. First hit is resolved to a piece via `findPieceData` (walks up the parent chain checking `userData.pieceData` / `userData.pieceName`). On hover: emissive boost with lerp factor `0.15` per frame toward `HIGHLIGHT_COLOR` (`0xffd9a0`), intensity bump `+0.6`, plus the piece-info card + cursor ring "hovering" class.
- **Click:** if a piece is hovered and we're not already walking in and not in debug mode, call `walkIntoPiece`.
- **Mobile CTA:** taps `mobCta`, uses the currently active piece from the zone detector.

## 9. What's NOT built yet (the roadmap)

1. **The five project case-study pages** — `#project-easel`, `#project-canvas`, `#project-palette`, `#project-zephyr`, `#project-bloom`. Each should be a long-scroll editorial page matching the dark brand. Wire the redirect by uncommenting `window.location.href = piece.link` in `walkIntoPiece`.
2. **About page** — architecture → product design story, OCAD, inclusive design research, approach (co-design, parametric, digital fabrication — Baltic birch, PLA joints, screwless assembly).
3. **Playground** — experiments, prototypes, sketches. Format TBD.
4. **Resume** — downloadable PDF + in-page summary.
5. **Contact** — email (`hello@simratsinghgandhi.com` is the placeholder), LinkedIn, Behance, Instagram.
6. **Router** — decide client-side vs. multi-page. Current hash links (`#work`, etc.) are placeholders.
7. **Performance pass** — lazy-load the GLB only after preloader appears, preload fonts, compress Studio.glb further with glTF-Transform if it's over 6MB combined.
8. **SEO + meta** — OG image, descriptions, sitemap.
9. **Analytics** — Plausible or similar, privacy-first.
10. **Accessibility pass** — reduced-motion fallback (static hero image instead of 3D scene), keyboard navigation, focus states, ARIA on custom buttons.

## 10. House rules for Claude Code working on this project

- **Preserve the camera path constants and closeups.** They were hand-tuned in-scene. If they need to change, use the debug tool (`J`) to reshoot poses and paste new keyframes — do not lerp or recompute them from object positions.
- **Match the design system exactly.** Any new UI element must use the existing CSS variables, type pairing, and spacing language. Don't introduce new accent colors, new fonts, or playful elements.
- **Single-file stays single-file** until the project case-study pages ship. At that point, split into a folder (`/`, `/projects/*`, shared `/assets/*`, `/styles/*`) but keep zero-build — vanilla HTML/CSS/JS, CDN imports.
- **Don't replace Three.js r128.** The loader CDN paths are pinned to this version; upgrading touches `GLTFLoader`, `RoomEnvironment`, `DRACOLoader`, and the encoding APIs (`outputEncoding` vs `outputColorSpace`). Out of scope unless explicitly asked.
- **Mobile parity is non-negotiable.** Every interaction has a touch/mobile path already — respect that pattern when adding new ones.
- **Name things the way Blender names them.** The `INTERACTIVE_PIECES` config is the source of truth. New pieces must ship with matching Blender object names (or a `namePattern` regex like Bloom's) plus a hand-tuned `closeup` pose.
- **Console logging stays minimal** — there's one `console.log('=== Studio objects ===')` traversal on load for debugging unknown object names. Keep that; don't add more.

## 11. Current file inventory

- `Draft26.html` — single-file homepage, ~1060 lines. This is the latest working version.
- External: `Studio.glb` in the `Pachenko7/3D-Assets` GitHub repo.

## 12. First task (when I give you one)

When I hand you a task, assume:
- You can edit `Draft26.html` directly, or split it into files if the task warrants it (ask first before splitting).
- You have access to my local machine — you can run a static server (`python3 -m http.server` or similar) to test.
- Don't push to GitHub unless I say so.
- Always test on a narrow viewport (~380px) and a desktop viewport after any UI change.

---

**End of brief.** Reply with "Studio ready" once you've read this, and wait for my first task.
