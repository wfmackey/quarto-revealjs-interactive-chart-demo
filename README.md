# Interactive charts in Quarto revealjs — a demo deck

> Built largely with AI. The deck, the four chart implementations and this
> README were written by Claude working from my direction, and I have reviewed
> and tested the result. Read it as a worked example rather than as hand-tuned
> reference code.

Live: <https://wfmackey.github.io/quarto-revealjs-interactive-chart-demo/>

The same slider chart built four different ways, so the trade-offs are visible
rather than argued about.

This is a demo of mechanics, not of design. The revealjs theme, fonts and
colours are stock. Nothing depends on a Quarto extension, a custom format, a
Lua filter or a theme file, and `tidyverse` is the only R dependency. Every
chart uses simulated data, so the deck carries no data files and none of the
numbers mean anything.

## Requirements

- Quarto 1.4 or later
- R 4.1 or later
- One R package: `tidyverse`

```r
install.packages("tidyverse")
```

## Build and view

```bash
quarto preview demo.qmd
```

Use `preview`, not `render`. It builds the deck *and* serves it over `http://`,
which the Observable charts need — see below.

## Observable charts need a web server

This is the one thing that catches people out, so it is worth stating plainly.

If you `quarto render demo.qmd` and then double-click `demo.html`, the browser
opens it at a `file://` address. Slides 1 and 4 will work. Slides 2 and 3 will
be blank, and the page pops up an alert saying the Observable runtime does not
work with `file://` URLs.

The reason is that Quarto loads Observable as an ES module
(`<script type="module">`), and browsers apply strict cross-origin rules to
modules. Over `file://` every file counts as a separate opaque origin, so the
module is blocked before it runs. This has nothing to do with the internet: the
deck fetches nothing external. It just needs to be served from an origin.

Which parts are affected:

| Slide | What it uses | Works from `file://` |
|---|---|---|
| 1, hand-written SVG | a classic `<script>` block | yes |
| 2, Observable Plot | `ojs` cells, ES modules | no |
| 3, R + `ojs_define` | `ojs` cells, ES modules | no |
| 4, ggplot frames | a classic `<script>` block | yes |

Any of these serves it correctly:

```bash
quarto preview demo.qmd
```

Quarto's own server, with live reload on save. Easiest, and the one to use while
you are working on the deck.

```bash
quarto render demo.qmd && python3 -m http.server --directory . 8000
```

Then open <http://localhost:8000/demo.html>. Any static file server does — this
one just happens to ship with macOS. In R, `servr::httd()` is the equivalent.

For sharing, publish it rather than emailing the file: `quarto publish quarto-pub`,
GitHub Pages, Netlify, or any web host. A published deck is served over `https://`,
so the Observable slides work for everyone who opens the link.

If you must hand someone a single file, know that the two Observable slides will
be blank when they open it locally. Versions 1 and 4 exist partly for that
reason — a hand-written `<script>` and a stack of pre-rendered images both
survive being opened from a USB stick.

`embed-resources: true` is set, so the built `demo.html` inlines its images,
fonts and the Observable runtime, and needs no network. It still needs an origin.

## Contents

| Path | Function |
|---|---|
| `demo.qmd` | The deck, including its four rules of CSS |
| `_flip-javascript.qmd` | Version 1 of the chart, kept in its own file because it is long |
| `figs/` | Charts written by R at render time; not tracked, the build makes them |

There is no theme file and no stylesheet. The deck uses the built-in revealjs
`simple` theme, and the `<style>` block in `demo.qmd` holds four rules: two for
the image stack the frame slider toggles, a height cap so the charts do not
overrun a slide, and a wider label column for Observable Inputs, whose own
stylesheet caps it at 120px and wraps every label. None of it is styling.

## One chart, four ways

The chart plots after-tax against before-tax return for a stylised leveraged
property investment. Because a capital gains tax discount taxes only part of
the gain, the after-tax line sits above the 45-degree no-tax line over a range
of losses — the gold band. Tax runs through the personal income schedule rather
than a flat rate, so the line is kinked wherever the investor crosses a bracket.

| Version | Approach | Lines |
|---|---|---|
| 1 | Hand-written SVG built and moved by plain JavaScript | ~270 |
| 2 | Observable JS with Observable Plot (Quarto built-in `ojs` cells) | ~30 model + ~55 spec |
| 3 | R computes a lookup grid, `ojs_define()` ships it, Observable draws | ~35 R, ~10 JS, same spec |
| 4 | ggplot renders eleven frames, a slider picks one | ~30 R |

The same model sits behind all four, so the numbers agree. Versions 1 to 3 are
also pixel-identical — same 640x610 frame, margins, ticks, type sizes and
annotation placement — so the only thing left to compare is the code. Version 4
is left in ggplot's own idiom, because looking like an ordinary R chart is the
whole point of that approach.

## The trade-offs

Version 1 is the fastest and the most controllable. It re-solves the model on
every slider tick and mutates SVG attributes rather than redrawing, so it feels
instant; and effects that depend on history — the fading trail, the ghost of the
previous line — are trivial because the code already knows which slider moved.
It is also the version nobody can read: the economics is fifteen lines
interleaved with two hundred lines of pixel arithmetic, nothing transfers to
the next chart, and there is no line you can point at and call the equation.

Version 2 cuts that to about a third and makes the model a named function with
no pixels in it. Worth noticing what the Plot spec spends its lines on: more
than half of it turns the library's defaults back off — `tickSize: 0`, explicit
tick lists, `label: null`, a `tickFormat` to stop Plot printing a Unicode minus
sign. Left alone, Plot draws a good chart that is not your chart. The trail and
ghost line are also gone: "what changed since last time" is exactly what a
reactive dataflow discards. In exchange, the spec is a function you can call
again, which is more than version 1 ever offers.

Version 3 puts the economics in R, where it can be reviewed and tested, and
leaves the browser two lines of arithmetic. It calls version 2's `flipPlot()`
unchanged, which is the one clear win of the library route — the drawing work
transfers, where nothing in version 1 does. It works here only because the model
is linear in the tax parameters once the before-tax payoff is known, so just the
expensive part had to be tabulated. The general cost is grid size: five
parameters at ten steps each is 100,000 rows.

Version 4 is all R. Its ceiling is arithmetic — one parameter at eleven steps is
eleven images and about a megabyte, two parameters is 121 images, and the full
seven-slider version is unreachable.

Deliberately excluded: `shinylive`, which would run the R model in the browser
through WebAssembly. It needs a Quarto extension, ships tens of megabytes, and
re-renders the whole ggplot on every slider tick. Also Vega-Lite, whose
declarative slider bindings are close to ideal until you need a loop, which the
mortgage amortisation schedule requires.

## If you only take one thing

Split the file. Even in version 1, pulling the economics into a pure
`model(params)` function and leaving `draw(state)` separate fixes most of the
readability objection without changing the technology.
