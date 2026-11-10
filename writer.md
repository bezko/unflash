---
layout: default
title: Recovered writer API
permalink: /writer/
description: The procedural writer struct, its primitive emitters, the tag map with caller chains, and the tags the published format does not define.
---

Reverse-engineered from `extracted/InstallShield/App_Files/Flash.exe`
(PE32, 12 107 776 bytes, sha256 `64280f22…d9510da`, imagebase `0x400000`).

Every claim below is tied to a disassembly or decompilation of a named address. Where a statement is
an inference rather than an observation it says so. Addresses are virtual addresses (VA).

Companion pages: [the format reference]({{ '/format/' | relative_url }}) and
[the hunt log]({{ '/hunt-log/' | relative_url }}).
This document is the **API surface**: the object, the primitives, and the tag map.

---

## 1. Method

Two facts make ordinary cross-referencing useless on this binary and are worth stating first, because
they invalidate most naive scans:

1. **There are almost no direct calls.** 52 291 `jmp` thunks carry the call graph
   (`E8` → thunk → `E9` → target). A scan for `E8` to a target returns zero even for functions with
   dozens of callers. Every caller count here was produced by resolving thunks first, and every
   emitter below was located by scanning references to its **thunk**, not its body.
2. **The writer is not a C++ class.** It has no RTTI descriptor and no vtable. Searching the 1283
   type descriptors for anything named `Writer` finds nothing, because the writer is a plain
   procedural struct passed as `this` in `ecx`.

The findings in §4.2 were additionally checked against the 62 SWF files shipped inside the
installer (Flash 5 and 6 output, §4.1) — the only ground truth available without the player.

---

## 2. The writer object

Recovered from the header writer, the finaliser, the tag primitives and the teardown path. Offsets
are byte offsets from the struct base (`ecx`).

| offset | size | meaning | evidence |
|---|---|---|---|
| `+0x008` … `+0x207` | 512 | output buffer | `putbyte` indexes `[esi+ecx+8]` |
| `+0x208` | u32 | bytes in buffer | `putbyte` increments it; flush at `>0x200` |
| `+0x214` | u32 | pending tag code; cleared by `endTag` | `beginTag` stores `param_2`, `endTag` zeroes it |
| `+0x218` | u32 | file offset of the tag header | `beginTag` stores `tell()` |
| `+0x21c` | u32 | long-header flag for the pending tag | `beginTag` stores `param_3` |
| `+0x230` | ptr | owned object, released by `FUN_007a0180` | `FUN_007a0180` |
| `+0x234` | i32 | offset of `FileLength`; **-1 means "no FileLength"** | header writer stores `tell()`; finaliser tests `>= 0`; timeline init stores `-1` |
| `+0x238` | u32 | offset of `FrameCount` | header writer and timeline init store `tell()` |
| `+0x23c` | u16/u32 | frame count value to patch | finaliser reads it into `put16` |
| `+0x240` | u32 | SWF version | header writer stores `version & 0xff` |
| `+0x280` | ptr | owned object, freed on teardown | `FUN_007a0070` |
| `+0x2bc` | u32 | frame rate, raw 8.8 fixed | header writer stores `param_3` |
| `+0x2d0`, `+0x2d4` | ptr | owned objects / list, freed on teardown | `FUN_007a0070` |
| `+0x2d8` | u8 | compressed flag (0 = `FWS`, 1 = `CWS`) | header writer; finaliser branches on it |
| `+0x2dc` | u32 | body start = end of header | header writer stores `tell()`; compressor reads from it |

Note the two-set pattern at `+0x234`: the same struct is used for the root movie (patch the length)
and for nested timelines (no length field at all). That is the mechanism behind §5.3.

---

## 3. The API

### 3.1 Output primitives

| name | body | thunk | contract |
|---|---|---|---|
| `putbyte` | `0x79f6c0` | `0x4070fe` | appends one byte; flushes via `0x4128b4` when the buffer would exceed 512 |
| `put16` | `0x79f710` | `0x4181d8` | little-endian UI16 (writes `al`, then `ah`) |
| `put32` | `0x79f780` | `0x417de6` | little-endian UI32 |
| `putrect` | `0x79f910` | `0x4110ef` | 5-bit `Nbits`, then four `SB[Nbits]` values (see §5.7) |
| `putstr` | `0x79fe50` | `0x40f1b4` | `strlen`, raw bytes, then a `0x00` terminator |
| `putmem` | `0x79f820` | — | raw block copy of `n` bytes |
| `varwidth` | `0x79f8c0` | `0x414dcb` | 3 bytes of the value, a 4th only when its flag argument is non-zero |
| `tell` | `0x79f4f0` | `0x40b721` | absolute file position |
| `seek` | `0x79f530` | `0x40d404` | reposition output |
| `flush` | `0x79f670` | `0x4128b4` | drain the 512-byte buffer |

