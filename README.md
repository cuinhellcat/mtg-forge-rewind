# Multi-step rewind (undo) for Forge

A patch for [Forge](https://github.com/Card-Forge/forge) that lets whoever runs the game
take back their last few actions — including everything that happened afterwards, such as
the opponents' and the AI's moves.

Forge already ships an undo, but it only covers your own spell while it is still on the
stack and you still hold priority. In a multiplayer game against friends and AI, that
rarely covers the "wait, I misclicked" case. This patch adds a **Rewind last action** entry
to the in-game *Game* menu that jumps the whole game back to just before your previous
action, up to three times in a row.

Built and tested against tag `forge-2.0.13`.

## What it does

* Keeps a short history of rewind points instead of the single snapshot Forge kept before.
  One is taken every time a player is about to act, which is machinery Forge already had —
  it was only used to unwind a spell you cancelled mid-cast.
* One rewind step = back to the moment you last held priority. Whatever happened since is
  discarded and gets played out again from there.
* **Host only.** The player running the game rewinds; everyone else keeps their stock Forge
  and simply receives the restored state. The network protocol is untouched, so unpatched
  clients stay compatible. They will see a version-mismatch warning in the lobby chat,
  which is harmless.

## Included snapshot fixes

Rewinding leans on Forge's snapshots far harder than cancelling a cast does, which brings
four separate defects in them to the surface. All four are in the patch, and each is
offered to Forge on its own, since they matter to anyone using undo restore:

| What was wrong | Upstream |
|---|---|
| Cards that exist in the running game but not in the snapshot — tokens, copies, effect cards — were never removed on restore, so the attachment loop looked them up and dereferenced null. Harmless while snapshots only unwound a cancelled cast, fatal for any real rewind. | [PR #11416](https://github.com/Card-Forge/forge/pull/11416) |
| A foretold card lost its foretold state on restore and could no longer be cast with Foretell — the two setters for it had sat commented out since the method was written. | [#7844](https://github.com/Card-Forge/forge/issues/7844) |
| Cards were filed under their controller rather than the player whose zone they are in, so a stolen card ended up in the wrong graveyard. | [#9297](https://github.com/Card-Forge/forge/issues/9297) |
| Rolling back through a snapshot skipped the ability cleanup entirely: announced values, targets and the frozen stack survived the cancelled cast. | [#10049](https://github.com/Card-Forge/forge/issues/10049), [#8762](https://github.com/Card-Forge/forge/issues/8762) |

## Files touched

| File | Change |
|---|---|
| `forge-game/…/Game.java` | rewind point history, `rewindToActionOf`, `getAvailableRewindSteps` |
| `forge-game/…/GameSnapshot.java` | the first three snapshot fixes above |
| `forge-game/…/GameActionUtil.java` | the fourth: ability cleanup on a snapshot rollback |
| `forge-game/…/phase/PhaseHandler.java` | `PriorityState` for the bookkeeping snapshots miss; rewind check in `mainLoopStep` |
| `forge-game/…/player/PlayerController.java` | hooks for requesting a rewind |
| `forge-gui/…/player/PlayerControllerHuman.java` | request handling, full resync of clients afterwards |
| `forge-gui/…/net/server/FServerManager.java` | `resyncAllClients()` |
| `forge-gui-desktop/…/menus/GameMenu.java` | the menu entry |
| `forge-gui/res/languages/en-US.properties` | three new strings |
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
(279 tests in total, all green).

Not covered by automated tests: clicking the menu entry in a live match. The engine side is
tested, the GUI wiring is not.

## Caveats

* Forge labels these snapshots experimental. Exotic card interactions may restore wrongly.
* Rewinding cannot un-see information. If an opponent already saw your hand, it stays seen.
* Random outcomes are re-rolled, and the AI may decide differently the second time around.

## Licence

GPL v3, same as Forge — this is derived work and cannot be anything else. See `LICENSE`.

---

**Kurz auf Deutsch:** Dieser Patch gibt Forge im Mehrspieler-Modus einen echten
Rückgängig-Knopf: Der Gastgeber kann bis zu drei eigene Aktionen zurücknehmen, samt allem,
was seitdem passiert ist. Mitspieler brauchen keine geänderte Version. Enthält nebenbei
vier Fehlerkorrekturen an Forges Schnappschuss-Code, die beim Bauen aufgefallen sind — vom
Absturz durch nicht entfernte Spielsteine bis zum eingefrorenen Stapel nach einem
abgebrochenen Zauberspruch. Jede davon ist Forge auch einzeln angeboten.
