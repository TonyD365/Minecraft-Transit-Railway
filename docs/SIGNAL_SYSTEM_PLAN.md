# Signal Desk & Interlocking System — Design Plan

> Status: design / not yet implemented
> Target: this fork (`master`). Assets are placeholders; upstream supplies final models & textures.

---

## 0. Scope

Add a prototypical **interlocking (联锁) and signal control system** on top of MTR's existing
rail + signal-colour mechanics:

1. **Signal Dashboard** (handheld item, OP-only) — administration: define desks, control areas,
   which signals/points each desk may control, and the desk's player allow/block list.
2. **Signal Desk** (block + seat) — operation: sit down, get a Hong Kong OCC-style control
   screen, set and cancel routes, operate points individually.
3. **Interlocking engine** (server side) — route locking, conflict checking, automatic point
   setting, approach locking, route release, and holding trains at a signal until the route is
   correct.

The UI is a **re-creation of the visual style** of a Hong Kong MTR OCC / NX signaller workstation,
built from public reference material. No third-party code is copied into this GPL project.

---

## 1. Engine constraints (verified against the code)

These are facts established by reading `libs/Transport-Simulation-Core-0.0.1.jar` and the mod
source. Everything in this plan is shaped by them.

### 1.1 The simulation core is a pre-compiled black box

`Transport-Simulation-Core-0.0.1.jar` owns train movement and block reservation. We cannot modify
it. We can only call its public API.

### 1.2 Train stopping points are defined by rail signal colours, NOT by signal blocks

From `org.mtr.core.data.Rail`:

```java
public IntAVLTreeSet getSignalColors();
public void applyModification(SignalModification);
public void blockRail(LongArrayList colors);
private static void reserveRail(...);
boolean isBlocked(long, Rail$BlockReservation);
```

Every one of these is **rail-level**. The core has no concept of a signal *block* in the world.

`BlockSignalBase` only:
```java
public int getActualAspect(boolean occupied, boolean isBackSide)  // reads state -> lamp colour
public void checkForRedstoneUpdate(...)                           // redstone -> trigger a block
```

**Consequence:** a train stops at the boundary of a *signal-colour span on the rails*. We choose
where trains stop by choosing where to paint colour spans. Physical signal blocks are annunciators
(标示器) only. No signal block is needed at a set of points.

### 1.3 Manual blocking expires after 1 second

```java
private static final int MANUAL_BLOCK_DURATION = 1000;   // milliseconds
private long manualBlockCooldown;
```

`blockRail()` holds for **1000 ms only**. To hold points locked for the life of a route, the server
must **re-issue the block every tick** — a heartbeat (心跳/保活). This is the single largest
technical risk in the plan and is validated first (Phase 2).

### 1.4 There is no player permission system in this mod

No `hasPermissionLevel` check exists anywhere in `org/mtr/mod`. We are adding the first one.
This creates a bootstrapping hole: if the Dashboard were craftable by anyone, a blocked player
could simply craft one and re-add themselves. Therefore **the Dashboard is OP-gated server-side**,
not merely hidden.

---

## 2. Domain model

### 2.1 `SignalDesk` — 信号桌

| Field | Type | Meaning |
|---|---|---|
| `id` | `long` | unique |
| `name` | `String` | shown on the screen header |
| `areaMin`, `areaMax` | `BlockPos` | the cuboid this desk controls |
| `controlledSignalIds` | `LongOpenHashSet` | signals the desk may operate |
| `controlledPointIds` | `LongOpenHashSet` | point groups the desk may operate |
| `access` | `SignalDeskAccess` | player allow/block list |
| `protectionMode` | `ProtectionMode` | `AREA_ENTRY` or `AT_POINTS` |
| `deskBlockPos` | `BlockPos` | the physical desk |

### 2.2 `SignalDeskAccess` — 黑白名单

```java
enum Mode { OPEN, ALLOWLIST, BLOCKLIST }
Mode mode;                                  // default OPEN
ObjectOpenHashSet<UUID> players;            // UUIDs, never names (players rename)
boolean opBypass;                           // default true
```

