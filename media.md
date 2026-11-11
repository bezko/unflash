---
layout: default
title: Video & audio
permalink: /media/
description: What Flash MX does with sound and video — import formats, bundled codecs, and the tags it writes.
---

What the authoring tool actually does with sound and video: the file formats it opens, the codecs
bundled in the binary, and the SWF tags it emits for them.

Target: `extracted/InstallShield/App_Files/Flash.exe` (PE32, sha256 `64280f22…d9510da`, imagebase
`0x400000`). Addresses are VAs. Observations come from strings, disassembly and decompilation;
anything else is labelled inference. Companions: [the format reference]({{ '/format/' | relative_url }}), [the writer API reference]({{ '/writer/' | relative_url }}),
[the ActionScript notes]({{ '/actionscript/' | relative_url }}), [the hunt log]({{ '/hunt-log/' | relative_url }}).

---

## 1. Three layers, and the evidence for each

"Support" splits into three independent questions, and this build answers them differently:

| Layer | Where it lives | How it was read |
|---|---|---|
| **File I/O** — what can be opened and saved | resource strings (`.rsrc`), Win32 file dialogs | filter strings `Name (*.ext)\|*.ext` |
| **Codecs** — what can be decoded/encoded | `.rdata` class names, third-party source paths, error text | RTTI descriptors, UTF-16 resources |
| **SWF output** — what reaches the movie | tag emitters in `.text` | disassembly at each `beginTag` call |

The three do not agree. The import layer accepts `*.avi`, `*.mov`, `*.mpg`, `*.dv` and `*.flv`, but
the writer has exactly one video codec path, and the streaming-sound head is written by an emitter
whose tag number is computed rather than literal (§3.6).

---

## 2. Summary

| Capability | Status | Evidence |
|---|---|---|
| Import audio: WAV, AIFF, N.AIFF, MP3 | yes | `WAV Audio (*.wav)`, `AIFF Sound (*.aiff)`, `N.AIFF Sound`, `MP3 Sound (*.mp3)` |
| Import video: FLV, AVI, MOV, MPEG, DV | yes | five filter strings, §4.1 |
| Export audio file | yes — WAV only | `Export Windows WAV…` |
| Export movie as video | yes — AVI, QuickTime | `Export Windows AVI…`, `Export QuickTime…` |
| SWF sound compression | ADPCM 2/3/4/5-bit, MP3 8–160 kbps, none | sound-export option list §3.1 |
| SWF video codec | H.263 (Sorenson Spark) only | `CVideoImport::H263Player`/`H263Consumer`; no VP6 strings |
| Video in SWF below version 6 | refused, object dropped | `cmp [ebp+0x110],5` at `0x6591d2` |
| MP3 in SWF outside 6–17 | refused | `0x7a2f48`, `0x7a2f4d`; budget table ends at 17 |
| Streaming sound head tag (18/45) | yes — one emitter, tag computed | `FUN_007a3470`, §3.6 |
| Camera / microphone capture | no authoring path | player-side natives only, §5.5 |

---

## 3. Audio

### 3.1 The options the UI offers

Two identical option blocks sit in `.rsrc` (each label list appears twice, presumably for the event-
and the stream-sound panels):

```
None
2 bit ADPCM      3 bit ADPCM      4 bit ADPCM      5 bit ADPCM
8 kbps MP3      16 kbps MP3      24 kbps MP3      32 kbps MP3
48 kbps MP3     56 kbps MP3      64 kbps MP3      80 kbps MP3
112 kbps MP3   128 kbps MP3     160 kbps MP3
5kHz Mono / 5kHz Stereo / 11kHz Mono / 11kHz Stereo
22kHz Mono / 22kHz Stereo / 44kHz Mono / 44kHz Stereo
```

