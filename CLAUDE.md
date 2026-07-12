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

1. **FeedRate / PlungeRate / RampRate are stored in units PER SECOND,
   not per minute**, even though the UI displays and labels them
   "mm/m" (per minute). Confirmed by creating a tool manually in
   MillMage with UI values Feed Rate 2000mm/m, Plunge Rate 300mm/m, and
   reading back the saved file: `"FeedRate": 33.333` (=2000/60),
   `"PlungeRate": 5` (=300/60). **Always divide CSV feedrate/plungerate
   (mm/min or in/min) by 60 before writing them into the JSON.** Set
   `RampRate` to the same (already-divided) value as `PlungeRate`.
   Diameter, Length, PassDepth, StepOver, RampAngle and SpindleSpeed
   (RPM) need no such conversion, they're stored as displayed.

2. **`TipDiameter` is a required field on every tool**, not just
   tapered geometries. Older reference files may omit it; the current
   app version writes `"TipDiameter": 0` even on plain End Mills.
   Include it on every entry.

3. **Geometry classification for V-bit/ball-ish rows** (CSV
   `type=vee` or `type=engraver`): compute `ratio = cornerradius /
   diameter`.
   - `ratio ~ 0` -> `Type: "V-Bit"`, `Radius: 0`, `IncludedAngle: <angle>`.
   - `0 < ratio < 0.5` -> `Type: "V-Bit"`, `Radius: cornerradius`,
     `IncludedAngle: <angle>` (a rounded-tip V-bit, geometrically
     valid).
   - `ratio >= 0.5` -> geometrically there's no V left (radius equals
     or exceeds half the diameter). Use `Type: "Ball Mill"`,
     `Radius: diameter/2` (clamp), `IncludedAngle: 0`. Do **not** force
     `Type: "V-Bit"` here: MillMage will flag Flute Length and Tip
     Radius as invalid (red X) because a sharp/tapered V cannot have a
     tip radius that wide.
   - We deliberately avoid `Type: "Tapered Ball Mill"` for these rows.
     It's a valid MillMage geometry (confirmed via UI) and requires
     `Radius` strictly `< diameter/2` plus a recalculated `Length`
     (cone height + ball height) that MillMage derives internally; we
     couldn't reliably reverse-engineer that formula from one manually
     edited sample, and reclassifying as plain "Ball Mill" sidesteps
     the problem entirely with no loss of accuracy for these tools.
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
- No V-Bit with `Radius >= Diameter/2`.
- No Ball Mill with `Radius != Diameter/2`.
- Spot check a few tools' `FeedRate * 60` / `PlungeRate * 60` against
  the original CSV mm/min values.