Checked **on sit-down and again on every action packet** — a player may be removed from the list
while already seated.

### 2.3 `Signal` — 逻辑信号机

Anchored to an **existing MTR signal block** placed in the world. Auto-discovered by scanning the
desk's area; no new block type is needed.

| Field | Meaning |
|---|---|
| `id` | derived from `BlockPos` |
| `pos`, `facing` | where it stands, which way it reads |
| `guardedRailIds` | the rails immediately beyond it |
| `signalColors` | the colours it owns (from the existing block entity) |
| `aspect` | current displayed aspect, see §4 |
| `heldRouteId` | the route currently cleared from this signal, or `null` |

Signals are placed at **route boundaries** (station throats, platform ends) — not at points.

### 2.4 `PointGroup` — 道岔组

Auto-discovered: any rail **node with more than two connections** is a junction.

| Field | Meaning |
|---|---|
| `id` | derived from node `Position` |
| `nodePos` | the junction |
| `normalRailId` | 定位 — the straight / main leg |
| `reverseRailId` | 反位 — the diverging leg |
| `currentPosition` | `NORMAL` / `REVERSE` |
| `lockedByRouteId` | non-null = locked, cannot be moved by hand |
| `isFlankFor` | routes using it purely as flank protection |

"Moving the points" = choosing which leg stays unblocked. The other leg is held blocked by the
heartbeat.

### 2.5 `SignalRoute` — 进路

| Field | Meaning |
|---|---|
| `id` | unique |
| `entranceSignalId` | N — where the train starts |
| `exitSignalId` | X — where the route ends |
| `railIds` | every rail in the route, in order |
| `requiredPoints` | `Map<pointId, NORMAL/REVERSE>` |
| `overlapRailIds` | safety overrun beyond the exit signal (§5.4) |
| `flankPointIds` | points locked away from the route (§5.5) |
| `state` | see §3 |
| `approachLockTimer` | ms remaining before a cancellation takes effect |

---

## 3. Route state machine — 进路状态机

```
                  reject (reason shown)
                 ┌──────────────┐
                 │              │
  IDLE ──set──> REQUESTED ──ok──> LOCKED ──train approaches──> APPROACH_LOCKED
    ^                                │                              │
    │                                │ cancel (no train near)       │ train passes signal
    │                                │ = immediate                  v
    │                                v                          OCCUPIED
    └────────────── RELEASING <──────┴──────── train clears route ──┘
```

| State | 中文 | Meaning |
|---|---|---|
| `IDLE` | 空闲 | nothing set; entrance signal at Danger |
| `REQUESTED` | 请求中 | checks running (one tick) |
| `LOCKED` | 已锁闭 | points set & locked, entrance signal cleared |
| `APPROACH_LOCKED` | 接近锁闭 | a train is approaching on a proceed aspect |
| `OCCUPIED` | 占用 | train is inside the route |
| `RELEASING` | 解锁中 | train clear, locks dropping |

---

## 4. Signal aspects — 信号显示

MTR's `getActualAspect` returns `0..3`. Mapping:

| Value | Aspect | 中文 | Shown when |
|---|---|---|---|
| `1` | **Danger** (red) | 停车 | no route set from this signal, or route occupied |
| `3` | **Caution** (yellow) | 注意 | route set, but the section beyond the exit is occupied |
| `2` | **Preliminary caution** (double yellow) | 预告注意 | route set, two sections ahead occupied (4-aspect only) |
| `0` | **Clear** (green) | 进行 | route set and the road ahead is clear |

A signal is only cleared by the interlocking — never directly by the player. The player asks for a
*route*; the aspect is a consequence.

---

## 5. Interlocking logic — 联锁逻辑

Runs server-side, every tick, in `SignalInterlocking.tick()`.

### 5.1 Setting a route (NX)

Player clicks entrance signal N, then exit signal X.

