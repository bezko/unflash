# unflash

Notes on the internals of Macromedia Flash — specifically the SWF exporter inside
**Flash MX** (`Flash.exe`, FileVersion 6.0.21.0, March 2002), recovered by static
reverse engineering.

Published at **https://bezko.github.io/unflash/**.

## Contents

| Page | What it covers |
|---|---|
| [The SWF file format](https://bezko.github.io/unflash/format/) | A working description of the SWF container, written as the reference for reading what this writer emits. |
| [Recovered writer API](https://bezko.github.io/unflash/writer/) | The exporter's object, its primitives, the tag map, and the undocumented tags. |
| [ActionScript](https://bezko.github.io/unflash/actionscript/) | The authoring tool's ActionScript engine: the documented core-object surface and the undocumented features beside it. |
| [Video & audio](https://bezko.github.io/unflash/media/) | The media side of the exporter: import formats, bundled codecs, and the sound/video tags it emits. |
| [Hunt log](https://bezko.github.io/unflash/hunt-log/) | Chronological record of the analysis: what was tried, what failed, what each address turned out to be. |

## Method, in one paragraph

The target is a 12.1 MB PE32 image with 52 291 `jmp` thunks, so the call graph is
recovered by resolving thunks before counting callers. Decompilation and
disassembly come from Ghidra run headless over a project-local analysis database;
findings are cross-checked against the 62 SWF files shipped inside the installer,
which is the only ground truth available without running the player. Claims are
labelled *observation* (read off a named address) or *inference*; unresolved
questions are listed rather than smoothed over.

## What is not here

No binaries, installers, licence keys or extracted application files. Every page
refers to addresses in a copy of the software that the reader must supply; the
image is identified only by size, FileVersion and SHA-256.

## Building the site locally

GitHub Pages builds these files directly. To preview locally:

```sh
gem install jekyll
jekyll serve --baseurl ""
```

## Licence

No licence is granted for the prose or the layout at this time.
