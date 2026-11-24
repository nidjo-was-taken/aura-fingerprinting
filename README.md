# Aura Fingerprinting under Addon Restrictions

## What a fingerprint is

A fingerprint is a set of constraints on aura properties that lets exactly one aura through, chosen from the auras that can coexist in the same context.

It is declared, not computed. In combat you give the game constraints and it filters; you cannot read the result. Learning happens out of combat, or when combat ends and values become readable again.

Properties fall into four families. Choose the most robust family that is enough, and add a more precise one only if collisions remain.

| Family | Examples (names as ForeverAuras exposes them) | Robustness | Precision |
| --- | --- | --- | --- |
| Semantic (Blizzard's classification) | `isBossAura`, `isRoleAura`, `isBossOrRoleAura`, `isPriorityAura`, `CROWD_CONTROL`, `IMPORTANT`, `BIG_DEFENSIVE`, `EXTERNAL_DEFENSIVE`, `RAID_IN_COMBAT` | Stable against talents; breaks if Blizzard reclassifies | Low |
| Categorical | Buff or debuff, dispel type (Magic, Curse, Disease, Poison, Bleed, Enrage), `isFromPlayerOrPlayerPet`, `PLAYER` | Very stable | Low |
| Quantitative | Total duration (`<=` native; `=` and `>=` through a gate, ±0.5 s), stack count, remaining-time window | Sensitive to haste, talents, diminishing returns | High |
| Contextual | Which unit carries it, order of application, time since a known event | Depends on the event being reliable | High within its context |

Constraints combine with AND only. An OR needs several displays, one fingerprint each. Section 7 lists the combinations the client refuses.

## Building fingerprints out of combat

Out of combat, aura fields are readable, so this is where fingerprints are measured, compared and checked. ForeverAuras itself reads auras only when neither `InCombatLockdown()` nor `ShouldAurasBeSecret()` is true.

All examples below use a hypothetical encounter, "Boss X". Its values are illustrative, not game data.

### 1. Property catalogue (the fingerprint sheet)

Record, for every aura that can appear on the target unit type, the properties the client can filter on. This sheet is the raw material of every other technique.

*Example.* Boss X puts three debuffs on allies. Sated, the Bloodlust-family debuff that BigWigs also excludes by ID, can sit on the same allies.

| Aura | Dispel type | Total duration | Max stacks | `isBossAura` | Who gets it |
| --- | --- | --- | --- | --- | --- |
| Debuff A | Magic | 8 s | 1 | yes | 1 random ally |
| Debuff B | Magic | 12 s | 1 | yes | 2 allies |
| Debuff C | Curse | 12 s | 3 | yes | tank |
| Sated | none | long | 1 | no | whole raid after Bloodlust |

### 2. Universe of coexisting auras

A fingerprint only has to be unique among auras that can be present together, not among all auras in the game. List the coexisting auras first, then solve for that small set.

*Example.* For Debuff A in the table above, the universe is {B, C, Sated}. Two constraints, boss aura and duration 8 s, already isolate A.

### 3. Collision matrix

Compare every pair in the universe and note which property separates them. The fingerprint is the smallest set of properties that separates the target from all others.

*Example.* A and B share dispel type (Magic) but differ in duration (8 s against 12 s). B and C share duration (12 s) but differ in dispel type (Magic against Curse). So B needs duration plus dispel type; A needs only duration.

### 4. Before and after snapshot

Snapshot the auras on a unit before and after an event, out of combat, and keep what is new. Compare by properties, not by instance ID: instance IDs are reported to be re-randomised when you enter an encounter, a challenge mode or a PvP match **\[to verify\]**.

*Example.* Take a training dummy or a test target, apply a known debuff, and diff the two snapshots to read its dispel type, duration and flags without guessing.

### 5. Durations from spell descriptions

ForeverAuras can derive a duration from the spell's description text (it calls `C_Spell.GetSpellDescription` and keeps the best match), so you can estimate a duration for an aura you cannot reproduce.

*Example.* A journal entry reads "for 8 sec": enter Total Duration = 8 s. Cross-check it with a measured value, since descriptions can include talent-modified or conditional text.

### 6. Regression check per patch

Re-run the catalogue and the collision matrix after every patch or hotfix. Blizzard can change durations and reclassify flags.

*Example.* A hotfix moves Debuff A from 8 s to 10 s. The gate tolerance is ±0.5 s, so a fingerprint set to 8 s stops matching and the check flags it before the next raid night.

## Using fingerprints in combat

In combat a fingerprint is a configuration of native filters and gates, and the game does the matching. The examples continue with Boss X from the previous section (illustrative values).

### 1. Declarative filter

Combine native constraints on the unit type that carries the aura: debuff filter, `isBossAura`, a dispel type, and a total duration.

*Example.* Unit: raid. Filter: debuff, `isBossAura`, dispel type Magic, Total Duration = 8 s (±0.5 s). Debuff A passes; B (12 s) and C (Curse) do not. A duration gate cannot be anchored to unit frames (Section 7).

### 2. Exclusion by subtraction

Remove known noise so that what remains is the target. Excluding by spell ID works only for `NeverSecret` spells; otherwise exclude by dispel type or flag.

*Example.* BigWigs' own player-debuff container uses this: its `excludeSpellIDs` list drops Deserter (26013), the Bloodlust-family debuffs (57723, 57724, 80354, 95809, 160455, 264689) and Stagger (124255), and sets `isFromPlayerOrPlayerPet = false`. For Boss X, `excludeDispelTypes = Curse` drops Debuff C from the display that tracks A and B.

### 3. Order as a discriminator

Sort the matching auras and show only the first. Available sort orders include order applied, remaining time, importance, name, and Blizzard's own debuff order.

*Example.* Sort by order applied, show one icon: the newest boss debuff on the raid. Combined with a boss-mod warning that a mechanic starts in 3 s, the newest aura is almost certainly the new one.

### 4. Duration bands

Several displays, each with a different duration window, sort auras into bands. `<=` is native; `=` and `>=` use a gate.

*Example.* Display 1: Total Duration = 8 s (A). Display 2: Total Duration = 12 s (B and C). Display 3: Total Duration >= 20 s (long-lived noise). Whichever display lights up tells you the band.

### 5. Absence as a signal

"Aura(s) Missing" and "Always" show state even when nothing matches.

*Example.* Show an icon on a healer's frame only when no boss debuff is present: the debuff was dispelled or has expired.

### 6. Delegating to Blizzard's classification

Where a Blizzard flag already means what you need, use it and keep nothing else to maintain.

*Example.* `RAID_IN_COMBAT` (auras Blizzard flags for raid-frame display in combat) or `IMPORTANT` for buffs on enemies. Blizzard updates these lists with its own patches.

### 7. Two-stage display **\[to verify\]**

Because gates cannot anchor to unit frames, split the job in two. Stage A, anchored to the frames, uses a loose filter and shows who carries a boss debuff. Stage B, a separate list, uses the precise gate and shows which one is active.

*Example.* Stage A: raid frames, `isBossAura` + Magic. Stage B: list, Total Duration = 8 s. When only A's debuff is up, the two stages together point at the right ally.

### 8. Event window **\[to verify\]**

BigWigs and DBM events reach ForeverAuras when they are not secret (the integration discards messages whose spell ID or text is secret). Use them to open a time window.

*Example.* The boss mod announces "Debuff A in 3 s": the fingerprint display is only relevant for the next 10 s. Whether a boss-mod trigger can gate an aura display in ForeverAuras is not confirmed.

## Validating fingerprints

A fingerprint is only trusted after it has been compared with a source of truth, because in combat you cannot read what it matched.

### 1. Sound-by-ID as an oracle

ForeverAuras registers native aura sounds through `C_UnitAuras.AddAuraSound` for three triggers: aura added, aura removed, and stack gained. The editor's status line says these sounds still play for spell IDs whose display Blizzard refuses in combat. Only sound files work, not sound kit IDs.

*Example.* Set a distinct sound on "added" for Debuff A's spell ID, and the fingerprint display for A shows an icon. Icon without sound is a false positive; sound without icon is a false negative. Whether group units register correctly is **\[to verify\]**.

### 2. Acceptance criteria

Decide before testing what counts as good, because a critical alert tolerates fewer false positives than a reading aid.

*Example.* Accept a fingerprint for a critical alert only after zero false positives over three pulls; accept one for a reading aid at no more than one wrong icon per pull. Record each pull as a tally: hits, misses, wrong icons.

### 3. Discovery pull with a permissive display

On a first attempt, show every boss debuff on allies, sorted by order applied. The game draws icons, names (in the tooltip) and countdowns, so you can read them by eye.

*Example.* Note with a stopwatch that the first icon counts down from 8 s and the second from 12 s, then use those numbers in the catalogue of Section 4.

### 4. Calibration between pulls

When combat ends, values become readable again and ForeverAuras re-reads the auras that are still up. Its automatic learning only looks at your player, pet, target and focus, not at group members.

*Example.* If an ally still carries Debuff C at the end of the pull, target or focus them before it expires. ForeverAuras records the duration and re-applies the displays that were waiting for it.

## Trade-offs and limits

The main trade-off is precision against position: the precise gates cannot be anchored to unit frames or nameplates. Each limit below is read from ForeverAuras 1.4.8.

| Limit | Effect | Workaround |
| --- | --- | --- |
| Total Duration `=` and `>=` cannot anchor to unit frames or nameplates | Precision or position, not both | Two-stage display (Section 5, technique 7) |
| Total Duration `=` / `>=` and Stack Count need "Aura(s) Found" and no Remaining Time | No single-aura mode with these gates | Use a list display and accept a cluster |
| Stack Count cannot be combined with Total Duration `=` / `>=` | One quantitative gate per display, except in a cell chain | Split into two displays |
| Remaining Time is for one unit, and on icons only | No group-wide time windows | Use the duration gate on the remaining time for bars and texts |
| Gate tolerance is ±0.5 s | Two auras 0.5 s apart collide | Add dispel type or a flag |
| Sounds ignore Stack Count, Total Duration and Remaining Time filters | The sound plays on any stack, not when the filter is met | Treat the sound as an oracle, not as a filtered alert |
| Some gates need client features (`C_AuraContainerUtil` options, duration text bindings) | A gate may be unavailable on a given build | The editor status line reports it **\[to verify\]** |

Two hard limits remain whatever the technique. The addon cannot count auras or compute on durations, so an uptime percentage cannot be derived from a fingerprint. And a fingerprint classifies; it does not prove that the matched aura is the one you meant.

## Worked example: Debuff A on the raid

The goal is to show Debuff A of Boss X on raid members in combat, and to know that it is A and not B, C or Sated. All values are illustrative.

&#91;embedded content: fingerprint workflow · 6 steps, 1 loop\]

Steps 1 to 3 build the fingerprint out of combat; steps 4 to 6 use and test it, and any mismatch sends you back to the sheet.

1. **Sheet.** Debuff A: Magic, 8 s, boss aura, one random ally (Section 4, technique 1).
2. **Universe.** Debuff B, Debuff C, Sated (technique 2).
3. **Collisions.** A differs from B and C by duration, and from Sated by the boss flag (technique 3).
4. **Stage A (who).** Raid frames, `isBossAura` + Magic. It lights for A and for B, because both are Magic boss debuffs.
5. **Stage B (which).** A separate list, Total Duration = 8 s. Only A passes. A sound on A's spell ID acts as the oracle (Section 6, technique 1).
6. **Validation.** Three pulls, tally hits, misses and wrong icons, and accept at zero false positives.
7. **Maintenance.** Re-run the check after each patch.

The residual weakness is attribution. If A and B are on two allies at once, Stage A lights both frames and Stage B only confirms that A exists; a person has to infer which frame it is.
