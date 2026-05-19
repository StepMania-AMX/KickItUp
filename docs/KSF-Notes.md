# Kick It Up KSF Format Notes

## Scope

This document describes the KSF behavior implemented by the Kick It Up (KIU)
source tree one directory above this document.

In the broader AMX documentation, this behavior is referred to as the
`KIU Variant`.

This is intentionally KIU-specific:

- it does not try to describe later DirectMove extensions
- it does not explain StepMania conversion policy
- it focuses on what the KIU source actually parses and uses

This document does not describe:

- `KIU Cerbo+AMX Variant`
- `DirectMove Experimental Variant`
- `DirectMove Variant`
- `DirectMove AMX Variant`

Those are later or broader variant names used by the AMX-side documentation.

## File Structure

KIU reads KSF as a text file:

- header tags start with `#`
- step data begins after `#STEP:`
- step rows continue until EOF or the sentinel row `2222222222222`

The parser is in [data/Ksf.cpp](../data/Ksf.cpp:24).

## Tags KIU Parses

KIU parses these tags in [data/Ksf.cpp](../data/Ksf.cpp:37):

- `#TITLE:`
- `#TICKCOUNT:`
- `#BUNKI:`
- `#BUNKI2:`
- `#BPM:`
- `#BPM2:`
- `#BPM3:`
- `#DIFFCULTY:`
- `#MADI:`
- `#STARTTIME:`
- `#STARTTIME2:`
- `#STARTTIME3:`
- `#STEP:`

Observed units:

- `STARTTIME*` is multiplied by `10`, so the stored file unit is `10 ms`
- `BUNKI*` is parsed as an integer in `Ksf`, and the older backup code also
  multiplies it by `10`, which strongly indicates the same `10 ms` storage unit
- `TICKCOUNT` is rows per beat

## Parsed Fields

The main KSF object stores:

- `m_title`
- `m_bpm[3]`
- `m_madi`
- `m_tick`
- `m_dummy`
- `m_track`
- `m_start[3]`
- `m_bunki[2]`
- `m_step`

See [data/Ksf.h](../data/Ksf.h:25).

## Meaning Of `MADI`

KIU parses `#MADI:` and stores it in `m_madi`:

- [data/Ksf.cpp](../data/Ksf.cpp:69)
- [data/Ksf.h](../data/Ksf.h:27)

However, this field does not appear to be used anywhere else in the current KIU
source tree. A repo-wide search finds only parsing/storage references.

Interpretation:

- `MADI` is very likely Korean `마디`, meaning bar/measure
- in practice, KIU treats it as parsed-but-unused metadata

So the safest documented meaning is:

- `MADI` = beats per measure / bar grouping metadata
- not part of playback timing

## Step Row Semantics

KIU stores each step row as a 13-character string in `m_step`.

During gameplay, `StageNormal` only treats `'1'` as a hittable note:

- [StageNormal.cpp](../stage/StageNormal.cpp:188)
- [StageNormal.cpp](../stage/StageNormal.cpp:374)

Inside the KSF parser itself:

- `'1'` becomes a note
- `'4'` is treated as a long-note marker
- `'0'` and `'2'` fall through the non-note path

See [data/Ksf.cpp](../data/Ksf.cpp:85) and the
older, more explicit state handling in
[backup/Main.cpp](../backup/Main.cpp:658) /
[backup/Double.cpp](../backup/Double.cpp:2053).

Practical KIU note-row observations:

- rows are 13 characters wide in the parser
- the gameplay path shown in `StageNormal` only uses the first 5 columns for
  single play
- `2222222222222` is the end-of-chart marker

## Timing Actually Used By Current KIU Gameplay

The current `StageNormal` gameplay path initializes timing from:

- `STARTTIME[0..2]`
- `TICKCOUNT`
- `BPM[0]`

See:

- [StageNormal.cpp](../stage/StageNormal.cpp:212)
- [StageNormal.cpp](../stage/StageNormal.cpp:217)
- [StageNormal.cpp](../stage/StageNormal.cpp:220)

Then it computes one fixed step gap:

```text
stepGapTime = 60000 / (BPM0 * TICKCOUNT)
```

See [StageNormal.cpp](../stage/StageNormal.cpp:233).

The judging and indexing logic then uses that constant step gap for the whole
chart:

- [StageNormal.cpp](../stage/StageNormal.cpp:325)
- [StageNormal.cpp](../stage/StageNormal.cpp:333)

This means the current KIU gameplay path does not appear to apply runtime BPM
changes from `BPM2` / `BPM3` or boundary changes from `BUNKI` / `BUNKI2`.

## Legacy Timing Intent vs Current KIU Gameplay

The parsed fields strongly suggest the intended legacy timing model was:

- `BPM`, `BPM2`, `BPM3`
- `BUNKI`, `BUNKI2`
- `STARTTIME`, `STARTTIME2`, `STARTTIME3`
- `TICKCOUNT`

The older backup code reinforces that interpretation:

- it stores `bunki1`, `bunki2`
- it multiplies them by `10`
- it checks current time against the bunki thresholds

See:

- [backup/Main.cpp](../backup/Main.cpp:634)
- [backup/Main.cpp](../backup/Main.cpp:658)
- [backup/Main.cpp](../backup/Main.cpp:668)

So the evidence suggests:

- KIU's format design included legacy multi-segment timing
- the current visible `StageNormal` path is much simpler and effectively uses
  only the first BPM plus one tickcount

## Source Quirks And Bugs

The KIU parser has a few notable issues in the current source:

1. `#BPM2:` and `#BPM3:` are both written into `m_bpm[0]` instead of `m_bpm[1]`
   and `m_bpm[2]`.
   - [data/Ksf.cpp](../data/Ksf.cpp:57)

2. `#STARTTIME3:` is parsed with the key `#STARTTIME2:` again, so
   `STARTTIME3` is effectively not read by this parser.
   - [data/Ksf.cpp](../data/Ksf.cpp:81)

3. `#DIFFCULTY:` is misspelled in the parser and source files.
   - [data/Ksf.cpp](../data/Ksf.cpp:65)

4. `_addTick()` exists, but the call site is commented out.
   - [data/Ksf.cpp](../data/Ksf.cpp:88)
   - [data/Ksf.cpp](../data/Ksf.cpp:131)

These quirks matter if someone wants to preserve KIU behavior exactly rather
than reconstruct an idealized KSF reader.

## Confirmed vs Inferred

Confirmed by current KIU source:

- the tag names listed above are parsed
- `STARTTIME` is stored in `10 ms` units after multiplying by `10`
- `MADI` is parsed and stored
- `MADI` is not used elsewhere in the current source tree
- `StageNormal` uses a fixed `BPM0` + `TICKCOUNT` step gap

Strongly inferred from source history:

- `BUNKI` is a time boundary value, not a row number
- `BUNKI` uses the same `10 ms` storage unit family as `STARTTIME`
- KIU originally intended to support old multi-segment timing, even if the
  current gameplay path does not clearly do so

## Preservation Notes

For KIU preservation work, the safest archival stance is:

- preserve all parsed tags, including `MADI`
- document `MADI` as measure metadata
- do not assume current KIU gameplay accurately represents every intended KSF
  timing feature
- treat the current parser bugs as part of the historical implementation record
