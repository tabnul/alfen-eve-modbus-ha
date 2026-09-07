# Alfen Eve — native Modbus EV charging control for Home Assistant

Solar-surplus and dynamic-price control of an **Alfen Eve Pro-line (NG9xx)** EV charger
over **native Modbus TCP**, driven entirely from Home Assistant. No custom integration —
just the built-in `modbus` platform plus template sensors, a couple of scripts, and four
automations (one Solar control loop, a keep-alive, a fuse guard, and mode routing).

The charger runs in EMS/slave mode: Home Assistant is the sole brain. It writes a single
current setpoint, switches between 1- and 3-phase, and keeps every phase under the fuse.
Charging follows one of three modes selected from a dropdown.

## What it does

- **Solar** — charges from genuine solar surplus only. It always starts on 1-phase,
  modulates the car's current to track the surplus, and switches up to 3-phase once
  there's enough sustained surplus to run three phases (then back down as surplus fades).
  Start, stop and phase changes each wait out a configurable hold, and phase switches are
  further capped by a cooldown, so it stays calm on a cloudy day.
- **Fast** — 3-phase at maximum current (clamped by the car max and the fuse guard).
  Ignores surplus — will import from grid. Recoveries use a dedicated, configurable buffer loop
  gated by a post-stop recovery cooldown (`input_number.alfen_stop_cooldown`).
- **Off** — pauses the car.
- **Per-phase fuse protection** — an always-on guard that keeps every phase under your
  fuse rating by *adjusting* the charge current down when a phase is loaded and letting recovery
  loops handle stepping power back up. It includes configurable telemetry settle-filtering
  and optional 6 A hard-floor protection to eliminate false triggers and charge session restarts.
- **Survives restarts** — the selected mode and all tuning values persist across an HA
  restart. Fast re-applies on boot; Solar picks back up on the next control tick.
- Full metering, status and control surfaced as HA entities, plus a dashboard.

## How the Solar loop works

One automation (`alfen_solar`) runs every 20 s (and on entering Solar) and dispatches a
single action per tick via a `choose`, so at most one setpoint write happens per tick —
this is what keeps it from disturbing the car with rapid writes. It is **level-triggered**:
each tick re-checks the current conditions rather than relying on a threshold-crossing
edge, so it can't get stuck because a level was already crossed when a setting changed.

Durations ("surplus above X for N minutes") are measured from the `last_changed` of three
threshold binary sensors (`alfen_surplus_over_start/up`, `alfen_surplus_under_down`). The
branches, in priority order: start (if stopped and surplus has held), stop (if charging
and surplus has fallen — resets to 1-phase as it stops), fuse-pause, phase-up 1→3,
phase-down 3→1, then modulate (with a ≥1 A deadband). Phase switches use a
pause→switch→resume script and are gated by a cooldown.

## Requirements

- Alfen Eve Pro-line / S-line on the **NG9xx** platform (this was built and tested on
  firmware **7.4.5**).
- The **Active Load Balancing** licence on the charger (paid unlock from Alfen). Without
  it the Modbus slave functionality can't be enabled.
- **ACE Service Installer** access (a service-level Alfen account) to configure the
  charger — see below. **Set the Modbus setpoint Validity time to 300 s** there (not the
  60 s default) — this keeps charging alive across an HA restart and is EVCC's recommended
  value; details in the charger-configuration section.
- A **HomeWizard P1** meter (or any P1 meter) exposing **total** and **per-phase** active
  power. Grid import must read positive, export negative.
- Home Assistant with the built-in `modbus` integration, packages enabled, and the
  **apexcharts-card** HACS card for the dashboard graphs.

### Site assumptions to adapt

Two things are specific to the reference install and must be changed for another setup:

- **1-phase charging happens on L1.** The per-phase headroom and fuse logic treat L1 as
  the phase that carries the car in 1-phase mode. If your charger uses a different phase,
  adjust the L1 references.
- **P1 entity names** are `sensor.p1_meter_power` (total) and
  `sensor.p1_meter_power_phase_1/2/3`. Change these throughout if yours differ.

## Charger configuration (ACE Service Installer)

