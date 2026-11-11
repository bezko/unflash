---
layout: default
title: ActionScript core objects
permalink: /actionscript/
description: The authoring tool's ActionScript engine — its documented core objects and the undocumented features beside them.
---

What the authoring tool's ActionScript engine actually implements, and what it hides.

Target: `extracted/InstallShield/App_Files/Flash.exe` (PE32, sha256
`64280f22…d9510da`, imagebase `0x400000`). Addresses are VAs. Observations are read off a named
address; inferences say so. Companion pages: the [format reference]({{ '/format/' | relative_url }}), the
[writer API]({{ '/writer/' | relative_url }}), the [hunt log]({{ '/hunt-log/' | relative_url }}).

---

## 1. There are two surfaces, and they disagree

The binary contains **two unrelated descriptions of the same language**, and neither is complete:

1. **The Actions panel's built-in reference**, an XML data file embedded in `.rsrc` from
   `0xf40000`–`0xf60000`. It holds **491 `<string>` entries**: 285 members across 17 named objects
   plus 206 global/operator entries, each with a `tiptext`, `helpid` and a `version` gate (`4`, `5`
   or `6`). This is what Macromedia decided to document *inside the product*.
2. **The interpreter itself**, a compact engine at `0x78f000`–`0x79f000`, entirely separate from
   the SWF writer at `0x7a0000`. Its string pool is at `0xbe1700`–`0xbe1900` and its initialiser is
   `FUN_0079c200`.

The interesting material is in the gap. Names that exist in the runtime but in no reference entry
are the undocumented surface (§5). Names in the reference that no runtime path resolves are
candidates for the reverse gap — the panel documents `ASSetPropFlags`-era player features this build
does not carry (§6).

---

## 2. The documented surface, as the product counts it

Entries per named object, from the embedded reference XML:

| Object | Entries | Object | Entries |
|---|---:|---|---:|
| `MovieClip` | 60 | `Capabilities` | 10 |
| `TextField` | 41 | `LoadVars` | 9 |
| `Date` | 36 | `XMLSocket` | 7 |
| `XML` | 32 | `Object` | 6 |
| `TextFormat` | 17 | `Color` | 4 |
| `Sound` | 16 | `Function` | 3 |
| `Button` | 15 | `Boolean` | 2 |
| `Array` | 13 | `Number` | 2 |
| `String` | 12 | | |

The version gates are informative in themselves: `Object` gets `watch`, `unwatch` and `addProperty`
only at version 6, as do `Array.sortOn` and `Function.apply`/`Function.call`. Everything else in
that table is marked version 5, except the operator/global block, which runs back to version 4.

The global block includes several entries that are documented but barely known, all marked v4:

```
mbchr          Convert an ASCII/S-JIS code to a character (multibyte aware)
mblength       Calculate the length of the string (multibyte aware)
mbord          Convert a character to an ASCII/S-JIS code (multibyte aware)
mbsubstring    Extract count characters, starting at index'th character (multibyte aware)
```

and two v6-only globals that are flags rather than functions:

```
showMenu       context menu visibility: show (true), hide (false)
align          movie alignment: centered(C), top(T), top-left(TL), ...
```

`align` and `showMenu` are marked `object=""` — they are host-behaviour switches, not library
functions, and their reference text reads like a preference description rather than a call signature.

**Coverage caveat.** Seventeen objects is not the whole language. The same XML carries a separate
`<object id="…"/>` declaration block (`0xf59980`–`0xf59be0`) naming 24 objects: `Date`, `Math`,
`Number`, `Object`, `String`, `Button`, `Camera`, `Color`, `Key`, `Microphone`, `Mouse`,
`MovieClip`, `Selection`, `Sound`, `Stage`, `System`, `TextField`, `LoadVars`, `NetConnection`,
`NetStream`, `SharedObject`, `XML`, `XMLSocket`, `CustomActions`. Twelve of them carry `<string>`
entries; twelve are declared with none — `Math`, `Camera`, `Key`, `Microphone`, `Mouse`,
`Selection`, `Stage`, `System`, `NetConnection`, `NetStream`, `SharedObject`, `CustomActions`. In
the other direction, five objects with entries — `TextFormat`, `Array`, `Capabilities`, `Function`,
`Boolean` — have no declaration in that block. The file is two overlapping lists, neither
complete, so the panel's object inventory cannot be read off the entry count.

---

## 3. Initialisation: `FUN_0079c200` (437 bytes)

This is the engine's bootstrap, and it is where the first undocumented feature lives.

```
alloc(0x468)                    -> the interpreter object
[ctx+0xab4] = obj
[obj+0x58]  = ctx+0x468
[obj+0x1ec] = ctx
[obj+0x50]  = 15999
FUN_00799130(obj+0x144, "ASnative", &LAB_00402040, 0)     ; 0x79c2ad
NaN      <- helper call (float10)                          ; string at 0xbb9674
Infinity <- helper call (float10)                          ; string at 0xbb9684
```

Three observations worth separating:

- **`ASnative` is installed as a global by name at startup.** The string sits at `0xbe1870` and has
  exactly one reference in the whole image, `0x79c2ad`, inside this function. It is registered with
  a code pointer, `LAB_00402040`, which resolves to a thunk: `jmp 0x79c170`. Script can therefore
  call it; the published ActionScript reference never mentions it. This is the classic AVM1 bridge
  into the native function table, and in this build it is reachable from any movie script the
  authoring tool evaluates.
- **`NaN` and `Infinity` are computed, not stored.** Both are produced by helper calls returning a
  `float10` and then stored under the literal names. The decompile does not show the arithmetic
  inside those helpers, so "computed rather than constant" is an observation; *which* arithmetic is
  an inference — the conventional pair is 0/0 and 1/0.
