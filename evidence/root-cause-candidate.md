# Root-cause candidate — a JSON roster serialised into the `.ess`

**Status: strongly indicated, NOT confirmed.** The decisive test — a fresh character with the mod
disabled — has not been run yet. Everything below is measured; the conclusion it points at is not
yet proven. Five earlier hypotheses in this investigation looked comparably good and all five died
under a controlled test, so treat this as the current best lead, not a verdict.

**Credit for the lead: `Nutella` in the Bordello crashlog thread**, who suggested disabling
Sanguine's Trade. Everything below is the follow-up on that hunch.

---

## 1. Every `RAX` fragment is a field name from one mod's data structure

`Sanguine's Trade - An Economy Mod` keeps a client roster. From its own UI source
(`PrismaUI/views/SanguinesTrade/index.html`):

```js
scenes:12, totalSpent:6000, regular:true, prospect:true, lastSeen:"2 days ago"
c => !c.blacklisted && (c.regular || c.prospect)
data-action="unreg-blacklist" data-id="${v.id}"
```

| `RAX` decoded | source |
|---|---|
| `rospect"` | `prospect` — client field |
| `blacklis` | `blacklisted` — client field |
| `"regular` | `regular` — client field |
| `},{"id":` / `,{"id":3` | the roster array, keyed by `id` |

Five distinct fragments, 20 crashes, two machines, two list versions — all five are fields of the
same structure.

## 2. The payload is physically present in the crashing save

Byte offset 2,772,248 of the crashing `.ess`. Readable in fragments because the save data is
LZ4-compressed — literal runs decode, back-references do not:

```
"cb":[{"id":132421,"name":"Brynjolf...","wealth":…,"female","spec":"None",
"disposition":…,"scenes":…,"totalSpent":…,"regular":"false","prospect":…,
"blacklisted":…,"lastSeen":…,"homeId":null,"favorite":…,"nextAppt":…,
"appointment":…}]
```

followed by a roster of named NPCs, each with a FormID — Mercer Frey, General Tullius, Esbern,
Arngeir, Kodlak Whitemane, Sheogorath, Nocturnal, Meridia, Cicero, Nazir, Festus Krex, Gabriella,
Veezara, Arnbjorn, Emperor Titus Mede II, Skjor, Galmar Stone-Fist, Septimus Signus — and then the
mod's own configuration keys (`fXPMultActive`, `MoodDegradeMult`, `FondnessGain`,
`showUtilityToolkit`, …).

> Some further config keys are omitted here. They are adult-content setting names, they add nothing
> diagnostic, and this is a public repo. The roster schema above is the part that matters.

## 3. A/B against the working save

The two saves that produced the 33-second reproduction:

| | `AutoRotate_2` — **loads** | `ManualRotate_3` — **crashes** |
|---|---|---|
| playtime | `000.48.41` | `000.49.14` |
| `{"id":` in `.ess` | **0** | **1** |
| `prospect` in `.ess` | **0** | **1** |

**The payload is absent from the save that loads and present in the save that does not.**

This also explains the 33 seconds. Loading the save builds the roster in memory; the next save
persists it. Gameplay was never the trigger — *loading* was.

## 4. The mechanism, and why the co-save test came out the way it did

`Source/Scripts/ST_PrismaBridge.psc:71`:

```papyrus
Function Invoke(string asJsCode) Global Native
{Runs a JS expression in the view. Used to push state JSON into the page.
    ST_PrismaBridge.Invoke("window.STRender(" + jsonString + ")")
```

The roster is assembled into a **Papyrus `String`** and pushed through a native bridge to a PrismaUI
web view. Papyrus strings are serialised into the **`.ess`**, not the SKSE co-save.

That resolves the result that had been hardest to place: deleting the `.skse` changed nothing. It
also rules out JContainers by elimination — JContainers state lives in the co-save, and the co-save
is demonstrably not where this lives.

It makes `RBX` legible too. 12,741 / 16,019 / 16,036 / 16,278 are consistent with **string lengths** —
constant per save, different across saves, growing as the roster fills, and zero on a new game.

## 5. Disabling the mod does NOT fix an existing save

Full unmount (MO2 `-` prefix, so `.esp`s, BSA, `SanguinesTrade.dll` and PrismaUI views all out),
then reload the same save:

| | before unmount | after unmount |
|---|---|---|
| `RAX` | `,{"id":3` | `,{"id":3` |
| `RBX` | 12741 | 12741 |
| `R15` | 102911 | 102911 |

**Byte-for-byte identical.** The blob is already written into the `.ess`; removing the mod cannot
retract it. So this test is **not** evidence against the hypothesis — it is what "already
serialised" looks like, and it means **existing saves are unrecoverable**.

## 6. What would actually confirm it

**A fresh character with the mod disabled**, played past the point where the roster populates
(~45 min in the observed case), then save → quit → reload.

- Loads → confirmed.
- Crashes → the mod is exonerated and the JSON came from somewhere else.

## 7. Caveat: PrismaUI has a second consumer

`Tailor - An Outfit and Wig Manager` also ships PrismaUI views, and the PrismaUI framework itself is
a separate mod that remains enabled. If the defect is in how PrismaUI marshals large strings rather
than in Sanguine's Trade specifically, Tailor is a second vector and disabling one mod will not
clear it.

The *data* in these crashes is unambiguously Sanguine's Trade's schema, so that is the right place
to start — but if the new-game test still fails, PrismaUI's other consumer is the next thing to pull.