All of this is set in the **ACE Service Installer** app, connected to the charger, under
**Load balancing**. The Eve Manager (end-user) app cannot set most of it.

**Active balancing:**

- Active Load Balancing — **on**
- Data Source — **Energy Management System** (this puts the charger in slave role)
- Safe current — the current the charger falls back to when HA stops writing for longer
  than the validity time. See note below.
- Allow single-/multiphase charging — **on** *(required, or the phase register does
  nothing)*
- Phase rotation — as wired (e.g. L1L2L3)

**TCP/IP EMS:**

- Mode — **Socket**
- Validity time — **300 s** *(recommended — see below; do not leave at the 60 s default)*

**Set the validity time to 300 s, not the 60 s default.** This is the window the charger
keeps a written setpoint valid before falling back to Safe current. EVCC recommends 300 s
for Alfen, and it buys two things: (1) an HA restart (which takes well under 5 min) no
longer causes the charger to drop the setpoint mid-charge — charging continues across the
reboot; and (2) generous margin against any write-timing jitter, so a delayed keep-alive
can never trip the fallback. The keep-alive still rewrites every 30 s, now comfortably
inside the window. There is no downside to 300 s for an EMS/slave setup.

On firmware 7.4.5 there are **no separate "Allow reading / Allow writing maximum
currents" checkboxes** — enabling Active Load Balancing with the EMS data source and
Socket mode is what permits Modbus writes. On some other firmware those toggles exist
(sometimes only under *Advanced Settings*); if present, both must be on.

### Safe current

Safe current is the charger's own fallback for when HA goes silent (restart, network
drop, the keep-alive dying). During normal operation your setpoint drives everything and
safe current is dormant. Because HA is the sole controller, a fallback should ideally be
conservative — but any sane value works. Note: while a car is connected and HA is *not*
writing, the charger will charge at safe current on its own, outside all HA logic.

## Files

| File | Contents | Where it goes |
| --- | --- | --- |
| `alfen_modbus.yaml` | The `modbus:` read layer — every register as a sensor. | `modbus: !include alfen_modbus.yaml` in `configuration.yaml` |
| `packages/alfen.yaml` | The generic engine: helpers, template sensors, scripts, and the control + protection automations. | `packages/` directory (packages must be enabled) |
| `optional_alfen_price.yaml` | **Site-specific, not reusable.** Dynamic-price/window wiring that only selects the charge mode. Depends on this install's price binary_sensors and input_booleans. | `packages/` directory |
| `alfen_dashboard.yaml` | The sections-view dashboard. | Lovelace dashboard config |

