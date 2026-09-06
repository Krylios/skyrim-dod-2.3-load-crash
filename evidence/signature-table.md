# Signature table — all 19 crashes

Generated directly from the CrashLogger files. `RAX` is decoded little-endian to ASCII.

**Timestamps in log filenames are UTC; file mtimes are local.** A `crash-2026-09-06-15-30-39.log`
is a 09:30 local event. Worth knowing before you line these up against save times.

| # | log (UTC) | uptime | RAX | RAX as ASCII | RBX | RBX dec | R15 dec |
|---|---|---|---|---|---|---|---|
| 1 | `2026-09-06-03-51-53` | 01:41 | `2274636570736F72` | `rospect"` | `0x3EA4` | 16036 | 103465 |
| 2 | `2026-09-06-03-53-53` | 01:40 | `2274636570736F72` | `rospect"` | `0x3EA4` | 16036 | 103465 |
| 3 | `2026-09-06-03-56-06` | 01:48 | `2274636570736F72` | `rospect"` | `0x3EA4` | 16036 | 103223 |
| 4 | `2026-09-06-04-00-01` | 02:12 | `2274636570736F72` | `rospect"` | `0x3EA4` | 16036 | 103223 |
| 5 | `2026-09-06-05-47-48` | 01:54 | `73696C6B63616C62` | `blacklis` | `0x3E93` | 16019 | 103174 |
| 6 | `2026-09-06-05-51-51` | 02:08 | `73696C6B63616C62` | `blacklis` | `0x3E93` | 16019 | 103174 |
| 7 | `2026-09-06-06-03-07` | 01:51 | `73696C6B63616C62` | `blacklis` | `0x3E93` | 16019 | 103174 |
| 8 | `2026-09-06-06-31-39` | 01:56 | `73696C6B63616C62` | `blacklis` | `0x3E93` | 16019 | 103174 |
| 9 | `2026-09-06-06-41-02` | 01:44 | `73696C6B63616C62` | `blacklis` | `0x3E93` | 16019 | 103174 |
| 10 | `2026-09-06-06-49-14` | 06:43 | `73696C6B63616C62` | `blacklis` | `0x3E93` | 16019 | 103046 |
| 11 | `2026-09-06-15-01-47` | 01:38 | `3A226469227B2C7D` | `},{"id":` | `0x31C6` | 12742 | 102936 |
| 12 | `2026-09-06-15-04-05` | 01:44 | `3A226469227B2C7D` | `},{"id":` | `0x31C6` | 12742 | 102990 |
| 13 | `2026-09-06-15-10-36` | 03:47 | `3A226469227B2C7D` | `},{"id":` | `0x31C6` | 12742 | 102862 |
| 14 | `2026-09-06-15-13-54` | 02:05 | `3A226469227B2C7D` | `},{"id":` | `0x31C6` | 12742 | 102677 |
| 15 | `2026-09-06-15-30-39` | 01:35 | `333A226469227B2C` | `,{"id":3` | `0x31C5` | 12741 | 103032 |
| 16 | `2026-09-06-15-35-40` | 01:38 | `333A226469227B2C` | `,{"id":3` | `0x31C5` | 12741 | 103138 |
| 17 | `2026-09-06-15-42-26` | 04:35 | `333A226469227B2C` | `,{"id":3` | `0x31C5` | 12741 | 102911 |
| 18 | `2026-09-06-15-48-41` | 05:58 | `333A226469227B2C` | `,{"id":3` | `0x31C5` | 12741 | 102911 |
| 19 | `2026-09-06-15-55-16` | 01:51 | `333A226469227B2C` | `,{"id":3` | `0x31C5` | 12741 | 102911 |
| — | **third party, unmodified 2.3.1 profile, other machine** | 02:37 | `72616C7567657222` | `"regular` | `0x3F96` | 16278 | 102438 |

## What the columns show

**`RAX` is ASCII JSON on every single crash.** Five distinct fragments across 20 crashes, at one
instruction. I dismissed this as incidental twice before checking all of them — one ASCII register
is noise, five at an identical fault address is not.

**`RBX` is constant per save and varies by save.** Four values across my 19, one per save I was
loading, plus a fifth from the other machine. Reloading the same save reproduces the same `RBX`;
a different save gives a different one. So it derives from **save content**.

> I first said these clustered "just under 16,384" and put that in the report. The 12,741/12,742
> values disprove it. Leaving the wrong version visible because it is superficially appealing and
> someone else will reach for it.

**`R15` barely moves at all** — 102,438 to 103,465 across 20 crashes, two machines, two list
versions. **~1% spread.** A number that stable across independent installs looks like a count over
**list content** rather than save content.

**Uptime** is 1:35–2:12 for 15 of the 19. The four longer ones (3:47, 4:35, 5:58, 6:43) are sessions
where I made more than one load attempt before the crash landed; I have not proven that, so treat
the range as 1:35–6:43 and the cluster as the signal.

## Reproducing the decode

```bash
echo 3A226469227B2C7D | fold -w2 | tac | tr -d '\n' | xxd -r -p    # -> },{"id":
```
