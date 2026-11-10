---
layout: default
title: The SWF file format
permalink: /format/
description: A working description of the SWF container, written as the reference for reading what the Flash MX writer emits.
---

A working description of the SWF ("Shockwave Flash") binary container, written for
this project: it is the reference for reading the tag stream that **Flash MX**
(`Flash.exe`, SWF 6 era) produces, and for recognising the emitter when reading
decompiled code.

Structure and field layouts follow the published Macromedia/Adobe *SWF File
Format Specification* (versions 6–10). Everything marked **inference** is my
reasoning about this specific binary, not spec text.

---

## 1. What a SWF is

A SWF is a **movie**: a dictionary of reusable *characters* (shapes, bitmaps,
sounds, fonts, text, sprites, buttons) plus a *timeline* of *frames*. Each frame
is a list of *control tags* that mutate a per-depth *display list*. Playback
advances one frame per `FrameRate` tick, executing `ShowFrame` to present the
frame just built and `DoAction` tags to run ActionScript.

Three consequences that matter when reading a writer:

* It is **streaming, not random-access**. Tags are emitted in playback order.
* It is **length-prefixed but not index-carrying**. There is no offset table, no
  cross-reference block, no trailer.
* Two header fields (total length, frame count) are only known once the last
  frame has been emitted, so a writer must either buffer everything or
  **seek back and patch byte 4 and the frame count**.

## 2. Conventions

| Item | Rule |
|---|---|
| Integers | **Little-endian** (`UI16`, `UI32`, `SI16`) |
| Bit fields | Packed **most-significant bit first**, and only inside bit-packed structures |
| Alignment | Bit-packed structures start byte-aligned; most end byte-aligned (`RECT`, `MATRIX`, style records) — `CXFORM`/shape record runs do not, they continue in the bit stream |
| Coordinates | **twips** = 1/20 pixel; `20 twips = 1 px` |
| Colour | `RGB` = 3 bytes R,G,B; `RGBA` = 4 bytes, alpha 0 = transparent |
| Strings | NUL-terminated bytes; SWF 6+ treats them as **UTF-8** (older files were CP1252) |
| `FIXED` | `SI32`, 16.16 fixed point |
| `FIXED8` | `SI16`, 8.8 fixed point |
| Version | 1 byte; `SWF 6` == Flash Player 6 (Flash MX authoring) |

Bit-packed scalars are **signed two's complement** unless the spec says
otherwise (`UB[n]` vs `SB[n]`).

## 3. File layout

```
offset  size  field
------  ----  -----------------------------------------------------------
0x00    3     Signature: 'FWS' | 'CWS' | 'ZWS'
0x03    1     Version
0x04    4     FileLength  (UI32, length of the *uncompressed* file, header included)
0x08    var   FrameSize   (RECT)  -- stage bounds in twips
...     2     FrameRate   (UI16, 8.8 fixed: low byte = fraction)
...     2     FrameCount  (UI16)
...     var   tag stream  (repeated: tag header + body)
```

`FrameSize` is a bit-packed RECT, so the first tag does **not** start at a fixed
offset. Its size is `ceil((5 + 4*Nbits) / 8)` bytes, where `Nbits` is the first
5 bits of byte 8.

The tag stream always ends with an `End` tag (code 0, length 0) followed by
nothing — no padding is required after it, though many writers leave none.

### 3.1 Compression

* **`FWS`** — no compression.
* **`CWS`** — introduced in SWF 6. Bytes 0–7 stay plain; **everything from byte
  8 onward is a single zlib stream** (`78 9C`/`78 DA`/`78 01...`). `FileLength`
  still describes the decompressed size. Tag data inside the deflate stream is
  ordinary uncompressed SWF — decompress once, then parse normally.
* **`ZWS`** — SWF 13+ (LZMA). Byte 8 is a `UI32` compressed length, bytes
  12–16 the 5-byte LZMA properties, then the LZMA stream.

`CWS` compresses the **whole body at once**, so a writer cannot stream-compress
tag by tag and still patch the body; Flash MX's exporter therefore buffers (see
§12).

## 4. Tag framing

Every tag begins on a byte boundary:

```
UI16  CodeAndLength = (code << 6) | length      ; little-endian on disk
      if length == 0x3F:  UI32 length          ; long form
      body: length bytes
```

So the first byte written is `(length & 0x3F) << 6 | (code & 0x3)`... in
practice: compute `w = code*64 + len` (for `len < 63`), write `w` as a
little-endian `UI16`. Codes go up to 1023; a body of 63 bytes or more requires
the long form. Bodies are never split across tags.