- **`15999` is written to `[obj+0x50]`** before any script runs. What that field means (a stack
  depth, a call budget, a recursion limit) is not established.

---

## 4. The object model, read from the four functions that implement it

### 4.1 Construction: `FUN_0079c430` (1544 bytes)

Called with a function object, a `this` value and the argument count, this is the `new`/`call` path.
It writes the prototype link explicitly, by name:

```
0x79c545   "prototype" (0xbe1758)    push the constructor's prototype
0x79c559   "__proto__"  (0xbe173c)   install it as the new object's __proto__
0x79c7da   "arguments"  (0xbe1884)   create the arguments object
0x79c7f1   "callee"     (0xbe187c)   point it at the running function
```

Then it pads the argument list to the callee's declared arity: it reads the parameter count from the
function's descriptor (`*local_68`), copies the supplied arguments, and fills the remainder from a
helper (`thunk_FUN_007824f0`) in a loop. So **missing parameters exist inside the callee as
`undefined` for every declared slot** — the callee never sees a short argument list.

`arguments.callee` and the `__proto__` write are both properties the reference XML never describes:
`__proto__` appears once in the panel data, as an `Object` member, and `callee` once, with no object.

### 4.2 Property lookup: `FUN_0078fa30` (774 bytes)

Resolution order, read from the branches:

1. an active `watch` registration wins (`thunk_FUN_0079d410` on `[obj+8]`, cleared through
   `thunk_FUN_0078a060`);
2. names beginning with `_` take a **separate dispatch path** (`*param_3 == '_'`), resolved through
   `thunk_FUN_00788ab0` instead of the normal property table — the underscore namespace is not just
   a naming convention, it is a different lookup;
3. ordinary names go through the property index (`thunk_FUN_00795070`), then through the prototype
   chain (`[obj+0x5c]+0xc`), then fall back to a **`MovieClip` lookup**
   (`thunk_FUN_0079cc40(obj, "MovieClip")`) when the scope has no other answer.

That fallback is why an undeclared name can still resolve to a clip property in the authoring
environment, and it is invisible in the documentation.

### 4.3 Prototype walking: `FUN_00794680` (173 bytes)

Walks `__proto__` recursively and flags each level, with a depth budget of **6** (`local_1c = 6`)
before it stops descending. Chain depth is therefore capped by the engine, not by the movie.

### 4.4 Default string conversion: `FUN_0079e470` (366 bytes)

Produces the `[type Object]` and `[type Function]` texts (strings at `0xbe18a4` and `0xbe1890`) for
values with no `toString` of their own. Both literals live in the same pool as `#ERROR#`
(`0xbe1784`), `_flash` (`0xbe17ac`), `_changed` (`0xbe17b4`) and `soundId` (`0xbe1864`) — engine
internals leaked into script-visible output.

---

## 5. The undocumented surface

| Name | Where | What the binary shows | In the panel reference? |
|---|---|---|---|
| `ASnative` | `0xbe1870`, installed at `0x79c2ad` | global callable bound to `LAB_00402040` → `jmp 0x79c170`; native-function bridge | **no** |
| `__proto__` | `0xbe173c`, written at `0x79c559` | real slot, assigned on construction; resolvable by script | one mention, as a member of `Object` |
| `arguments.callee` | `0xbe187c`, written at `0x79c7f1` | set on every call's `arguments` object | one mention, no object |
| `_`-prefixed names | `FUN_0078fa30` | separate lookup table, ahead of the prototype chain | no |
| `[type Object]` / `[type Function]` | `0xbe18a4`, `0xbe1890` | default string forms | no |
| `#ERROR#` | `0xbe1784` | error string in the value pool | no |
| `_flash`, `_changed` | `0xbe17ac`, `0xbe17b4` | internal flags in the script-visible pool | no |
| `NaN`, `Infinity` | `0xbb9674`, `0xbb9684` | globals created by computation at init, not constants | `NaN` only, as a value |

Two of these deserve emphasis because they change how code behaves:

- **`ASnative` lets a movie reach the host function table.** Any script evaluated by the authoring
  tool can call it. There is no reference entry, no panel completion, and no help id.
- **Missing arguments are padded with `undefined`.** A function written as `f(a,b)` receives two
  arguments even when called as `f(1)`, so `arguments.length` and the parameter count diverge —
  observable behaviour that the reference text does not describe.

---

## 6. Negative finding: no `ASSetPropFlags`

`ASSetPropFlags` — the player's documented-nowhere mechanism for marking properties as
non-enumerable or read-only — has **zero occurrences** in this image: not as ASCII, not as UTF-16,
and not in the 491 reference entries. Nor is `isPropertyEnumerable`, `propertyIsEnumerable` or
`isPrototypeOf` present.

The engine does implement the *effects* (`FUN_00794680` flags properties while walking the
prototype chain), but no script-callable name reaches them. So the authoring tool's engine is a
subset of the player's AVM1, and code that relies on `ASSetPropFlags` will not run there. This is a
difference between the tool and the player, not a difference between SWF versions — the binary is
the authoring application.

---

## 7. What is not established

- **The `ASnative` table itself.** `LAB_00402040` is a thunk into `0x79c170`; the table of native
  functions indexed by `ASnative(table, index)` was not enumerated, so which natives are reachable
  is unknown.
- **Whether the reference XML's 491 entries equal the implemented set.** The XML describes the
  panel's completion data; matching it member-by-member against runtime registration would require
  enumerating every native registration call, which was not done.
- **`[obj+0x50] = 15999`.** Written at init, meaning unresolved.
- **The arithmetic behind `NaN`/`Infinity`.** Computed by helpers; the helpers were not decompiled.
- **Whether the authoring engine and the standalone player share code.** Nothing here compares the
  two binaries.
