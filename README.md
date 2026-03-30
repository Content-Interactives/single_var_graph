# Single-Variable Graph (Number Line) Interactive

**Live Site Link:** https://content-interactives.github.io/single_var_graph/

A React interactive that presents a **horizontal number line** on a grid. Users **draw** with the pointer: horizontal strokes become **closed intervals** (segments) on the line with optional **blue endpoint arrows** when a segment reaches the extended ends; small vertical **loops** near a tick register as **open** or **filled** circles at integer positions (\(-5\) through \(5\)). Undo/redo walks an explicit **action list** (segments and circle markers).

## Tech stack

| Layer | Choice |
|--------|--------|
| UI | React 19 |
| Build | Vite 7 with `@vitejs/plugin-react` |
| Styling | `App.css` (includes segmented control “glow” styles); component uses inline layout on the root |
| Deploy | `gh-pages` publishing `dist/` |

Production asset URLs assume the app is served under **`/single_var_graph/`** (`vite.config.js` → `base`). The live link above matches that path on the Content-Interactives GitHub Pages org.

## Repository layout

```
src/
  main.jsx
  App.jsx              # Imports App.css, renders SingleVarGraph
  App.css              # #root layout + segmented-glow-button / .compact styles
  index.css
  components/
    SingleVarGraph.jsx # Number line, gestures, SVG, history reducer
```

There is no separate `glow.css`; control chrome lives in `App.css`.

## Geometry and coordinates

- **Canvas**: `500×500` SVG pixels; horizontal line at **`LINE_Y = HEIGHT / 2`**.
- **Labeled ticks**: integers **`MIN` … `MAX`** (\(-5\) … \(5\)).
- **Drawing span**: The visible tick segment is inset by **`PADDING`** (40px) from left/right; **`GRID_CELL`** (30px) is one **unit** on the line. **`lineMargin`** centers the \([-5,5]\) span so consecutive ticks are exactly **`fullUnit`** apart.
- **Extended range**: **`EXTENDED_MIN` / `EXTENDED_MAX`** (\(-6\) / \(6\)) so gray **number-line arrows** sit one unit beyond the outer labels (`valueToX` maps this range to `[lineStart, lineEnd]`).
- **`xToValue` / `valueToX`**: Linear map between pixel \(x\) along the drawable line and real “value” positions, including the extended ends for arrow placement.
- **While drawing**: `clientToSvg(..., { forDrawing: true })` clamps \(x\) to `[lineStart, lineEnd]` and \(y\) to **`[drawYMin, drawYMax]`** (`LINE_Y ± DRAW_VERTICAL_MARGIN`) so strokes stay near the line but can rise/fall for circle detection.

Background **grid** is a full-plot pattern with **`GRID_OFFSET_X` / `GRID_OFFSET_Y`** so partial cells don’t clip asymmetrically at edges.

## Gesture → history actions

On **pointer up**, the current polyline is classified:

1. **“Empty circle” (open endpoint)**: `path.length >= 4`, horizontal span `< EMPTY_CIRCLE_MAX_SPAN`, vertical span `>= EMPTY_CIRCLE_MIN_VERTICAL`. Center \(x\) is converted with `xToValue`; tick is **`Math.round`**, clamped to `[MIN, MAX]`. If **`pathLength / bboxPerimeter >= FILLED_CIRCLE_INK_RATIO`** (heavy scribble), commit **`ACTION_FILLED_CIRCLE`**; else **`ACTION_EMPTY_CIRCLE`**.
2. **“Filled circle” (closed endpoint on the line)**: small \(x\) span and **small** \(y\) span (`< EMPTY_CIRCLE_MIN_VERTICAL`), `path.length >= 2` → **`ACTION_FILLED_CIRCLE`** at nearest tick.
3. **Otherwise**: Treat as a **interval along the line**. Bounding \(x\) → `minVal` / `maxVal` via `xToValue`; endpoints snap to **`Math.round`** ticks, clamped to `[EXTENDED_MIN, EXTENDED_MAX]`, converted back to **`valueToX`**. If collapsed to one \(x\), leave a single point in `path` without pushing history; else push **`ACTION_SEGMENT`** with `data: [startPt, endPt]` at **`LINE_Y`**.