## 5. Tag codes

Core set (SWF ≤ 7). Codes marked *dep* are obsolete, codes in the gap list were
never shipped in public SWF 7 or earlier and should not appear in a Flash MX file.

| Code | Tag | Notes |
|---:|---|---|
| 0 | End | terminates the stream |
| 1 | ShowFrame | present the frame just built |
| 2 | DefineShape | v1: 8-bit style counts, RGB only |
| 3 | FreeCharacter | *dep* |
| 4 | PlaceObject | v1: no name/ratio/clip-depth |
| 5 | RemoveObject | v1 |
| 6 | DefineBits | JPEG, shared `JPEGTables` |
| 7 | DefineButton | v1 |
| 8 | JPEGTables | shared JPEG quantisation/huffman tables |
| 9 | SetBackgroundColor | `RGB` |
| 10 | DefineFont | glyphs only, no codes |
| 11 | DefineText | v1 |
| 12 | DoAction | action stream |
| 13 | DefineFontInfo | code→glyph mapping, name, style |
| 14 | DefineSound | event sound |
| 15 | StartSound | play a `DefineSound` |
| 16 | StopSound | *dep* |
| 17 | DefineButtonSound | |
| 18 | SoundStreamHead | declares the streaming sound |
| 19 | SoundStreamBlock | one block of streaming sound |
| 20 | DefineBitsLossless | zlib-compressed paletted/15/32-bit bitmap |
| 21 | DefineBitsJPEG2 | self-contained JPEG |
| 22 | DefineShape2 | 16-bit style counts |
| 23 | DefineButtonCxform | |
| 24 | Protect | optional MD5 password + licence text |
| 26 | PlaceObject2 | the workhorse (§9) |
| 28 | RemoveObject2 | depth only |
| 32 | DefineShape3 | styles become `RGBA` |
| 33 | DefineText2 | `RGBA` text |
| 34 | DefineButton2 | |
| 35 | DefineBitsJPEG3 | JPEG + zlib alpha channel |
| 36 | DefineBitsLossless2 | with alpha |
| 37 | DefineEditText | editable text field |
| 39 | DefineSprite | nested timeline (§10) |
| 43 | FrameLabel | name + (SWF 6+) named-anchor flag |
| 45 | SoundStreamHead2 | as 18, allows 16-bit |
| 46 | DefineMorphShape | |
| 48 | DefineFont2 | codes + layout (§11) |
| 56 | ExportAssets | `(tagId, name)` pairs — the linkage table |
| 57 | ImportAssets | |
| 58 | EnableDebugger | MD5 password |
| 59 | DoInitAction | `SpriteId` + actions run once on load |
| 60 | DefineVideoStream | |
| 61 | VideoFrame | |
| 62 | DefineFontInfo2 | |
| 64 | EnableDebugger2 | |
| 65 | ScriptLimits | recursion/time limits |
| 66 | SetTabIndex | |

Later versions add: 69 `FileAttributes`, 70 `PlaceObject3`, 71 `ImportAssets2`,
73 `DefineFontAlignZones`, 74 `CSMTextSettings`, 75 `DefineFont3`,
76 `SymbolClass`, 77 `Metadata`, 78 `DefineScalingGrid`, 82 `DoABC`,
83 `DefineShape4`, 84 `DefineMorphShape2`, 86 `DefineSceneAndFrameLabelData`,
87 `DefineBinaryData`, 88 `DefineFontName`, 89 `StartSound2`,
90 `DefineBitsJPEG4`, 91 `DefineFont4`, 93 `EnableTelemetry`.

Gaps (25, 27, 29–31, 38, 40–42, 44, 47, 72) are unused or internal-only.

### 5.1 What a plain authored movie actually contains

For a static Flash MX document, the overwhelmingly common stream is:

```
SetBackgroundColor, DefineShape2/3 xN, DefineBitsLossless2, DefineFont2,
DefineText2, DefineSprite, ExportAssets, FrameLabel,
DoAction, PlaceObject2 xN, ShowFrame, [more frames], End
```

`DefineShape` v2 (22) and v3 (32) dominate; v1 tags appear only from very old
documents. A minimal frame is `PlaceObject2`… then `ShowFrame`.

## 6. Primitive data types

