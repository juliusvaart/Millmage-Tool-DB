# Spetool CSV -> MillMage .tools conversion

Notes on converting Spetool feeds-and-speeds CSV exports into a MillMage
tool library (`.tools`, JSON format). Reference script: `build_library.py`
in this folder.

## Source CSV format

Columns: `number,vendor,model,URL,name,type,diameter,cornerradius,
flutelength,shaftdiameter,angle,numflutes,stickout,coating,metric,notes,
machine,material,plungerate,feedrate,rpm,depth,cutpower,finishallowance,
3dstepover,3dfeedrate,3drpm`

- One CSV per material (Hardwood, Softwood, Aluminum, Steel, etc.), plus
  one `Any+Any` file for bits with no material-specific tuning (v-bits,
  ball nose, engravers).
- `metric` column: `1` = values in mm, `0` = values in inches. Respect
  this per row, don't assume a global unit.
- Some files (HARDWOOD, MDF Laminate, SOFTWOOD) are encoded `gb18030`,
  not UTF-8 (curly quotes `“”` show up as garbage bytes `0xa1 0xb0` if
  read as UTF-8). Try `utf-8-sig` first, fall back to `gb18030`.
- Tool numbers repeat across material files with different feed/speed
  for the same physical tool. Don't dedupe: keep one category per
  material so the CNC operator picks the right speeds for their job.
- `cornerradius` is not reliable as a literal "tip radius" spec. For
  `type=vee`/`engraver` rows it's often exactly `diameter/2` (a full
  ball, not a V taper) or even larger (bad data). See geometry mapping
  below.

## MillMage .tools schema

Flat JSON: `{ "<Category Name>": { "{uuid}": { ...tool fields... }, ... }, ... }`