plus the checkbox **"Use imported MP3 quality"** (`0xf03ea0`), which is the passthrough switch: an
imported MP3 can be copied into the movie without re-encoding. The dialog templates next to these
strings (`%i kbps MP3`, `%i bit ADPCM`, `Mono`, `Stereo`, `5KHz`…`44KHz` at `0xbce9a8`) are the
summary line the sound properties panel renders.

Note the rates: 5, 11, 22, 44 kHz. There is no 8 kHz and no 48 kHz in the export set.

### 3.2 A full ISO MPEG Layer-3 encoder is statically linked

The `.rdata` source paths name the encoder's own tree:

```
C:\proj\flash4\mp3encoder\core\bitmux\enc\l3\bit_fram.c
C:\proj\flash4\mp3encoder\wrappers\enc\l3\encifc.c
C:\proj\flash4\mp3encoder\core\psych\enc\l3\psymain.c
```

That is a Layer-3 encoder with a bit reservoir (`bitmux`) and a psychoacoustic model (`psych`) —
built from a **Flash 4-era** source tree, so the MP3 encoder predates Flash MX by several releases
and was carried forward unchanged. Its diagnostics survive as strings:

```
bitrates > 320000 not supported by ISO MPEG Layer-3
bitrates < 8000 not supported by ISO MPEG Layer-3
non-ISO bitrate          bitrate too high          bitrate too low
only mono and stereo case implemented
mismatch between encoder interface version and client version
```

The 8–160 kbps list in §3.1 is a strict subset of the 8–320 kbps the encoder itself accepts, so the
UI, not the codec, is what caps the bitrate.

There is a decoder too: `(mp3dec) error: invalid handle` and `(Mp3Ssc) warning…` (`0xee5c00`), next
to the `Sound Mapper` device string — the playback side of the authoring tool, which is why MP3
files can be auditioned without re-encoding.

**Inference:** "statically linked" is supported by the absence of any MP3/encoder DLL in the import
table combined with these in-image paths, not by a direct observation of the linker.

### 3.3 `DefineSound` (14): `0x7a2ed0`

The emitter is a `thiscall` on the writer; `ecx` is the writer, `[ebp+8]` the character id,
`[ebp+0xc]` the sound object, `[ebp+0x14]` the SWF version.

```
0x7a2ef3  push 0x1000 ; call 0x8e657c        allocate 4 KiB scratch
0x7a2f15  push 1 ; push 0x0e ; call beginTag tag 14, long header
0x7a2f26  put16([ebp+8])                     character id
0x7a2f2e  esi = [obj+4]                      packed format byte
0x7a2f33  ecx = esi & 0xf0 ; cmp cl,0x20     SoundFormat nibble == 2 ?
0x7a2f48  cmp version,6 / jl  -> non-MP3     MP3 needs SWF 6+
0x7a2f4d  cmp version,0x11 / jg -> non-MP3   ... and stops at SWF 17
```

The MP3 branch then writes

```
putbyte(flags)          ; format|rate|size|type, as passed in
put32([obj+8])          ; SampleCount
put16(0)                ; SeekSamples = 0, always
<data block>            ; via 0x412661 (length) + 0x40c7b1 (putmem-like)
```

Three observations worth keeping:

- **`SeekSamples` is hard-wired to zero.** A re-encoder would need it; this one never sets it.
- **The MP3 version window is `[6, 17]`**, both ends enforced — and it is not arbitrary: the writer
  reads its bitrate budget from `[version*4 + 0xaf8888]`, a table whose entries run from version 6
  to version 17 (§3.4). A higher version would index past its end.
- **The format dispatch is a nibble test, not an enum compare**: `(flags & 0xf0) == 0x20`. The low
  nibble (rate/size/type) rides along untouched.

### 3.4 ADPCM depth is computed, not chosen

The non-MP3 branch (`0x7a2f89` onward) does arithmetic before it writes anything:

