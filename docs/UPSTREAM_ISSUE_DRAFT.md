# 上游 Issue 草稿 — Suggestion 模板

仓库：Minecraft-Transit-Railway/Minecraft-Transit-Railway
模板：Suggestion (`.github/ISSUE_TEMPLATE/suggestion.yaml`)
标题：`Signal desk and interlocking system with manual route setting`

---

## 字段 1 — Suggestion Type（下拉框）

选择：**Train simulation mechanics**

---

## 字段 2 — Suggestion

Right now signals in MTR work as automatic block signals: a train reserves the coloured rail
spans ahead of it, and the aspect shown is a consequence of occupancy. There is no way for a
player to act as a signaller — to decide which way a train goes through a junction, to hold a
train at a signal, or to give one train priority over another.

I would like to suggest a **signal desk** and a **server-side interlocking system**, so that
junctions can be worked by hand the way a real signalling control centre works.

Two new pieces of content:

**1. Signal Dashboard** — a handheld, OP-gated item for administration. It defines *signal
desks*: each desk gets a control area, a list of signals and sets of points it is authorised to
operate, and a per-desk player allow/block list. This lets a server owner split a large network
into control areas and hand each one to a different player, without giving anyone the ability to
touch the whole map.

**2. Signal Desk** — a block you sit at. Sitting down opens a control screen showing a schematic
track diagram of the desk's area, with live track occupancy, signal aspects and point positions.
Routes are set with the standard **entrance–exit (NX)** method used by real signalling
workstations:

| Input | Action |
|---|---|
| Click the entrance signal, then the exit signal | Set the route; points move and lock automatically |
| Right-click the entrance signal | Cancel the route; the signal returns to danger |
| Click a set of points | Move them individually (refused while locked by a route) |

The interlocking runs on the server and refuses any route that conflicts with another, is already
occupied, or needs points that are locked. If a route is set the wrong way for a train, the train
is held at the protecting signal until the signaller corrects it — which is the whole point of the
feature, and something that is currently impossible to build in MTR.

This would give players a completely new role to play on a server. Today a multiplayer MTR network
has drivers and builders; this adds **signallers**, and makes busy station throats something a
person actually operates rather than something that resolves itself automatically.

---

## 字段 3 — Assets

None. No third-party assets are proposed, so there is no licensing question to resolve.

I am not suggesting any particular look for the new block or item — those are yours to design. The
control screen, however, is intended to be drawn entirely in code (rectangles, lines and the
vanilla font, with colours as constants), so it carries no texture dependency and needs no art
work from your side.

---

## 字段 4 — Implementation Details and References

I have written a full design document for this. It is in my fork, in Chinese:

https://github.com/TonyD365/Minecraft-Transit-Railway/blob/master/docs/SIGNAL_SYSTEM_PLAN.md

I am happy to translate it to English if that would help review.

### Why this fits the existing engine

Before designing anything I checked what the simulation core already allows, since the whole
proposal depends on it. Three findings:

**1. Manual rail blocking already exists.** `org.mtr.core.data.Rail` exposes
`blockRail(LongArrayList)`, and `BlockSignalBase` already calls it via `PacketBlockRails` on a
redstone input. An interlocking can therefore hold points by blocking the diverging leg — no
change to the simulation core is needed.

**2. Stopping points come from rail signal colours, not from signal blocks.** Everything in the
core that decides whether a train may proceed (`getSignalColors`, `reserveRail`, `isBlocked`) is
rail-level; `BlockSignalBase` only reads state to pick a lamp aspect. This means a route can hold
a train at a chosen boundary without requiring a signal block to be placed at every set of points,
which matches real practice — one home signal protects a whole station throat.

**3. `MANUAL_BLOCK_DURATION` is 1000 ms.** A manual block lapses after a second, so locks have to
be re-issued every tick as a heartbeat. This is the main implementation risk and the thing I would
validate first. Because a lapsed block makes a rail *passable*, the design additionally drops the
entrance signal to danger whenever a heartbeat is missed, so the failure mode stays safe.

### Scope

The design covers route locking, conflict checking, automatic point calling, **approach locking**
(cancelling a route in a train's face holds the points for a timed release instead of releasing
them immediately), **overlap**, **flank protection**, route release, and an OP-only emergency
release. All validation is server-side; the screen is a renderer and an input device only.

Deliberately left out of a first version, to keep the scope reviewable:

- **Sectional release** — a first version releases the whole route at once, so one train at a time
  through a throat. The route stores its rails as an ordered list so this can be added later
  without a data migration.
- **Automatic route setting (ARS)** — every route is set by hand. Timetable-driven route setting
  would be a natural follow-on.
- Swinging overlaps; unusual point layouts (slips, crossings, ladders) may need manual correction
  from the dashboard.

### References

- Entry/exit (NX) route setting — https://www.jmri.org/help/en/html/tools/EntryExit.shtml
- Evolution of signalling control — https://www.railengineer.co.uk/evolution-of-signalling-control/
- Overlap and flank protection — https://www.railwaysignallingconcepts.in/overlap-flank-protection-railway-signalling/
- Route locking circuits — https://www.railwaysignallingconcepts.in/route-locking-circuit-railway-signalling/
