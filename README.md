# Image Fusionner

A PowerShell 7 + ImageMagick pipeline that turns background images into complete grids of
90×90 PNG tiles, decorating individual tiles (or merged blocks of tiles) with buttons, icons,
logos, overlays, spanning images, text, transparent cut-outs, and window/island borders — all
driven by plain-text layout files.

It's built for generating tiled artwork sets — for example menu/launcher grids where each cell
is a separate 90×90 image and some cells carry a button, a spanning banner, a label, or a
framed "window."

---

## Requirements

- **PowerShell 7** (`pwsh`) is the target. It also runs under Windows PowerShell 5.1.
- **ImageMagick** — bundled with the project at `Bin\magick.exe` and already set up. Nothing is
  installed system-wide; the script calls that executable directly.
- **Windows.** Stale-file cleanup uses the Recycle Bin API and the optional launcher uses
  `taskkill`, so the tooling assumes Windows.
- *(Optional)* The .NET Framework C# compiler (ships with Windows) if you want to build the
  hidden launcher — see [Hidden launcher](#hidden-launcher-with-progress-window).

---

## Folder structure

Everything lives under one project folder. The script finds its own location at runtime, so it
can be run from any working directory.

```
Image_Fusionner\
    script.ps1                 <- the main script
    Launcher.exe               <- optional hidden launcher (built from launcher.cs)
    launcher.cs                <- launcher source (optional)
    build.bat                  <- builds Launcher.exe (optional)

    Bin\
        magick.exe             <- ImageMagick (required)
        db\                    <- processing cache (auto-created)

    Grid_layout\               <- layouts that get PROCESSED (one run per .txt)
        Apps-Roku.txt          <- ships pre-populated with sample device layouts
        Marantz.txt               you can study, run, or edit
        Samsung.txt
        Main&&rokuTV.txt
        ...

    Layout Templates\          <- BLANK starter templates, one per grid size.
        layout_2x3.txt            NOT processed here - copy one into Grid_layout\
        layout_4x6.txt            to begin a fresh layout.
        layout_6x9.txt
        layout_8x12.txt

    Source_Images\
        Backgrounds\           <- source images to be tiled
        Buttons\               <- *.png  (!button)
        Icons\                 <- *.png  (!icon)
        Logos\                 <- *.png  (!logo)
        Overlays\              <- *.png  (!overlay)
        Images\                <- *.png  (!image / spanning pictures & banners)

    Output_Images\             <- generated tiles + preview sheets (auto-created)
```

**Only files in `Grid_layout\` are processed.** `Layout Templates\` is just a library of blank
starters to copy from — nothing there is rendered until you copy it into `Grid_layout\`.

Any missing source folder is created automatically on first run (with a warning), so you can
start from just `script.ps1` and `Bin\magick.exe`.

> **Folder names are plural** — `Logos`, `Overlays`, `Images` — even though the layout
> instructions are singular (`!logo`, `!overlay`, `!image`).
>
> There is **no `Banners` folder** — banners are done with `!image` from the `Images\` folder
> (`!banner` is still accepted as an alias for `!image`).

---

## Quick start

`Bin\magick.exe` comes bundled and set up. `Grid_layout\` already contains working sample
layouts (device remotes like Marantz, Samsung, Xbox-One, and the Roku/Shield menus — see
[Included sample layouts](#included-sample-layouts)), and `Layout Templates\` holds a blank
starter for each grid size. So you can either run the samples as-is or start your own:

1. Drop background images into `Source_Images\Backgrounds\`.
2. Add PNG assets to whichever element folders you'll use (`Buttons`, `Icons`, `Logos`,
   `Overlays`, `Images`).
3. Pick a layout to work with:
   - **To learn from or tweak an example**, open one of the sample `.txt` files already in
     `Grid_layout\`.
   - **To start fresh**, copy a blank from `Layout Templates\` into `Grid_layout\`, rename it,
     and uncomment the tiles you want. (Add `&&` to the name for output variants — see
     [the `&&` filename trick](#the--filename-trick).)
4. Run the script:

   ```powershell
   pwsh -File .\script.ps1
   ```

5. Find the results under `Output_Images\<LayoutName>\<Background>\`, plus a stitched preview
   image one level up.

Every `.txt` in `Grid_layout\` is processed in turn, so **all** the sample layouts run unless
you move or delete the ones you don't want. The samples pin a specific background with
`!background = name`, whereas the blank templates use `!background = *all*`.

---

## Included sample layouts

`Grid_layout\` ships with finished layouts that double as a feature tour — each one leans on
different parts of the syntax. Open them alongside this README to see the techniques in context.

| Layout | Grid | What it shows |
|--------|------|---------------|
| `Apps-Roku` / `Apps-Shield` | 4×6 | A grid of streaming-app logos stacked on framed buttons (`!button` + `!logo` with per-tile `[size / radius / x / y]` overrides), a two-tile `!image` banner, a `!border` window, and a labelled Return button. |
| `Jump-Menu` | 4×6 | A large title spanning the whole top row in a custom font (`!text` with `[font=…, size=55]` across four tiles), over a grid of device/app logos on black buttons. |
| `Marantz` | 6×9 | A dense AV-receiver remote: `!border` windows with hex colours, a `!frame` running down a whole column, an `!image` d-pad across a 3×3 block, `!image` up/down pads across vertical tile pairs, a bottom banner, and many labelled buttons. |
| `Samsung` | 6×9 | A TV remote: a top `!image` logo banner, an `!image` d-pad, buttons pairing a recoloured `!icon` with a `!text` label, `!border` and `!frame` groupings, and an HDMI-input row. |
| `Xbox-One` | 6×9 | A sparser controller layout with a transparent SNES d-pad `!image` across a 3×3 block plus menu buttons. |
| `Main&&rokuAV`, `Main&&rokuTV`, `Main&&shieldAV`, `Main&&shieldTV` | 6×9 | Full remote "home" screens that demonstrate the `&&` trick (four variants → folders `Main-rokuAV`, `Main-rokuTV`, … all sharing the `main_` file prefix), plus `!clear` transparent tiles, `!overlay` glass plates, `!frame` islands grouping button rows, spanning `!text`, and `!image` banners. |

If you just want to experiment, the `Main&&…` set is the most complete reference for combining
`!clear`, `!overlay`, `!frame`, and banners on one screen; `Marantz` and `Samsung` are the
best for dense button/label grids.

---

## Layout files

A layout file is a `.txt` file in `Grid_layout\`. It selects a grid size, chooses which
background(s) to use, optionally sets a diorama effect, and lists per-tile instructions.

- Blank lines are ignored.
- `#` starts a comment — everything after it on the line is ignored (whole-line or inline).

### Directives (file-level)

```text
!grid = 4x6                 # grid size: one of 2x3, 4x6, 6x9, 8x12 (invalid -> 4x6)
!background = *all*         # every image in Backgrounds\
!background = sunset        # only sunset.<ext>, matched without extension
!diorama = Tilt-shift       # optional per-layout effect (see Diorama effects)
```

`!grid` is `<cols>x<rows>`, so `4x6` = 4 columns × 6 rows = 24 tiles. All four supported grids
are 2:3 aspect. `!diorama` (`none` / `Tilt-shift` / `Shadowbox`, case-insensitive) overrides the
global `$DioramaMode` for that layout only; omit it to use the global setting.

### Tile instructions

A tile is `r<row>c<col>` (1-based, row then column) — `r1c1` is top-left. An instruction maps
one or more tiles to a comma-separated list of elements:

```text
r1c1 = !button = play, !icon = star, !text = "Watch Now"
```

**Layer order is left → right = bottom → top.** In the example, the button is drawn first, the
star icon on top of it, then the text on top of that.

Listing several tiles before the `=` acts on the whole group:

```text
r1c1, r1c2, r2c1, r2c2 = !logo = netflix
```

### Elements

| Element    | Example                       | Source folder | Notes                                                        |
|------------|-------------------------------|---------------|--------------------------------------------------------------|
| `!button`  | `!button = play`              | `Buttons\`    | inline overlay: size, opacity, X/Y                           |
| `!icon`    | `!icon = star`                | `Icons\`      | rounded corners, opacity, X/Y, optional **recolour**         |
| `!logo`    | `!logo = netflix`             | `Logos\`      | rounded corners, opacity, X/Y                                |
| `!overlay` | `!overlay = glass`            | `Overlays\`   | full-tile overlay (e.g. a glass effect): size, opacity, X/Y  |
| `!image`   | `!image = topbar`             | `Images\`     | scales one picture **across a block** of tiles, then slices  |
| `!text`    | `!text = "Play"`              | —             | font, size, color, opacity, gravity, offset, faux styles     |
| `!border`  | `!border`                     | —             | greyscale "window" tiles + frame their neighbours            |
| `!frame`   | `!frame`                      | —             | inverse of border: frame an "island," optionally grey moat   |
| `!clear`   | `!clear`                      | —             | make the tile background transparent (or faded)              |

Asset names are given **without** the `.png` extension, and all assets must be PNG.
`!boarder` is accepted as an alias of `!border`; `!banner` as an alias of `!image`.

> **`!button = *all*` was removed.** Name a specific button instead. Because of that, output no
> longer has a per-button subfolder.

Text supports line breaks with `<br>`:

```text
r3c3 = !text = "First line<br>Second line"
```

`!clear` makes the tile's background fully transparent by default (handy for letting icons/text
float on transparency); `!clear [opacity=40]` leaves a faded background instead. Other elements
on the same line are still composited on top.

### Multi-tile behaviour

Listing more than one tile behaves differently depending on the element:

- **`!logo` / general merge** — the listed tiles are cut from the background as one region,
  resized to a **single** 90×90, decorated, and that one image is copied into every tile slot in
  the group (so the full tile count still holds). Good for one big centred logo.
- **`!image`** — one picture is scaled across the whole block (fitted by default, or stretched
  with `[aspect=no]`), then **sliced back** into individual 90×90 tiles so the picture spreads
  across them. This is also how you make a banner: list a full row and use `[aspect=no]`.
- **`!text`** — the text is drawn **once across the block** at full resolution, then sliced into
  tiles, so it spans them. Putting `!text` on a multi-tile line switches the whole line into this
  block/span mode, so any other element on that line is rendered once across the block too.
- **`!border` / `!frame`** — the frame is drawn per sub-cell so it lines up along the abutting
  edges.

### Per-element overrides

Any element value may be followed by a `[ ... ]` block that overrides the global settings **for
that element only**. Anything you leave out keeps its global default.

```text
!button = play    [size=50x50, x=20, y=20, opacity=85]
!icon   = star    [size=40x40, radius=8, opacity=90, color=white]
!logo   = netflix [size=70x70, x=10, y=10, radius=10]
!overlay = glass  [size=90x90, opacity=60]
!image  = topbar  [aspect=no, x=10, y=10, opacity=85]
!text   = "LIVE"  [font=DejaVu-Sans, size=16, color=yellow, gravity=South, bold=yes]
!border           [style=Raised, color=white, thickness=12, roundness=20]
!frame            [style=Raised, color=white, thickness=10, roundness=20, greyscale=yes]
```

Recognised keys (aliases in parentheses):

| Key                    | Applies to                          | Meaning                                            |
|------------------------|-------------------------------------|----------------------------------------------------|
| `size` (`s`)           | button / icon / logo / overlay / image / text | ImageMagick geometry, e.g. `50x50`       |
| `x` (`posx`)           | all                                 | horizontal offset in pixels                        |
| `y` (`posy`)           | all                                 | vertical offset in pixels                          |
| `opacity` (`op`, `o`)  | button / icon / logo / overlay / image / clear | 0–100                                   |
| `radius` (`r`, `corner`)| icon / logo                        | rounded-corner radius in pixels                    |
| `color` (`colour`, `c`)| icon (recolour) / text              | colour name or `#RRGGBB`                           |
| `font` (`f`)           | text                                | ImageMagick font name                              |
| `bold` (`b`)           | text                                | `yes`/`no` — faux bold                             |
| `italic` (`i`, `italics`)| text                              | `yes`/`no` — faux italic (slant)                  |
| `underline` (`u`, `ul`)| text                                | `yes`/`no` — underline                            |
| `gravity` (`g`)        | text / image                        | ImageMagick gravity (`Center`, `South`, …)         |
| `aspect` (`ar`)        | image                               | `yes` = fit, `no` = stretch/fill                   |
| `style` (`st`)         | border / frame                      | `Flat Solid` / `Beveled 3D` / `Raised`             |
| `thickness` (`t`, `width`)| border / frame                   | stripe width in pixels                             |
| `roundness` (`round`)  | border / frame                      | outer-corner radius in pixels                      |
| `greyscale` (`grey`, `gs`)| frame                            | `yes`/`no` — desaturate the surrounding tiles      |

Yes/no keys accept `y/yes/on/true` and `n/no/off/false`.

> **Don't use `rgb(r,g,b)` colours inside `[ ]`** — the commas are read as key separators. Use a
> name (`white`) or hex (`#1a2b3c`).

### Text styling

`!icon color` recolours the whole icon to that colour (great for turning a black glyph white or a
brand colour); `!text color` sets the text colour. Bold / italic / underline are **faux** styles
drawn with ImageMagick primitives, so they work with any font without needing a separate bold or
italic font face. Their look is tuned by the text-style config variables
(`$TextBoldStrokeWidth`, `$TextItalicSlant`, `$TextUnderlineThickness`, `$TextUnderlineGap`).

---

## Diorama effects

A single global switch, `$DioramaMode` (or a per-layout `!diorama` directive), selects one
treatment:

- **`none`** — no effect.
- **`Tilt-shift`** — a miniature / "tiny model" look: a sharp horizontal focus band with the rest
  blurred, plus a saturation and contrast boost. It is applied to the **whole background before it
  is split**, so every tile, merged region, and image-slice inherits it; buttons/icons/logos/text
  are composited afterward and stay crisp. Tuned by `$DioramaBlurSigma`, `$DioramaFocusCenter`,
  `$DioramaFocusHeight` (0 = blur the whole background evenly), `$DioramaFeather`,
  `$DioramaSaturation`, `$DioramaContrast`.
- **`Shadowbox`** — 3D recessed-window depth that **enhances `!border`**: the greyscale window
  tiles get an inner shadow along their outward edges so the cut-out looks like it sinks into a
  box. It has no effect unless a layout uses `!border`, and doesn't touch `!frame`. Tuned by
  `$DioramaShadowDepth` and `$DioramaShadowOpacity`.

The two effects are gated separately in the render code; the switch is a single either/or.

---

## Configuration reference

All tunables live in clearly-commented blocks at the top of `script.ps1`. Paths are resolved
automatically. Positions are pixel offsets from a gravity anchor; sizes are ImageMagick geometry
strings; opacity is `0–100`. Current defaults:

**Tile** — `TileSize` `90` (the whole pipeline assumes 90×90 output).

**Button** — `85x85`, X/Y `2.5`/`2.5`, opacity `100`.

**Icon** — `60x60`, X/Y `15`/`15`, opacity `100`, corner radius `4`, `IconColor` empty (no recolour).

**Logo** — `60x60`, X/Y `15`/`15`, opacity `100`, corner radius `10`.

**Border** — style `Flat Solid`, color `black`, thickness `3`, roundness `25`.

**Frame** — style `Flat Solid`, color `black`, thickness `5`, roundness `16`, greyscale `no`.

**Overlay** — `90x90`, X/Y `0`/`0`, opacity `100`.

**Image** — aspect `yes`, size empty (auto-fit the block), X/Y `0`/`0`, opacity `100`, gravity `Center`.

**Text** — font `DejaVu-Sans-Mono` (validated at startup; falls back if missing), size `14`,
color `white`, X/Y `0`/`0`, opacity `100`, gravity `Center`; bold/italic/underline `no`;
`TextBoldStrokeWidth` `0.6`, `TextItalicSlant` `-12`, `TextUnderlineThickness` `1`, `TextUnderlineGap` `2`.

**Diorama** — `DioramaMode` `none`; tilt-shift: blur `2.5`, focus centre `50`, focus height `0`,
feather `20`, saturation `115`, contrast `2x50%`; shadowbox: depth `18`, opacity `0.55`.

**Output quality / size** — compression level `9`, filter `5`, strip metadata `$true`,
**use palette `$true`** (PNG8 — biggest size win but can band gradients), palette colours `64`,
depth `8`. Tiles carrying transparency automatically skip palette quantization.

**Cleanup / preview / cache** — `RecycleStaleOutputs` `$true`; `BuildPreview` `$true`,
`PreviewScale` `1`; `UseManifestCache` `$true`, `CacheVersion` (internal — bumped by the author
when render logic changes).

**Performance** — `MagickThreadLimit` empty (ImageMagick default), `ParallelPasses` `1`
(sequential; the `>1` parallel path is a hook that needs a refactor before it will run — leave it
at `1`).

---

## Output structure & file naming

```
Output_Images\
    <LayoutName>\
        <LayoutName>_<background>.png     <- stitched preview sheet (if BuildPreview)
        <Background>\
            <tiles>.png
```

Every `<Background>` folder contains the **complete** tile set for the grid (24 files for 4×6,
54 for 6×9, etc.), whether or not each tile was modified.

Filenames are prefixed with the layout name and encode the tile position, background, and any
decorating elements:

```
<LayoutName>_r<row>c<col>-<background>[-icon_<x>][-logo_<y>][-image_<z>][-border][-frame][-text_<t>].png
```

Example: `layout_4x6_r1c1-beach-icon_star.png`

Merged groups use the row/column span plus an index within the group:

```
<LayoutName>_r<R0>-<R1>c<C0>-<C1>-<background>[-tags]-<n>.png
```

Example: `layout_4x6_r1-2c1-2-beach-logo_netflix-1.png`

### The `&&` filename trick

Putting `&&` in the **layout filename** splits the folder name from the file prefix. The whole
name (each `&&` → `-`) becomes the folder; only the part **before** the first `&&` becomes the
per-file prefix:

```
Main.txt        -> folder Main\        files main_r1c1-...
Main&&app.txt   -> folder Main-app\    files main_r1c1-...   (the "app" part is dropped from the prefix)
Main&&game.txt  -> folder Main-game\   files main_r1c1-...
```

Handy for keeping several variants of one layout in separate folders while their files share a
single prefix.

---

## Caching, cleanup & preview

- **Manifest cache** (`Bin\db\`): each background pass records a signature of the layout text,
  every source asset, and the relevant config. On re-run, an unchanged pass whose output files all
  still exist is skipped — the big speed win. Any change under `Source_Images`, or to the
  config/layout, invalidates the cache, so it never serves stale output. Set
  `UseManifestCache = $false` to always regenerate.
- **Stale-file cleanup**: after writing a folder, any *other* file there (from a renamed element,
  removed asset, etc.) is sent to the **Recycle Bin** rather than permanently deleted, so it can be
  restored. Controlled by `RecycleStaleOutputs`.
- **Preview / contact sheet**: with `BuildPreview` on, the tiles are stitched back into one full
  image at `Output_Images\<LayoutName>\<LayoutName>_<background>.png` for quick inspection. It's
  rebuilt from whatever tiles are on disk, so it reflects the current output even when the tile
  pass itself came from cache. `PreviewScale` upscales it (nearest-neighbour, so edges stay crisp).

---

## Hidden launcher (with progress window)

`Launcher.exe` (built from `launcher.cs` via `build.bat`) runs `script.ps1` with the PowerShell
console **completely hidden** and shows a small window with a progress bar and live status.
Closing that window cancels the run.

Build it once by double-clicking `build.bat` (it uses the C# compiler that ships with the .NET
Framework — no Visual Studio needed), then run `Launcher.exe` from the project folder.

The launcher shows an **animated (indeterminate) bar** plus the latest log line as status. It can
also render a real percentage bar if the script prints `@PROGRESS:<done>/<total>` lines — but the
current `script.ps1` does **not** emit those, so you'll get the animated bar unless you add that
emission yourself.

---

## Troubleshooting

- **"ImageMagick not found"** — check that `Bin\magick.exe` exists.
- **Font warning at startup** — the configured `TextFont` isn't installed; the script falls back
  automatically. Run `magick identify -list font` to see valid names.
- **"`!button = *all*` was removed"** — name a specific button; the all-buttons wildcard no longer
  exists.
- **Unknown element warning** — you used a `!something` that isn't a recognised element (check
  spelling against the [Elements](#elements) table).
- **A background is skipped** — a `!background = name` filter didn't match any file (matching
  ignores the extension), or `Backgrounds\` is empty.
- **A tile is skipped** — the instruction referenced a tile outside the chosen grid
  (e.g. `r5c1` in a `2x3` grid).
- **Colours inside `[ ]` misbehave** — you used `rgb(r,g,b)`; use a name or hex instead.
- **Parallel passes error** — `$ParallelPasses` must stay at `1` until the render functions are
  made runspace-safe.
- **Launcher can't find the script** — `Launcher.exe` must sit in the same folder as `script.ps1`.