Constants: **`EMPTY_CIRCLE_*`**, **`FILLED_CIRCLE_INK_RATIO`**, etc., are tuned at the top of `SingleVarGraph.jsx`.

## History model

- **`history`**: Ordered list of actions:  
  - `{ type: ACTION_SEGMENT, data: [ {x,y}, {x,y} ] }`  
  - `{ type: ACTION_EMPTY_CIRCLE, tick: number }`  
  - `{ type: ACTION_FILLED_CIRCLE, tick: number }`
- **`historyIndex`**: Prefix of `history` that is “active”.
- **`reduceHistoryToState(slice)`** rebuilds:
  - **`segments`**: all segment actions in order,
  - **`emptyCircleTicks`** / **`filledCircleTicks`**: sets by tick; later circle actions **override** the same tick (empty vs filled mutually exclusive).

**Undo/redo**: Move `historyIndex`; no mutation of past actions. **Reset** clears `history` and index.

**`pushHistory`**: Truncates any redo tail (`[...h.slice(0, idx), action]`), then increments index (uses **`historyIndexRef`** so deferred `endDrawing` commits see the latest index).

## Rendering

- **Open circles**: stroke only at **`valueToX(tick)`**, **`EMPTY_CIRCLE_RADIUS`**.
- **Filled circles**: same center, solid fill.
- **Segments**: Horizontal spans may be **split** by **`splitSegmentAtOpenCircles`** so blue strokes **do not pass through** the interior gap reserved for open circles (merged interval gaps in pixel space).
- **Axis**: Gray line from `lineDrawStart` to `lineDrawEnd` (slightly extended past ticks for arrow bases), ticks and labels (unicode minus for negative labels).
- **Blue arrows**: For each segment whose endpoint \(x\) is within **`END_ARROW_TOLERANCE`** of `lineStart` / `lineEnd`, draw blue triangles coincident with the gray arrowheads so user-drawn rays **cover** the axis arrows.

In-progress **`path`** is rendered as a blue `<path>` until it resolves to an action.

## Pointer and touch

- **Mouse**: `mousedown` / `mousemove` / `mouseup` / `mouseleave` (leave ends stroke).
- **Touch**: `touchstart` / `touchmove` / `touchend` / `touchcancel`; a **`useEffect`** registers **`touchmove`** with **`{ passive: false }`** and **`preventDefault`** while drawing so the page does not scroll.
- Root **`touchAction: 'none'`**, **`userSelect: 'none'`**.

Controls **do not** re-export pointer handlers from the overlay; they are plain buttons sitting above the SVG.

## Accessibility notes

The graph root is a plain `div` (no `role` / `aria-label` in the current code). Primary interaction is **drag-to-draw**; keyboard users may need an alternative if this is embedded in a course requiring full WCAG coverage.

## Scripts

| Command | Purpose |
|---------|---------|
| `npm run dev` | Vite dev server |
| `npm run build` | Output to `dist/` |
| `npm run preview` | Serve `dist/` locally |
| `npm run deploy` | Build then `gh-pages -d dist` |

## Tunables

Key literals in `src/components/SingleVarGraph.jsx`: **`MIN` / `MAX`**, **`WIDTH` / `HEIGHT` / `PADDING` / `GRID_CELL`**, **`DRAW_VERTICAL_MARGIN`**, **`END_ARROW_TOLERANCE`**, circle-detection thresholds, **`ARROW`** extension fudge. Changing **`MIN`/`MAX`** requires keeping **`EXTENDED_*`** consistent with the “one past” tick convention if arrows should stay aligned.