Enable packages in `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Set the charger's **IP** in `alfen_modbus.yaml` (`host:`). Give the charger a **static
IP / DHCP reservation** — the modbus hub reads `host:` only at startup and doesn't
follow a helper, so a moving IP silently breaks the connection.

The engine (`alfen_modbus.yaml`, `packages/alfen.yaml`, `alfen_dashboard.yaml`) is
generic and portable. `optional_alfen_price.yaml` is an example of a site layer bolted on
top; its only contract is **"set `input_select.alfen_charge_mode`, nothing else"** — it
never writes current or phases directly, so it can't fight the engine.

## How it works

**Setpoint.** All current control is a single float32 write to register **1210** (the
"Modbus Slave Max Current" setpoint). The `alfen_set_current` script packs the amps into
two 16-bit words (big-endian IEEE-754, single multi-register write, as the charger
requires) and records the value into `input_number.alfen_target_current`. Any write below min current
automatically timestamps `input_datetime.alfen_last_stop`.

**Keep-alive.** The setpoint falls back to Safe current if it isn't refreshed within the
configured validity time (set to 300 s — see charger config). `alfen_setpoint_renew`
rewrites 1210 every 30 s whenever a car is connected — far inside the window. It is a
**standalone** automation (not part of the control loop) so nothing can delay it — this
is what keeps the charger off its safe-current fallback.

**Phase switching.** Writing `1` or `3` to register **1215** switches phases, but doing so
live makes the car re-negotiate and can fault. `alfen_switch_phases` therefore does
**pause → wait 10 s → write 1215 → wait 5 s → resume at min**. Expect a ~15 s charging gap
on every phase change. Each switch stamps `input_datetime.alfen_last_phase_switch`
(as a Unix timestamp) for the cooldown guard.

**Control loop.** `alfen_solar` runs every 20 s and on entering Solar, dispatching one
action per tick via a `choose`. Modulation computes the target as
`min(surplus / (phases × 230), max_current, headroom)`, floored at 6 A, and writes only on
a ≥1 A change (deadband). Because it re-checks every tick (level-triggered) rather than
firing on a threshold-crossing edge, it can't get stuck if a level was already crossed
when a setting changed.

**Solar surplus.** `alfen_solar_surplus = charger_power − grid`. It is independent of the
car's own draw (the car's power appears in both terms and cancels), so a running car
doesn't hide the surplus. Three binary sensors flag it against the start / phase-up /
phase-down thresholds; the loop measures how long each has held from its `last_changed`.

**Phase decisions.** Solar always starts on **1-phase**. It switches up to 3-phase only
once surplus holds above `alfen_phase_up_threshold` for the phase hold, and drops back to
1-phase when surplus holds below `alfen_phase_down_threshold`. A **cooldown**
(`alfen_phase_cooldown`) caps how often a switch can happen in either direction, and a
full stop resets to 1-phase so a paused charger never sits on 3-phase.

**Anti-oscillation.** Configurable holds on start/stop (`alfen_solar_hold`) and on phase
switches (`alfen_phase_hold`); a phase-switch cooldown (`alfen_phase_cooldown`); a configurable
post-stop recovery cooldown (`input_number.alfen_stop_cooldown`); asymmetric step buffers (`alfen_step_buffer_down` / `alfen_step_buffer_up`);
and the 1 A modulate deadband. Together these keep it from thrashing on a flickering cloudy day or bouncing on household loads.

## Fuse protection

The charger's own load balancing is off (pure EMS), so keeping the main fuse safe is HA's
job. It's a **soft** limit — reactive at the 20 s guard cadence, not a substitute for the
physical breaker — but it prevents nuisance trips in normal operation, and rather than
cutting the car off it **adjusts** the current to fit.

- **Per-phase headroom** (`alfen_headroom_l1/l2/l3`) = fuse rating − other house load on
  that phase. `alfen_headroom_active` governs the current mode: L1 in 1-phase, tightest of
  the three in 3-phase.
- **Fuse guard** (`alfen_fuse_guard`, every 20 s, **all modes**) keeps every phase under
  `alfen_fuse_per_phase`. When household appliances turn on, it steps the target current down.
- **Telemetry Settle Filter** (`alfen_guard_settle_time`): Requires headroom to remain below
  target for a configurable duration (default 5 s, max 20 s) before executing a step-down.
  This ignores phantom headroom collapses caused by P1 meter polling lag when the car renegotiates power.
- **Asymmetric Step Buffers** (`alfen_step_buffer_down` / `alfen_step_buffer_up`): Decouples
  downward safety stepping (fast/sensitive, e.g., 1 A) from upward recovery stepping
  (cautious/buffered, e.g., 3 A) to stop setpoint hunting.
- **Enforce Hard Floor** (`input_boolean.alfen_enforce_hard_floor`): When enabled, clamps the
  minimum current calculation strictly at 6 A (the IEC 61851 minimum). This prevents illegal intermediate
  setpoints (1–5 A) from forcing the charger to drop the Mode 3 state and triggering unexpected charge session restarts.

**Set `alfen_fuse_per_phase` to the total per-phase current you'll allow** (car + other
household load on that phase), a little below your real fuse — e.g. **24 A under a 25 A
fuse**. The guard holds total phase current under this figure.

## Metering — charging energy

`sensor.alfen_energy_consumed_total_kwh` is the total energy consumed by charging, in kWh
(`device_class: energy`, `state_class: total_increasing`), so it can be added to the HA
**Energy dashboard** and long-term statistics. It's derived from the charger's Wh
delivered-energy register (÷1000).

This is **AC energy fed from the charger to the car** — i.e. what you drew and paid for.
The Alfen is an AC charger, so this is the only meaningful energy figure it can report;
the AC→DC conversion losses (~10–15%) happen inside the car's onboard charger, downstream
of any meter the charger or HA can see, so "energy actually stored in the battery" is not
obtainable here — only the car's own telemetry knows that. For cost and consumption
tracking, the AC figure is the correct one. (The charger's separate `energy_consumed`
register is dead on this meter and is not used.)

## Tunable settings (helpers)

All tuning is via `input_number` / `input_select` / `input_boolean` helpers — no YAML logic edits.
None have a fixed `initial:`, so **your values persist across restarts**; each shows a suggested
starting value on the dashboard, and the templates fall back to that suggestion if a
helper is ever left blank.

| Helper | Meaning | Suggested |
| --- | --- | --- |
| `alfen_max_current` | Car's charge-rate ceiling per phase. | 16 A |
| `alfen_min_current` | The 6 A IEC floor; never charges below this. | 6 A |
| `alfen_stop_current` | The "off" value written to pause (below 6 A). | 5 A |
| `alfen_fuse_per_phase` | Total per-phase current cap (car + house). | 24 A |
| `alfen_step_buffer_down` | Minimum Ampere drop required to trigger a safety step-down. | 1 A |
| `alfen_step_buffer_up` | Minimum Ampere headroom required before Fast mode ramps back up. | 3 A |
| `alfen_guard_settle_time` | Seconds low headroom must persist to ignore P1 telemetry drops. | 5 s |
| `alfen_stop_cooldown` | Minutes Fast mode recovery must wait after a stop before restarting. | 5 min |
| `alfen_enforce_hard_floor` | Boolean: locks min setpoint to 6 A to prevent session restarts. | On |
| `alfen_phase_up_threshold` | Surplus (W) to switch 1→3 phase. | 4800 W |
| `alfen_phase_down_threshold` | Surplus (W) to switch 3→1 phase. | 4140 W |
| `alfen_phase_cooldown` | Minutes blocking any further phase switch. | 12 min |
| `alfen_solar_hold` | Minutes surplus must hold before start/stop. | 3 min |
| `alfen_phase_hold` | Minutes surplus must hold before a phase switch. | 3 min |
| `alfen_charge_mode` | Off / Solar / Fast. | — |
| `alfen_price_fallback` | Mode entered when a price window closes (price layer). | Solar |

## Register reference

Slave **200** = station; slave **1** = socket. All values big-endian, no word swap.

| Register | Meaning | Type | R/W |
| --- | --- | --- | --- |
| 1100–1101 | Station Active Max Current (hard ceiling) | float32 | R |
| 1102–1103 | Board temperature | float32 | R |
| 1201–1205 | Mode 3 state (IEC 61851 A/B/C/D/E/F) | string | R |
| 1206–1207 | Actual Applied Max Current | float32 | R |
| 1208–1209 | Setpoint remaining valid time | uint32 | R |
| **1210–1211** | **Modbus Slave Max Current (setpoint)** | float32 | R/W |
| 1212–1213 | Active Load Balancing safe current | float32 | R |
| 1214 | Setpoint accounted for (1 = charger is using it) | uint16 | R |
| **1215** | **Charge using 1 or 3 phases** | uint16 | R/W |
| 306–336 | Per-phase voltage, current, power factor, frequency | float32 | R |
| 374 / 390 | Real energy delivered / consumed (sum) | float64 | R |

`sensor.alfen_setpoint_accounted_for` (1214) is the definitive "are my writes landing"
check: **1** means the charger is using your setpoint.

## Bring-up / testing order

Do this in order so a failure points at one layer:

1. Set the real charger IP in `alfen_modbus.yaml`, load it, restart HA. Confirm sensors
   populate — voltages near 230 V, temperature sane.
2. Load `packages/alfen.yaml`. In Developer Tools call `script.alfen_set_current` with
   `amps: 6` and watch `sensor.alfen_setpoint_readback` → 6 and
   `sensor.alfen_setpoint_accounted_for` → 1. That validates the whole write path.
3. **Set `alfen_fuse_per_phase` to your real safe rating** before anything else — a wrong
   (too-low) value clamps or trips everything.
4. Try **Solar** and **Fast**, and the Force-phase buttons; in Fast, add a household load
   on a phase and confirm the fuse guard trims the car and recovers when it clears.
5. Load the dashboard; wire the price layer last.

## Known limitations & gotchas

- **Reallin meter (post-2021 units)** only expose a subset of measurement registers over
  Modbus; several aggregate registers read NaN / never populate. On this build,
  `alfen_current_sum` and `alfen_energy_consumed_sum` are dead and are not used. Voltages,
  per-phase current, charger power, and **energy delivered** all report fine — and energy
  delivered is what matters (see Metering below). Verify `sensor.alfen_power_sum` reports
  before relying on Solar.
- **Integer-amp control** — the setpoint steps in whole amps, so ~230 W (1-phase) or
  ~690 W (3-phase) of granularity per step is unavoidable. Inherent to amp-stepped
  charging, not a bug.
- **Solar has holds by design** — surplus must hold above the start threshold for the
  start/stop hold before charging begins (and below it before stopping), and above the
  phase-up threshold for the phase hold before going 3-phase. On a marginal day it pauses
  more and switches less; that's the trade-off for not thrashing. All holds are tunable.
- **DSMR4 vs. DSMR5 Smart Meter Polling Frequency:** The `alfen_fuse_guard` automation defaults
  to a 20-second evaluation tick (`seconds: "/20"`), which works reliably for DSMR4 meters
  (10-second updates). If you have a DSMR5 smart meter (1-second updates) and want the fuse
  guard to react faster to heavy household loads, open `packages/alfen.yaml` and adjust the
  trigger interval in `alfen_fuse_guard`:
  ```yaml
  triggers:
    - trigger: time_pattern
      seconds: "/5"  # Faster evaluation for DSMR5 1-second updates (or "/2", "/10")
  ```
  When lowering this interval, set **Fuse guard settle time** (`alfen_guard_settle_time`)
  on the dashboard to `2 s`–`3 s` to keep filtering out transient telemetry drops during EV current renegotiations.
- **After an HA restart**, the selected mode and all tuning values are restored (no helper
  has a fixed `initial:`, so HA restores the last value). Fast re-applies on boot; Solar
  resumes on the next 20 s control tick if there's surplus. *On the very first reload after
  removing the `initial:` values, the helpers may be blank — set them once from the
  dashboard (suggested values are shown inline) and they persist thereafter.*
- **Autonomous charging during an HA outage** — while HA is *down* longer than the
  setpoint validity window (300 s), the charger falls back to **Safe current** and charges
  on its own, outside all HA logic. The 300 s window means a normal HA restart rides
  through without a fallback; only a genuine multi-minute outage triggers it. Nothing in HA
  can prevent it while HA is down; the only fix is to set **Safe current to 0** (or the
  lowest accepted) in the Service Installer, so the fallback pauses instead of charging.
- **Phase switches cost ~15 s** of charging each (the pause-switch-resume), and are capped
  to at most one per `alfen_phase_cooldown` in either direction.
- **2 Modbus TCP connections max** — if EVCC or another master is still connected, HA's
  writes can be refused for lack of a slot. Run only HA against the charger.
- **Fuse protection is soft** — reactive at the 20 s guard cadence, and it fails *open*
  (does nothing) if P1 data is missing. It reduces nuisance trips; it is not the breaker.
- **The car is a second controller.** Some EVs refuse to charge at low currents, drop the
  pilot (Mode 3 → B1/E/F), or won't hold 3-phase without enough power — behaviour that can
  look like a control bug but is car-side. Mode 3 = C means it's actually drawing; A/E/F or
  B1 flapping points at the cable or car, not the integration.

## The optional price layer

`optional_alfen_price.yaml` is included as a working example, not part of the reusable
engine — hence the name. It reacts to this install's cheap-window binary_sensors and
override/night input_booleans and simply selects **Fast** (or the configured fallback)
on `input_select.alfen_charge_mode`. To adapt it, swap in your own price entities; to
drop price control entirely, just don't install the file. The engine runs fine without
it.