```
cmp [writer+0x240],4          ; the writer's SWF version field (see the writer API reference, §2)
cmp version,0x11
mov edx,(flags>>2)&3          ; SoundRate, 2 bits
cmp [edx*4+0xaf7f8c],0x1588   ; rate table; 0x1588 = 5512 = the 5.5 kHz entry
mov eax,[edx*4+0xaf7f8c]      ; rate value
mov ebx,[version*4+0xaf8888]  ; bitrate budget for this SWF version
lea ebx,[ebx+ebx] ; idiv      ; 2 * bitrate / (rate * channels)
cmp eax,5 ; ja  -> depth 5
jmp [eax*4+0x7a3334]          ; cases 2,3,4,5
```

So the ADPCM bit depth (2–5 bits) is derived from the sample rate, the channel count and a
per-version bitrate budget, then passed to the ADPCM writer at `0x418b83`. Both tables are now read
as data rather than inferred:

```
0xaf7f8c  SoundRate -> Hz     5512  11025  22050  44100
0xaf8888  SWF version -> bps  v6 8000   v7 16000  v8 20000   v9 24000  v10 32000  v11 48000
                              v12 56000 v13 64000 v14 80000  v15 112000 v16 128000 v17 160000
                              (v0–v5 are zero)
```

The quotient is `2 × bitrate(version) / (sampleRate × channels)` — a bits-per-sample figure with a
factor of two in it — and the six-entry jump table at `0x7a3334` maps it to a depth: index 0 and 2 to
2-bit, 3 to 3-bit, 4 to 4-bit, 1 and 5 to 5-bit, with anything above 5 clamped to 5. **Inference:**
the doubling reads like deliberate headroom, but a units error would look identical here; what is
certain is that the depth comes from the rate and the version, and never from the dialog's dropdown
value. The 5.5 kHz entry is additionally tested on its own (`0x7a2fb6`), and when the rate is not
5512 Hz and the writer's header version is ≥ 4, control leaves the arithmetic path entirely.

### 3.5 `StartSound` (15): `0x653e07`

```
if ([ebp+8] > 0) {                       sound id
  beginTag(0x0f, 1); put16(id);
  eax = [obj+0x174] + 0xb8;              SoundInfo source fields
  edx = [eax+8]; eax = [eax+4];
  eax = (eax >> 2) & 3;
  ecx = [eax*4 + 0xaf7f9c];              shift table
  shl edx, cl;  cmp ebx, edx;            loop-count / style packing
}
```

The `SoundInfo` flags are packed by shifting through a 4-entry table at `0xaf7f9c`, i.e. the
SyncType/SyncStop/loop-count byte is assembled bitwise. `DefineButtonSound` (17) has its own site at
`0x658d93`.

### 3.6 Streaming: a computed head, five block emitters

Tag 19 `SoundStreamBlock` is emitted from five places, all in the `0x7a38f0`–`0x7a3f20` range:

| site | function | what it writes |
|---|---|---|
| `0x7a390d` | `FUN_007a38f0` | payload size from `[obj+0x24c]` (or `rate × channels`), then the block data |
| `0x7a3a15` | `FUN_007a3a10` | **empty block** — `put16(0); put16(0)`, then `endTag` |
| `0x7a3ab1` | `FUN_007a3a70` | running frame counter at `[obj+0x2c4]`, sample offset at `[obj+0x2cc]`, then data |
| `0x7a3dc5` | `FUN_007a3dc0` | opens the tag and parks a back-pointer at `[obj+0x1c]` for a later fill |
| `0x7a3f15` | `FUN_007a3f10` | `beginTag(0x13,1)` immediately followed by `endTag` |

All five are called from **one** function, `FUN_00652e00` — the frame sound compiler, which also
performs the version refusals of §3.7.

The head, which the previous round reported as missing, is **`FUN_007a3470`**, and its tag number is
computed from an argument rather than pushed:

```
mov eax,arg                  ; tag selector
neg eax ; sbb eax,eax        ; 0 stays 0, non-zero becomes -1
and al,0xe5                  ; -1 & 0xe5 = 0xe5
add eax,0x2d                 ; 0x2d = 45, or 0xe5 + 0x2d = 0x12 = 18
call beginTag                ; long-form flag = 0, so the short header
```

