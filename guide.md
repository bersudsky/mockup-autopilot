# Figma mockup plugin — mechanics for Claude

Request protocol **v1**. This file is the only detailed description of the
mechanics: the skill loads it on every run. When the mechanics change, this file
changes; the skill stays as it is.

## How it works

Mockups are drawn by the mockup plugin in the user's Figma (its name is in
`manifest.plugin` or in the skill). Claude does not launch it and renders
nothing itself: it reads the file through Figma MCP (`use_figma`), writes a
**request** into the file, and the open plugin polls the file every 1.5 s, picks
the request up and runs it with the same renderer as its Insert button.

Load the `figma-use` skill before every `use_figma` call.

All communication goes through shared plugin data, namespace `mockupautopilot`.
Values are JSON strings. Plain `setPluginData` does not work in `use_figma` —
use `setSharedPluginData` / `getSharedPluginData`.

| Node | Key | Written by | What |
|---|---|---|---|
| `figma.root` | `manifest` | plugin | what the plugin can do right now |
| `figma.root` | `request` | Claude | the request |
| `figma.root` | `status` | plugin | progress and result |
| mockup group | `params` | plugin | what the mockup is made of |

Each mockup is a **group** (not a frame) with the phone, covers, shadows, glass
and cards inside.

## manifest — what the plugin can do

**Never take angles, devices or colours from memory or from this file** — only
from the manifest. The set grows: new angles and colour schemes appear, and the
manifest learns about them by itself.

```json
{"v":1, "plugin":"<plugin name>", "mockupHeight":1080,
 "templates":[
   {"id":"iphone-right", "device":"iphone", "angle":"right", "name":"Right 3/4",
    "frame":[402,874], "colors":["<colour 1>","<colour 2>"], "hint":"…"}
 ]}
```

- `v` — the plugin's protocol version. If it is higher than the one at the top
  of this file, the plugin is newer than this description: follow this file but
  tell the user.
- `templates` — in the order of the plugin panel. The first template of a device
  is its default angle.
- `device`, `angle` — how to name them in a request. `name` — the panel label.
- `hint` — optional: how people call the angle. Match the user's words
  ("front", "from the top", "three quarters left") against `angle`, `name` and
  `hint`. Not sure which template is meant — ask, listing the options by `name`.
- `frame` — the design size in pt that the device screen is made for. It also
  tells the device of a frame: ≈402×874 is iPhone, ≈360×780 is Android.
- `colors` — colours of **this** template. The first one is the default. Sets
  may differ between templates: check the colour for each one.
- `mockupHeight` — height of a finished mockup in pt. Width depends on the
  angle (currently 0.53–0.6 of the height), so leave rows to the `layout` grid:
  it steps by each mockup's actual width.

An empty manifest means the plugin has never been opened in this file.

## request

```json
{"v":1, "id":"req-1790000000000",
 "layout":{"x":0, "y":1100, "cols":4, "gap":100},
 "items":[
   {"source":"10:62", "template":"iphone-right", "color":"<colour>", "name":"Home — Right"},
   {"source":"10:26", "device":"android", "angle":"front"},
   {"source":"10:94", "template":"iphone-front",
    "highlights":[{"node":"10:120"}, {"node":"10:131"}], "glass":true},
   {"replace":"12:5", "color":"<another colour>"},
   {"source":"10:94", "template":"iphone-front", "x":3000, "y":0, "parent":"10:1"}
 ]}
```

- `id` — new and unique for every request, e.g. `"req-" + Date.now()`. A request
  whose `id` was already done is not run again.
- One item = one mockup. Item order = grid order.
- `source` — id of the frame with the design. Exported at 4×.
- `template` — id from the manifest, or `device` + `angle`. Nothing given — the
  first template in the manifest.
- `color` — a name from the template's `colors`; case and spaces do not matter.
  Not given — the default one. A colour the template does not have is an item
  error.
- `name` — name of the mockup group. Default: "<frame name> — <template name>".
- `replace` — redraw an existing mockup in place: same parent, layer order and
  position. Whatever is not given comes from its `params` (source, template,
  colour, highlights, glass, cover). A new screen in an old mockup is
  `replace` + `source`; its old highlights are then dropped.
- `x`, `y` — mockup position in the parent's coordinates; `parent` — id of a
  section or frame. Without `x`/`y` — the `layout` grid; without `parent` — the
  page of the source frame.
- `layout`: `x`, `y` — top-left corner of the grid. Without them the grid starts
  400 pt to the right of all source frames, aligned to their top. `cols` — per
  row (4), `gap` — spacing (100).
- `fit` — `false` turns off height fitting for this item (see below).

### Highlights

A highlight lifts part of the screen out of the phone as a floating card, with a
shadow, and optionally frosted glass under it and a cover patch over the place it
came from.

- `highlights` — up to **4** per mockup. Each is either `{"node":"<layer id>"}` —
  a layer **inside the source frame** (preferred) — or `{"x", "y", "w", "h"}` in pt
  of the frame. Optional per highlight: `r` — corner radius in pt (by default the
  layer's own radius, otherwise the device's usual one), `dy` — vertical offset
  of the card in pt, as in the plugin's area picker.
- Neighbouring areas merge into one card automatically.
- `glass` — frosted glass under the cards (default `false`).
- `cover` — paint over the place the card came from (default `true`).
- On `replace` without `highlights` the old ones are kept (same screen only).
  `"highlights": []` removes them.

