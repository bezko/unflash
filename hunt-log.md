---
layout: default
title: Hunt log
permalink: /hunt-log/
description: Chronological record of the static reverse engineering of the Flash MX SWF exporter.
---

> **Note.** This is the working log, kept in the order the work happened. Names such as
> `.tools/emit4052.out` are local scratch captures and are not published; every address is a
> VA in the image identified on the [home page]({{ '/' | relative_url }}).

Target: `extracted/InstallShield/App_Files/Flash.exe`
PE32, 12 107 776 bytes, sha256 `64280f22c156cff5d4aa2011aeed0a58c4b555b381d64395b8d52d548d9510da`
imagebase 0x400000, linker 6.0, timestamp 0x3C88A4AE (2002-03-08), FileVersion 6,0,21,0.

All addresses below are VAs in Flash.exe's image space.

## 1. Command surface (strings live in .rsrc as UTF-16, so no code xrefs)

    "Export Flash Player"                  va 0xefe57e
    "Export FutureSplash Player"           va 0xefe14e
    "Export SWF File:"                     va 0xf6dad8
    "Export Movie" (filter "Flash Movie|SWFL|FutureSpla...")  va 0xf6c93a
    "Publish: SWF Compression"             va 0xf75338
    "Publish: SWF Player Version %d"       va 0xf7536a

These are RT_MENU/RT_STRING resources, fetched by ID (LoadString) — that is why
string->code xref scans find nothing for them.

## 2. Export-SWF command  (FUN_005e5bf0 @0x5e5bf0, 442 bytes)

    FUN_009150e6(&ctx, "Import"@0xbbfa20, "Export Dir"@0xbc97cc, "")
    FUN_008e65b0(0x9d); FUN_008e65b0(0x85)          // resource/string ids
    thunk_FUN_00437510(0,0,0,0x1006, filter, 0)     // GetOpenFileName-style
    ... dialog template id 0x813b ...
    FUN_008d878c / thunk_FUN_00745310               // path -> CString
    FUN_008ffd8e("Import","Export Dir", path)       // last-dir bookkeeping
    if (ext != ".swf") { CString += '.' ; CString += "swf" }   // ".swf" from 0xbc97dc
    (**(code **)(*doc + 0x84))(local_44)            // slot 33  @0x5e5da2

So: pick a name -> force a `.swf` suffix -> hand it to the document object's
virtual method #33.

## 3. Slot 33 == "open an output file object"  (FUN_008f6fde @0x8f6fde, 114 bytes)

Resolved owners of that slot (all documents, i.e. a shared base implementation):

    CSketchDoc        vftable 0xaf1e4c   82 slots
    CFlashPlayerDoc   vftable 0xad8c38   44 slots
    COleDocument      vftable 0xb02228
    COleLinkingDoc    vftable 0xb04b44

Body: `new(0x14)` -> write vftable `0xb00068` -> call virtual slot 0x20 (index 8)
with (path, flags, CFileException*), return null on failure.

The class at vftable 0xb00068 is **CMirrorFile** (COL 0xb39b28 ->
TypeDescriptor `.?AVCMirrorFile@@`). MFC mirror-file semantics: bytes go to a
temporary file and the finished file is renamed over the target on close.

### CMirrorFile vtable 0xb00068 (indices 0..20)

    0  0x914d8f      1  0x8f7050  scalar deleting dtor
    2  0x41adf2      3  0x8eb847
    4  0x8ecb6c      5  0x8ecbee
    6  0x8ecc70      7  0x40f82b
    8  0x8f6c40  <- Open override
    9  0x8eb601
   10  0x8eb818  <- Seek  (SetFilePointer(m_hFile, ...); m_hFile = this+4)
   11  0x8eb944     12  0x8eb970     13  0x8eb793
   14  0x8eb7cd  <- Write (WriteFile(m_hFile, buf, len, ...); err state this+0xc)
   15  0x8eb8f2     16  0x8eb91b     17  0x8f6de7
   18  0x8eb870     19  0x8f6e00     20  0x8eb99d

So the whole byte sink for SWF output is MFC `CFile::Write` on a CMirrorFile.
There is no bespoke `CSwfWriter`/`CTagWriter` class (full RTTI inventory: 1283
type descriptors, no writer class exists).

## 4. Why the SWF header is invisible

`'FWS'` / `'CWS'` byte triples do **not** occur anywhere in `.text` or `.rdata`
of Flash.exe. Every occurrence is in `.data` or `.rsrc`:

    .rsrc 0xedb25c  "CWS\0" "FWS\0" "FWS\x06\0\0\0\0"   (mini-SWF header template)
    .data / .rsrc blobs shared verbatim with authplay.dll and SAFlashPlayer.exe

The same cluster appears in `authplay.dll` (file 0xac754/0xac758, .data) and in
`SAFlashPlayer.exe` (file 0xa3b50/0xa3b54, .data), i.e. it is shared *player*
engine data (built-in ActionScript library mini-SWF), not authoring export code.
Byte-immediate store scan for 0x46/0x57/0x53/0x5A in byte-sized operands found
647 sites, **none** in the 0x800000-0x840000 engine region.