A zero selector writes **tag 45 `SoundStreamHead2`**; a non-zero selector writes **tag 18
`SoundStreamHead`**, the older form without MP3 fields. That is the split the SWF 6 format
description makes, arrived at by arithmetic instead of two code paths — which is why a scan for the
literals `0x12` and `0x2d` never found it.

The body then branches on a codec code at `[+0x27c]`: 2–5 are the ADPCM depths from §3.4; 6–17 index
the MP3 quality ladder (`thunk_FUN_0067b2a0` is called with `code - 6`); and anything above 17 takes
a third path that writes the format byte `0x60` — or `0x50` when the code is exactly 19 — the ids the
SWF 6 speech format occupies.

Its only caller is in `FUN_00652e00` (`0x653026`), and it does **not** pass zeros: the decompiled call
is `FUN_007a3470(FUN_0077cd20(rateA, rateB), &config, code)`, where `FUN_0077cd20` is a 63-byte merge
of two rate fields — `(a | b) & ~0xc | max(a & 0xc, b & 0xc)`, keeping the higher `SoundRate` bit-pair
— and `&config` is a stack struct filled immediately before. The all-zero argument sequence I read in
the raw disassembly (`push ebx` three times) belongs to a fallback branch that `jmp 0x653022`
short-circuits; the decompiler shows the arguments that actually reach the call. The head's
`if (config == NULL) putbyte(0)` branch therefore fires only when that config is genuinely absent.

So streaming sound is written head-and-blocks; the open question is narrower than before — not "is
there a head" but "does the merged flags word produce a head consistent with the blocks". None of the
62 shipped movies contains tag 18, 19 or 45, so there is still no sample to check it against.

### 3.7 Version refusals are user-visible strings

```
"MP3 sound codec not supported in SWF versions less than 4"
"Speech sound codec not supported in SWF versions less than 6"
```

The second names a codec the binary never calls by name: **"Nellymoser" has zero occurrences** in
either encoding. The shipped help credits page does name it — `Help/Flash/html/90cred2.html`,
"Speech compression and decompression technology licensed from Nellymoser, Inc." — so the codec is
present and licensed, just never named in code. In this build the SWF 6 speech codec is only ever "Speech sound", and its message
gates it below version 6 like MP3.

Note the disagreement in the first message: it says **4**, while the `DefineSound` gate at `0x7a2f48`
is **6**. The 4 belongs to the writer's own header version at `+0x240`, which the non-MP3 branch
tests first (`0x7a2fa0`) — two different notions of "version" meet in that one function, and the
string records the older one.

---

## 4. Video

### 4.1 Import pipeline: DirectShow plus QuickTime

The RTTI block at `0xbe4e08`–`0xbe5188` is a `CVideoImport` namespace:

```
.?AVCBaseFilter@@            .?AVCBaseRenderer@@         .?AVCMediaGrabber@@
.?AUIMemInputPin@@           .?AUIQualityControl@@       .?AUIPin@@
.?AVCBasePin@@               .?AVCBaseInputPin@@         .?AVCRendererInputPin@@
.?AVCMediaGrabberRenderInputPin@@
.?AVResampler@CVideoImport@@     .?AVQuickTime@CVideoImport@@
.?AVDirectShow@CVideoImport@@    .?AVBaseProducer@CVideoImport@@
.?AVBasePlayer@CVideoImport@@    .?AVH263Player@CVideoImport@@
.?AVBaseConsumer@CVideoImport@@  .?AVH263Consumer@CVideoImport@@
.?AVVanillaConsumer@CVideoImport@@
```

`CBaseFilter`/`CBasePin`/`CBaseInputPin`/`CMediaGrabber`/`UIMemInputPin` are the **DirectShow SDK
base classes**, linked in rather than imported as `strmbase` — so the authoring tool hosts a real
DirectShow filter graph. Two backends sit over it: `DirectShow` and `QuickTime`, both under
`CVideoImport`, with a `Resampler` between the source and the encoder.

