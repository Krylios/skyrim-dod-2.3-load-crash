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

That resolves the result that had been hardest to place: deleting the `.skse` changed nothing.

> **Correction.** An earlier version of this file said the co-save test "rules out JContainers."
> That was wrong, and the mod's own documentation says so: **JContainers is a hard dependency of
> Sanguine's Trade** (`Docs/ST_API_FOR_EXTENSIONS.md`). What the co-save test actually shows is
> narrower and still useful — *the crashing payload* is not in the co-save, so whatever holds it is
> not JContainers-backed storage. JContainers is used by the mod for other state, and it is a
> shared framework many mods depend on. It is not a suspect and must not be disabled.

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

---

## 8. What removing the mod does and does not require

**Nothing else depends on it.** A scan of every `.esp` / `.esm` / `.esl` in the install for a
Sanguine's Trade master reference returns **zero** hits. No patch, no extension, no compatibility
plugin points at it, so unmounting it orphans nothing.

**What it depends on, and what must stay:**

| framework | status |
|---|---|
| **JContainers SE** | hard dependency of ST — but a shared framework across the list. **Leave enabled.** |
| **PrismaUI** | ST's UI transport — also used by `Tailor - An Outfit and Wig Manager`. **Leave enabled.** |
| SexLab P+ / OStim | ST routes scenes through them; framework-agnostic. Untouched by this. |

**Tool re-runs, if the removal becomes permanent** (Rule 11 triggers, blanket by design):

| tool | triggered | why |
|---|---|---|
| **Synthesis** | **yes** | four plugins removed — the trigger is any plugin added/removed/replaced |
| **BodySlide** | **yes** | ships armor, wig and hair meshes |
| ParallaxGen | probably | ~70 loose textures |
| Pandora | unknown | no loose animations or behavior files; the 207 MB BSA could not be enumerated (BSArch not installed) |
| DynDOLOD / TexGen / xLODGen | **no** | the plugin contains no worldspace or landscape records |

> The record-signature counts behind that last row come from a raw scan of a binary plugin, which is
> indicative rather than a proper census. The absence of worldspace records is consistent with what
> the mod is — an interior, quest-and-script mod — but it has not been confirmed in xEdit.

**Do not run any of them while the test is in progress.** A Synthesis re-run changes the load order
mid-bisection, and if the mod turns out to be innocent it goes back in and the run has to be redone.
Re-runs are for a settled list, not for a list under test.

---

## 9. RESOLVED — the test passed, and the author confirmed it

### The controlled test

A fourth character, created fresh with `Sanguine's Trade` fully unmounted, played past the boundary
where every previous character had already broken. Nine saves, all clean:

| playtime | cell | payload |
|---|---|---|
| `000.05.28` … `000.46.19` | various | clean |
| `000.46.52` | Hall of the Elements ("Before FL") | clean |
| **`000.49.23`** | Hall of the Elements (**"After FL"**) | **clean** |
| **`000.50.56`** | Hall of the Elements | **clean** |

The matched pair is the result:

| | old run (mod active) | new run (mod unmounted) |
|---|---|---|
| save name | `Vania After FL` | `Vania After FL` |
| playtime | `000.50.54` | `000.49.23` |
| cell | Hall of the Elements | Hall of the Elements |
| payload | **ROSTER PRESENT** | **none** |
| loads? | **no** | **yes** |

Same character name, same cell, same quest beat, past the boundary where the old run was already
unloadable. `000.49.23` clears the old first-bad point of `000.49.14`. The save loads and play
continues normally.

### Confirmed upstream

`Sanguine's Trade - An Economy Mod` **v1.0.7b** (2026-09-06) — the author's changelog:

> **FIXED: Saves made with 1.0.7 could crash on load, every time, even with the mod removed. The
> Ledger's instant first open parked a large block of text inside the save, and the game cannot read
> it back.** That feature is gone; the first open of a session builds the page live again, as in
> 1.0.6.

Shipped in **DoD 2.4.0** (2026-09-06), whose release note reads: *"This non-save safe update was
unfortunately mandatory due to everyone's saves not functioning correctly."*

The trigger is visible in the **1.0.7** changelog that introduced it: *"FIXED: Opening the Ledger
cost four to seven seconds of frozen game. It paints instantly on the first open of a session now."*
That optimisation cached the Ledger's rendered page content into a Papyrus string — the
`ST_PrismaBridge.Invoke("window.STRender(" + jsonString + ")")` path traced in §4 — and Papyrus
strings are serialised into the `.ess`.

### Correspondence between what was measured and what the author states

| measured here | author's wording |
|---|---|
| a JSON block inside the `.ess`, extracted at offset 2,772,248 | "parked a large block of text inside the save" |
| the fault is on the save-load path | "the game cannot read it back" |
| unmounting the mod left `RAX`/`RBX`/`R15` byte-identical | "even with the mod removed" |
| `RBX` = 12,741 / 16,019 / 16,036 / 16,278, varying per save | "a large block of text" |

The diagnosis was reached independently, before the fix was published.

### Scope — this explains the third-party case too

`Sanguine's Trade` is part of the **curated list**, not a personal addition: it sits at line 271 of
the stock `Diaries of Dibella - Lord's Vision` profile, enabled. So the corroborating crash on an
unmodified 2.3.1 profile on another machine had the same mod, at the same broken version. Nothing in
the modified fork was ever required to reproduce this.

### If you are hitting this

1. Update `Sanguine's Trade` to **v1.0.7b**, or take **DoD 2.4.0**, which ships it.
2. **Saves already carrying the payload are unrecoverable.** Removing the mod does not retract it —
   verified. 2.4.0 is a minor version bump and is not save-safe regardless.