Conclusion: the 8-byte SWF header is not assembled from literals in this binary.
It must be produced either by a bit/byte writer fed from the document model, or
by the movie serializer in the 0x80xxxx engine.

## 5. Related paths that are *not* the SWF writer (all checked and excluded)

- FUN_00993b10 @0x993b10 — the only `mmioOpen/mmioCreateChunk/mmioWrite` user:
  a RIFF **WAVE** writer (fccType 'WAVE'=0x45564157, 'fmt '=0x20746d66,
  'data'=0x61746164). The mmio path is sound export only.
- FUN_00803200 @0x803200 — Test Movie / playtest: iterates entries, rejects
  anything whose first dword != 0x666c7368 ("hslf"), loads the movie into the
  player (FUN_00839ef0), builds `%TEMP%\FlashTmp.swf`, launches it.
- FUN_0083518c @0x83518c — MSVC CRT error reporter ("Microsoft Visual C++
  Runtime Library").
- FUN_00832d3b @0x832d3b / FUN_00830548 @0x830548 — CRT `_write`/`_lseek`
  (fd table DAT_00ce4120, CRLF translation).
- FUN_005c6350 @0x5c6350 — EPS/PostScript exporter ("%!PS-Adobe-3.0 EPSF-3.0",
  "%%BoundingBox", metafile rendering).
- FUN_009581e0 / 0x957e70 / 0x958020 — dump a resource to `%sflash_i.png`,
  `%sflash_i.ai`, `%sflash_i_%s`, then post a message with the path.
- FUN_006bf070 / 0x6bf3e0 — "Mmfdata"/"DashBoard" preference writers.
- FUN_005de9d0 @0x5de9d0 — extracts the embedded authplay.dll resource (801,
  815104 bytes) to disk and `ExternalPlayer_Initialize`s it. authplay.dll
  exports exactly one symbol, so it cannot write SWFs.

## 6. Module architecture

    Flash.exe        authoring app + document model + export/publish UI
                     + embedded authplay.dll as resource 801 (815104 bytes)
    authplay.dll     player engine, single export ExternalPlayer_Initialize
    SAFlashPlayer.exe standalone player (same engine data, no exports)

No separate SWF-writer DLL is shipped, so all writing is inside Flash.exe.

## 7. Open question

The tag emitter itself (the function that walks the MFI document modules and
emits DefineShape/PlaceObject2/ShowFrame... tags) has not been isolated yet.
Candidates left to examine: the 0x80xxxx "movie file" layer around
FUN_00801b80 / 0x802530 / 0x804c60 / 0x80bb30, and the MFI module vtable
cluster (MFIMovieModule vftable 0xaba1d8, slots 10..15 ->
0x482dd0/0x482ec0/0x482fa0/0x483080/0x483160/0x483230, which are thin
exception-frame wrappers around thunk_FUN_00482ab0).

## Tooling

`.tools/ghidra_12.2_PUBLIC` + `.tools/jdk-25.0.5+8` (JDK 21 is rejected),
scripts in `.tools/ghscripts`: RefScan, DecompFn, Insight, MagicScan, SigScan,
VtableDump, Sites, ImpXref, HdrBuild, VCallScan. Helper `.tools/rtti.py`
(RTTI -> COL -> vftable), `.tools/swfscan.py`, `.tools/venv` (capstone, pefile).

---

## 8. Refinement: slot 33 is a *file-opening service*, not the serializer

Raw-dword verification (not inference):

    vtable 0xaf1e4c CSketchDoc      +0x80=0x92af98  +0x84=0x8f6fde  +0x88=0x8f70a1
    vtable 0xad8c38 CFlashPlayerDoc +0x80=0x8f6a2e  +0x84=0x8f6fde
    vtable 0xb02228 COleDocument    +0x80=0x8f6a2e  +0x84=0x8f6fde
    vtable 0xb04b44 COleLinkingDoc  +0x80=0x92af98  +0x84=0x8f6fde

`FUN_008f6fde` full body: alloc 0x14, stamp vftable `0xb00068`, call slot `+0x20` (Open) with
three stack arguments, on failure call slot 1 (deleting dtor) and return null; otherwise
return the object. **It has zero direct `E8` callers** — reachable only through slot 33.

So slot 33 = *create/open an output file object* and returns a `CFile*`. Callers such as
`FUN_00805810` / `FUN_009cd1b0` show the idiom:

    p = doc->v[33](path);
    if (!p) p = doc->v[33](alternate_path);      // retry: temp dir vs document dir
    if (p) p->v[6]();                            // release/close

Hence the serializer receives that `CFile*` and writes through **virtual** `Write`
(slot 14, byte offset `0x38`). It never calls `0x8eb7cd` directly — which is exactly why
call-site hunting for the writer kept failing.

## 9. Excluded this round (all decompiled, not guessed)

- **Entire `0x80xxxx` "movie layer" cluster** — it is import-side:
  `FUN_00801b80` = WMF/metafile reader (`SetWinMetaFileBits`, `DeleteEnhMetaFile`, METAFILEPICT);
  `FUN_00802530` = `CreateFile`+close wrapper;
  `FUN_00802cb0` = file loader (seek+read, progress callbacks);
  `FUN_0080bb30` = asset importer reading `0x400`-byte blocks, progress via caller slots `0xdc`/`0xe8`;
  `FUN_00805810` = library/component template path, builds `...\libraries\...\dummy.fla`.
- **All 18 direct calls to `CFile::Write`** are now accounted for. The last unclassified one,
  `0x81b726` inside `FUN_0081b64b`, is a resource dumper (`FindResourceA`/`SizeofResource`/
  `LockResource` + write).
- `FUN_005e5dbb` is a destructor/unwind stub (callees: `~CDialog`, `FUN_008e5eab`, `FUN_008eb385`)
  — the export dialog performs nothing after the slot-33 call.
- `FUN_004ede90` / `FUN_004e6e60` = "seek to offset, then read N bytes into a `LocalAlloc` buffer"
  helpers (used to pull strings/blobs out of a file) — **not** header back-patching.
- **Tag framing is not `shl reg,6`.** There are 118 `C1 Ex 06` sites in `.text`; the 66 in the
  engine region are all `lea eax,[ecx+ecx*8]; shl eax,6` = index × 576 (array/hash sizing).
  So `(code<<6)|len` is built some other way (`lea`/`imul`), or the word is assembled from two
  byte stores.

Method note: a first attempt at "decode all of `.text` with capstone and pattern-match" silently
yielded zero instructions because the section range was unpacked from a **dict** (keys reversed
the bounds). The numbers above come from the corrected scan plus targeted disassembly of
windows around byte-pattern hits; full-text capstone decoding with `detail=True` is too slow here.

## 10. Export command identified at the data level

The export dialog's handler is reached through an MFC message map, not a vtable:

    0xad80a4: {pfn=0x41b473, msg=0x111 (WM_COMMAND), 0, id=5, idLast=5, sig=0x0c}
    0xad80bc: {pfn=0x40b4b5, msg=0x111,              0, id=5, idLast=5, sig=0x0c}

`0x40b4b5` is a `jmp` thunk to `FUN_005e5bf0` (the export dialog), so **the export command is
WM_COMMAND id 5** in that map. (The vtable walk-back for `0x40b4b5` earlier landed on `0xad80bc`
with `COL=0xc`, which is what proved this is a 24-byte `AFX_MSGMAP_ENTRY` stream and not a vtable.)

## 11. Consequence / next probe

The writer must be found by **virtual `Write` dispatches**, not direct calls. Seed list:
`call dword ptr [reg+0x38]` sites (opcode `FF /2 38`), intersected with functions reachable from
the export command (id 5) rather than from the import paths excluded above.

Seed inventory measured: **104** `FF /2 38` sites in `.text`, 44 of them at `0x800000+`, and 30 of
those fall outside functions already excluded. Densest engine pages: `0x855000` (5), the rest
scattered across `0x8f3xxx`-`0xa18xxx`.

## 12. More exclusions (checked after writing §11)

- **`0x855000` cluster** — `FUN_008559cc` and neighbours are **DIB/bitmap** code: it fills a
  `BITMAPINFOHEADER` (`*(this+0x10) = 0x30` biSize, `+0x14 = 0`, palette fields), queries an
  interface via `&DAT_00b093d0` and calls `+0x14`, `+0x34`, `+0xc`, `+0x4c`, `+0x38` on
  lock/surface objects. That is a DIB write path, not SWF tag emission. All five `0x855xxx`
  `+0x38` calls are this.
- `cmp reg,0x3f` in the engine region (`0x820d16`, `0x82c7xx`-`0x82d4xx`, 12 sites) is capacity
  clamping in string/hash code (`sar edx,4; dec edx; cmp edx,0x3f; push 0x3f; pop edx`), not the
  SWF long-tag-form check.

So the exporter is still unlocated, but the search space is now: 30 `FF /2 38` sites, minus
DIB/bitmap and resource paths, with the export command pinned to WM_COMMAND **id 5**.

## 13. Hard negative: the signature is not a literal *in any form*

Measured over the whole of `.text` (`0x401000`-`0xab7000`):

| probe | result |
|---|---|
| `46 57 53 00` (`"FWS\0"`) | **0** occurrences |
| `43 57 53 00` (`"CWS\0"`) | **0** occurrences |
| `53 57 46 00` (`"SWF\0"`, reversed) | **0** occurrences |
| code references to any `.data`/`.rsrc` `"FWS"`|	**0** (all 10 data + 5 resource sites checked) |
| `FF 15`/`E8` refs into `mmio*` IAT slots | all inside `0x993xxx` only |

For contrast, `hslf` (`0x666c7368`) *does* appear as an instruction immediate, at `0x6fc103`,
`0x6fd1df` and `0x80351d` (the latter inside the Test Movie function `FUN_00803200`) — so this
binary does use 4-byte magics as immediates when it wants to.

Consequence: the header cannot be produced by copying a static `"FWS"` and cannot be a 32-bit
immediate in either byte order. It must be either (a) three separate **byte-sized** immediates or
byte computations (`'F'`=0x46, `'W'`=0x57, `'S'`=0x53), or (b) written by code reached only through
a resource/ID (e.g. a template blob fetched by `FindResourceA`, which leaves no address xref).

Also note: these early scans were weaker than they looked — an initial "decode all `.text` with
capstone `detail=True`" pass silently returned 0 instructions (dict-unpack reversed the section
bounds) and a full-detail decode then timed out, so **byte-immediate stores of `'F'`/`'W'`/`'S'`
were never reliably enumerated** until now.

