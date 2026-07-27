# Turn rewind (undo) for Forge

A patch for [Forge](https://github.com/Card-Forge/forge) that lets whoever runs the game
restart one of their own turns from the beginning, discarding everything since — including
the opponents' and the AI's moves.

Forge already ships an undo, but it only covers your own spell while it is still on the
stack and you still hold priority. In a multiplayer game against friends and AI, that
rarely covers the "wait, I misclicked" case. This patch adds a **Rewind** submenu to the
in-game *Game* menu, plus saving and loading positions, which the same machinery makes
possible.

Built and tested against tag `forge-2.0.13`.

> **Related upstream work:** [PR #10294](https://github.com/Card-Forge/forge/pull/10294)
> implements multi-step undo via Ctrl-Z on top of Forge's snapshot mechanism, and repairs
> that mechanism systematically, including a reflection test that finds uncovered fields.
> If you want undo in Forge itself, that is the one to watch. This patch takes a different
> route — see below — and is a working build with a GUI on top.

## How it works, and why that way

An earlier version of this patch copied the whole game before every action, using Forge's
`GameSnapshot`. That kept breaking. `GameSnapshot` transplants field by field into live
objects, so every field nobody thought of is a bug: stolen creatures came back twice,
damage was not restored, life totals drifted.

This version stores a rewind point as **Forge's own save format** (`GameState`) — the one
behind dev mode, puzzles and saved games, which the Forge developers maintain and which
covers damage, control, libraries, the stack and the phase.

The price is that the format describes no "until end of turn" effects and no combat in
progress. So a point is only taken at one moment:

> your own turn, first main phase, empty stack, once per turn

That is the only moment where the format is complete: the previous turn's temporary
effects expired in its cleanup step, no combat is running, upkeep and draw have resolved.
Rewinding therefore always means "play this turn again", never "step back one action" —
a deliberate trade of granularity for correctness.

## What it does

**Rewind** — a submenu naming the turn each entry leads back to (`Restart turn 5`). Depth
0–10, set before the game under *Settings → Preferences → Rewind turns*. Does not need
Forge's experimental snapshot preference.

* Auto-pass is stopped on a rewind. Without that it carries the game straight back past the
  position you rewound to, and the next rewind lands on the same spot.
* Emblems and the hidden command-zone cards behind lasting effects are carried across by
  the rewind point itself, because the save format can only name printed cards.
* **Host only.** The player running the game rewinds; everyone else keeps their stock Forge
  and simply receives the restored state. The network protocol is untouched, so unpatched
  clients stay compatible. They will see a version-mismatch warning in the lobby chat,
  which is harmless.

**Save and load** — *Game → Save game…* writes the position to a text file; *Game → Load
game…* puts one back mid-match; *Forge → Load saved game…* starts a match from one without
needing a game running first. Loading starts the match under puzzle rules so there is no
mulligan prompt over an empty opening hand, and restores the default maximum hand size,
which the saved format does not carry.

## Included Forge fixes

Rewinding leans on Forge's own machinery far harder than its normal use does, which brings
defects to the surface. Each is in the patch and offered to Forge separately, with a test
that fails without it.

| What was wrong | Upstream |
|---|---|
| Cards that exist in the running game but not in the snapshot — tokens, copies, effect cards — were never removed on restore, so the attachment loop dereferenced null. | [PR #11416](https://github.com/Card-Forge/forge/pull/11416) |
| Rolling back through a snapshot skipped the ability cleanup: announced values, targets and the frozen stack survived the cancelled cast. | [PR #11418](https://github.com/Card-Forge/forge/pull/11418) |
| Cards were filed under their controller rather than the player whose zone they are in, so a stolen card ended up in the wrong graveyard. | [PR #11419](https://github.com/Card-Forge/forge/pull/11419) |
| A foretold card lost its foretold state on restore — the two setters for it had sat commented out since the method was written. | [PR #11420](https://github.com/Card-Forge/forge/pull/11420) |
| A melded permanent came back as two, or as one that was both melded and loose on the battlefield. | [PR #11421](https://github.com/Card-Forge/forge/pull/11421) |
| A creature stolen after the snapshot stayed with the thief, and ended up in both battlefields at once. | not reported |
| **Loading a state lost a card's Adventure.** The permission that keeps it castable in exile was created while the exile zone was being read, and setting up the command zone straight afterwards emptied it again. Also, the loader carried its own copy of the Adventure definition, which had drifted from the one the game builds. | not reported |
| **Applying a state never cleared the stack**, it only added back what it had recorded, so anything in flight survived the load. Worked around here rather than fixed in `GameState`. | not reported |

The Adventure bug is a good illustration of the shape of these: the loader empties every
zone, then builds the cards, then fills the zones — so anything it creates while building
cards is wiped when the zones are filled. Forge had already been caught by this twice, and
patched it in place both times; the comment `would have been erased by setCards` next to
`createCommanderEffect` is the fossil record.

## Files touched

| File | Change |
|---|---|
| `forge-game/…/Game.java` | turn rewind points, `stashTurnRewindPoint`, `rewindToActionOf`, carrying command-zone effects |
| `forge-game/…/GameState.java` | Adventure permission rebuilt after the zones are in place |
| `forge-game/…/card/CardFactoryUtil.java` | one shared Adventure definition instead of two |
| `forge-game/…/GameSnapshot.java` | the snapshot fixes above |
| `forge-game/…/GameActionUtil.java` | ability cleanup on a snapshot rollback |
| `forge-game/…/phase/PhaseHandler.java` | `PriorityState` for the bookkeeping the format misses; rewind and load checks in `mainLoopStep` |
| `forge-game/…/player/PlayerController.java` | hooks for requesting a rewind, and the pending saved state |
| `forge-gui/…/player/PlayerControllerHuman.java` | rewind requests, save and load, stopping auto-pass, client resync |
| `forge-gui/…/gamemodes/match/HostedMatch.java` | reads the rewind depth from the preferences |
| `forge-gui/…/localinstance/properties/ForgePreferences.java` | `MATCH_REWIND_STEPS` |
| `forge-gui/…/net/server/FServerManager.java` | `resyncAllClients()` |
| `forge-gui-desktop/…/menus/GameMenu.java` | rewind submenu, save and load entries |
| `forge-gui-desktop/…/menus/LoadSavedGame.java`, `ForgeMenu.java` | starting a match from a saved position |
| `forge-gui-desktop/…/settings/*Preferences.java` | the *Rewind turns* setting |
| `forge-gui/res/languages/en-US.properties` | new strings |
| `forge-gui-desktop/src/test/java/forge/ai/*Test.java` | tests for the rewind and for each fix |

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
name, and copy `forge-gui/res/languages/en-US.properties` over the installed one — without
it the new menu entries show their raw keys instead of labels.

## Tests

```sh
mvn -B -pl forge-gui-desktop -am test
```

299 tests, all green. The rewind ones:

| Test | Covers |
|---|---|
| `RewindTest` | taking points, counting, depth, jumping back, turn/phase/priority |
| `RewindHandSizeTest` | maximum hand size, in both directions |
| `RewindLastingEffectTest` | rest-of-game effects (Praetor's Counsel) |
| `AdventureGameStateTest` | a card on an Adventure stays castable after a state is loaded |
| `SnapshotStolenCardTest`, `SnapshotControlTest`, `SnapshotMeldTest`, `ForetellSnapshotTest`, `RollbackCleanupTest` | the Forge fixes above |

One thing worth knowing when writing more of these: applying a state only runs on the game
thread, which `GameAction.invoke` recognises by the thread's name. A test that calls it from
an ordinary thread silently does nothing. Hence `new Thread(…, "Game-test")` in the tests.

Not covered by automated tests: the menus themselves, and the network side — a second player
receiving a rewound state has not been tried yet.

## Caveats

* Rewinding cannot un-see information. If an opponent already saw your hand, it stays seen.
* Random outcomes are re-rolled, and the AI may decide differently the second time around.
* Effects that remember a specific card for the rest of the game ("you may play it for as
  long as it remains exiled") may come back pointing at a card object the restore replaced.
  Untested — no such card has come up in play yet.

## Licence

GPL v3, same as Forge — this is derived work and cannot be anything else. See `LICENSE`.

---

**Kurz auf Deutsch:** Dieser Patch gibt Forge einen Rückgängig-Knopf: Der Gastgeber kann
einen seiner eigenen Züge von vorn beginnen, samt allem, was seitdem passiert ist.
Mitspieler brauchen keine geänderte Version. Dazu Speichern und Laden von Stellungen, auch
aus dem Hauptmenü heraus, etwa nach einem Absturz.

Speicherpunkte entstehen bewusst nur am Anfang eines eigenen Zuges — das ist der einzige
Moment, den Forges Speicherformat lückenlos beschreibt. Dafür ist der Rücksprung verlässlich
statt feinkörnig.

Beim Bauen sind acht Fehler in Forge aufgefallen, vom Absturz durch nicht entfernte
Spielsteine bis dahin, dass eine Karte auf Abenteuer nach dem Laden eines Spielstands nicht
mehr spielbar war. Jeder ist einzeln angeboten oder dokumentiert, mit Test, der ohne die
Korrektur fehlschlägt.