### RECT
```
UB[5]  Nbits
SB[Nbits] Xmin, SB[Nbits] Xmax, SB[Nbits] Ymin, SB[Nbits] Ymax
-> byte-align
```

### MATRIX (2×3 affine)
```
UB[1] HasScale
  if HasScale:  UB[5] NScaleBits; FIXED8 ScaleX; FIXED8 ScaleY
UB[1] HasRotate
  if HasRotate: UB[5] NRotateBits; FIXED8 RotateSkew0; FIXED8 RotateSkew1
UB[5] NTranslateBits; SB[NTranslateBits] TranslateX; SB[NTranslateBits] TranslateY
-> byte-align
```
Scale/rotate fields are 8.8 fixed **as raw bit fields**, not byte-aligned. In an
identity matrix `HasScale = HasRotate = NTranslateBits = 0`, which is 7 zero bits
— useful as a signature when reading hex.

### CXFORM / CXFORMWITHALPHA
```
UB[1] HasAddTerms, UB[1] HasMultTerms, UB[4] Nbits
if HasMultTerms: SB[Nbits] Rmult, Gmult, Bmult [, Amult]
if HasAddTerms : SB[Nbits] Radd,  Gadd,  Badd  [, Aadd]
```
Multiply terms are 8.8 fixed; add terms are signed twips-ish integers scaled by
256. `CXFORMWITHALPHA` (used by `PlaceObject2`) has the alpha terms.

### Gradients
```
GRADIENT:  UB[2] SpreadMode (SWF 8+), UB[2] InterpolationMode (SWF 8+),
           UB[4] NumGradients, then per record: UI8 Ratio + RGB|RGBA
GRADIENTMATRIX: MATRIX then GRADIENT
```
`SpreadMode`: 0 pad, 1 reflect, 2 repeat.

## 7. Shapes — the complex part

`DefineShape3` (tag 32) body:

```
UI16   ShapeId
RECT   ShapeBounds
SHAPEWITHSTYLE
```

```
SHAPEWITHSTYLE:
  FILLSTYLEARRAY:
    UI8 FillStyleCount; if 0xFF: UI16 FillStyleCountExtended
    FILLSTYLE[count]:
      UI8 FillStyleType
        0x00 solid              -> RGB (shape v1/2) or RGBA (v3)
        0x10 linear gradient    -> GRADIENTMATRIX + focal point only if 0x13
        0x12 radial gradient    -> GRADIENTMATRIX
        0x13 focal radial (SWF8)-> GRADIENTMATRIX + FIXED8 FocalPoint
        0x20 repeating bitmap   -> UI16 BitmapId + MATRIX
        0x40 clipped bitmap     -> UI16 BitmapId + MATRIX
        0x41 unsmoothed clipped bitmap
        0x42 unsmoothed repeating bitmap
  LINESTYLEARRAY:
    UI8 LineStyleCount; if 0xFF: UI16 count
    LINESTYLE[count]: UI16 Width (twips) + RGB|RGBA
  then a run of SHAPERECORDs until the end record
```

Each `SHAPERECORD` starts with a 1-bit type flag read **from a continuous bit
stream** (no re-alignment between records):

