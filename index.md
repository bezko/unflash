---
layout: default
title: unflash
permalink: /
description: Reverse-engineering notes on the Macromedia Flash MX SWF exporter.
---

# unflash

<p class="lead">What Macromedia Flash MX actually wrote into a <code>.swf</code> file, recovered by
reading <code>Flash.exe</code> rather than the specification.</p>

The published SWF format documents describe what a player must *accept*. They say
nothing about what a particular authoring tool chooses to *emit*: which tag variants
it prefers, which fields it fills with a constant, which records exist only in that
build. Those choices are the interesting part if you are writing a compatible
exporter, diffing two files, or reading a decompiled writer.

These pages are the result of taking the Flash MX exporter apart statically. Every
statement is tied to a virtual address in the image, and every conclusion labels
itself as an observation or an inference.

<ul class="cards">
  <li>
    <h3>The SWF file format</h3>
    <p>A working description of the container — header, tag framing, the character
    dictionary, the timeline, and the tag table — written to explain why this writer
    emits what it emits.</p>
    <a href="{{ '/format/' | relative_url }}">Read →</a>
  </li>
  <li>
    <h3>Recovered writer API</h3>
    <p>The exporter's procedural writer struct, its primitive emitters, the tag map
    with caller chains, and the tags it emits that the published format never
    defines.</p>
    <a href="{{ '/writer/' | relative_url }}">Read →</a>
  </li>
  <li>
    <h3>ActionScript</h3>
    <p>The authoring tool's own engine: the core-object surface its panel documents, and the
    runtime features it never mentions — <code>ASnative</code>, <code>__proto__</code> on
    construction, argument padding.</p>
    <a href="{{ '/actionscript/' | relative_url }}">Read →</a>
  </li>
  <li>
    <h3>Video &amp; audio</h3>
    <p>What the exporter really does with sound and video: a statically linked MPEG Layer-3
    encoder, a DirectShow/H.263 import graph, ADPCM depth chosen by arithmetic, and a
    sound-stream head whose tag number is computed rather than pushed.</p>
    <a href="{{ '/media/' | relative_url }}">Read →</a>
  </li>
  <li>
    <h3>Hunt log</h3>
    <p>The chronological record: the false starts, the thunk problem, the corpus
    checks, and what each suspicious address turned out to be.</p>
    <a href="{{ '/hunt-log/' | relative_url }}">Read →</a>
  </li>
</ul>

## Target

| | |
|---|---|
| File | `Flash.exe` (Macromedia Flash MX) |
| Format | PE32, imagebase `0x400000` |
| Size | 12 107 776 bytes |
| SHA-256 | `64280f22c156cff5d4aa2011aeed0a58c4b555b381d64395b8d52d548d9510da` |
| Version | FileVersion 6.0.21.0, timestamp 2002‑03‑08 |
| Disassembly | Ghidra 12.2, headless |

The image is not redistributed here. Addresses in these pages are virtual addresses
in that image, so they index a local copy; the file identity above is what they are
anchored to.

## Two things that mislead a first pass

1. **Almost no direct calls exist.** 52 291 `jmp` thunks carry the call graph
   (`E8` → thunk → `E9` → target), so a scan for `E8` to a target returns zero even
   for a function with dozens of callers. Every caller count here was produced by
   resolving thunks first, and every emitter was found by scanning references to its
   *thunk*, not its body.
2. **The writer is not a class.** No RTTI descriptor, no vtable: it is a plain
   procedural struct passed as `this` in `ecx`. Searching the 1283 type descriptors
   for anything called `Writer` finds nothing at all.

## How to read a claim

*An observation* is a fact read off a named address — an instruction, a constant, a
branch. *An inference* is reasoning about the bytes that the code does not literally
contain. Anything unsupported is listed as an open question on the page it belongs
to, with what would settle it. The [writer API page]({{ '/writer/' | relative_url }})
closes with the current list.

## Corpus grounding

Installer contents include 62 SWF movies written by Flash 5 and Flash 6. They are
used as the only available ground truth: a byte pattern that the decompiled writer
predicts and the corpus contains is evidence; one the corpus contradicts is treated
as a decompiler error and re-read in assembly. The format pages cite which movie
demonstrates which rule.