`putstr` is the only string writer in the binary; because it appends the NUL itself, no emitter ever
computes a string length, and `0x79f820` is used directly only for binary blobs.

### 3.2 Tag framing — `beginTag` / `endTag`

`beginTag` reserves the header and remembers where it is:

```c
void beginTag(writer, code, longHeader)      // body 0x79f560, thunk 0x413afc
{
    writer->[0x214] = code;
    writer->[0x218] = tell();                // header position
    writer->[0x21c] = longHeader;
    put16(0);                                // reserve the code|length word
    if (longHeader) put32(0);                // reserve the extended length
}
```

`endTag` — **the actual closer, `0x79f5b0`, thunk `0x4123a5`** — computes and patches both forms:

```c
void endTag(writer)                          // thiscall
{
    int end  = tell();
    int hdr  = (writer->[0x21c] != 0 ? 6 : 2);
    uint len = end - hdr - writer->[0x218];
    seek(writer->[0x218]);
    if (writer->[0x21c] == 0)  put16((writer->[0x214] << 6) | len);
    else                     { put16((writer->[0x214] << 6) | 0x3f); put32(len); }
    seek(end);
    writer->[0x214] = 0;
}
```

So the framing is: reserve in `beginTag`, compute in `endTag`. An earlier note in this document
claimed the binary contains no `code << 6 | length` arithmetic — that was wrong, but only because the
arithmetic lives in the closer rather than the opener, and because `endTag`'s thunk was not in the
63-site census used to build §4.

Larger writers do not use `endTag`; `FUN_007a1860` (`DefineShape`) patches by hand because it spans
two passes (§5.5), and several emitters (§4.2) do the same.

### 3.3 Document-level entry points

| name | body | role |
|---|---|---|
| `writeHeader` | `0x7a01c0` | signature, version, `FileLength` slot, `RECT`, `FrameRate`, `FrameCount` slot |
| `initTimeline` | `0x7a02d0` | same state fields but `+0x234 = -1` — no `FileLength`; used for nested timelines |
| `finalise` | `0x7a0340` | patch `FileLength` (only if `+0x234 >= 0`), patch `FrameCount`, flush, compress if `+0x2d8` |
| `compress` | `0x7a03e0` | zlib deflate of `[+0x2dc .. EOF)`, with an `FWS` fallback |
| `teardown` | `0x7a0070` | frees `+0x280`, `+0x2d0`, `+0x2d4`, then calls `0x7a0180` |
| `release230` | `0x7a0180` | `if (+0x230) { FUN_00890900(+0x230); FUN_008e65a5(+0x230); +0x230 = 0; }` |

`0x7a0180` is 46 bytes and does nothing else: it is the destructor for the one owned pointer the
teardown path does not handle inline. It is called by the finaliser and by teardown, so the object at
`+0x230` is released on both the success and the abort path.

`writeHeader`:

```c
c = (flags != 0 && (byte)version >= 6);       // CWS requires SWF 6+
this->[0x2d8] = c;
putbyte(c ? 'C' : 'F'); putbyte('W'); putbyte('S'); putbyte(version);
this->[0x234] = tell();  put32(0);
this->[0x2dc] = tell();
putrect(stageRect);
put16(fps >> 8);  put16(0);                   // FrameRate 8.8 with zero fraction
this->[0x238] = tell();  put16(0);
this->[0x2bc] = fps;  this->[0x23c] = 0;  this->[0x240] = version & 0xff;
```

### 3.4 Shape serialiser

| name | body | role |
|---|---|---|
| `fillStyleArray` | `0x7a1060` | count + entries, style de-duplication, gradient matrices |
| `defineShape` | `0x7a1860` | bounds, header reservation, two style passes, records |
| `shapeRecord` | `0x7a06d0` | initialises the record/count struct |
| `matrix` | `0x7a0ff0` | 66 bytes: identity fast-path, else transform then emit |

`fillStyleArray` entry types observed as immediates: `0x00`, `0x10`, `0x12` (solid, linear gradient,
radial gradient), with bit tests against `0x10` and `0x40` inside the bitmap/focal branches.

---

## 4. Tag map

