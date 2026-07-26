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

## Included bug fix

`GameSnapshot` did not remove cards that exist in the running game but not in the snapshot
— tokens, copies and effect cards created after it was taken. Restoring then walked the
battlefield, looked those cards up in the snapshot, and hit a `NullPointerException`. This
never showed up in Forge's own use of snapshots, because cancelling a cast creates no new
objects, but it makes any real rewind crash. The fix removes such cards from the game
during the restore, which is what rewinding past their creation means anyway.

This fix is useful on its own and would be worth upstreaming.

## Files touched

| File | Change |
|---|---|
| `forge-game/…/Game.java` | rewind point history, `rewindToActionOf`, `getAvailableRewindSteps` |
| `forge-game/…/GameSnapshot.java` | remove cards the snapshot does not know (the fix above) |
| `forge-game/…/phase/PhaseHandler.java` | `PriorityState` for the bookkeeping snapshots miss; rewind check in `mainLoopStep` |
| `forge-game/…/player/PlayerController.java` | hooks for requesting a rewind |
| `forge-gui/…/player/PlayerControllerHuman.java` | request handling, full resync of clients afterwards |
| `forge-gui/…/net/server/FServerManager.java` | `resyncAllClients()` |
| `forge-gui-desktop/…/menus/GameMenu.java` | the menu entry |
| `forge-gui/res/languages/en-US.properties` | three new strings |
| `forge-gui-desktop/src/test/java/forge/ai/RewindTest.java` | new tests |

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
mvn -B -pl forge-gui-desktop test -Dtest=RewindTest
```

Five tests cover taking back one and several own actions, undoing an opponent's move along
with your own, restoring turn/phase/priority, and the step limit. Forge's own desktop suite
(276 tests) and its TCP network integration test pass with the patch applied.

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
einen Fehler-Fix in Forges Schnappschuss-Code (Spielsteine und Kopien wurden beim
Zurückholen nicht entfernt, was zum Absturz führte).
