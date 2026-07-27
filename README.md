# Multi-step rewind (undo) for Forge

A patch for [Forge](https://github.com/Card-Forge/forge) that lets whoever runs the game
take back their last few actions — including everything that happened afterwards, such as
the opponents' and the AI's moves.

Forge already ships an undo, but it only covers your own spell while it is still on the
stack and you still hold priority. In a multiplayer game against friends and AI, that
rarely covers the "wait, I misclicked" case. This patch adds a **Rewind** submenu to the
in-game *Game* menu that jumps the whole game back to just before one of your previous
actions — plus saving and loading positions, which the same machinery makes possible.

Built and tested against tag `forge-2.0.13`.

> **Related upstream work:** [PR #10294](https://github.com/Card-Forge/forge/pull/10294)
> implements multi-step undo via Ctrl-Z and repairs the snapshot state more systematically
> than this patch does, including a reflection test that finds uncovered fields. If you want
> undo in Forge itself, that is the one to watch. This repository is a working build with a
> GUI on top, and the fixes below that #10294 does not cover.

## What it does

**Rewind** — a submenu with one entry per step, each saying where it leads
(`1 action back — back to turn 5, Main 1`), greyed out where no rewind point exists. One
step = back to the moment you last held priority; everything since is discarded and played
out again from there. Depth is a preference (*Settings → Preferences → Rewind steps*).

* Keeps a short history of rewind points instead of the single snapshot Forge kept before.
  One is taken every time a player is about to act, which is machinery Forge already had —
  it was only used to unwind a spell you cancelled mid-cast.
* Auto-pass is stopped on a rewind. Without that it carries the game straight back past the
  position you rewound to, and the next rewind lands on the same spot — visible in the log
  as the same snapshot timestamp restored over and over.
* **Host only.** The player running the game rewinds; everyone else keeps their stock Forge
  and simply receives the restored state. The network protocol is untouched, so unpatched
  clients stay compatible. They will see a version-mismatch warning in the lobby chat,
  which is harmless.

**Save and load** — *Game → Save game…* writes the position to a text file; *Game → Load
game…* puts one back mid-match; *Forge → Load saved game…* starts a match from one without
needing a game running first. This uses Forge's own `GameState` format, the one behind dev
mode and puzzles, which covers rather more than the snapshot mechanism does. Loading starts
the match under puzzle rules so there is no mulligan prompt over an empty opening hand, and
restores the default maximum hand size, which the saved format does not carry.

## Included snapshot fixes

Rewinding leans on Forge's snapshots far harder than cancelling a cast does, which brings
four separate defects in them to the surface. All four are in the patch, and each is
offered to Forge on its own, since they matter to anyone using undo restore:

| What was wrong | Upstream |
|---|---|
| Cards that exist in the running game but not in the snapshot — tokens, copies, effect cards — were never removed on restore, so the attachment loop looked them up and dereferenced null. Harmless while snapshots only unwound a cancelled cast, fatal for any real rewind. | [PR #11416](https://github.com/Card-Forge/forge/pull/11416) |
| A foretold card lost its foretold state on restore and could no longer be cast with Foretell — the two setters for it had sat commented out since the method was written. | [#7844](https://github.com/Card-Forge/forge/issues/7844) |
| Cards were filed under their controller rather than the player whose zone they are in, so a stolen card ended up in the wrong graveyard. | [#9297](https://github.com/Card-Forge/forge/issues/9297) |
| Rolling back through a snapshot skipped the ability cleanup entirely: announced values, targets and the frozen stack survived the cancelled cast. | [PR #11418](https://github.com/Card-Forge/forge/pull/11418) |
| A melded permanent came back as two, or as one that was both melded and loose on the battlefield. | [PR #11421](https://github.com/Card-Forge/forge/pull/11421) |
| A creature stolen after the snapshot stayed with the thief, and ended up in both battlefields at once. | partly in [#10294](https://github.com/Card-Forge/forge/pull/10294) |

The zone-owner and foretold fixes went upstream as [#11419](https://github.com/Card-Forge/forge/pull/11419) and, in #10294's wider form, are already covered there.

## Files touched

| File | Change |
|---|---|
| `forge-game/…/Game.java` | rewind point history with turn/phase labels, `rewindToActionOf`, `describeRewindPoints` |
| `forge-game/…/GameSnapshot.java` | the snapshot fixes above |
| `forge-game/…/GameActionUtil.java` | ability cleanup on a snapshot rollback |
| `forge-game/…/phase/PhaseHandler.java` | `PriorityState` for the bookkeeping snapshots miss; rewind and load checks in `mainLoopStep` |
| `forge-game/…/player/PlayerController.java` | hooks for requesting a rewind, and the pending saved state |
| `forge-gui/…/player/PlayerControllerHuman.java` | rewind requests, save and load, stopping auto-pass, client resync |
| `forge-gui/…/gamemodes/match/HostedMatch.java` | reads the rewind depth from the preferences |
| `forge-gui/…/localinstance/properties/ForgePreferences.java` | `MATCH_REWIND_STEPS` |
| `forge-gui/…/net/server/FServerManager.java` | `resyncAllClients()` |
| `forge-gui-desktop/…/menus/GameMenu.java` | rewind submenu, save and load entries |
| `forge-gui-desktop/…/menus/LoadSavedGame.java` | starting a match from a saved position |
| `forge-gui-desktop/…/menus/ForgeMenu.java` | that entry in the always-available menu |
| `forge-gui-desktop/…/settings/*Preferences.java` | the *Rewind steps* setting |
| `forge-gui/res/languages/en-US.properties` | new strings |
| `forge-gui-desktop/src/test/java/forge/ai/*Test.java` | tests for the rewind and for each snapshot fix |

## Build

Needs JDK 17+ and Maven.

```sh
git clone --branch forge-2.0.13 --depth 1 https://github.com/Card-Forge/forge.git
cd forge
git apply ../rewind.patch
mvn -B -DskipTests install
```

The result is `forge-gui-desktop/target/forge-gui-desktop-2.0.13-jar-with-dependencies.jar`.
Drop it into an existing Forge installation next to `res/`, replacing the jar of the same
name, and copy `forge-gui/res/languages/en-US.properties` over the installed one so the new
menu entry has its label.

To switch on the snapshots the rewind builds on, enable **EXPERIMENTAL Undo restore** in
Forge's preferences (`MATCH_EXPERIMENTAL_RESTORE=true`).

Number of steps, at startup:

```sh
java -Xmx8192m -Dforge.undoSteps=3 -jar forge-gui-desktop-2.0.13-jar-with-dependencies.jar
```

Each rewind point holds a full copy of the game, so give the JVM room — the default 4 GB
Forge launcher is tight with several of them.

## Tests

```sh
mvn -B -pl forge-gui-desktop -am test -Dtest='RewindTest,ForetellSnapshotTest,SnapshotStolenCardTest,RollbackCleanupTest'
```

`RewindTest` covers taking back one and several own actions, undoing an opponent's move
along with your own, restoring turn/phase/priority, and the step limit. The other three
reproduce the snapshot defects listed above: each fails without its fix and passes with it.
Forge's own desktop suite and its TCP network integration test pass with the patch applied
(282 tests in total, all green).

Not covered by automated tests: the menus themselves. Rewind, save and load have been used
in real games against the AI; the network side — a second player receiving a rewound state —
has not been tried yet.

## Caveats

* Forge labels these snapshots experimental. Exotic card interactions may restore wrongly.
* Rewinding cannot un-see information. If an opponent already saw your hand, it stays seen.
* Random outcomes are re-rolled, and the AI may decide differently the second time around.

## Licence

GPL v3, same as Forge — this is derived work and cannot be anything else. See `LICENSE`.

---

**Kurz auf Deutsch:** Dieser Patch gibt Forge einen echten Rückgängig-Knopf: Der Gastgeber
kann eigene Aktionen zurücknehmen, samt allem, was seitdem passiert ist — im Menü mit
Angabe, wohin der Sprung führt. Mitspieler brauchen keine geänderte Version. Dazu Speichern
und Laden von Stellungen, auch aus dem Hauptmenü heraus, etwa nach einem Absturz.

Beim Bauen sind fünf Fehler in Forges Schnappschuss-Code aufgefallen — vom Absturz durch
nicht entfernte Spielsteine über verlorene Foretell- und Kontrollzustände bis zum
eingefrorenen Stapel nach einem abgebrochenen Zauberspruch. Jeder ist Forge einzeln
angeboten worden, mit Test, der ohne die Korrektur fehlschlägt.