**Finding the layer.** People point at an element in one of three ways — all
must work:

1. **Words** ("the balance card", "the Buy button"): search the frame's layers
   by name and by the text inside them (TEXT nodes).
2. **A screenshot of the element in the chat:** look at it, take a screenshot of
   the frame (`get_screenshot` or `node.screenshot()`), list the frame's layers
   with their boxes and texts, and match by text and look. Prefer the smallest
   layer that has its own background and fully contains the element.
3. **A selection in Figma** ("this one" with a layer selected): `get_metadata`
   shows the user's current selection at the top of its answer. Most precise.

If several layers fit, show the options briefly and ask. Highlight a whole
visible block (a card, a button, a row), not a text label inside it.

### Height fitting

Designs are not always drawn to the device screen: Android screens are often
360×732 while the mockup needs 360×780. Before export the plugin fits such a
frame: it clones it off-canvas, resizes the clone to the template's `frame`
height (the background stretches, layers pinned to the bottom such as a tab bar
move down), exports the clone and deletes it. **The frame in the file never
changes.**

Fitting applies only when the width matches the template exactly and the frame
is up to 15% shorter. Width is never stretched; a taller frame is not cut. It is
on by default; `"fit": false` turns it off for an item. Highlights are measured
on the fitted copy, so a tab bar highlight moves down together with the tab bar.

## status — progress and result

```json
{"id":"req-…", "state":"running", "at":1790000000000,
 "done":3, "total":5, "ok":3,
 "results":[{"i":0, "id":"20:1"}, {"i":1, "error":"Frame 10:26 not found"},
            {"i":2, "id":"20:9", "replaced":"12:5", "warning":"…"}]}
```

`state`: `running` → `done` (or `failed` if the whole request failed, then with
`error`). `done` also has `failed` — the number of failed items. `results[].i` is
the item index, `id` the created mockup, `warning` something worth telling the
user.

## params — on every mockup

```json
{"v":1, "source":"10:62", "template":"iphone-right", "color":"<colour>",
 "fitted":false, "glass":false, "cover":true,
 "highlights":[{"x":16, "y":100, "w":370, "h":80, "r":24, "dy":0}]}
```

A mockup made by the plugin is recognised by non-empty `params`, including ones
inserted by hand. `source: null` — the design was loaded from disk; it can only
be redrawn with a new `source`. `highlights` are in pt of the exported frame. In
mockups made by older versions `highlights` is a number: such highlights cannot
be restored on redraw — warn the user.

## Workflow

1. **Link.** `fileKey` and `node-id` from the URL (`10790-24017` → `10790:24017`).
   No link — ask for one.
2. **One read-only call:** `manifest`, `status`, and the frames of the page or
   section — id, name, type, size, position, parent, `params`.
3. **Parse.** Turn the instruction into a table "frame → template, colour,
   highlights, place". Ambiguous (which frames, which angle, which element,
   order) — show the table briefly and ask. Unambiguous — go on.
4. **Request.** Write `request`. Place the row so it does not overlap the
   frames: e.g. `y` = bottom of the lowest frame + 200.
5. **Wait.** Read `status` every ~5 s. No status with this `id` after 15 s — the
   plugin is closed: ask the user to open it in this file (Plugins menu, or the
   plugin's button in the right panel when nothing is selected). The request
   waits in the file; do not rewrite it.
6. **Result.** `done` → check `results`, report briefly; explain errors and
   warnings and offer a fix.

Move, arrange, rename, group, delete mockups yourself through `use_figma`, no
request needed. Redraw (angle, colour, screen, highlights) — only by request.

## Snippets

Read (page `PAGE_ID`):

```js
const page = await figma.getNodeByIdAsync("PAGE_ID");
await figma.setCurrentPageAsync(page);
const g = k => figma.root.getSharedPluginData("mockupautopilot", k);
const frames = page.children.map(n => ({ id: n.id, name: n.name, type: n.type,
  x: Math.round(n.x), y: Math.round(n.y), w: Math.round(n.width), h: Math.round(n.height),
  params: n.getSharedPluginData("mockupautopilot", "params") || null }));
return { manifest: g("manifest"), status: g("status"), frames };
```

For a section — the same over `section.children`.

Layers of a frame, to find a highlight (`FRAME_ID`):

```js
const f = await figma.getNodeByIdAsync("FRAME_ID");
const fb = f.absoluteBoundingBox, out = [];
const text = n => n.type === "TEXT" ? n.characters
  : ("findAll" in n ? n.findAll(t => t.type === "TEXT").map(t => t.characters).join(" | ") : "");
f.findAll(n => n.visible !== false && n.absoluteBoundingBox && n.width >= 16 && n.height >= 16)
 .slice(0, 400).forEach(n => {
   const b = n.absoluteBoundingBox;
   out.push({ id: n.id, name: n.name, type: n.type, x: Math.round(b.x - fb.x), y: Math.round(b.y - fb.y),
              w: Math.round(b.width), h: Math.round(b.height), text: text(n).slice(0, 80) });
 });
return out;
```

Write a request:

```js
const request = { v: 1, id: "req-" + Date.now(), layout: { x: 0, y: 1100 }, items: [ /* … */ ] };
figma.root.setSharedPluginData("mockupautopilot", "request", JSON.stringify(request));
return { id: request.id };
```

Status:

```js
return figma.root.getSharedPluginData("mockupautopilot", "status");
```