The producer/consumer split is the transcode model: `BaseProducer` grabs and resamples frames,
`H263Player`/`H263Consumer` encode, `VanillaConsumer` is the no-encode passthrough (**inference**
from the naming and the `Vanilla` vs `H263` pairing).

### 4.2 Import formats

```
Macromedia Flash Video (*.flv)|*.flv
Video for Windows (*.avi)|*.avi
MPEG Movie (*.mpg,*.mpeg)|*.mpg;*.mpeg
Digital Video (*.dv,*.dvi)|*.dv;*.dvi
QuickTime (*.mov)|*.mov
```

and in the codec-download list (`0xf80aa2`, UTF-16 in `.rsrc`):

```
All Video Formats|*.avi;*.mpg;*.mpeg;*.mov;*.dv;*.asf;*.wmv;*.flv
All Video Formats|MooV;VfW ;MPG ;dvc!;MFLV[*.mov;*.avi;*.mpg;*.dv;*.flv http://www.sorenson.com/sparkpro
```

`MFLV` is the FourCC at the end of the second list — this tool's own FLV variant, one letter away from
the `FLV ` it puts in a file-name filter. The `MooV`/`VfW`/`MPG`/`dvc!` entries beside it are the
QuickTime/DirectShow/MPEG/DV identifiers for the same job, and `[` is the separator this string uses
instead of `|` after the FourCC list. The Spark Pro URL in the same string is the codec-licence
prompt; the neighbouring string ` try import via QuickTime?` is the fallback offer shown when the
DirectShow side has no decoder for a stream.

This build never writes such a file: the `FLV` version-1 signature and an `MFLV` immediate are both
absent from the whole image, so both names exist only on the import side, where they are matched
against the extension and the codec keys above.

Two filters in the same block matter above: `Windows Media (*.asf,*.wmv)` is importable although
nothing here encodes those codecs, and `Video for Windows (*.avi)|*avi` ships with its dot missing —
the Win32 dialog matches anything ending in `avi`, not in `.avi`.

Encoder settings exposed in the dialog, as strings: `Compress video`, `Smooth`, `Quality:`,
`Keyframe interval:`, `Video Format:`, `Maintain aspect ratio`,
`Synchronize video to Macromedia Flash document`,
`Number of video frames to encode per number of…`.

### 4.3 `DefineVideoStream` (60): `0x659205`

Already recovered in the writer round, restated here because it is the whole video format decision:

```
put16([obj+8])                      character id
put16(([obj+0x138]-[obj+0x134])>>4) frame count, from two frame indices
put16([obj+0x100])                  width
put16([obj+0x104])                  height
putbyte(0)                          flags: no smoothing, not deblocking
putbyte(codec)                      [obj+0xd8] mapped 1->1, 2->2, anything else -> 0
```

Codec id 2 is H.263 / Sorenson Spark, and it is the only id the mapping can produce besides 1. The
gate above this call site is `cmp [ebp+0x110],5 / jle`, so **a movie with a video stream cannot be
exported below SWF 6**: below that the object is dropped with diagnostic `0x1e` and nothing is
written.

### 4.4 `VideoFrame` (61): `0x653fcd`

```
if ([esp+0x34] != 0 && [esp+0x30] != 0) {     payload present
  beginTag(0x3d, 1);
  put16([ebx+8])        stream id
  put16([esi+0x10])     frame number, the object's own counter
  putmem(payload, size) via 0x40c7b1
  endTag();
}
[esi+0x10]++                                   always increments
```

Note the asymmetry: the frame counter advances even when no tag is written, so a dropped frame is
silent — the movie's frame numbering can skip with no record of the gap.

### 4.5 Tag 38, the video record

