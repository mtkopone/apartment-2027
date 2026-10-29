# New apartment: interactive 3D models

Single-file three.js pages modelling furniture and rooms for the owner's apartment in Helsinki
(Punavuorenkatu). No build step, no framework. Published on GitHub Pages from `main` at
https://mtkopone.github.io/apartment-2027/ (repo `mtkopone/apartment-2027`, Pages serves the repo root).

## Layout

- `index.html`: root index linking each project.
- `aava-wardrobe/`: wardrobe versions `wardrobe*.html` plus an index. Older, finished project.
- `mikko-bedroom/`: the headboard wall.
  - `bed-wall.html` is **v1, the original as drawn. Never change it.** It is the reference.
  - `bed-wall-2.html` is **v2, the live working model**. All new work goes here. Everything is
    one configurable file; do not create v3, v4… copies. New ideas become switchable options.
  - `index.html`: lists v1, v2 and preset links into v2 (`bed-wall-2.html?f=…`).
- `reference/`: the designer's elevation JPG, floor-plan PDF and screenshots. Read these when a
  request says "look at the jpeg". Measure from them with PIL rather than eyeballing
  (the elevation is 3508 px wide; at 2000 px display width 1 mm ≈ 0.33 px).
- `package.json` + `serve.json`: `npm start` serves on :3000 with clean URLs off (query strings
  must survive). `node_modules/` is gitignored.

## How v2 is organised (`bed-wall-2.html`)

Read the file in parts; it is ~2400 lines. Rough order:

1. CSS, then markup: left panel `#spec` (Options checkboxes, Interior fold, Dimensions fold),
   right panel `#look` (moods, surfaces), intro modal `#intro`, tooltip `#tip`, busy badge `#busy`.
2. `CONFIG` at the top of the script. **All measurements in cm.** Coordinates: x along the
   headboard wall (0 = box's left edge), z into the room from the wall, y up from the floor.
   - `CONFIG.features`: the Options switches (`alcove`, `niches`, `tray`, `pullouts`, `frame`,
     `nightstand`). `CONFIG.interior`: swappable things (lamp model).
   - `CONFIG.wall` and `CONFIG.bed` hold every dimension. Feature-specific numbers live in
     sub-blocks (`wall.alcove`, `wall.niches.slots`, `wall.lids`, `wall.alcovePower`, `bed.frame`…).
   - `effective()` derives `EFF.wall` / `EFF.bed` from CONFIG + features. Builders read `EFF`,
     never mutate `CONFIG.wall` directly.
3. Texture generators (procedural canvases: parquet, walnut, gingham, bouclé, linen, floral
   stand-in) and `mat` (the material table). The real wallpaper loads from the Rebel Walls CDN.
4. The Look system: `LOOK_SCHEMA` (surfaces and their fields), `MOODS`, `applyLook()` which
   mutates materials in place (no rebuild), `encodeSurface/decodeSurface` for the URL.
5. Builders: `buildRoom`, `buildWall` (+ `addUpright`, `addWell`, niches, plates…), `buildDoors`,
   `buildBed` (+ `addBedFrame`), `pillow()`, `buildDimensions`, `renderSpec`.
6. `rebuild()` tears down and rebuilds the scene from `EFF`; keeps open/closed state of pieces.
7. State, setters, URL read/write, click/hover, animation loop.

Helpers: `box(w,h,d,x,y,z,mat)` takes size and **min corner** in cm; `rbox` is rounded;
`placed(geo, mat, x,y,z)` positions by centre; `M = 0.01` converts cm to scene metres.

## Conventions that matter

- **Everything shareable is in the URL query.** Any new user-facing state must be written in
  `writeUrl()` (only when it differs from the default) and read in the deep-link block near the
  end. Hash tokens are a fallback for `file://`. Test both directions.
- **Options**: add the flag to `CONFIG.features`, a checkbox `data-feature="…"` in the Options
  fieldset, a name in `FEATURE_NAMES`. Incompatible pairs go in `EXCLUDES`; dependencies in
  `REQUIRES` (dependants are dropped when their parent turns off).
- **Movable pieces** (doors, lids, pullouts, bed drawers, nightstand lid) carry
  `userData = { open, target, click, … }` and are registered via `pieces(kind)`. They animate in
  the loop, sync to Interior controls in `syncPieceControls()`, and go into the URL as `all` or a
  list of 1-based indices.
- **Clickable things** set `userData.click`; **notes** set `userData.tip` (HTML, built from the
  `TIPS` table using CONFIG numbers). Hover/click raycasting skips anything hidden.
- **Loose items** (lamp, bottles, books, bedding, cupboard contents) go into `itemsGroup(parent)`
  so the Items switch can hide them. Built-ins (pads, plates, grommets) do not.
- **Lights / materials / rebuilds** call `markBusy()` so the progress cursor and badge show.
  Do not name a body class the same as a component class (that caused a full-page layout flash).
- Colours derived in code must be set as sRGB (`setRGB(r,g,b, THREE.SRGBColorSpace)`).
- Avoid coplanar faces between different materials: inset or shorten one (several z-fighting
  bugs came from this).
- The left wall and everything beyond it live in `room.userData.westSide`, hidden when the camera
  is outside it.
- Dimension tags: add to `buildDimensions()`; spec rows to `renderSpec()`. When a feature changes
  a number the owner cares about, show it in both.

## Workflow

1. Edit `mikko-bedroom/bed-wall-2.html` with exact string replacements (a Python script asserting
   each `old` string occurs once works well for multi-site edits). Never touch `bed-wall.html`.
2. **Verify by rendering, not by assumption.** Start a throwaway server
   (`npx serve -l 3123 .` in the background, kill it after) and use headless Chromium via
   Playwright to load a URL, set the camera through `window.bedwall.camera/controls`, click
   things, and screenshot. Check for page errors. Look at the screenshot before reporting.
   Sample pixels with PIL when judging colour; downscaled screenshots mislead.
   - Playwright lives in the npx cache; the chromium headless shell is in
     `~/Library/Caches/ms-playwright/chromium_headless_shell-*/`. Pass `executablePath` and
     `--use-angle=swiftshader --enable-unsafe-swiftshader`.
   - Keep scripts and screenshots in the session scratchpad, not the repo.
3. Commit only when asked. Commit messages describe the change in a short paragraph. Pushing to
   `main` deploys; wait ~1 min and confirm with `curl` that the live page contains the change.
4. Do not publish anything as a claude.ai Artifact; the owner views pages locally or on Pages.

## The owner's preferences (learned)

- Wants realistic, buildable details: real product dimensions (Familon 60×80 pillows, Innolux
  Rondo Air, Schneider Exxact anthracite sockets, Rebel Walls Blooming Bouquet Dark), electrics
  that a carpenter and electrician could actually do, and honest flags when something is not
  feasible (e.g. pull-out uprights are probably not structurally sound).
- Measures carefully: when asked to match the drawing, sample it.
- Prefers small, direct changes reported briefly with the resulting numbers.
- Text in the page is English; the owner is Finnish and some references are in Finnish.