## 14. Verified false lead

`0x84c580` does **not** contain a file-position bit-masking writer. It lies inside
`FUN_0084c570`, a 76-byte stub that builds `{byte0=0, byte1=0x0c, word=0x8b}` on the stack and
calls `FUN_00838e90` — no file I/O, no bit math. `OpenFile` is not imported by this binary at all
(`KERNEL32` provides `CreateFileA`/`ReadFile`/`WriteFile`; `WINMM` provides the `mmio*` family).

## 15. Method warning: substring matching on `op_str` lies

Searching capstone's `op_str` for `", 0x43"` also matches `", 0x4303c5"`, which inflated a
byte-immediate hunt to 455 bogus "hits" (e.g. `mov eax, 0x4303c5`, `add esp, 0x434`). Any scan of
this kind must split `op_str` on `", "` and compare **whole tokens**, not substrings. Earlier
mistakes in the same family: a dict-unpack that reversed a section range (silently 0 instructions)
and a `detail=True` full-`.text` decode that exceeded the timeout. Byte-level pattern scans
(`C1 Ex 06`, `FF /2 38`, `46 57 53 00`) are the only cheap-and-trustworthy primitives found so far.

---

## FOUND: the SWF writer

> The consolidated API reference is on the [writer API page]({{ '/writer/' | relative_url }}) (writer
> struct layout, primitives, `beginTag`, tag-code table, undocumented behaviour). This page remains
> the chronological hunt log.

