# DoD 2.3.x — save-load crash at `SkyrimSE.exe+143E7DD`

Evidence pack for the Modding Bordello crashlog thread. Everything here is **measured, not
inferred**; where I guessed, it is labelled as a guess, and where I guessed wrong the wrong
version is left in with a correction.

**Symptom in one line:** the game plays fine, then a save written during that session cannot be
loaded from the main menu. A new game always works.

**Status:** reproduced on **two machines**, **two list versions** (2.3.0 and 2.3.1), and — critically
— on **one unmodified profile that is not mine**.

> ### RESOLVED — confirmed by the mod author and fixed upstream
>
> **Fixed in `Sanguine's Trade` v1.0.7b (2026-09-06); shipped in DoD 2.4.0.** The author's changelog:
> *"Saves made with 1.0.7 could crash on load, every time, even with the mod removed. The Ledger's
> instant first open parked a large block of text inside the save, and the game cannot read it back."*
>
> The diagnosis below was reached independently before that fix was published, and matches it.
> All five JSON fragments found in `RAX` are **field names from one mod's client roster**
> (`Sanguine's Trade - An Economy Mod`), the payload is **physically present in the crashing save and
> absent from the working one**, and the mod pushes that roster through a **Papyrus `String`** — which
> is serialised into the `.ess`, explaining why deleting the SKSE co-save changed nothing.
>
> **The decisive test passed.** A fresh character with the mod unmounted, played past the boundary
> where every earlier character had already broken, produced saves with no payload — and they load.
> Full working: **[`evidence/root-cause-candidate.md`](evidence/root-cause-candidate.md)**.
>
> Lead credited to **`Nutella`** in the Bordello crashlog thread.

| | |
|---|---|
| Fault | `SkyrimSE.exe+143E7DD` — `mov byte ptr [rbx+rax*1], 0x00` |
| Exception | `EXCEPTION_ACCESS_VIOLATION` reading `0xFFFFFFFFFFFFFFFF` |
| Runtime | 1.6.1170, `SkyrimSE.exe` MD5 `7a44a52dfc92d78f934c4d12ed92f494` |
| Occurrences | 20 of mine + 1 third-party, all identical |
| Reproduction | ~2 minutes, no gameplay required (below) |

### Contents

| path | what |
|---|---|
| `README.md` | this report |
| [`logs/`](logs) | three of my crash logs, **trimmed** to header + registers + call stack |
| [`evidence/signature-table.md`](evidence/signature-table.md) | all 19 crashes, register by register |
| [`evidence/save-bisection.md`](evidence/save-bisection.md) | how the failing window was narrowed |
| [`evidence/root-cause-candidate.md`](evidence/root-cause-candidate.md) | **the current lead** — schema match, save A/B, mechanism |

> **On the trimmed logs:** the `MODULES` / `SKSE PLUGINS` / `PLUGINS` sections are removed. Nothing
> diagnostic is lost — the fault address, registers and call stack are intact — I just did not want a
> 3,700-line plugin list and my user paths indexed on a public repo. Full untrimmed logs on request
> in the thread.
>
> **A second person's crash log is deliberately NOT republished here.** They posted it in Discord for
> support, not for a public repo. The comparison against it is below in full; the raw file is theirs
> to share.

---


## ⚠ Up front: this is a MODIFIED list

**I run a Rule 11 fork of Diaries of Dibella 2.3.0 with my own mods added.** Flagging that before
anything else — my crash may not be the same one others are hitting, and nobody should spend time on
it under the assumption it is a clean install. Treat everything below as "one modified install",
not as a clean-list data point.

**Setup:** DoD 2.3.0, runtime 1.6.1170, custom profile, clean 2.3.0 install.
Curator profile has 3,691 active plugins; mine has 3,700.

**What I added** (9 extra plugins, the rest are asset/preset-only):

