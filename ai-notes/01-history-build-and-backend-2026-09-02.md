# moongrader — ML MoonBoard grade predictor (frontend). History, how to rebuild, how the backend works

Live: https://rt567.github.io/moongrader/  ·  Repo: github.com/RT567/moongrader (branch `main`, Pages serves `/`)
Was called `moonboard` until 2026-09-02; the repo was renamed and every absolute path inside changed to `/moongrader/`.

## What it is

A one-page app: a photo of a 2019 MoonBoard (`mb2019.jpg`) with an invisible 11×18 grid of buttons over it. Clicking a hold toggles a blue ring on it. "grade climb" POSTs the selected hold list to a Python backend that renders the climb and runs a classifier; the predicted V-grade comes back and is shown. "clear holds" resets.

**Owner's warning: "it was finnicky to setup, the screenshots need to be sent exactly."** In practice that means the JSON body the frontend sends must stay *exactly* `{"holds":["F18","H15","C10","H5","D1"]}` — an array of `<Letter><Row>` strings, key `holds`, nothing else — because the backend script builds the climb image from that list. Verify the payload in the browser Network tab after any frontend change (recipe below).

## Where things live (three repos + one deploy)

| piece | where | notes |
|---|---|---|
| deployed frontend (compiled JS) | **this repo** `js/compiled/app.js`, `index.html`, `vendor/css/*`, images | Only the compiled bundle. No source here. |
| frontend **source** | github.com/RT567/moonboard-stack → `my-app2/` (local clone `~/silly/moonboard-stack/my-app2`) | shadow-cljs + Reagent + re-frame + re-com 2.13.2. `src/my_app2/{core,db,events,subs,views,styles,routes,config}.cljs`. `src - Copy/` is a stale duplicate; ignore it. |
| backend | github.com/RT567/moonboard-fastapi → deployed on **Render free tier** at `https://moonboard-fastapi.onrender.com` | `main.py` FastAPI, `POST /grade_climb` runs `python-image-creator/grade-singular-climb-from-holdlist.py <holds…>` as a subprocess and greps `Grade predicted as:` from stdout. CORS allows `*`. `moonboard-stack` also contains copies of `moonboard-fastapi/` and `python-image-creator/` but Render deploys from the `moonboard-fastapi` repo. |

Render free tier **spins the service down when idle**. First request after a quiet period: 50 s – 2 min (observed 2026-09-02: root GET 200 in 8.7 s, then 48 s, then 0.3 s). The frontend request timeout is 180 s for this reason (was 60 s until 2026-09-02).

## Timeline

| date | event |
|---|---|
| 2024-10-17 | First GitHub Pages deployment of the re-frame app (repo `moonboard`). Same day: stylesheet links switched to https, 404ing paths fixed, tab title set to "MoonGrader". |
| 2024-10-22 | "working comms between front and back" — the Render backend and the frontend talk. |
| 2024-11-04 | POST timeout raised to 60 s (twice), minor UI changes. This deploy's source was slightly *newer* than what's in `moonboard-stack` (title got the "Warning: this is a prototype and inference can take up to two minutes..." suffix; the idle text became "click some holds and then click grade climb"). Those two strings were re-ported into the source on 2026-09-02, so source and deploy match again. |
| 2024-11-04 | `moonboard-stack` repo created as the "entire directory" for the project. Companion repos `moonboard-fastapi` (backend) also date from here. |
| 2026-09-02 | Repo renamed `moonboard` → `moongrader`. `index.html` and `working_for_prod.html` paths `/moonboard/…` → `/moongrader/…`. Landing page link updated. |
| 2026-09-02 | Frontend **rebuilt from source** with: a re-com throbber + "grading... the server sleeps when idle, so the first one can take a minute or two" while `:flag` is true; red "grading failed, try again" when `:error` is set; idle hint instead of a fake "V3" default grade; timeout 180 s; `:grade-climb` now clears `:error`/`:grade` when it starts. Verified old vs new bundle send byte-identical POST bodies. |

## How the frontend works (re-frame)