## 16. How it was located

An exact-token sweep of every `mov`/`push` whose last operand is *exactly* `0x46`, `0x57` or `0x53`
produced 64 sites. Looking for an `F` then `W` then `S` within 0x80 bytes left **exactly one**
candidate:

    0x7a01ed push 0x46   ('F')
    0x7a01f6 push 0x57   ('W')
    0x7a01ff push 0x53   ('S')

(9 bytes apart each — the spacing is the intervening `mov ecx,esi; call putbyte`.)

## 17. `FUN_007a01c0` — the SWF header writer (200 bytes)

```c
void __thiscall FUN_007a01c0(this, rect /*param_2*/, fps /*param_3*/,
                            version /*param_4*/, compressWanted /*param_5*/)
{
    c = (param_5 != 0 && (byte)param_4 >= 6);      // CWS requires SWF 6+
    this->[0x2d8] = c;                             // "compressed" flag
    putbyte(c ? 'C' : 'F');                        // 0x46 / 0x43
    putbyte('W'); putbyte('S'); putbyte(version);
    this->[0x234] = tell();                        // FileLength field offset
    put_dword(0);                                  // placeholder FileLength
    this->[0x2dc] = tell();                        // == start of compressed body
    put_rect(rect);                                // FrameSize
    put_word_hilo(fps >> 8); put_word_hilo(0);     // FrameRate (UI16, 8.8)
    this->[0x238] = tell();                        // FrameCount field offset
    put_word_hilo(0);                              // placeholder FrameCount
    this->[0x2bc] = fps;  this->[0x23c] = 0;  this->[0x240] = version & 0xff;
}
```

Helper thunks resolved to real bodies (all verified by disassembly, not inference):

| role | thunk | body | verified behaviour |
|---|---|---|---|
| `putbyte` | `0x4070fe` | `0x79f6c0` | `buf[this+8][count] = b; count = [this+0x208]+1`; if `count+1 > 0x200` call `0x4128b4` to flush. **512-byte output buffer** |
| `tell` | `0x40b721` | `0x79f4f0` | returns absolute file position ≈ `[0x208] + bytes already flushed` |
| `put_dword` | `0x417de6` | `0x79f780` | little-endian 32-bit store into the same buffer |
| `put_word` | `0x4181d8` | `0x79f710` | writes `al` then `ah` (UI16 little-endian), reserving 2 bytes |
| `put_rect` | `0x4110ef` | `0x79f910` | reads the 4 rect ints (`+0,+4,+8,+0xc`), `abs()`s Ymax, takes min/max via `0x41c89b`, then `push 5` — emits the **5-bit `Nbits`** field then the four `SB[Nbits]` values |
| `compress` | `0x40b753` | `0x5ba930` | `deflateEnd` |

This settles the `FrameRate` question raised in [the format reference]({{ '/format/' | relative_url }}) §3: `put_word(fps >> 8)` writes
`0x0C 0x00` for 12 fps — integer part in the low byte, zero fraction in the high byte — and the
`tell()` recorded at `+0x238` sits **between** that and the `put_word(0)` that is the `FrameCount`
placeholder, exactly as the spec requires.

### Writer object layout (recovered)

```
+0x008 .. +0x207   512-byte output buffer
+0x208             buffer fill count           (flush via 0x4128b4)
+0x234             offset of FileLength field  (4 bytes to back-patch)
+0x238             offset of FrameCount field  (2 bytes to back-patch)
+0x23c             0 (init)
+0x240             SWF version byte
+0x2bc             frame rate, raw 8.8 fixed
+0x2d8             compressed flag (0 = FWS, 1 = CWS)
+0x2dc             offset of end-of-header == start of body (deflate input start)
```