`0x65911b`, guarded by `IsKindOf(CMediaVideo)`: `UI16`, then the name written with `putmem` and an
explicit `strlen` (**no NUL terminator**, the only STRING field in this writer without one), then a
`UI8` that takes 0, 1 or 2 from three converging paths. Unlike tag 60 it is **not** version-gated.
What the flag selects is unknown.

---

## 5. What is not there

### 5.1 No VP6 and no Screen video

`VP6`, `Screen video`, `On2` (as a codec name) have no occurrences. The only video encoder classes
are H.263. This matches the era — VP6 arrives with Flash 8 — but it is worth stating because the
*import* side advertises AVI/MOV/MPEG, which can carry many codecs; the transcode target is always
Spark.

### 5.2 No audio codec newer than MP3

No AAC, no ADPCM-with-alpha, no 8/48 kHz, and the speech codec has no name beyond "Speech sound".
The encoder-compression set is exactly: none, ADPCM 2–5 bit, MP3 8–160 kbps.

### 5.3 Streaming sound exists but cannot be validated

There is a head writer after all (`FUN_007a3470`, §3.6), plus five block emitters, all reached from
the frame sound compiler. What cannot be checked is whether the emitted head is *complete*: its only
call site passes a null configuration, so the output is a tag 45 whose entire body is one zero byte.
No shipped movie contains tags 18, 19 or 45, and the authoring tool was not asked to export a
streaming-sound movie here, so there is nothing to compare the bytes against.

### 5.4 No capture path in the exporter

There is no camera-capture, microphone-capture or `NetStream` publish path in the SWF writer: the
`CVideoImport` graph is fed from a file, and the only `NetStream` code in the image is player-side.

### 5.5 But the player inside the tool does have them

The scripting engine's native property tables (`0xedf6f0`–`0xedf8dc`) include the Camera and
Microphone objects in full:

```
Camera:   bandwidth, keyFrameInterval, nativeModes, onActivity, ati, vfw, defaultcamera, quality,
          motionTimeOut, motionLevel, loopback, muted, activityLevel, entFps
Microphone: defaultmicrophone, echosuppression, gain, useEchoSuppression, rate,
          silenceTimeOut, silenceLevel
NetStream: play, publish, publish.BadName, bufferLength, bufferTime, time,
          NetStream.Buffer.Full / .Empty, attachVideo, attachAudio, seek, pause,
          receiveVideo, receiveAudio
```

So `Camera`, `Microphone` and `NetStream` exist for *script*, while nothing in the exporter consumes
them: a movie can reference `Camera.get()` and the authoring tool will play it, but it cannot author
a captured stream.

The Actions panel's own object inventory agrees — its `<object id="…"/>` block (`0xf59980`) lists
`Camera`, `Key`, `Microphone`, `Mouse`, `NetConnection`, `NetStream`, `SharedObject` and 17 others,
even though its `<string>` reference entries cover only some of them.

---

## 6. Open questions

- **Whether the head's flags match its blocks.** The call itself is understood now (§3.6): a merged
  rate word, a config struct, a codec code. What stays unverified is whether that combination writes a
  self-consistent stream, because no sample carries tags 18, 19 or 45.
- **Why 5.5 kHz is special-cased** in both `DefineSound` (`0x7a2fb6`) and the head writer
  (`0x7a34da`), leaving the arithmetic path whenever the rate is not 5512 Hz.
- **Whether the ADPCM factor of two** is deliberate headroom or a units error. The tables and the
  jump table are read now (§3.4); the intent is not.
- **What tag 38's `UI8` selects** (0, 1, 2) and why a video record exists beside tag 60.
- **What `MFLV` denotes on import.** It is a codec key in the filter list, not a container this build
  produces — no signature is written anywhere in the image. Which decoder the key resolves to, and
  whether the QuickTime backend can take an FLV stream at all, is untested.
- **Whether the QuickTime backend can decode FLV** or only hands it to DirectShow. The ` try import
  via QuickTime?` string shows the fallback is offered; no decoder load was observed.