- `db.cljs` default: `{:name "re-frame" :selected-holds [] :grade nil :flag false :error nil}`.
- Events (`events.cljs`): `:hold-clicked` toggles a coord in `:selected-holds`; `:clear-holds`; `:grade-climb` → `:http-xhrio` POST (json request/response formats from cljs-ajax via `day8.re-frame/http-fx`), sets `:flag true`; `:process-response` stores `(:grade response)` and clears `:flag`; `:process-fail` stores `:error`, clears `:flag`.
- Subs (`subs.cljs`): `::db` (= selected holds), `::grade`, `::loading` (= `:flag`), `::error`.
- Views (`views.cljs`): `main-panel` = banner (`nice-size.png` + "MoonGrader" title) → title with warning → two buttons → `grade-value` (cond: loading → throbber; error → red text; grade → "This climb has been classified by the model as: Vx"; else → idle hint) → the board image at absolute top 300px / left 20% → grid of toggleable rings → grid of invisible buttons. Grid geometry is hard-coded pixels (rows 30 px apart, columns 30.8 px, ring 45 px); don't move the image without moving both grids.

## Rebuild + deploy recipe

```bash
cd ~/silly/moonboard-stack/my-app2
npm install                       # needs node (nvm has v22) and java (openjdk 21 present)
npx shadow-cljs release app       # ~10 s after deps are cached; output resources/public/js/compiled/app.js
cp resources/public/js/compiled/app.js ~/silly/moongrader/js/compiled/app.js
cd ~/silly/moongrader && git commit -am "..." && git push
```
Commit the `.cljs` changes to `moonboard-stack` too, so the deployed bundle stays reproducible.

`index.html` here uses **absolute** `/moongrader/...` paths, so a local test must serve a parent directory:
```bash
mkdir -p /tmp/srv && ln -sfn ~/silly/moongrader /tmp/srv/moongrader && (cd /tmp/srv && python3 -m http.server 8352)
# → http://localhost:8352/moongrader/
```

## Verification recipe (do this after any frontend change)

1. Open the live site and the local build side by side in Chrome (devtools MCP works well).
2. In each, run in the console: click the same hold buttons (`[...document.querySelectorAll('button')].filter(b=>!b.textContent.trim())` are the 198 hold buttons, in DOM order), then click "grade climb".
3. Compare the `POST …/grade_climb` request bodies — they must be identical.
4. Wait for the 200 (may take a minute). Expect `{"grade":"V3"}`-style JSON. `V3` for the test holds `["F18","H15","C10","H5","D1"]`.
5. `favicon.ico` 404 in the console is normal.

## Known limitations / ideas not done

- Only the 2019 MoonBoard layout; holds grid is pixel-tuned to `mb2019.jpg` at 400×600.
- No "wake the server" ping on page load (would hide most of the cold start). Easy win if wanted: `GET https://moonboard-fastapi.onrender.com/` on mount.
- Error text is generic; the backend's own error strings arrive as `grade` ("Error occurred: …") and will be shown as if they were a grade.

## 2026-10-05 — header tidy (Rob: spinner + slow-inference text were hiding behind the board)

The board, rings and hold buttons are absolutely positioned at top 300px / left 20% (untouched, by Rob's
request: "leave the risky areas and work around them"). The fix only changes what sits above them, which
must end before y=300: smaller gaps, one short intro line + a muted "Prototype…" line, buttons side by
side (grade climb is primary), a fixed 60px status box, and plain `[:p]` instead of `re-com/p` (that one
forces width 450px). The header column is 400px wide with `margin-left: 20%` so it lines up over the board.
`index.html` hides the horizontal scrollbar at ≥600px only (the ring rows are 1000px-wide boxes); narrow
screens still scroll to reach the right-hand holds. Request body verified unchanged
(`{"holds":["A18","J17","F13","K8","D1"]}` for the same clicks). Grading status ends at y≈245.
Still not done: the board overflows on phones (needs the board's 20% offset changed in three places).

### Later 2026-10-05 — board centred and fully on screen on phones
- Board image, rings and hold buttons now all use `board-left` = `max(0px, calc(50% - 200px))` (views.cljs)
  instead of three separate `"20%"`s. Verified against the live site: all 198 buttons + 198 rings sit at the
  same offsets from the board image (max 0.125px subpixel difference); POST body unchanged.
- `index.html`: phones narrower than 430px get `<meta viewport width=430>` (browser scales the page to fit;
  no layout maths changes). `#app` is `position: relative` + `overflow-x: clip` (fallback: hidden +
  min-height 940px) so the 1000px-wide ring rows can't widen the page. `#app` is now the board's
  containing block, at the same top-left as the page.
- Logo 80→64px wide (was stretched).
- Backend check: direct POST of the test climb → V3 in 51s with the service already awake; through the live
  site 98s. It works, it's just slow.