This is the definitive proof of the format model in [the format reference]({{ '/format/' | relative_url }}) §3: signature, version,
`FileLength` **saved as a position for later back-patching**, then `FrameSize`/`FrameRate`/
`FrameCount` with the frame count likewise saved at `+0x238`.

## 18. `FUN_007a03e0` — the CWS compressor (361 bytes)

```c
len   = file->GetLength();                       // vtable slot 0x30
body  = len - this->[0xb7*4];                    // 0x2dc = header/body split
buf   = malloc(body);
file->Seek(this->0x2dc, 0); file->Read(buf, body);
z_stream zs; deflateInit_(&zs, Z_DEFAULT_COMPRESSION, s_1_0_4, sizeof(z_stream)=0x38);
deflate(&zs, 4);                                 // Z_FINISH
deflateEnd(&zs);
if (zs.total_out < body) { file->Seek(0x2dc,0); file->Seek(0,2/*END*/); file->Write(compressed, total_out); }
else                     { byte b='F';        file->Seek(0,0);                  file->Write(&b,1); }
```

The `else` branch is the tell-tale: when deflate does not shrink the body, the writer flips the
signature byte from `'C'` back to `'F'` — producing an uncompressed file. `s_1_0_4` at `0xbc37dc`
is **zlib's `ZLIB_VERSION` string**; zlib **1.0.4 is statically linked**, which resolves the earlier
puzzle that `Flash.exe` imports no `zlib1.dll` yet obviously compresses.

## 19. Consequence for the remaining hunt

The writer object is a plain **procedural struct** (a file handle + state at `+0x234`, `+0x238`,
`+0x23c`, `+0x240`, `+0x2bc`, `+0x2d8`, `+0x2dc`), not a C++ class — consistent with the absence of
any writer class in RTTI. Tag emission is therefore reached through `putbyte`/`put_dword` helpers on
this object, and the tag layer should be found by walking **callers of `0x79f6c0` (putbyte)** and the
`0x2d8`/`0x2dc` state users rather than by any vtable reasoning.

## 20. `FUN_007a0340` — the finaliser / back-patcher (115 bytes)

```c
put16(0);                                        // reserve FrameCount
if (this->[0x234] >= 0) {
    len = tell();                                // real FileLength
    seek(this->[0x234]);  put32(len);            // patch FileLength
}
seek(this->[0x238]); put16(this->[0x23c]);       // patch FrameCount
flush();                                         // 0x79f670
if (this->[0x2d8]) compress();                   // FUN_007a03e0, only for CWS
close_thing();                                   // FUN_007a0180
```

Note the endianness detail: `put32`/`put16` write little-endian, but the *header count* goes through
`put16(CONCAT22(...))`, and the deflate input start (`+0x2dc`) is fixed at header end, so the
compressed stream replaces the body **in place** while the header stays literal.

## 21. Tag-layer helpers identified

| function | size | role | evidence |
|---|---|---|---|
| `FUN_0079f8c0` | 61 | variable-width integer: emits 3 bytes of the value, 4th only when its flag byte is non-zero | `putbyte(v); putbyte(v>>8); putbyte(v>>16); if (flag) putbyte(v>>24)` |
| `FUN_007a1060` | 1631 | **fill-style array** writer | emits `count`, and when `count >= 0xFF` emits `0xFF` then a UI16 — the exact `FillStyleCount`/`FillStyleCountExtended` rule of [the format reference]({{ '/format/' | relative_url }}) §6; per-entry type bytes are `0x00`, `0x10`, `0x12` (= solid, linear gradient, radial gradient) |

`FUN_007a1060` de-duplicates styles by identity (`+0x28`/`+0x2c` caching per entry) and drives
`0x79f9d0` for the gradient `MATRIX` — so the shape serialiser is procedural and starts here.

## 22. Caller census (Ghidra, 53 151 functions scanned)

61 functions drive the writer helpers. Sorted by output volume the leading emitters are
`FUN_00656770` (46 bytes / 56 words / 5 rects, touches `+0x208`, `+0x234`, `+0x240`),
`FUN_007a1060` (21 bytes), `FUN_007a1cb0`, `FUN_007a3470`, `FUN_007a29a0`, `FUN_0079f8c0`.
Only two functions touch `+0x2d8`/`+0x2dc`: `FUN_007a01c0` (header writer) and `FUN_007a0340`
(finaliser) — confirming the tag layer never inspects the compression state.

## 23. `FUN_007a1860` — the DefineShape tag writer (872 bytes)

This is the first confirmed **tag** emitter, and it shows how tag framing is actually done here:

```c
walk display list -> transform each entry's matrix (0x99e7f0, 0x9a34c0, 0x9a31b0 = concat/translate)
putRect(shapeBounds);                        // local_98
putRect(edgeBounds);                         // local_88   (two rectangles)
lenPos = tell();
put32(0);                                    // tag length PLACEHOLDER
fillStyleArray(styleList, &nFills, &nLines); // FUN_007a1060
for (pass = 0; pass < 2; pass++) {
    if (pass == 1) { end = tell(); seek(lenPos); put32((end - lenPos) - 4); seek(end); }
    lineStyleArray(pass ? nFills : 0, pass ? nLines : 0);   // FUN_007a06d0
    ... shape records for the display list ...
}
```

So the tag code byte is written by the caller *before* entering this function, the 4-byte length is
reserved and later back-patched by arithmetic on `tell()`, and the `-4` accounts for the length field
itself. The two-pass loop emits the shape once for each of two fill/line style sets, selected by
`~-(uint)(pass != 0) & count` — i.e. `0` on the first pass, the real count on the second.

## 24. Confirmed graph so far

```
export driver (one of 0x64e7e0 / 0x65f57b / 0x67d8a0 / 0x7b01d2 / 0x809800 family)
   |-- FUN_007a01c0  header writer      (FWS/CWS + version + FileLength/FrameCount slots)
   |-- FUN_007a1860  DefineShape        (bounds, fill/line styles, shape records)
   |      |-- FUN_007a1060  fill-style array (0x00/0x10/0x12 + 0xFF escape)
   |      |-- FUN_007a06d0  line-style / shape-record emitter
   |      |-- FUN_007a0ff0  matrix helper (gradient + placement matrices)
   |      +-- putRect / put32 / tell  (0x79f910 / 0x79f780 / 0x79f4f0)
   +-- FUN_007a0340  finaliser          (patch FileLength + FrameCount, flush)
          +-- FUN_007a03e0  CWS compressor (zlib 1.0.4, statically linked)
```

## 25. `FUN_00656770` — the tag emitter (12 111 bytes)

This was the object of the whole hunt. `0x656eb9` (the sole caller of the `DefineShape` writer
`FUN_007a1860`) resolves *inside* `FUN_00656770`, and an independent Ghidra census had already flagged
this function as the heaviest writer-helper consumer: **46 `putbyte`, 56 `put16`, 5 `putRect`**, plus
direct calls to `put32`, the variable-width writer, the matrix writer and the finaliser.

Its callee set is the export vocabulary: `0x7a1860` (DefineShape), `0x7a1060` (fill-style array),
`0x7a06d0`, `0x7a0a60`, `0x7a0d20`, `0x7a0ff0` (matrix), `0x7a02d0`, `0x7a0070`, `0x7a25f0`,
`0x7a2280`, `0x7a4390`, `0x7a5630`, `0x7a1cb0`, `0x79fe50/0x79fe90/0x79ffa0`,
`0x79f8c0`/`0x79f6c0`/`0x79f710`/`0x79f780`/`0x79f910`, `0x7a0340` (finaliser), and
`GlobalAlloc`-free helpers (`GlobalFree`, `GlobalUnlock`).

Small immediates inside the function, as candidates for tag codes:

| imm | count | first site | reads as |
|---|---|---|---|
| `0x39` | 1 | `0x6567de` | `DefineSprite` |
| `0x20` | 1 | `0x656d15` | near `0x656d1b` (`0x16`, DefineButtonCxform) |
| `0x26` | 1 | `0x659113` | `PlaceObject2` |
| `0x32` | 1 | `0x657bfe` | `DefineShape3` |
| `0x0a` | 1 | `0x657db4` | `DefineFont` |
| `0x0b`/`0x0d` | 3 | `0x657773` | `DefineText` / `DefineFontInfo` |
| `0x0c` | 1 | `0x657ddc` | `DoAction` |
| `0x0e`/`0x0f` | 1/1 | `0x657e7c`/`0x657eef` | `DefineSound` / `StartSound` |
| `0x12` | 2 | `0x657bf0` | `DefineButtonSound` |
| `0x14` | 2 | `0x65902a` | `DefineBitsJPEG2` |
| `0x16` | 2 | `0x656d1b` | `DefineButtonCxform` |
| `0x17` | 2 | `0x658d13` | `Protect` |
| `0x1a` | 1 | `0x657c1c` | `SoundStreamHead` |
| `0x2b` | 1 | — | `FrameLabel` (43, per [the format reference]({{ '/format/' | relative_url }}) line 139) |

**Caveat, stated deliberately:** a small immediate is only a *candidate* tag code. The values
`0x10`, `0x11`, `0x27`, `0x45`, `0x38`, `0x3c`, `0x13`, `0x1d`, `0x1e`, `0x25`, `0x2a`, `0x2e` also
appear and are probably **not** tag codes (array strides, enum members, buffer sizes). Confirming any
row needs its surrounding code read — the cluster at `0x656d15`–`0x656d1b` and the `DefineSprite`
site `0x6567de` are the cheapest first checks.

Relation to the format model: [the format reference]({{ '/format/' | relative_url }}) §4 predicted that tag framing is a bit-field
(`code << 6 | shortLen`) and that the back-patch is unavoidable. Both are confirmed here — the code
byte comes from these immediates, the length is reserved with `put32(0)` and patched as
`(tell() - lenPos) - 4` in `FUN_007a1860`.

---

## 26. API mapping: corrections, and what 38/40/49/51/52 really are