Method: every reference to the `beginTag` thunk was enumerated and disassembled. There are **62 call
sites** (the earlier count of 63 included the thunk's own `jmp`). The last immediate `push ≤ 0xFF`
before the call gives the code at 37 sites; the rest compute it (§4.3). Codes are compared against
the verified table in [the format reference]({{ '/format/' | relative_url }}) §5.

### 4.1 Corpus check

The installer ships 62 SWF files: 33 `CWS` version 6, 25 `FWS` version 5, 4 `FWS` version 6.
Parsing every tag stream gives the ground truth for what this writer actually puts in a movie:

- present: `0` `1` `2` `6` `8` `9` `10` `11` `12` `13` `14` `15` `17` `20` `22` `24` `26` `28`
  `32` `34` `35` `36` `37` `39` `43` `45` `46` `48` `56` `57` `58` `59` `60` `61` `62` `63`
- absent everywhere: `38` `40` `49` `51` `52`
- `63` appears exactly once — in `Tutorials/ActionScript/Finished/puzzle.swf`.

Notably absent as well are `4`/`5` (`PlaceObject`/`RemoveObject`), `18`/`19` (`SoundStreamHead`/
`SoundStreamBlock`), `21`/`23` (`DefineBitsJPEG2`/`DefineButtonCxform`) and `33` (`DefineText2`):
this corpus was authored with the modern tag set. So a code's absence here is evidence about which
paths the samples exercise, not about whether the emitter is reachable.

That survey is what makes §4.2 more than guesswork.

### 4.2 Confirmed tags

| code | dec | documented as | sites | first site |
|---|---|---|---|---|
| `0x0a` | 10 | DefineFont | 1 | `0x65a8ba` |
| `0x0c` | 12 | DoAction | 1 | `0x42e79c` |
| `0x0d` | 13 | DefineFontInfo | 1 | `0x65abea` |
| `0x0e` | 14 | DefineSound | 1 | `0x7a2f1b` |
| `0x0f` | 15 | StartSound | 1 | `0x653e15` |
| `0x11` | 17 | DefineButtonSound | 1 | `0x658d93` |
| `0x13` | 19 | **SoundStreamBlock** (was mis-listed as DefineBitsLossless; the lossless codes are 20/36, see §4.3) | 5 | `0x7a390f` |
| `0x15` | 21 | DefineBitsJPEG2 | 1 | `0x7a29a7` |
| `0x16` | 22 | DefineShape2 | 2 | `0x656d20` |
| `0x17` | 23 | DefineButtonCxform | 1 | `0x658d15` |
| `0x18` | 24 | Protect | 1 | `0x7a0688` |
| `0x1a` | 26 | PlaceObject2 | 1 | `0x7a2c62` |
| `0x1c` | 28 | RemoveObject2 | 1 | `0x7a2e97` |
| `0x25` | 37 | DefineEditText | 1 | `0x657bba` |
| `0x27` | 39 | DefineSprite | 1 | `0x6578f2` |
| `0x2b` | 43 | FrameLabel | 2 | `0x7a43e0`, `0x7a4420` |
| `0x2e` | 46 | DefineMorphShape | 1 | `0x656eb6` |
| `0x30` | 48 | DefineFont2 | 1 | `0x65b7e6` |
| `0x38` | 56 | ExportAssets | 3 | `0x659618` |
| `0x39` | 57 | ImportAssets | 4 | `0x6567e6` |
| `0x3c` | 60 | DefineVideoStream | 1 | `0x659205` |
| `0x3d` | 61 | VideoFrame | 1 | `0x653fda` |
| `0x3f` | 63 | **nothing** — see §5.10 | 1 | `0x8088ce` |
| `0x40` | 64 | EnableDebugger2 | 2 | `0x8087c5` |

`FrameLabel` has **two emitters**, which the immediate heuristic missed because both push `0x2b`
followed by the long-header flag `0x1`:

| body | emitter |
|---|---|
| `STRING` + `UI8 = 1` | `FUN_007a43e0` (44 B) |
| `STRING` only | `FUN_007a4420` (35 B) |

Both are called only from `FUN_00652e00` (at `0x65317d`/`0x65319c` and `0x6531da`/`0x6531f9`), which
picks one or the other at the call site. Neither variant uses the short header.

### 4.3 The unassigned codes

Five codes are emitted by the movie writer but appear in no file of the shipped corpus; a sixth, 63,
is documented nowhere yet does occur. Bodies were recovered from the emitters themselves, so the
layouts below are observations even where the meaning is not.

| code | dec | body | emitter | call sites | called from |
|---|---|---|---|---|---|
| `0x26` | 38 | `UI16`, `STRING` (**no NUL**), `UI8` = 0/1/2 | inline `0x65911b`–`0x659275` | 1 | `IsKindOf(CMediaVideo)` |
| `0x28` | 40 | `UI16`, `STRING`, `UI16` | `FUN_007a4390` (59 B) | 6 | `FUN_00656770` ×4, `FUN_0065c960` ×2 |
| `0x31` | 49 | `UI16 = packed & 0x0f`, then a 0–3 `UI16` payload chosen by that nibble, then `STRING` | `FUN_007a4460` (138 B) | 9 | `FUN_00652e00` ×6, `FUN_008084b0` ×3 |
| `0x33` | 51 | `UI16`, `UI16` | `FUN_007a4540` (47 B) | 1 | `0x808a32` |
| `0x34` | 52 | `UI16`, `STRING`, `UI16`; **not emitted at all when the string is empty** | `FUN_007a4580` (79 B) | 2 | `FUN_00656360` |
| `0x3f` | 63 | 16 raw bytes | inline | 1 | `0x8088ce` |

Details that are worth knowing when parsing or forging these:

- **40 and 52 have the identical body shape** (`UI16`, `STRING`, `UI16`) but different callers and
  different guards: 52 is fully suppressed for an empty string, 40 never is. 52 also writes its
  arguments in the opposite order to its parameters — `FUN_007a4580(name, a, b)` emits `b`, the name,
  then `a`; 40 (`FUN_007a4390(a, name, b)`) emits them in order. Both are hard-wired emitters: 40
  always calls `beginTag(0x28, 1)` and takes no code argument, which kills the earlier hypothesis
  that `0x409dd1` is a shared body writer reading its code from `[esp+0x103c]`.
- **49 is a variant record driven by a jump table, not a variable-length switch.** The fourth
  argument is packed; its low nibble is written as a `UI16` first (`put16(arg4 & 0x0f)`), then the
  value is decremented and range-checked (`lea eax,[edi-1]; cmp eax,7; ja tail`), and the table at
  `0x7a44ec` has exactly eight entries, one per discriminator 1…8:

  | disc | payload `UI16`s |
  |---|---|
  | 1, 2 | one — `param_3` (both entries emit identical bytes, via two different registers) |
  | 3 | none |
  | 4 | three — `param_3`, `param_4 & 0xf0`, `param_4 >> 16` |
  | 5, 6, 7 | none |
  | 8 | one — `param_2` |
  | 0 or > 8 | none (the `ja` short-circuits before the table) |

  The `STRING` is written last in every case. An earlier revision of this table claimed "1 for 1, 2
  and 8; 3 for 4; 1 by default"; that was wrong twice — the default emits no numbers at all, and
  discriminators 1/2 and 8 carry different values (the third and second parameter respectively).
- **49 is a per-timeline-element record, and its discriminator is data, not code.** All nine call
  sites live in two functions: `FUN_00652e00` (the frame writer, six sites) and `FUN_008084b0` (the
  publish driver, three sites). The frame-writer sites pass either a literal `0` or a `UI16` field
  copied out of the layer/frame object at `+0x62`, alongside the frame count `[obj+0x160]`; the
  publish-driver sites pass `(CString, 0, 0)` guarded by a non-empty string, so they emit the
  discriminator `0` form — code, a name, no numbers. Because the packed argument is a copied field,
  the eight-entry payload table of §4.3 is reachable only through whatever those objects hold; no
  site in this build hard-wires anything but `0`.
- **51 is called with a boolean and a zero**: `0x808a32` pushes `GetSystemMetrics(0x2a) != 0` and `0`,
  so the emitted body is `UI16 0/1`, `UI16 0`. Metric `0x2a` is 42, `SM_DBCSENABLED`, so the bit is
  "this build ran on a double-byte-character-set (CJK) system". The call sits in the publish driver
  `FUN_008084b0` behind `[doc+0x54] != 0` and immediately before the run of `0x6ed2d0` … `0x6eed00`
  that assembles the font tables, so tag 51 is a once-per-file header record describing the font/text
  environment of the machine that compiled the movie, not a per-object tag.
- **38 is a video record.** The guard is an `IsKindOf(CMediaVideo)` test at `0x6590f1`; the body is
  `UI16` (the id stored in the object at `[edi+8]`), a name taken from `[ebx+0x88]` and written with
  `putmem` from an explicit `strlen` — **no NUL terminator** — then a `UI8` that takes 0, 1 or 2 from
  the three paths that converge on the shared `putbyte`/`endTag` tail at `0x659265`. The `version >= 6`
  test at `0x659130` picks only how the name is marshalled (temporary `CString` copy for v6+, raw byte
  pointer below), not whether the tag is written. What the three flag values mean is not established.

### 4.4 The 25 "indirect" sites, resolved

The sites that do not push a literal are mostly one pattern — **tag promotion by alpha** — which is
why they look computed:

```
0x7a26e0:  EAX = [ESP+0x8a4]        ; nonzero flag
0x7a26e9:  NEG EAX / SBB EAX,EAX     ; -> 0 or -1
0x7a26ef:  AND EAX,0x10
0x7a26f2:  ADD EAX,0x14              ; 20 or 36
0x7a26f6:  CALL beginTag
```

`0x14` = 20 `DefineBitsLossless`, `0x24` = 36 `DefineBitsLossless2`. The parallel pair at `0x7a237a`
computes `0x15 + (flag ? 0x0e : 0)` → 21 `DefineBitsJPEG2` / 35 `DefineBitsJPEG3`. Since those are
exactly the alpha-bearing variants, the flag is almost certainly "has alpha channel" (inference; the
guard that fills the flag was not traced).

This also corrects §4.2's single `DefineBitsLossless` row: the five sites in `0x7a38f0`–`0x7a3f10`
are the lossless pair, while code 19 is the sound-block emitter.

### 4.5 The dispatch: class tests, version gates, and tag promotion

`FUN_00656770` is not a switch on an enum. It is a chain of `IsKindOf` tests: `0x8eb3b9(object,
&descriptor)`, where the descriptor's first dword points at a class-name string, so every test names
the authoring class it accepts. Eleven of them sit in the function:

| test site | descriptor | class |
|---|---|---|
| `0x6568e6` | `0xbd5a80` | `CPicShape` |
| `0x656e01` | `0xbcf4a0` | `CPicMorphShape` |
| `0x656ef2` | `0xbcd188` | `CCommandObj` |
| `0x6570a5` | `0xbd5e80` | `CPicText` |
| `0x65897f` | `0xbd5bf8` | `CPicSprite` |
| `0x658993` | `0xbdd5e0` | `CDocumentPage` |
| `0x6589a7` | `0xbd0968` | `CPicButton` |
| `0x658e62` | `0xbce6d8` | `CMediaBits` |
| `0x6590f1` | `0xbcfd20` | `CMediaVideo` |
| `0x6591c5` | `0xbd73d8` | `CPicVideoStream` |
| `0x659290` | `0xbdd5e0` | `CDocumentPage` |

Both video chains were read instruction by instruction, so their gates are observed rather than
inferred:

- `IsKindOf(CMediaVideo)` → `0x65911b`, tag 38 (§4.3), always emitted.
- `IsKindOf(CPicVideoStream)` → `0x659205`, tag 60 `DefineVideoStream` — but only when
  `[+0x110] > 5`. The gate is `0x6591d2 cmp [ebp+0x110],5` / `0x6591d9 jle 0x65927a`, and `0x65927a`
  pushes `0x1e` and calls `0x40c342` → `FUN_006511f0` → `FUN_00650af0(obj+8, arg, 0, arg)`: the
  reporting path, with nothing written. Video export is *rejected* below version 6, not downgraded.
  **Correction:** an earlier revision named `0x6fa8d0` as the diagnostic raiser. That function is a
  seven-byte getter, `return *(param_1 + 0x94)`, and is called on the *success* path (`0x6591e1`, via
  thunk `0x401f5a`) and inlined at `0x65929f`.
  The reporting path itself is `FUN_00650af0`, and it is a message formatter rather than an abort:
  behind `DAT_00c155f8 != 0` it tests the object against `CPicObj`, `CMediaBits`, `CMediaElem`,
  `CPicFrame` and `CPicLayer` to build a "what and where" clause, then switches on the code — `0x1e`
  maps to string resource `0x5cc` — and loads that string for display. The refusal is therefore a
  user-visible warning, and a headless export can drop the object silently: nothing is written and
  nothing is raised if the message path is disabled.

The remaining eight descriptors are paired to tag sites in §4.6, which follows the taken edge from
each test instead of physical adjacency.

Where a tag exists in several generations the exporter picks one at the call site, and the choice is a
feature test, not a version number:

| site | codes emitted | reading |
|---|---|---|
| `0x656d20` | `2` / `0x20` / `0x16` | `DefineShape` / `DefineShape3` / `DefineShape2` |
| `0x657178`, `0x657fee` | `0x0b` / `0x21` | `DefineText` / `DefineText2` |
| `0x658b2b` | `7` / `0x22` | `DefineButton` / `DefineButton2` |
| `0x7a26f2`, `0x7a237a` | `0x14`/`0x24`, `0x15`/`0x23` | `DefineBitsLossless`/`Lossless2`, `JPEG2`/`JPEG3` (§4.4) |

The text and button pairs compute their code arithmetically, which is why an immediate-scan census
misses them:

```
0x657171:  AND EDX,0x16 / ADD EDX,0x0b      ; 11 DefineText    or 33 DefineText2
0x658b24:  AND EDX,0x1b / ADD EDX,0x07      ;  7 DefineButton   or 34 DefineButton2
```

For shapes the three codes are separate literal branches off `[obj+4]` (`0x656d02`, `0x656d11`), so a
shape with per-fill bitmap data is written as `DefineShape3` and a plain one as `DefineShape`; which
condition separates `0x20` from `0x16` was not resolved.

### 4.6 Class test → tag site, by control flow

`FUN_00656770` is one `if (IsKindOf(A) == 0) { if (IsKindOf(B) == 0) … }` ladder, so a handler is the
*taken* edge after its test, not the code physically nearest it. Following that edge and walking
fallthrough / `jmp` / `jcc` until the first `beginTag` call (`.tools/pairdesc.py`) replaces the
adjacency guesses with reachability:

| test | class | first tag reached | what it is |
|---|---|---|---|
| `0x6568e6` | `CPicShape` | `0x656d20` | `2` / `0x20` / `0x16` — DefineShape family |
| `0x656e01` | `CPicMorphShape` | `0x656eb6` | `0x2e` DefineMorphShape |
| `0x656ef2` | `CCommandObj` | `0x656fe6` | `0x27` DefineSprite |
| `0x6570a5` | `CPicText` | `0x657178`, `0x657bba` | text / text2 promotion, `0x25` DefineEditText |
| `0x65897f` | `CPicSprite` | `0x659369` (via `0x659289`) | `0x27` DefineSprite |
| `0x658993`, `0x659290` | `CDocumentPage` | `0x659369` | `0x27` DefineSprite |
| `0x6589a7` | `CPicButton` | `0x658b2b` | `7` / `0x22` DefineButton family |
| `0x658e62` | `CMediaBits` | `0x659095`, `0x6590bf` | code 40, the two marshalling variants |
| `0x6590f1` | `CMediaVideo` | `0x65911b` | `0x26` = 38 |
| `0x6591c5` | `CPicVideoStream` | `0x659205` | `0x3c` DefineVideoStream |

`CPicSprite` and both `CDocumentPage` tests share a handler: the chain jumps to `0x659289`, which
re-tests `CDocumentPage` only to choose between `[obj+0x94]` and the object itself (`0x65929f` vs
`0x659299`), then falls into `0x6592a9`. That block builds a child writer (`0x40e606`, `0x4181a1`,
`0x4017f8`), copies the parent version from `[writer+0x240]` and `[+0x1ac]`, calls `0x40a7bd`, and
only then writes `0x27`. So `DefineSprite` is emitted for a page or a sprite that owns a nested
timeline — one handler, three classes.

`CPicText` reaches three sites from one handler: the two `0x0b`/`0x21` promotion sites and `0x25`
`DefineEditText`. A text object is therefore written as text, text2 or edit text depending on its
fields, not on its class.

**`ExportAssets` (56) is a per-object flag test, not a name lookup.** The site `0x659618` is preceded
by `call 0x4107df` → `0x5408c0`, twelve bytes long:

```c
bool FUN_005408c0(obj) { return *(int *)(obj + 8) == 1; }
```

`beginTag(0x38)` runs only when that returns non-zero, and the same guard sits on several handlers'
tails, so export records are emitted per object carrying the flag. `CCommandObj`, `CMediaBits` and
`CPicVideoStream` all reach it. The flag's authoring meaning is not established.

### 4.7 The `DefineVideoStream` (60) body

`0x659205` onwards, in order: `put16([obj+8])` (the character id), `put16(FUN_006843f0(x))` where
`FUN_006843f0` returns `([x+0x138] - [x+0x134]) >> 4` (a frame count), `put16([x+0x100])`,
`put16([x+0x104])` (width, height), `putbyte(0)` for the flags byte, then a final `putbyte` from
`[x+0xd8]` mapped as 1 → 1, 2 → 2, anything else → 0. That is the documented field order with the
codec id reduced to three cases.

---

## 5. Undocumented / undocumented-in-practice behaviour

These are the findings that are absent from, or contradicted by, the published SWF specification.
Each is stated with the evidence that supports it.

### 5.1 A compressed file can be silently rewritten to uncompressed

`FUN_007a03e0` always attempts deflate, then compares the result with the input size:

```c
if (zs.total_out < body) { seek(bodyStart); seek(0, SEEK_END); Write(compressed, total_out); }
else                     { b = 'F'; seek(0, SEEK_START); Write(&b, 1); }
```

Flash writes `CWS` in the header *before* it knows whether compression helps, then overwrites byte 0
with `'F'` if it does not. There is no repacking and no second pass — the uncompressed body is
already on disk, so flipping the signature is sufficient. A reader therefore cannot assume that a
`CWS`-capable exporter produced `CWS`. The shipped corpus confirms both outcomes from the same
toolchain: 33 `CWS` files and 29 `FWS` files, four of those `FWS` version 6 — files this very build
could have written compressed.

### 5.2 zlib 1.0.4 is statically linked

The compressor passes `0xbc37dc` to `deflateInit_` as the version argument; that string is `"1.0.4"`,
and the fourth argument is `0x38` = `sizeof(z_stream)`. `deflateInit_(strm, -1, …)` uses
`Z_DEFAULT_COMPRESSION` (-1), not an explicit level. The import table contains no zlib DLL, which is
why earlier rounds wrongly concluded the binary could not compress.

### 5.3 The same struct writes both movies and sprites

`writeHeader` stores `tell()` into `+0x234`; `initTimeline` (`0x7a02d0`) stores **-1** there and only
sets up `FrameCount`. The finaliser tests `if (this->[0x234] >= 0)` before patching `FileLength`:

```c
put16(0);
if (this->[0x234] >= 0) { len = tell(); seek(this->[0x234]); put32(len); }
seek(this->[0x238]); put16(this->[0x23c]);
flush(); if (this->[0x2d8]) compress();
```

So one writer serves the root movie and every nested timeline, distinguished only by a sentinel
value. Nested timelines never get the `FWS` header, which is correct for `DefineSprite` bodies but
means the sprite path is not a separate code path.

### 5.4 String and blob writing is centralised

`putstr` (`0x79fe50`) computes `strlen`, copies the bytes through `putmem`, and appends `0x00`; most
emitters never pass a length for a name, so a `STRING` field costs `len+1` bytes and the length
arithmetic is `tell()`-based. **Code 38 is the exception** (§4.3): it runs its own `strlen` and calls
`putmem` directly, so its name carries no terminator. A reader that assumes every `STRING` ends in
`0x00` will misparse that tag by one byte. Binary payloads also call `0x79f820` directly — which is
how the 16-byte body of §5.10 is written.

### 5.5 Shapes are emitted twice, into one tag

`FUN_007a1860` runs `for (pass = 0; pass < 2; pass++)`, and the style counts are selected as
`~-(uint)(pass != 0) & count` — i.e. `0` on the first pass and the real count on the second. The tag
length is patched at the start of the second pass, so the recorded length covers both. This is why a
reader sees two style arrays and interleaved records where the spec describes one.

### 5.6 `FillStyleCount` escape is `0xFF` + UI16

`FUN_007a1060` writes `count` as a byte, and when `count >= 0xFF` writes `0xFF` followed by a UI16
count. The published format describes the escape but says nothing about the exact threshold
behaviour; here the boundary is inclusive at 255 and the extended count is written even when the
byte form *could* have been used for 255 values.

### 5.7 The RECT writer abs()es `Ymax` first

`FUN_0079f910` does `abs(rect->ymax)` before feeding the four values to the min/max reduction, then
emits `5` as the `Nbits` field. A naive reader that trusts `Ymax` sign will disagree with Flash on
movies whose bounds are wholly above the origin.

### 5.8 `ImportAssets` (57) outnumbers `ExportAssets` (56)

Four sites push `0x39` (57) and three push `0x38` (56). The corpus agrees: `shim.swf` carries two
`ImportAssets` tags and 56 `ExportAssets` tags, while `lib.swf` and `panel.swf` carry exports only.
Placing an asset on the stage of a document whose library entry is shared therefore produces an
import record even in a single-file export — worth knowing when reconstructing why a movie references
an asset it never defines.

### 5.9 `FrameLabel` is always long-form, and claims an anchor only when the label says so

Both emitters push the long-header flag, so a frame label costs a 6-byte header even though its body
is a short string, and both hard-wire code `0x2b` (43):

```
0x7a43e0  beginTag(0x2b, 1); putstr(name); putbyte(1); endTag();    44 bytes
0x7a4420  beginTag(0x2b, 1); putstr(name);            endTag();    35 bytes
```

**Correction:** an earlier revision said every label is marked as a named anchor. It is not
unconditional — `FUN_00652e00` branches on `[label+0x1e4]`, taking the 44-byte form only when that
field is non-zero and the 35-byte form otherwise, and the version test on the same path selects only
how the name string is marshalled (temporary `CString` copy for v6+, raw pointer below), exactly as
in §4.3's code-38 case. So the anchor byte reflects a per-label document field. Writing it into a
pre-6 file would be invalid, and nothing at the call site enforces that — the caller must (inference).

### 5.10 Tag 63 carries a 16-byte binary ID, and there is a companion file

`FUN_008084b0` — the publish driver — writes, when the debug flag is set:

```
putmem(&DAT_00be4b64, 4)     ; bytes 'F' 'W' 'D' 0x06  -> a second, separate file object
beginTag(0x3f);  putmem(buf16, 16);  endTag();
put32(...);  putmem(buf16, 16);  flush();   ; same 16 bytes to the same second file
```

`puzzle.swf` shows the movie-side half of this: tag `0x3f`, length 16, body
`7f6a765d 0d2311d6 af270050 e450cf68`, placed immediately after `EnableDebugger` (58). So code 63 is
a real, unassigned tag whose entire payload is a 16-byte id/digest; no length, no string, no flags.
The published format assigns nothing to 63 and describes no such record.

The 4-byte `"FWD"`+`0x06` prefix and the trailing UI32 + 16 bytes go to a different writer object
than the movie: the `.swd` companion file, whose path this same function derives (below). Whether the
id is byte-identical to the movie's tag-63 payload cannot be proven from the decompile (the same
local is used, but the two writes are 140 bytes apart); the corpus movie carries no `FWD` marker
anywhere, so the prefix is definitely not part of the movie.

