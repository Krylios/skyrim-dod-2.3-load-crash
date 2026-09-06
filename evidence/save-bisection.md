# Save bisection — how the failing window was narrowed

Parsed straight out of the `.ess` headers. **The Skyrim save header is uncompressed**, so player
name, level, location and playtime read out without launching the game — see the bottom of this file
for the snippet. That is what turned "somewhere in a ten-minute window" into a named cell.

**Note the two clocks.** `wall` is the file mtime; `playtime` is the in-game counter from the header.
They diverge whenever a save is reloaded, and the playtime column is the one that matters.

---

## Character 2 — `FEB81BDF`, first bisection

Five launches, from "somewhere in 14 saves" to an exact boundary:

```
001.06.06 ✅  ->  001.12.18 ✅  ->  001.14.26 ❌  ->  back to 001.12.18 ✅
                          last good = 001.12.18      first bad = 001.14.26
```

| wall | playtime | lvl | location | .ess | .skse | verdict |
|---|---|---|---|---|---|---|
| 22:33 | `000.59.53` | 1 | Sleeping Giant Inn | 10,146,120 | 1,271,737 | loads |
| 22:39 | `001.06.06` | 1 | Skyrim | 10,332,421 | 1,322,247 | loads |
| 22:45 | `001.12.18` | 1 | Hall of the Elements | 10,151,015 | 1,341,096 | **LAST GOOD** |
| 22:55 | `001.14.26` | 1 | Hall of the Elements | 10,137,210 | 1,335,732 | **FIRST BAD** |
| 22:57 | `001.17.15` | 1 | Hall of the Elements | 10,122,436 | 1,336,061 | crashes |
| 22:59 | `001.18.41` | 1 | Hall of Attainment | 10,256,744 | 1,345,700 | crashes |

**The break is 2 minutes 8 seconds of playtime wide**, inside one cell.

---

## Character 3 — `F1BE8262` ("Vania"), the 33-second reproduction

This is the important one. A fresh character, created after every local suspect had been disabled.

| wall | playtime | lvl | location | .ess | .skse | verdict |
|---|---|---|---|---|---|---|
| 03:04 | `000.48.41` | 1 | Hall of the Elements | 10,266,047 | 967,624 | **loads fine** |
| 09:40 | `000.49.14` | 1 | Hall of the Elements | 10,184,199 | **1,317,383** | **crashes** |

Load the 48:41 save, stand still, do nothing, save 33 seconds later — **that save is already
unloadable.**

```
playtime delta   +33 seconds
.ess             10,266,047 -> 10,184,199    SHRANK by 81,848 bytes
.skse co-save       967,624 -> 1,317,383     GREW 36.1%
```

**Read the co-save column carefully.** My first conclusion was that SKSE was therefore the culprit.
**It is not** — loading the crashing `.ess` with its `.skse` deleted crashes identically, same
address, same `RAX`. So this table is evidence of **rate**, not of **location**: something serialises
a large and growing amount of state on every save, but the corruption itself lives in the `.ess`.

Leaving the wrong inference visible because it is the obvious one to draw from this table.

---

## What this rules out

| hypothesis | test | result |
|---|---|---|
| gameplay actions (looting, quests) | save taken 33s after load, having done nothing | **still crashes** |
| completing *First Lessons* | save taken *before* completing it | **still crashes** |
| looting books / spell tomes | save taken *before* looting | **still crashes** |
| the SKSE co-save | deleted it, loaded the `.ess` alone | **still crashes, identical `RAX`** |
| a bad character / bad start | three fresh characters, all new games | **all three affected** |
| my added mods | a third party on an unmodified profile | **identical stack** |

**A new game always works.** That is what moved the whole investigation off the load order and onto
the save, and it cost exactly one launch. Ask it first.

---

## Three characters, all fresh on 2.3.0

No save was carried over from 1.7; each began as a brand-new game.

| character | saves | span |
|---|---|---|
| `FEB81BDF` | 13 | 09-05 13:46 → 09-06 00:58 |
| `1BC67476` | 1 | 09-06 00:37 (crashes logged 4 and 12 min later) |
| `F1BE8262` | 13 | 09-06 02:18 → 09-06 09:42 |

Starting over does not avoid it. It recurs.

---

## Reading save headers without launching the game

Layout after the 13-byte `TESV_SAVEGAME` magic and a `uint32` headerSize:
`version(u32), saveNumber(u32), playerName(wstr), playerLevel(u32), playerLocation(wstr),
gameDate(wstr), playerRaceEditorId(wstr), …` — where `wstr` is a `uint16` length then UTF-8 bytes.

```powershell
function RS($br){ $n=$br.ReadUInt16(); [Text.Encoding]::UTF8.GetString($br.ReadBytes($n)) }
Get-ChildItem $saves -Filter *.ess | Sort-Object LastWriteTime | ForEach-Object {
  $fs=[IO.File]::OpenRead($_.FullName); $br=New-Object IO.BinaryReader($fs)
  $br.ReadBytes(13)|Out-Null; $br.ReadUInt32()|Out-Null
  $br.ReadUInt32()|Out-Null; $br.ReadUInt32()|Out-Null
  $name=RS $br; $lvl=$br.ReadUInt32(); $loc=RS $br; $play=RS $br
  "$($_.LastWriteTime)  $play  Lv$lvl  $loc"
  $br.Close(); $fs.Close()
}
```

This also caught a false assumption: session 2 began at playtime `000.50.13`, **earlier** than
session 1's last save at `001.18.41` — so session 2 did not continue from where session 1 ended.
The timeline reconstruction would have been wrong without it.