New probes: `CtxScan.java` (disassembly window before every reference to a target),
`FnDump.java` (linear dump), `BigDecomp.java` (decompile with no size gate).
Outputs: `.tools/api1..5.mark`.

- The `beginTag` thunk `0x413afc` has **62** call sites; the 63rd reference is the thunk's own `jmp`.
- **`endTag` exists** — body `0x79f5b0`, thunk `0x4123a5`. It computes `code<<6 | len`
  (`| 0x3f` + UI32 for long form) and clears `[+0x214]`. The earlier claim that no such arithmetic
  exists anywhere in the binary was wrong; it lives in the closer, not the opener.
- `putstr` `0x79fe50` / thunk `0x40f1b4` = strlen + raw copy + `NUL`; `putmem` `0x79f820` = raw blob.
- `0x7a0180` (previously undecompiled) is a 46-byte release helper for `[+0x230]`, called by both the
  finaliser and teardown.
- The `0x7a4390..0x7a45cc` cluster are small emitters **with thunks** (`0x409dd1`, `0x4107d0`,
  `0x414b82`, `0x4194e3`, `0x419aa1`, `0x418070`), so their callers are findable after all:
  codes 40, 43 (two variants), 49, 51, 52.
- Code 43 = `FrameLabel`, always long-form; one variant appends `UI8 1` (`namedAnchor`).
- Code 63 = 16 raw bytes, written by the publish driver `FUN_008084b0` when the debug flag is set.
  Confirmed in `puzzle.swf` (body `7f6a765d0d2311d6af270050e450cf68`) immediately after
  `EnableDebugger`. The same function writes `FWD\x06` + UI32 + the same 16 bytes to a separate file
  object — the `.swd` companion.
- Corpus check (62 SWFs: 33 CWS v6, 25 FWS v5, 4 FWS v6): codes 38/40/49/51/52 occur **zero** times,
  63 exactly once. The four unassigned codes are movie-writer paths the shipped samples never hit.
- The "indirect" `beginTag` sites are mostly alpha promotion: `0x14 + (flag?0x10:0)` (20/36) and
  `0x15 + (flag?0x0e:0)` (21/35).
- Correction: code 19 is `SoundStreamBlock`, not `DefineBitsLossless`.

---

## 27. The dispatch, read — `IsKindOf` class chain and tag promotion

Method note: Ghidra's temp space was exhausted (`/tmp` at 100 %), so this round used a capstone
linear sweep of `FUN_00656770`'s byte range (`0x656770..0x6596bf`) instead. Outputs:
`.tools/dispatch.txt`, `.tools/dispatch2.txt`, `.tools/classmap.txt`. Cross-check: the sweep's
`put16` 56 / `putbyte` 46 / `putrect` 5 counts match the Ghidra census of §25 exactly, so the sweep
is sound. `endTag` 14, `beginTag` 16, `putmem` 5, `putstr` 9, `tell` 15; `put32` and `flush` read 0
because a primitive can have several thunks and the sweep only matched one address each.

- `0x8eb3b9` is `IsKindOf(object, &descriptor)`. Descriptor first dword -> class-name string.
  Eleven tests in the function: `CPicShape`, `CPicMorphShape`, `CCommandObj`, `CPicText`,
  `CPicSprite`, `CDocumentPage` (x2), `CPicButton`, `CMediaBits`, `CMediaVideo`, `CPicVideoStream`.
- `CMediaVideo` -> tag 38. `CPicVideoStream` -> tag 60, gated on version > 5; at <=5 the chain raises
  diagnostic `0x1e` through `FUN_006fa8d0` instead of emitting.
- Tag promotion is decided at the call site: `2`/`0x20`/`0x16` for shapes, `0x0b`/`0x21` for text
  (`AND EDX,0x16; ADD EDX,0x0b`), `7`/`0x22` for buttons (`AND EDX,0x1b; ADD EDX,7`), plus the
  alpha pair of §26 in the publisher. This is why immediate-only censuses undercount the tag set.
- Code 38's body is now known: `UI16`, `STRING` written with `putmem` (no NUL), `UI8` in {0,1,2}.
  It is the only tag body in this writer whose name field is not NUL-terminated.
- New corpus tool `.tools/swftags.py` (validated: 62 files, one tag-63 hit in `puzzle.swf`).

## 28. Emitters decompiled: 40/49/51/52 are dedicated, and two corrections

Ghidra became usable again once `/tmp` was relieved (a project-local `TMPDIR=.tools/tmp` keeps scratch
on the 800 G disk instead of the 3.9 G tmpfs). `BigDecomp` ran on the whole of `FUN_00656770`
(12 111 bytes, 1449 decompiled lines → `.tools/emit56770.out`), on the four small emitters
(`.tools/emit4052.out`) and on the helpers (`0x5408c0`, `0x6fa8d0`, `0x6511f0`, `0x6843f0`).

New tools: `.tools/pairdesc.py` (control-flow walk from each `IsKindOf` test to its first tag call),
`.tools/switch49.py` (jump-table resolver), `.tools/thunk.py` (`E9` thunk resolver), `.tools/win.py`
(disassembly window).