**The second writer is the `.swd` file, and its path is computed in the same function.** Immediately
before the `"FWD"`+`0x06` write, `FUN_008084b0` copies the current document's file name and rewrites
its extension: if the last four characters are `.swf` the final `f` becomes `d`, otherwise the string
gets `.swd` appended. That path is opened as a `CStdioFile` into `local_30`, and `local_30` is the
object that receives the marker, the UI32 and the second copy of the 16 bytes. The companion file is
therefore not inferred from behaviour any more — it is named by the code, derived from the movie path,
and written through a separate handle.

### 5.11 Image tags are promoted by alpha, not chosen by the author

The two computed-code sites of §4.4 mean the exporter, not the document, decides between
`DefineBitsJPEG2`/`DefineBitsJPEG3` and `DefineBitsLossless`/`DefineBitsLossless2`: the presence of an
alpha channel selects the `2`/`3` form, which is the one that has a `BitmapAlphaData` field. A
reconstruction that always writes the classic tag will silently drop transparency.

### 5.12 A tag can be suppressed by its own payload

`FUN_007a4580` (code 52) inlines a `strlen`, tests it, and writes the tag **only when the string is
non-empty**:

```c
if (len != 0) { beginTag(0x34, 1); put16(param_3); putstr(param_1); put16(param_2); endTag(); }
```