```
1. PATH      find the rail path N -> X. No path -> reject "NO ROUTE".
2. AUTHORITY every signal and point on the path must be in this desk's
             controlledSignalIds / controlledPointIds. Otherwise -> reject "NOT CONTROLLED".
3. CONFLICT  no rail in railIds or overlapRailIds may belong to another
             non-IDLE route -> reject "CONFLICTING ROUTE".
4. POINTS    every required point must be free (lockedByRouteId == null)
             or already locked to the same position -> reject "POINTS LOCKED".
5. CLEAR     no train may currently occupy railIds -> reject "TRACK OCCUPIED".
6. OVERLAP   the overlap beyond X must be free -> reject "OVERLAP NOT AVAILABLE".
7. FLANK     flank points must be movable to their protecting position
                                                   -> reject "FLANK NOT AVAILABLE".

  -> all pass:
8. CALL      move every required point (including flank) to position.
9. LOCK      set lockedByRouteId on every point; state = LOCKED.
10. BLOCK    start the heartbeat blocking every diverging leg not in the route.
11. CLEAR SIGNAL  unblock the route's own rails; N shows a proceed aspect (§4).
```

Rejections are **not silent** — the reason is written to the desk's message log (§7.5) and the
attempted route flashes red for ~2 s.

### 5.2 Cancelling a route — 取消进路

Player right-clicks the entrance signal.

Two cases, matching real practice:

* **No train approaching** → immediate. Signal to Danger, points unlocked, route `IDLE`.
* **Train approaching on a proceed aspect** (`APPROACH_LOCKED`) → **approach locking (接近锁闭)**.
  The signal returns to Danger at once, but the points stay locked for a
  **timed release** (default 120 s, configurable). This is the real-world protection against
  pulling the points out from under a driver who has already seen a green.
  The desk shows a live countdown on the route.

A route already `OCCUPIED` cannot be cancelled at all — only an **Emergency Release** (§6.4) by an
OP can break it.

### 5.3 Releasing a route — 进路解锁

**v1 — route release (整进路解锁).** When the last rail of the route becomes clear of the train,
every lock drops at once.

**v2 — sectional release (分段解锁).** Locks drop rail-by-rail behind the train, so a following
train can enter the throat sooner. Not in v1, but `railIds` is stored as an ordered list
specifically so this can be added without a data migration.

### 5.4 Overlap — 安全超距

A short run of track beyond the exit signal, reserved with the route, so that a train that
overruns X by a little still has protected track. Default: the next rail beyond X, configurable
per signal. Locked and released with the route.

### 5.5 Flank protection — 侧防

Points *not on* the route but which could let another train run into its flank are forced to the
position that leads **away** from the route, and locked there. Catches the classic "another train
rolls out of a siding into my path" case.

### 5.6 The heartbeat — 心跳封锁

```java
// every server tick, for every route in LOCKED / APPROACH_LOCKED / OCCUPIED
for (long railId : route.blockedLegRailIds) {
    rail.blockRail(route.blockColors);   // expires after 1000 ms, so re-issue
}
```

If the server stalls for more than a second the locks lapse and re-apply on the next tick. That is
fail-safe in the right direction: a lapsed *block* makes a rail passable, so we additionally hold
the entrance signal at Danger whenever the heartbeat has missed a beat, rather than trusting the
block alone.

### 5.7 Holding a train at the wrong route — the behaviour we are after

The originally requested behaviour, restated precisely:

> A route is set to platform B, but the train is booked for platform A.
> The leg towards A is held blocked by the heartbeat. The train cannot reserve it, so it stops at
> the boundary of the blocked colour span. The signaller cancels the route and re-sets N -> A.
> The block lifts, the signal clears, the train proceeds.

Where exactly it stops is set by `protectionMode`:

| Mode | Stops at | Notes |
|---|---|---|
| `AREA_ENTRY` (default) | the entrance signal to the whole throat | Prototypical. One signal protects many points. No signal is needed at the points themselves. |
| `AT_POINTS` | immediately before the points | Easier to see while debugging. Still needs no signal block there. |

Both are the same code — only the position of the painted colour-span boundary differs.

---

## 6. Operations — 操作方式

