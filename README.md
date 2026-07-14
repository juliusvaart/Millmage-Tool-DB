# Millmage-Tool-DB

Repo for Millmage Tool DB's.


# Spetool

`Spetool Tool Library.tools` is a MillMage tool library generated from the
Spetool feeds-and-speeds CSV exports in `spetool_database_csv/`, one
category per material file (478 tools). See `CLAUDE.md` for the conversion
rules and MillMage file-format gotchas.

## Verification status

Every tool has been checked field-by-field against its source CSV row
(9,082 checks, zero errors):

- 1:1 mapping between CSV rows and tools; no orphans, duplicate UUIDs or
  duplicate tool numbers per category.
- All stored values are in MillMage's canonical units (mm, mm/sec), with
  inch-based CSV rows converted (×25.4). The tool's unit flag only affects
  how MillMage *displays* values.
- Feeds and speeds match the CSV: feed/plunge = CSV value ÷ 60 (×25.4
  first for inch rows), ramp = plunge, rpm/pass depth/stepover/flute
  count as specified.
- Geometry re-derived from every CSV row and compared, including the
  Tapered Ball Mill length formula MillMage enforces internally.

## Known deviations from the source data

Intentional differences, dictated by what MillMage's format can represent:

- **Tapered ball-nose engravers (W01001/W01003/W01005/W01008/W01009)**:
  the CSV records the tip-ball diameter and no shank size. MillMage's
  Tapered Ball Mill wants the shank diameter and extends the taper cone
  all the way up to it, so the shank sizes were taken from spetools.com
  product pages (W01001/W01003 = 1/8", W01005/W01008/W01009 = 1/4") and
  the stored flute length is MillMage's cone-extended value (e.g. 30.7mm
  for W01003 instead of the real 15mm flute). The cutting profile is
  identical within real cutting depths.
- **Round-bottom V-groove bits (W06003/W06004)**: MillMage has no
  "V-bit with rounded tip" geometry, so these are stored as Tapered Ball
  Mills — same cutter profile.
- **W06016 (round-nose bit)**: its CSV flute length (8.33mm) is shorter
  than its ball radius (8.7mm), which MillMage's ball-mill model cannot
  represent; the flute length is clamped up to the radius.
- **Single spindle speed per tool**: 7 CSV rows (W06016 ×4, W08505 ×3)
  specify a different rpm for 3D finishing (18000) than for 2D
  (15000/16000). MillMage stores one `SpindleSpeed`, so the 2D value is
  used.
- The CSV `notes` and `coating` columns are empty in every row, so
  nothing was lost by omitting them.