Tool fields (confirmed via MillMage's own save output):
`Category, Diameter, FeedRate, FluteCount, IncludedAngle, Index, Length,
MetricTool, Name, Notes, PassDepth, PlungeRate, Radius, RampAngle,
RampRate, SpindleSpeed, StepOver, TipDiameter, TipLength, ToolSpecURL,
Type, Vendor`

Geometry dropdown options (confirmed from the app UI): End Mill, Ball
Mill, V-Bit, Tapered Ball Mill, Tapered End Mill, Drill, Round-over,
Scribe.

## Critical gotchas (found by testing against the real app)

1. **All stored values are canonically METRIC (mm / mm-per-second),
   regardless of the tool's units.** `MetricTool` is a DISPLAY-ONLY
   flag: it picks the units the UI shows (and converts stored mm to
   inches for display when false); it does not change how values are
   stored. Confirmed by loading a file with inch values +
   `MetricTool: false`: the app displayed every value divided by 25.4
   (e.g. stored `0.685` intended as 0.685" showed as "0.02697in" =
   0.685mm converted). Also: toggling Tool Units in the UI does NOT
   convert the numbers, so a user toggling units on an existing tool
   silently changes its real size/feeds by 25.4x.
   - Geometry from an inch-unit CSV row (`metric=0`): multiply
     Diameter, Length, PassDepth, StepOver, Radius, TipDiameter,
     TipLength by 25.4.
   - **FeedRate / PlungeRate / RampRate are stored in mm PER SECOND**,
     even though the UI labels them "mm/m" (per minute). Confirmed by
     creating a tool with UI Feed Rate 2000mm/m and reading back
     `"FeedRate": 33.333`. So: CSV mm/min -> /60; CSV in/min ->
     *25.4/60. Set `RampRate` equal to `PlungeRate`.
   - RampAngle, IncludedAngle and SpindleSpeed (RPM) are unit-free,
     stored as displayed.

2. **`TipDiameter` is a required field on every tool**, not just
   tapered geometries. Older reference files may omit it; the current
   app version writes `"TipDiameter": 0` even on plain End Mills.
   Include it on every entry.

3. **Geometry classification for V-bit/ball-ish rows** (CSV
   `type=vee` or `type=engraver`): compute `ratio = cornerradius /
   diameter`.
   - `ratio ~ 0` -> `Type: "V-Bit"`, `Radius: 0`, `IncludedAngle: <angle>`.
     **A V-Bit must have `Radius: 0`.** MillMage has no rounded-tip
     V-bit: a stored V-Bit with `Radius > 0` is shown in the UI as a
     Tapered Ball Mill and red-X'd on Flute Length (seen on W06003/
     W06004, ratio 0.333). So:
   - `0 < ratio < 0.5 - epsilon` -> `Type: "Tapered Ball Mill"` (see
     the convention bullet below), `Radius: cornerradius`,
     `IncludedAngle: <angle>`, `Length` from the formula.
   - `ratio >= 0.5 - epsilon` (use e.g. `epsilon = 0.001`, NOT a strict
     `>= 0.5` check: the CSVs' mm->inch values are rounded, so a true
     half ratio shows up as e.g. `0.0295276 / 0.05906 = 0.499959` on
     the W0100x ball-tip engravers, which MillMage then rejects as
     invalid V-Bits) -> geometrically there's no V left; the tip is a
     full ball. If the tool is genuinely a tapered engraver (the
     SpeTool W0100x family, per vendor specs) use
     `Type: "Tapered Ball Mill"` with the SHANK diameter (next
     bullet). Otherwise (e.g. W06016, a round-nose bit) use
     `Type: "Ball Mill"`, `Radius: diameter/2` (clamp),
     `IncludedAngle: 0`, and **`Length` must be >= `Radius`** — the
     ball must fit inside the flute. W06016's CSV flutelength
     (0.32813") is shorter than its radius (0.3425"), which MillMage
     red-X's; clamp `Length` up to `Radius` in that case. Do **not**
     force `Type: "V-Bit"` here either way.
   - **Tapered Ball Mill convention** (reverse-engineered against the
     app UI, W01003 sample; derived Flute Length matched to <0.001mm):
     `Diameter` = the SHANK diameter (NOT the CSV `diameter`, which is
     the tip-ball diameter), `Radius` = tip ball radius (= CSV
     `cornerradius`), `IncludedAngle` = CSV `angle`, and `Length` is
     the cone-extended flute length MillMage derives internally:
     `Length = r*(1 - sin(a)) + (Diameter/2 - r*cos(a)) / tan(a)`
     with `r = Radius`, `a = IncludedAngle/2` (ball cap height + cone
     height, cone tangent to the tip ball). MillMage greys this field
     out and recomputes it, so the stored value must match the formula.
     `Radius` must be strictly `< Diameter/2`. For rounded-tip V rows
     (W06003/W06004) `Diameter` is just the CSV `diameter` (the V
     opens to the full cutting diameter). For the W0100x tapered
     engravers the CSV `diameter` is the tip-ball diameter, and
     `shaftdiameter` is 0, so shank sizes came from spetools.com
     product pages: W01001/W01003 = 1/8" (3.175mm),
     W01005/W01008/W01009 = 1/4" (6.35mm). Note the CSV `flutelength`
     (actual fluted length, e.g. 15mm) is discarded — MillMage's model
     extends the cone all the way to `Diameter` (e.g. 30.72mm for
     W01003); the cutter profile is identical within real cutting
     depths. Keep `StepOver` computed from the CSV tip diameter, not
     the shank diameter.
   - `type=end` -> always `Type: "End Mill"` (cornerradius is always
     0 in the source data). `type=ball` -> always `Type: "Ball Mill"`
     (cornerradius is always exactly `diameter/2`).

4. **`StepOver`** = `Diameter * (3dstepover_percent / 100)`. Confirmed
   against MillMage's own example data (6mm tool, 40% -> StepOver 2.4).

5. **`RampAngle`**: source CSVs have no equivalent column; MillMage's
   own example files consistently use `22.5` as a default, so we hard-code
   that.

6. Tool `Name` fields in the CSV use doubled quotes (`0.25""`) for the
   inch mark; normalize to a single `"`.

## Category naming used

One category per source CSV, prefixed `Spetool - `, e.g. `Spetool -
Hardwood`, `Spetool - Aluminum Copper Brass`, `Spetool - Any Material`
(from the `Any+Any` file, generic non-material-specific bits).

## Validation checklist before delivering

- JSON parses, keys match the confirmed schema on every tool.
- Every stored value is in mm / mm-per-second (canonical metric),
  including on `MetricTool: false` tools.
- No V-Bit with `Radius != 0`.
- No Ball Mill with `Radius != Diameter/2` or `Length < Radius`.
- Every Tapered Ball Mill has `Radius < Diameter/2` and `Length`
  exactly matching the derived-length formula.
- Spot check a few tools' `FeedRate * 60` / `PlungeRate * 60` against
  the original CSV values (x25.4 first for `metric=0` rows).