### 6.1 At the desk (`SignalDeskScreen`)

| Input | Action | 中文 |
|---|---|---|
| Left-click signal N, then left-click signal X | Set route N -> X; points move automatically | 排进路 |
| Right-click entrance signal | Cancel route (immediate, or approach-locked countdown) | 取消进路，信号转红 |
| Left-click a point group | Toggle NORMAL / REVERSE — refused if locked by a route | 单动道岔 |
| Left-click a signal, then Esc / right-click empty space | Abandon the pending entrance selection | 撤销选择 |
| Hover anything | Tooltip: id, state, owning route, lock reason | |
| Scroll / drag | Zoom and pan the track diagram | |

The pending entrance signal is highlighted so the operator always knows a half-finished NX
operation is outstanding.

### 6.2 At the dashboard (`SignalDashboardScreen`, OP only)

* List / create / delete / rename desks
* Drag a rectangle on a `WidgetMap` to define the control area
* Two checkbox lists: signals in the area, point groups in the area — tick to authorise
* Access panel: mode dropdown (`OPEN` / `ALLOWLIST` / `BLOCKLIST`), player list, add/remove,
  `opBypass` checkbox
* `protectionMode` dropdown
* **Validation warnings**, shown inline:
  * a point group in the area with no signal protecting any approach to it
  * two desks whose authorised sets overlap (both could fight over the same points)
  * a signal authorised but its rails lie outside the area

### 6.3 Sitting down

Right-click the desk → server checks `SignalDeskAccess` → spawns `EntitySignalDeskSeat`, player
rides it, view snapped towards the screen, then the GUI opens. Standing up, dying, or
disconnecting removes the seat entity. Access is re-checked on every subsequent action packet.

### 6.4 Emergency release — 紧急解锁

OP-only, confirmation dialog, written to the message log with the player's name. Force-drops every
lock on a route regardless of state. The escape hatch for a desynchronised or stuck route.

---

## 7. The screen — Hong Kong OCC style

Re-created in `ScreenExtension`, reusing `WidgetMap` for pan/zoom. Visual language follows a
Hong Kong MTR OCC / NX signaller workstation.

### 7.1 Palette

| Element | Colour |
|---|---|
| Background | near-black `#0A0A0A` |
| Track, unoccupied & unset | dark grey `#555555` |
| Track, route set | white `#FFFFFF` |
| Track, occupied by a train | red `#FF3030` |
| Track, locked but not yet occupied | amber `#FFB000` |
| Points, reverse | highlighted leg drawn thicker |
| Signal at Danger | red dot |
| Signal at Caution | yellow dot |
| Signal at Clear | green dot |
| Selected entrance signal | flashing white ring |

### 7.2 Track diagram

Schematic, not geographic: rails are straightened to horizontal runs with short 45° links at
points, the way a real control diagram is drawn. Point groups render as a stub diverging from the
main run, with the set leg drawn solid and the unset leg dimmed.

### 7.3 Train describer — 列车标示

Occupied sections carry a small box with the train's number / route, following it from section to
section as it moves. This is what makes the screen feel like the real thing.

### 7.4 Header

Desk name · controlled area · in-game time · connection state.

### 7.5 Message log

Bottom-right, scrolling, newest first. Every route set, cancellation, rejection (with reason),
point movement, and emergency release, timestamped and attributed to a player.

### 7.6 Bottom button bar

`Set Route` · `Cancel Route` · `Individual Points` · `Emergency Release` (OP) · `Zoom to Fit`
— mirroring the mode buttons on a real workstation, for players who prefer buttons to
click-sequences.

---

## 8. Security model

| Layer | Gate | Enforced where |
|---|---|---|
| Dashboard item | OP (`hasPermissionLevel(2)`) | server, in the packet handler |
| Sitting at a desk | `SignalDeskAccess` | server, in `onUse` |
| Any desk action | `SignalDeskAccess` **re-checked** + desk's authorised signal/point sets | server, in every action packet |
| Emergency release | OP | server |