| mod | plugin |
|---|---|
| Tullius SMP Hair 3 1004 | `Tullius Hair 3 SMP.esp`, `SMP2`, `SMP3` |
| Tullius Eyes Pack | `Tullius Eyes.esp` |
| Tullius Hair 3 XML Pack 1004 / textures Pack 1004 | — |
| Extended Makeup and Features | `ExMakeup.esp` |
| Female Basic Makeup STANDALONE | `Pocky Punk's Make Up Addon_females.esp` |
| Female Makeup Suite - Face - 2K | `FMS_FemaleMakeupSuite.esp` |
| Koralina's Makeup Tweaks 2k ESL | `Koralina's Makeup Tweaks.esp` |
| Auto Player Outfit Swap | `PlayerOutfitSwapper.esp` |
| Auto Track Nearby Quests 3.1 | — (SKSE DLL) |
| SG Female Eyebrows - Improved, Leyenda Skin | — |
| Body/RaceMenu presets: Vanta Luxe, Curvy Queen, Curvycious, Thicc Charm, Levathicc Body, Seranya, Vania, Yovanna | — |

**Also changed:** `ENB Frame Generation` enabled (curator ships it disabled). Custom BodySlide and
Synthesis output folders per the Rule 11 guide; Synthesis re-run after adding plugins.

**Disabled during testing and NOT present for the crashes below:** `Tullius SMP Hair` (the older 806
build), `Personal INI and MCM Settings`, and my own cell-ownership patch.

I am not claiming this is the listside issue. If it turns out to be mine, that is a fair outcome and
I will keep chasing it myself — I am posting because the signature looked identical to what was
described and the reproduction is unusually cheap.

---

## Independent corroboration — CLEAN profile, different machine, different list version

Someone else's crash log was compared against mine. Their setup shares nothing with mine except the
list itself:

| | mine | theirs |
|---|---|---|
| list version | 2.3.0 | **2.3.1** |
| profile | custom Rule 11 fork, 9 added plugins | **unmodified Lord's Vision** |
| install path | `I:\Modded Skyrim` | `E:\modlists` |
| machine | different | different |

**The call stack is identical frame for frame** — same functions, same offsets:

```
[0] 104661+0x2ED   [1] 104829+0x108   [2] 54049+0x9C    [3] 54018+0x22
[4] 35641+0xEE     [5] 35600+0x9A8    [6] skse64_1_6_1170.dll+001070B
[7] 442580+0x354   [8] 35772+0x46A
```

Same fault address `SkyrimSE.exe+143E7DD`, same instruction, same access violation on
`0xFFFFFFFFFFFFFFFF`, same executable MD5 `7a44a52dfc92d78f934c4d12ed92f494`, uptime 2:37 (inside my main
1:35–2:12 cluster), same symptom — **plays fine, then cannot load the save**.

**Their `RAX` is JSON too:** `0x72616C7567657222` → `"regular` — a fifth distinct JSON fragment,
from a machine that has never run any of my mods.

**`R13` is a `LoadStorageWrapper*` in both**, confirming the save-load path on both installs.

**`RBX` is constant per save, and varies by save.** It is the base of the write that faults. Across
my 19 crashes it takes exactly four values, one per save I was loading, and theirs is a fifth:

```
16,036  x4     16,019  x6     12,742  x4     12,741  x5     16,278  (theirs)
```

Reloading the same save reproduces the same `RBX` every time; a different save gives a different one.
So it is derived from save content — a count or index over something in the save — not a fixed
constant and **not** a buffer limit. (I initially thought these clustered just under 16,384 and said
so; the 12,741 values disprove that. Noting it because the wrong version is superficially appealing.)

**`R15`, by contrast, is tightly clustered across everything** — checked on all 19 of my crashes plus
theirs:

```
102,438  (theirs)  …  103,465  (mine)      ~1% spread, 20 crashes, two machines, two list versions
```

Unlike `RBX`, this barely moves between saves, characters, installs, or 2.3.0 vs 2.3.1. A value that
stable across independent machines running the same list looks like a **count over list content**
rather than over save content — the sort of number that would be near-identical for anyone on DoD
2.3.x and would shift if the list's content changed.

Combined with `RAX` holding JSON in all five distinct cases, the shape is: an index or length
computed from save content (`RBX`), applied against a structure holding JSON text, in a routine
whose other operand (`R15`) tracks list content — producing a write to
`[garbage_base + string_bytes]`.

**This clears my added mods.** An unmodified profile on another machine hits the identical stack. The
disclosure above stands, but nothing in my list is required to reproduce this.

They also note they enabled Texture Downscaler and ENB Frame Generation, **and that it still crashes
with both disabled** — so those are ruled out on their side. (I run ENB Frame Generation too, which
would otherwise have been a shared variable worth chasing.)

**It has happened on THREE separate new characters**, all created fresh on 2.3.0 — no save carried
over from 1.7, and each one started from a brand-new game rather than an imported one:

| character | saves | span |
|---|---|---|
| `FEB81BDF` | 14 | 09-05 13:46 → 09-06 00:58 |
| `1BC67476` | 1 | 09-06 00:37 (crashes logged 4 and 12 min later) |
| `F1BE8262` | 13 | 09-06 02:18 → 09-06 09:42 |

Two of the three were bisected save-by-save; both showed the same shape — early saves load, later
saves in the same session do not, with the boundary a couple of minutes of playtime wide. Starting
over on a new character does **not** avoid it. It recurs.

**Signature — identical across 19 crashes:**
```
SkyrimSE.exe+143E7DD    mov byte ptr [rbx+rax*1], 0x00
EXCEPTION_ACCESS_VIOLATION  reading 0xFFFFFFFFFFFFFFFF
Process uptime: 15 of 19 between 1:35 and 2:12; four outliers 3:47 – 6:43
Crash is on LOADING A SAVE from the main menu. A NEW GAME always starts fine.
```

