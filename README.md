# HOPE

**H**euristic **O**ffensive **P**ursuit **E**ngine — a survival-duelist bot for the
`mechfight/1` 16-mech battle royale.

HOPE plays for placement, not highlight reels. It loots efficiently, picks fights
it can win, refuses the ones it can't, and treats the shrinking zone as the real
opponent. Over 48 practice matches against a 15-bot squad that shares vision and
hunts it down, the current build averages a placement score of **~6.38**.

---

## How it plays

The squad never fights itself in practice, so every place HOPE gains has to be
taken from a hunter. That shapes the whole policy.

### 1. Farm the map, not the fight

- Grabs the nearest useful weapon first, ranked **lightning > laser > mortar**.
- Then cubes, uncontested supply drops (3 cubes + a full heal), NPC drones
  kited at ~60 m, and crates.
- Every loot target is filtered twice: it must sit inside the *next* zone, and
  it must be far from any hunter HOPE remembers.

### 2. Take the 1v1, dodge the 2v1

- Dueling is target-locked and range-aware, using the weapon triangle as a set
  of distance bands: lightning inside 30 m (240 DPS plus the gun stacked),
  laser around 85 m, mortar lobbed over cover with a ballistic solver when
  there's no line of sight.
- Against enemy lightning it stays outside 45 m.
- Strafing is randomized every tick and tightens into harder jukes when a laser
  starts charging.
- Outnumbered? It doesn't take the fight.

### 3. Break contact, then heal

When it's losing, low, or outnumbered, HOPE disengages through the building
graph toward the zone while still facing and shooting the pursuer. Once line of
sight is broken it exploits the regen window: 5 seconds out of combat, then
45 HP/s. Sneaking below 5 m/s produces no footstep audio.

### 4. Respect the clock

- Zone urgency is driven by the shrink timer.
- Loot is only chased if the next circle is still reachable in time.
- Full-heal drops are contested only when no hunter is nearby.

### 5. Upgrade with intent

Priority order: reactive armor → plating → repair nanites → gun damage/rate,
then weapon-specific picks for whatever is actually in hand.

---

## Architecture

| File | Role |
| --- | --- |
| `hope.py` | The bot: policy, world model, navigation, combat. Standard library only. |
| `mechfight.py` | Official kit wrapper — protocol loop, newest-observation rule. Vendored unmodified. |
| `Dockerfile` | `python:3.13-slim`, runs `hope.py` as the entrypoint. |

### World model

Built once during `hello`, from the map mounted at `/arena` in matches:

- **Terrain graph.** A 4 m heightfield covering ground and building roofs, read
  from `v2/world.json` prisms and `v2/terrain.i16`. Edge passability is
  precomputed against the 1.2 m step-up rule.
- **Pathing.** Bounded A* that replans at most every 1.6 s, with greedy
  line-of-sight shortcutting, jump/jet routes to roof loot, and stuck detection
  that recovers with a sidestep.
- **Visibility.** Ray casts against prism volumes and terrain, so the bot never
  dumps rounds into a wall — it switches to the mortar arc instead.

### Performance

Measured with the kit's `check/check.py` on the development machine:

| percentile | time |
| --- | --- |
| p50 | 0.2 ms |
| p99 | < 1 ms |
| worst | ~20 ms |

Budget is 50 ms per tick.

---

## Running it

Download `mechfight-kit.tar.gz` from the track page and unpack it next to this
repo. The `kit/` directory is development-only and is git-ignored.

Local check, no container:

```sh
python3 kit/check/check.py -- python3 -u hope.py
```

Containerized check, mirroring the match sandbox (no network, read-only root,
1 CPU, 512 MB, arena mounted read-only):

```sh
docker build -t hope .

python3 kit/check/check.py --map-dir /arena -- docker run --rm -i --network none \
  --cpus 1 --memory 512m --read-only --tmpfs /tmp:rw,size=64m \
  --user 65534:65534 -v "$PWD/kit/arena:/arena:ro" hope
```

---

## Notes

- Standard library only — no third-party dependencies, no build step.
- `mechfight.py` is the unmodified official wrapper and must keep its name: the
  bot imports it as `mechfight`.
- The only code beyond the kit is `hope.py`.
