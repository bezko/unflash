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
the writer has exactly one video codec path, and no literal `SoundStreamHead` emitter exists at all
(§5.3).

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
| MP3 in SWF below version 6 | refused | `cmp ebx,6 / jl` at `0x7a2f48` |
| Streaming sound head tag (18/45) | **no emitter found** | §5.3 |
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
- **The MP3 version window is `[6, 17]`**, both ends enforced. Above 17 the writer falls out of the
  MP3 branch, which is odd for a tool that can write higher versions elsewhere.
- **The format dispatch is a nibble test, not an enum compare**: `(flags & 0xf0) == 0x20`. The low
  nibble (rate/size/type) rides along untouched.

### 3.4 ADPCM depth is computed, not chosen

The non-MP3 branch (`0x7a2f89` onward) does arithmetic before it writes anything:

```
cmp [writer+0x240],4          ; the writer's SWF version field (see SWR_WRITER_API §2)
cmp version,0x11
mov edx,(flags>>2)&3          ; SoundRate, 2 bits
cmp [edx*4+0xaf7f8c],0x1588   ; rate table; 0x1588 = 5512 = the 5.5 kHz entry
mov eax,[edx*4+0xaf7f8c]      ; rate value
mov ebx,[version*4+0xaf8888]  ; per-version budget table
lea ebx,[ebx+ebx] ; idiv     ; quotient
cmp eax,5 ; ja  -> depth 5
jmp [eax*4+0x7a3334]          ; cases 2,3,4,5
```

So the ADPCM bit depth (2–5 bits) is derived from a rate table, a per-version table and a division,
then passed to the ADPCM writer at `0x418b83`. **Inference:** the quotient is a bit budget (bits per
sample the container can afford at that rate) and the depth is the largest of 2–5 that fits — the
mechanics are observed, the intent is not. What is *not* true is the natural assumption that the
"2/3/4/5 bit ADPCM" dropdown value is written straight through: no path takes a depth from the
sound object's format byte.

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

### 3.6 Streaming: five block emitters, no head

Tag 19 `SoundStreamBlock` is emitted from five places, all in the `0x7a38f0`–`0x7a3f20` range:

| site | notes |
|---|---|
| `0x7a390d` | builds a `CString` at `[obj+0x244]` with size `[obj+0x24c]`, writes it as the payload |
| `0x7a3a15` | **empty block** — `put16(0); put16(0)` then `endTag` |
| `0x7a3ab1`, `0x7a3dc5`, `0x7a3f15` | same family as the first |

The second one is a stub: a ten-byte tag whose entire body is two zero words. It exists to advance
the stream timeline during silent frames — **inference**, since nothing branches on it.

**No `SoundStreamHead` (18) or `SoundStreamHead2` (45) emitter exists as a literal.** A scan of
every `push 0x12` and `push 0x2d` byte pair in `.text` found none that reaches `beginTag`
(`0x413afc`) within 12 bytes, while the same scan finds all five tag-19 sites, tag 14, tag 60 and
tag 61. Either the head is composed through a computed code expression, or this build does not write
one — the corpus is no help, because none of the 62 shipped movies contains tag 18, 19 or 45.

### 3.7 Version refusals are user-visible strings

```
"MP3 sound codec not supported in SWF versions less than 6"
"Speech sound codec not supported in SWF versions less than 6"
```

The second names a codec the binary never calls by name: **"Nellymoser" has zero occurrences** in
either encoding. In this build the SWF 6 speech codec is only ever "Speech sound", and its message
gates it below version 6 like MP3.

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

and in the codec-download list (`0xf80aa2`):

```
Video Formats:|MooV;VfW ;MPG ;dvc!;MFXV[*.mov;*.avi;*.mpg;*.dv;*.flv
http://www.sorenson.com/sparkpro
```

`MFXV` is the FourCC this tool registers for its own FLV variant; the Spark Pro URL in the same
string is the codec-licence prompt that appears when QuickTime cannot decode the stream natively.

The FLV association is also in the registry block: `…\Software\Macromedia\Flash\6` with `.flv`
mapped to `Flash.Movie`.

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

### 5.3 No streaming head

See §3.6. Tag 19 blocks without a tag 18/45 head are unparseable as a stream; if this build can
export streaming sound, the head is written somewhere the literal scan did not reach.

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

- **Where the streaming head is written.** Five tag-19 emitters, no tag 18 or 45 anywhere as a
  literal. Either a computed code or a path not yet examined.
- **What tag 38's `UI8` selects** (0, 1, 2) and why a video record exists beside tag 60.
- **Whether the ADPCM depth decision is a bitrate fit.** The division and the 2–5 jump table are
  observed; the meaning of the two tables is not.
- **Why the MP3 branch stops at SWF 17** while the rest of the writer supports higher versions.
- **What `MFXV` is on disk.** The FourCC and the FLV export filter exist; no shipped sample uses
  either, so the container layout is unverified.
- **Whether the QuickTime backend can decode FLV** or only hands it to DirectShow — the Spark Pro
  URL hints at a third-party component, but no load of one was observed.