* **Style change record** — first bit `0`, then 5 flags:
  `StateNewStyles(0x80)` `StateLineStyle(0x40)` `StateFillStyle1(0x20)`
  `StateFillStyle0(0x10)` `StateMoveTo(0x08)`.
  If any of the last four is set: `UB[5] NumBits`, then
  `SB[NumBits] MoveX, SB[NumBits] MoveY` (if MoveTo),
  `UB[NumBits] FillStyle0`, `UB[NumBits] FillStyle1`, `UB[NumBits] LineStyle`.
  A style index of 0 means "unchanged"/"none". `StateNewStyles` re-emits both
  style arrays (used when a sprite's shape needs new paints mid-path).
* **End record** — six zero bits, then byte-align.
* **Edge record** — first bit `1`, then `UB[1] StraightFlag`:
  * straight: `UB[4] NumBits` (stored as `n-2`, i.e. 2 bits used), actually
    `UB[4]` where the real width is `NumBits + 2`; then `UB[1] GeneralLineFlag`;
    if general `SB[n] DeltaX, SB[n] DeltaY`, else `UB[1] VertLineFlag` and a
    single `SB[n] DeltaX` or `DeltaY`.
  * curved (quadratic): `UB[4] NumBits` (width `NumBits + 2`), then
    `SB[n] ControlDeltaX, ControlDeltaY, AnchorDeltaX, AnchorDeltaY`.

Edge deltas are **relative to the pen position**, in twips. Style indices are
1-based into the arrays emitted for the shape (0 = none).

Practical note: bit-level writing means the emitter needs a bit accumulator with
`putbits(value, nbits)` plus a flush-to-byte-boundary call. In decompiled code
this shows up as shifts/ORs into a DWORD and a "flush when >= 8 bits" store —
the reason a naive scan for the byte constant `'F'`/`'W'`/`'S'` finds nothing.

## 8. Bitmaps

| Tag | Body |
|---|---|
| 21 `DefineBitsJPEG2` | `UI16 CharacterId`, JPEG bytes (SWF 8+ also PNG/GIF) |
| 35 `DefineBitsJPEG3` | `UI16 CharacterId`, `UI32 AlphaDataOffset`, JPEG, zlib alpha plane (one byte per pixel) |
| 20 `DefineBitsLossless` | `UI16`, `UI8 BitmapFormat` (3=8-bit paletted, 4=15-bit, 5=32-bit), `UI16 Width`, `UI16 Height`, zlib data (paletted: `UI8 ColorTableSize` + RGB table + row-aligned indices) |
| 36 `DefineBitsLossless2` | same, colours are `RGBA` |
| 6 `DefineBits` + 8 `JPEGTables` | table-less JPEG data shared across bitmaps |

`DefineBitsLossless*` payloads are **zlib streams** — this is why libpng/zlib
strings live in an authoring tool even when SWF body compression is off.

## 9. Frames and the display list

`PlaceObject2` (26) is the only tag that needs care:

```
UI8   Flags
UI16  Depth
[UI16 CharacterId]        if Flags & 0x02 (HasCharacter)
[MATRIX Matrix]           if Flags & 0x04 (HasMatrix)
[CXFORMWITHALPHA Cxform]  if Flags & 0x08 (HasColorTransform)
[UI16 Ratio]              if Flags & 0x10 (HasRatio)
[STRING Name]             if Flags & 0x20 (HasName)
[UI16 ClipDepth]          if Flags & 0x40 (HasClipDepth)
[CLIPACTIONS]             if Flags & 0x80 (HasClipAction)
```

`Flags & 0x01` is `Move`: with a `CharacterId`, the existing object at that depth
is replaced; without one, the existing object's fields are updated in place.
Depth 1 is front-most; higher depths draw behind. `ClipDepth` makes the placed
object a mask for depths `Depth+1 .. ClipDepth`.

`RemoveObject2` (28) is just `UI16 Depth`. `RemoveObject` (5) adds a
`UI16 CharacterId` that players ignore.

`CLIPACTIONS`: `UI16 Reserved`, `UI16 AllEventFlags` (SWF 6+), then per event
flag a `CLIPACTIONRECORD` (`UI32 EventFlags`, `UI32 RecordSize`, action stream,
`UI8 KeyCode`); terminated by an all-zero `UI32`.

`ShowFrame` (1) closes the frame. There is no explicit "frame header" tag — the
frame boundary is positional.

## 10. Sprites and nesting

`DefineSprite` (39):

```
UI16 SpriteId
UI16 FrameCount
<control tags>          ; same tag stream, but no file header/FrameSize
```

A sprite is a self-contained timeline whose display list is independent; it must
contain `End`? — **no**: it is terminated by its declared tag length, and
conventionally its last control tag is `ShowFrame` per frame. Sprites are placed
with `PlaceObject2` like any other character, and may contain `DoAction`,
`SoundStreamHead`, nested sprites, etc. This is how a symbol becomes a movie
clip.

## 11. Fonts and text

`DefineFont2` (48):
```
UI16 FontId, UI8 Flags, UI8 LanguageCode,
UI8 NameLength,  BYTES Name,
UI16 NumGlyphs,  UI16 OffsetTable[NumGlyphs] (byte offsets from body start),
UI16 CodeTableOffset,
GLYPHSHAPE[NumGlyphs]           ; SHAPE records, as in §7 but no fills/styles array
UI16(UI8 if WideCodes) CodeTable[NumGlyphs]
[if HasLayout: SI16 Ascent, Descent, Leading; SB[16] widths x NumGlyphs;
               SI16 Left, Right, Top, Bottom indents (EM square 1024)]
SI16 ItalicAngle       (if Italic flag)
SI16 BoldPoint         (if Bold flag)
```
Flags: `0x80 HasLayout`, `0x40 ShiftJIS`, `0x20 SmallText`, `0x10 ANSI`,
`0x08 WideOffsets`, `0x04 WideCodes`, `0x02 Italic`, `0x01 Bold`.

`DefineText2` (33): `UI16 CharacterId`, `RECT TextBounds`, `MATRIX TextMatrix`,
`UI8 GlyphBits`, `UI8 AdvanceBits`, then `TEXTRECORD`s
(`UI8 Flags` + optional font/colour/offset triplets + glyph entries of
`GlyphBits` index and `AdvanceBits` advance), terminated by `UI8 0`.

`DefineEditText` (37): `UI16`, `RECT Bounds`, `UI16 Flags`, optional
font/colour/maxlength/layout/variable-name fields in a fixed order, then the
initial text string. `DefineFontInfo`/`DefineFontInfo2` (13/62) attach a name,
character codes and style to a v1 `DefineFont`.

## 12. Writing an SWF: the back-patching problem

Two header values are unknowable up front:

1. `FileLength` at byte 4.
2. `FrameCount` right after `FrameRate`.

Tag bodies (`DefineSprite` lengths, `DefineShape` style counts) also depend on
how many bytes the bit writer actually flushed. Three strategies exist:

* **buffer everything in memory, then emit** — simplest, what most third-party
  libraries do;
* **reserve and patch** — write the 8-byte header with a placeholder, stream
  tags, then `Seek(4)` and rewrite `FileLength` and `FrameCount`;
* **two-pass** — serialise once to count, then again to write.

**Inference from this binary:** Flash MX uses the third shape of the *output
plumbing* — an MFC `CMirrorFile` (write to a temp mirror, rename over the target
on close) whose vtable exposes `Seek` (slot 10 → `SetFilePointer`) and `Write`
(slot 14 → `WriteFile`). A mirror file is exactly what a writer needs when it
must go back and patch the header of a big file without holding it all in RAM.
Consistent with that, `'FWS'`/`'CWS'` never appear as immediate operands in
`Flash.exe`'s code — the signature bytes are almost certainly composed in a
buffer at run time, not stored as a literal triple.

## 13. Worked example: smallest useful movie

550×400 stage, 24 fps, one frame, white background. No shapes, no actions.
Nothing here is compressed, so every byte is accounted for.

```
46 57 53 06   "FWS", version 6
1E 00 00 00   FileLength = 30 (0x1E), uncompressed
78 00 05 5F 00 00 0F A0 00     FrameSize RECT
00 18         FrameRate = 0x1800 = 24.0 (low byte = fraction)
01 00         FrameCount = 1
43 02         tag (9 << 6) | 3 = 0x0243 -> SetBackgroundColor, 3 bytes
FF FF FF      white
40 00         tag (1 << 6) | 0 = 0x0040 -> ShowFrame, 0 bytes
00 00         tag 0 -> End
```

Decoding the RECT by hand: byte 8 = `0x78` = `0111 1000`, so `Nbits = 01111`
(15). The next 60 bits are four 15-bit signed values
`0, 11000, 0, 8000` → `(0,0)`–`(11000,8000)` twips = `550 × 400 px`. Five spare
bits plus the 4+15+15+15+15 alignment pad to 9 bytes.

Bit-exact layout of the 9 RECT bytes, MSB first:

```
01111 000000000000000 010101011111000 000000000000000 001111101000000 0000000
^Nbits^     Xmin=0        Xmax=11000        Ymin=0         Ymax=8000      pad
```

That trace was verified by round-tripping the file through a parser: version 6,
length 30, Nbits 15, rect `[0, 11000, 0, 8000]`, rate 24.0, 1 frame, then tags
9/3, 1/0, 0/0 at offsets `0x15`, `0x1a`, `0x1c`.

To make it a `CWS` file: keep bytes 0–7 (`'C','W','S'`, version, uncompressed
length) and replace everything from byte 8 with `zlib.compress(body)`; the file
on disk shrinks, `FileLength` does not change.

### 13.1 Cross-check against real movies shipped in Flash.exe

`Flash.exe` embeds seven complete movies in `.rsrc` (built-in component/library
assets). Parsing each with §14's algorithm recovers plausible, self-consistent
headers — independent confirmation of the RECT/framing rules above:

| .rsrc file offset | sig | ver | FileLength | stage | rate | frames |
|---|---|---|---|---|---|---|
| `0xa432c0` | FWS | 4 | 8471 | 210×115 | 12 | 32 |
| `0xa4d430` | FWS | 3 | 20883 | 410×210 | 12 | 16 |
| `0xa525c8` | FWS | 5 | 12197 | 469×255 | 120 | 30 |
| `0x9f88dc` | CWS | 6 | 57684 | — | — | — |
| `0x9fe510` | CWS | 6 | 998 | — | — | — |
| `0xa39830` | CWS | 6 | 73480 | — | — | — |
| `0xa453d8` | CWS | 6 | 58830 | — | — | — |

(The other ~10 raw signature hits are accidental byte sequences in unrelated
blobs, with impossible version/length fields — a useful reminder that signature
scanning alone is not identification.)

All four `CWS` files inflate to exactly `FileLength − 8` bytes
(57676, 990, 73472, 58822), which confirms the rule in §3.1: **`FileLength`
counts the whole decompressed file, header included**, not the on-disk size.

Their tag streams are exactly the shape predicted in §5.1 — the v3/v4/v5 movies
open with `SetBackgroundColor`, `Protect`, `JPEGTables`+`DefineBits` or
`DefineShape`, then `PlaceObject2`/`DefineSprite`/`ExportAssets`/`DefineFont2` —
while the v6 `CWS` assets are component templates using the same vocabulary.

## 14. Reading it back

Parser skeleton:

1. Read 3-byte signature. `CWS` → inflate from byte 8; `ZWS` → LZMA. `FWS` → read
   straight through.
2. `version = b[3]`, `fileLength = u32(b[4:8])` (sanity-check against the
   decompressed size).
3. Bit reader at byte 8: `RECT` → stage bounds. Re-align.
4. `frameRate = u16 / 256`, `frameCount = u16`.
5. Loop: `w = u16`; `code = w >> 6`; `len = w & 0x3F`; if `len == 0x3F` read
   `u32`; dispatch on `code`; advance `len`. Stop at code 0.

Recursive (`DefineSprite`) and bit-level (`SHAPEWITHSTYLE`) decoding are the only
places that need real care. Everything else is a flat `struct` read.

## 15. Common false conclusions

* **"The tag header is `(code | len << 6)`".** No — code occupies the **high** 10
  bits: `(code << 6) | len`, then written little-endian.
* **"There's a frame header tag".** No; frames are delimited by `ShowFrame`, and
  per-frame content is just the control tags in order.
* **"`FileLength` is the on-disk size".** No, it is the *decompressed* size; for
  `CWS`/`ZWS` the file is shorter.
* **"RGB in a shape style".** Only `DefineShape`/`DefineShape2`; v3 shapes and
  `CXFORMWITHALPHA` use `RGBA`.
* **"Style indices are 0-based".** They are 1-based, with 0 meaning
  none/unchanged.
* **"Shape edges are absolute".** They are relative deltas from the current pen.
* **"Sprites end with an `End` tag".** They end at their declared tag length.
* **"SWF 6 supports zlib-compressed tags".** It compresses the whole body once.

## 16. Where this fits the Flash MX binary

Established so far (see [the hunt log]({{ '/hunt-log/' | relative_url }})):

```
menu "Export Flash Player" / "Export SWF File:"
  -> FUN_005e5bf0    file dialog, forces ".swf", calls document vtable slot 33
  -> FUN_008f6fde    allocates CMirrorFile, calls its Open (0x8f6c40)
  -> slot 14 Write   FUN_008eb7cd -> WriteFile
```

What to look for in the not-yet-found emitter, in SWF-format terms:

* a bit accumulator — `putbits` inlined as shifts into a register plus a
  conditional flush, and a `byte-align` helper that stores the remaining bits;
* `(code << 6) | length` (i.e. `SHL 6` + `OR`) right before a store, and the
  `length >= 63` branch that appends a `UI32`;
* the constant `44` (the `ShowFrame` word `0x0040`) or `9` (background colour)
  appearing as small immediates near that writer;
* a `Seek(4)`-and-rewrite sequence at close, matching the mirror-file design.

## 17. References

* *SWF File Format Specification*, version 10 (Adobe, 2008) — the authoritative
  tag/struct tables; versions 6–8 cover everything Flash MX emits.
* *ActionScript 1/2 bytecode* tables in the same document, chapter "Action
  Model" — needed only for `DoAction`/`DoInitAction` bodies.
* `SWF 6` added: `CWS` compression, `DefineShape3` accents, `FrameLabel` anchors,
  `ScriptLimits`-era changes (SWF 7), `DefineFontInfo2`.