**Nothing is trusted from the client.** The client screen is a renderer and an input device; every
decision is made server-side and broadcast back.

---

## 9. File plan

```
data/
  SignalDeskData.java          desk record + NBT/JSON serialisation
  SignalDeskAccess.java        allow/block list
  SignalRoute.java             route record + state
  PointGroup.java              points record
  SignalRef.java               logical signal wrapper over the existing block entity
  SignalDeskRegistry.java      world-save persistence, all desks
  SignalInterlocking.java      the engine: tick, set, cancel, release, heartbeat
  SignalPathfinder.java        rail-graph search N -> X, collecting points

block/
  BlockSignalDesk.java         block + block entity   [placeholder model]

item/
  ItemSignalDashboard.java     OP-gated handheld      [placeholder texture]

entity/
  EntitySignalDeskSeat.java    invisible seat

screen/
  SignalDashboardScreen.java   admin GUI
  SignalDeskScreen.java        operator GUI (OCC style)
  SignalDeskDiagram.java       track diagram renderer

packet/
  PacketUpdateSignalDesk.java     dashboard -> server (config)
  PacketSignalDeskAction.java     desk -> server (set/cancel/toggle/emergency)
  PacketSignalDeskState.java      server -> desk clients (live state broadcast)

resources/
  blockstates, models, textures, loot table, recipe   [all placeholders]
  lang: en_us, zh_cn
```

Every placeholder asset carries `TODO: placeholder asset — to be replaced upstream`, so the art
and the logic can be reviewed separately.

---

## 10. Build order

| Phase | Deliverable | Validates |
|---|---|---|
| 1 | Data model, desk block, dashboard item, seat entity, placeholder assets | Place it, sit on it, stand up cleanly |
| 2 | **Heartbeat blocking** — hold one rail blocked indefinitely | The 1000 ms expiry is beaten. **Highest risk — done early.** |
| 3 | Dashboard GUI: OP gate, area selection, signal/point authorisation, allow/block list | Config persists across a restart |
| 4 | Desk GUI, static: track diagram, signals, points, occupancy colours | The OCC screen exists and is readable |
| 5 | Pathfinder + route setting + all seven interlocking checks + automatic point calling | Setting a route moves the points and clears the signal |
| 6 | Cancellation, approach locking countdown, route release, **holding a train for a wrong route** | The core requested behaviour |
| 7 | Overlap, flank protection, emergency release, message log, full server-side validation | Safety complete |
| 8 | Compile, in-game test, commit to `master` | Shipped |

Phase 2 is deliberately second: if the heartbeat cannot hold, phases 5–7 need a different
mechanism, and it is far cheaper to learn that before the GUI exists.

---

## 11. Known limitations of v1

* **Route release, not sectional release** — one train per throat at a time. Data is already
  shaped for the v2 upgrade.
* **No automatic route setting (ARS)** — every route is set by hand. Timetable-driven automatic
  route setting is a natural follow-on.
* **Overlap is a fixed single rail**, not a speed-dependent swinging overlap.
* **Point detection is inferred** from rail-graph topology; unusual layouts (slips, crossings,
  ladders) may need manual correction from the dashboard.
* **The core is a black box** — if a future core release changes `blockRail` semantics or the
  1000 ms constant, the heartbeat must be revisited.

---

## References

* Entry/Exit (NX) route setting — <https://www.jmri.org/help/en/html/tools/EntryExit.shtml>
* Evolution of signalling control — <https://www.railengineer.co.uk/evolution-of-signalling-control/>
* Overlap & flank protection — <https://www.railwaysignallingconcepts.in/overlap-flank-protection-railway-signalling/>
* Route locking circuits — <https://www.railwaysignallingconcepts.in/route-locking-circuit-railway-signalling/>
* Route release circuits — <https://www.railwaysignallingconcepts.in/railway-route-release-circuits/>
* SACEM — <https://en.wikipedia.org/wiki/SACEM_(railway_system)>
* MTR CBTC upgrade — <https://www.itsinternational.com/news/hong-kongs-mtr-upgrades-signalling-cbtc>