- **`FUN_007a4390` is code-40 only.** `beginTag(0x28,1); put16(a); putstr(name); put16(b); endTag()`.
  No code parameter, no guard — so the `[esp+0x103c]` marker-byte theory of §26/§27 is dead: the tag
  code is never taken from a local.
- **`FUN_007a4460` (49)** writes `put16(packed & 0xf)`, then `lea eax,[edi-1]; cmp eax,7; ja tail`
  and an 8-entry table at `0x7a44ec`: disc 1,2 → `put16(param_3)`; 3 → none; 4 → three (`param_3`,
  `packed & 0xf0`, `packed >> 16`); 5,6,7 → none; 8 → `put16(param_2)`; 0 or >8 → none. The §26
  reading ("1 for 1,2,8; 3 for 4; 1 by default") was wrong in two places.
- **`FUN_007a4580` (52)** = `if (strlen(name)) { beginTag(0x34,1); put16(param_3); putstr(name);
  put16(param_2); endTag(); }` — empty string suppresses the tag entirely.
- **`FUN_007a4540` (51)** = `beginTag(0x33,1); put16(a); put16(b); endTag()`.
- **Version gate direction corrected.** `0x6591d2 cmp [ebp+0x110],5 / jle 0x65927a`;
  `0x65927a push ebx; push 0x1e; call 0x40c342` → `0x6511f0` → `FUN_00650af0`. So below v6 the
  video-stream path reports `0x1e` and writes nothing (the claim stands), but the reporting function
  is `0x6511f0`, not `0x6fa8d0`. **`0x6fa8d0` is a 7-byte getter** (`return [ecx+0x94]`), also
  thunked at `0x401f5a` and inlined at `0x65929f`.
- **DefineVideoStream (60) body recovered**: `put16(id)`, `put16(([x+0x138]-[x+0x134])>>4)`,
  `put16([x+0x100])`, `put16([x+0x104])`, `putbyte(0)`, `putbyte([x+0xd8] mapped 1→1, 2→2, else 0)`.
- **`ExportAssets` (56) is gated on `[obj+8] == 1`** (`0x4107df` → `0x5408c0`, 12 bytes).
- **Class→site pairings by control flow**, not adjacency: shape→`0x656d20`, morph→`0x656eb6`,
  `CCommandObj`→`0x656fe6`, text→`0x657178`/`0x657bba`, sprite+page→`0x659369` (shared handler at
  `0x659289`), button→`0x658b2b`, bits→40, video→38, video-stream→60.

## 29. Callers traced: the diagnostic formatter, the frame writer, and the publish driver

`BigDecomp` takes one address per `-postScript`, so passing two numbers in one argument silently
decompiles only the first; repeats of `-postScript BigDecomp.java <addr>` are the way to get several
(see `.tools/emit_trace2.out`, `.tools/emit_frame.out`).

- **`FUN_00650af0` (1238 bytes) is a diagnostic message formatter, not an abort.** Gated on
  `DAT_00c155f8 != 0`, it classifies the object (`CPicObj`, `CMediaBits`, `CMediaElem`, `CPicFrame`,
  `CPicLayer`) to build a "where" clause, formats frame/layer positions (`0x3cc`, `0x3cd`, `0x3ce`,
  `0x3cf`, resource `1000`), then maps the error code through a switch — `0x1e` → `0x5cc` — and loads
  that string. So the v6 video-stream refusal is a user-visible warning; with messages disabled the
  object is dropped silently.
- **`FUN_00652e00` (5599 bytes) is the frame/timeline writer.** It walks the layer list at
  `[doc+0x50]`, pairs layers by a `short` at `+0x62`, gates frame iteration on
  `[doc+0x110] > 2`, and emits tag 49 from six sites with `([obj+0x160])` (frame count) or with a
  copied `UI16` from `+0x62`. Frame labels are written from `[frame+0x1d4]` via `0x7a4420`/`0x7a43e0`.
- **`FUN_007a43e0` (44 B) / `FUN_007a4420` (35 B) are the FrameLabel pair**, both `beginTag(0x2b,1)`;
  the larger adds `putbyte(1)` (the v6 `namedAnchorFlag`). Selector is `[frame+0x1e4] != 0`, *not* the
  version — §5.9's "always claims a named anchor" was wrong and has been corrected.
- **`FUN_008084b0` (2724 bytes) is the publish driver.** It derives the companion path from the movie
  name (`.swf` → `.swd`, else append `.swd`), opens it as a `CStdioFile`, writes `"FWD"`+`0x06`, the
  UI32 and the second 16-byte copy there, and emits tag 51 once behind `[doc+0x54] != 0` with
  `GetSystemMetrics(0x2a) != 0` as its argument. Metric `0x2a` is 42 = `SM_DBCSENABLED`, so the tag
  records that the movie was compiled on a double-byte (CJK) system; it precedes the
  `0x6ed2d0`…`0x6eed00` font-table run.
- **Tag 49's discriminator is data.** All nine sites are in those two functions; eight pass a literal
  or copied value whose low nibble is not fixed by the code, and the three publish-driver sites pass
  `(CString, 0, 0)`, i.e. the "no numbers" form. The 1–8 payload table is unreachable without objects
  that carry those nibbles.