**`RAX` holds an ASCII JSON fragment on every single crash.** Four distinct values across clusters (the last two both seen WITH and WITHOUT the co-save present):
```
0x2274636570736F72  ->  rospect"
0x73696C6B63616C62  ->  blacklis
0x3A226469227B2C7D  ->  },{"id":
0x333A226469227B2C  ->  ,{"id":3
```
The faulting instruction writes a null terminator using `RAX` as an **offset**. A string sitting
where a length or index belongs is what a parser looks like when its length field has been
overwritten by content. `RBX` is also small and non-pointer-like (e.g. `0x3E93`), so both operands
are junk — the structure being written through is corrupt, not just one field.

**Reproduction — no gameplay required:**
```
1. Load a save that works
2. Save immediately (seconds later, having done nothing)
3. Quit to desktop
4. Load that new save  ->  crash
```
The newly written save is already unloadable. Whatever the player does in game is irrelevant.

**Something is being serialised into the save, fast.** Across **33 seconds** of playtime, standing
still and doing nothing:

| | .ess | .skse co-save |
|---|---|---|
| save that loads | 10,266,047 | 967,624 |
| save 33s later that crashes | 10,184,199 | 1,317,383 |

The main save **shrank** while the co-save grew 36%. My first read was that the co-save was therefore
the problem — **the test below disproves that**, so treat this table as evidence of *rate*, not of
location. Something writes a lot of state on save within seconds; the corruption itself is in the
`.ess`. No JSON file on disk ever grew, so whatever it is, it is built in memory and persisted into
the save rather than read from a config.

**Ruled out on my install** (each disabled/reverted, crash persisted):
- personal INI/MCM override mod (disabled, still crashed)
- the older Tullius SMP Hair 806 build (disabled, still crashed)
- my own cell-ownership patch + the Synthesis re-run that followed it
- OCW posted notes (`OBS_StaticUntilTaken`) — never picked up, still crashed
- Modex (its user JSON never changed)
- looting books/tomes, and completing First Lessons — both looked like triggers until a save taken
  *before* either one also failed

**Red herring worth flagging:** `hdtSMP64` appears at frame [12] in every stack, but it hooks
`Main::Update`, so it is in essentially every main-loop stack on this list. Easy to read as physics.

**Environment:** Win11 26100, Ryzen 9 9950X, RTX 5080, 31 GB RAM. Working set ~8 GB at crash,
~24/31 GB physical in use.

---

## Co-save test — DONE, and it rules SKSE out

Loaded the crashing `.ess` with its `.skse` **removed**. Skyrim loads a save without its co-save.

**Result: still crashed. Same address, same uptime, and the identical `RAX` value `,{"id":3`.**

So the corrupt data is in the **`.ess` itself**, not the SKSE co-save — and the JSON string survives
with SKSE's serialised data entirely absent. The 36% co-save growth was correlation, not cause.

**What that leaves.** With the co-save out of the picture, JSON-shaped text in a loaded `.ess` most
plausibly comes from the **Papyrus data in the save** — script variables and the save's string
table, which is where a Papyrus `String` holding a JSON payload would live. Something on the list is
building a JSON string in a script variable, and it is being serialised into the save and grown over
play until the load path faults on it.

That is a direction, not a conclusion — I have not proven which script, and I am not able to from
outside. But it is consistent with everything measured:

| observation | fits |
|---|---|
| JSON fragment in `RAX` on all 19 crashes | a JSON string in the save's string data |
| survives co-save removal | Papyrus data lives in the `.ess`, not the co-save |
| no JSON file on disk ever grew | built in memory, persisted into the save |
| a save taken seconds after loading is already bad | written on save, not by gameplay |
| new game always fine | nothing accumulated yet |
| both operands junk (`RAX` a string, `RBX` = `0x3E93`) | a corrupt length/index, not one bad value |

**Suggested next step for whoever is dissecting this listside:** look for a script holding a growing
JSON string — JContainers/PapyrusUtil-style serialisation into a `String` property, an MCM or state
tracker that accumulates `{"id": …}` array entries. The array-element boundaries seen in `RAX`
(`},{"id":` and `,{"id":3`) suggest a list of objects keyed by a numeric id.

## Useful commands

```bash
# every crash log's signature at a glance
find "$SKSE" -maxdepth 1 -iname "crash-*.log" -print0 | sort -z | while IFS= read -r -d '' f; do
  printf "%s uptime %s addr %s RAX %s\n" "$(basename "$f")" \
    "$(grep -oP 'Process Uptime: \K[0-9:]+' "$f")" \
    "$(grep -oP 'at 0x[0-9A-F]+ \K[^\t]+' "$f" | head -1)" \
    "$(grep -m1 -oP 'RAX 0x\K[0-9A-F]+' "$f")"
done

# decode a register that looks like ASCII (little-endian)
echo 3A226469227B2C7D | fold -w2 | tac | tr -d '\n' | xxd -r -p    # -> },{"id":
```

> **Note on timestamps:** CrashLogger names files in **UTC**; file mtimes are **local**. A
> `crash-2026-09-06-15-30-39.log` corresponds to a 09:30 local save. This cost me a wrong search
> window — worth stating if you compare log names against save times.