The string is the *first* parameter while the first word written comes from the *third*, so the
argument order and the field order disagree. The consequence for a reader is the interesting part: an
object can be present in the document and leave no bytes in the file, so absence of this tag is not
evidence of absence of the object.

---

## 6. What is still unknown

- **What authoring feature triggers 40, 49 and 52.** The layouts are exact and each emitter is
  dedicated (it hard-wires both its code and its argument order). 49 is now pinned to two functions
  and is emitted per timeline element from the frame writer; 51 is resolved (§4.3). But no site says
  which *authoring* feature arrives at 40, 49 or 52. None of the three appears in the 62 shipped
  movies. Accessibility data, screen/form documents and the `.swd` companion remain candidates; none
  is proven.
- **What `FUN_00652e00` writes tag 49 *for*.** It is the frame/timeline writer — it walks the layer
  list at `[doc+0x50]`, matches layers by a `short` at `+0x62`, and calls tag 49 with a frame count
  and that same `short`. Whether the field is a layer id, a scene index or a sound/stream id is not
  established, so the meaning of the record is open even though its bytes are not.
- **What code 38's `UI8` selects.** Three converging paths produce 0, 1 and 2 (§4.3) and no code
  branches on the result.
- **What `[esp+0x1030]` and `[esp+0x103c]` are.** These frame locals take small immediates (`0x16`,
  `0x15`, `0x14`, `0x13`, `0x11`, `0x10`) around the tag calls and are reset to `0xffffffff`
  afterwards. They are demonstrably *not* the tag code — the emitters hard-wire that — so their role
  (SEH state, current-object kind, scratch) is open.
- **`varwidth`'s role.** It emits 3 or 4 bytes of a value depending on a flag, which does not match
  any single format field; likely a length or a 24-bit ID, unconfirmed.
- **Whether anything reads tag 51's DBCS bit.** The metric is `SM_DBCSENABLED` and the recorded value
  is a plain 0/1, but no code in this build reads the tag back, so whether a player uses it to pick a
  text-encoding path is unknown.
- **Which condition separates `DefineShape2` from `DefineShape3`.** The three shape codes are separate
  literal branches off `[obj+4]`; the `0x20` vs `0x16` split is unresolved.
