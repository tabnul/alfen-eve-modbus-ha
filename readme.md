### Note: Solar charging broken atm.


# Alfen Eve — native Modbus EV charging control for Home Assistant

Solar-surplus, dynamic-price and manual control of an **Alfen Eve Pro-line (NG9xx)** EV
charger over **native Modbus TCP**, driven entirely from Home Assistant. No custom
integration — just the built-in `modbus` platform plus template sensors, scripts and a
handful of automations (a control loop, a keep-alive, a fuse guard, a protection-lockout
clear, and a startup-resume).

The charger runs in EMS/slave mode: Home Assistant is the sole brain. It writes a
single current setpoint, switches between 1- and 3-phase, and protects the main fuse
per phase. Charging follows one of four modes selected from a dropdown.

## What it does

- **Solar** — charges from genuine solar surplus only. The loop targets a small grid
  *export* (a configurable margin), so it charges only when there's real surplus and
  backs off before it would import. It modulates the car's current to track the surplus
  and switches 1↔3 phase as surplus allows. A heavy anti-oscillation stack (3-minute
  start/stop holds, a per-phase-switch cooldown, deadband and export margin) keeps it
  calm on cloudy days.
- **Fast** — 3-phase at maximum current (clamped by your per-phase and fuse limits).
  Ignores surplus entirely.
- **Manual** — you set the target current directly; it's written live.
- **Off** — pauses the car.
- **Per-phase fuse protection** — an always-on guard that pauses charging (in any mode)
  before any phase reaches your fuse rating, with hysteresis so it can't oscillate.
- **Clean restart & mode handling** — mode and target survive an HA restart; on restart
  the engine re-asserts the restored mode (and only resumes Solar charging if there's
  actual surplus). Selecting Solar always re-baselines to stopped first, then charges
  only when surplus justifies it.
- Full metering, status and control surfaced as HA entities, plus a dashboard.

## Requirements

- Alfen Eve Pro-line / S-line on the **NG9xx** platform (this was built and tested on
  firmware **7.4.5**).
- The **Active Load Balancing** licence on the charger (paid unlock from Alfen). Without
  it the Modbus slave functionality can't be enabled.
- **ACE Service Installer** access (a service-level Alfen account) to configure the
  charger — see below.
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
- Validity time — **60 s** (default; keep it — the keep-alive rewrites every 30 s)

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
requires) and records the value into `input_number.alfen_target_current`.

**Keep-alive.** The setpoint falls back to safe current if not refreshed within ~60 s.
`alfen_setpoint_renew` rewrites 1210 every 30 s whenever a car is connected. It is a
**standalone** automation (not merged into the control loop) so the control queue can
never delay it — this is what keeps the charger off its safe-current fallback.

**Phase switching.** Writing `1` or `3` to register **1215** switches phases, but doing so
live makes the car re-negotiate and some fault. `alfen_switch_phases` therefore does
**pause → wait 10 s → write 1215 → wait 5 s → resume at min** every time. Expect a ~15 s
charging gap on every phase change. Each switch also stamps
`input_datetime.alfen_last_phase_switch` for the cooldown guard.

**Single control loop.** `alfen_control` handles everything by dispatching on trigger id
(`tick`, `mode_change`, `manual_slider`, `solar_start/stop`, `phase_up/down`). Solar
modulation runs on a **20 s tick** (aligned to a slow P1 meter) and computes the target
as `min(surplus_current, max, phase_limit, headroom)` bounded below by the 6 A floor,
writing only on a ≥1 A change (deadband).

**Solar surplus.** `alfen_solar_surplus = charger_power − grid − export_margin`. The
export margin (default 300 W) shifts the operating point so the loop targets a small
export rather than grid-zero — it charges only from genuine surplus and never imports at
the edge. Set the margin to 0 for old grid-zero behaviour, higher for stricter
solar-only. The sensor carries an `availability` guard so a brief `alfen_power_sum`
dropout doesn't corrupt the value.

**Mode entry re-baselines.** `mode_change` handles Off/disconnect (stop), **Solar (stop
— then the loop starts charging only when `solar_start` sees real surplus)**, and Fast
(3-phase + max). So switching into Solar from Fast never coasts at Fast's high current —
it drops to stopped immediately and comes up only on genuine sun.

**Anti-oscillation.** Symmetric 3-minute holds on solar_start and solar_stop; 3-minute
holds plus a 10 % margin on the phase up/down thresholds; a configurable
**per-phase-switch cooldown** (`alfen_phase_cooldown`, default 10 min) that hard-caps how
often 1↔3 switching can happen regardless of surplus; a 1 A modulate deadband; and the
300 W export margin. Together these keep it from thrashing on a flickering cloudy day.

## Fuse protection

The charger's own load balancing is off (pure EMS), so protecting the main fuse is HA's
job. It's a **soft** limit — reactive at the automation cadence, not a substitute for the
physical breaker — but it prevents nuisance trips in normal operation.

- **Per-phase headroom** (`alfen_headroom_l1/l2/l3`) = fuse rating − house load on that
  phase (P1 phase power minus the car's own draw). `alfen_headroom_active` is the one
  that governs the current mode: L1 only in 1-phase, tightest of the three in 3-phase.
- **Fuse guard** (`alfen_fuse_guard`, every 5 s, **all modes**) pauses the car if any
  phase's total current (house + car) reaches the fuse rating.
- **Phase-down safety** — a 3→1 switch only happens if L1 can actually hold a 1-phase
  charge afterward, so it never switches *into* an overload.
- **Hysteresis / anti-oscillation** — when protection stops the car it latches
  `input_boolean.alfen_protect_lockout`. Resume is blocked until headroom has been at
  `min + recovery_margin` (default +3 A) for a continuous **2 minutes**
  (`alfen_protect_clear`), then Solar is kick-started. This stops the "stop → headroom
  recovers → restart → overload → stop" bounce.

**Set `alfen_fuse_per_phase` to a safe per-phase value** — a couple of amps below your
real main fuse rating (e.g. 23 on a 25 A connection), so the guard acts before the
breaker does. This is per phase: a 3×25 A connection is 25 A *per phase*.

## Tunable settings (helpers)

All tuning is via `input_number` / `input_select` helpers — no YAML logic edits.

| Helper | Meaning |
| --- | --- |
| `alfen_max_current` | Full-power ceiling. |
| `alfen_phase_limit` | Hard per-phase cap on the car's own draw. |
| `alfen_fuse_per_phase` | Per-phase fuse rating the guard trips at (set with margin). |
| `alfen_export_margin` | Watts of export to keep in reserve; 0 = grid-zero, higher = stricter solar-only. Default 300. |
| `alfen_phase_cooldown` | Minutes to block further 1↔3 phase switches after one. Default 10. |
| `alfen_min_current` | The 6 A IEC floor; the loop never charges below this while intending to charge. |
| `alfen_stop_current` | The "off" value written to pause (below the 6 A floor). |
| `alfen_recovery_margin` | Hysteresis band: resume only when headroom ≥ min + this. |
| `alfen_charge_mode` | Off / Solar / Fast / Manual. |
| `alfen_price_fallback` | Mode entered when a price window closes (used by the price layer). |

The hysteresis hold times live in the automation (`for:` durations on the solar/phase
triggers, and the 2-minute dwell in `alfen_protect_clear`) — HA can't template a trigger
`for:`, so those stay in YAML; the cooldown, being a condition, is a live helper.

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
4. Mode → **Manual**, drag the target slider, confirm the readback tracks it live.
5. Try **Solar** and **Fast**, and the Force-phase buttons.
6. Load the dashboard; wire the price layer last.

## Known limitations & gotchas

- **Reallin meter (post-2021 units)** only expose a subset of measurement registers over
  Modbus; some power/energy/current-sum registers may read NaN. Voltages and per-phase
  current are what the control logic needs; the Solar loop's surplus depends on the
  charger power register — verify `sensor.alfen_power_sum` reports before relying on
  Solar.
- **Integer-amp control** — the setpoint steps in whole amps, so ~230 W (1-phase) or
  ~690 W (3-phase) of granularity per step is unavoidable; the export margin absorbs this
  so it stays on the export side. Inherent to amp-stepped charging, not a bug.
- **Solar starts slowly by design** — surplus must hold above threshold for 3 minutes
  before charging begins, and stop takes 3 minutes too (symmetric, to minimise relay
  wear). On a marginal day it will pause more and charge less; that's the "only charge on
  genuine solar" trade-off.
- **After an HA restart**, mode and target are restored (they survive the restart). The
  engine drops off the safe-current fallback, then re-asserts the restored mode — Fast
  resumes at max, Manual at its target, and Solar resumes charging only if there's actual
  surplus (otherwise it waits at stop). Solar can take up to ~3 min to (re)start once sun
  is present, due to the start hold.
- **Autonomous charging during an HA outage** — while HA is *down* longer than the ~60 s
  setpoint validity, the charger falls back to **safe current** and charges on its own,
  outside all HA logic. Nothing in HA can prevent this while HA is down; the only fix is
  to set **Safe current to 0** (or the lowest accepted) in the Service Installer, so the
  fallback pauses instead of charging.
- **Phase switches cost ~15 s** of charging each (the pause-switch-resume), and are capped
  to at most one per `alfen_phase_cooldown` (default 10 min) in either direction. A
  blocked phase-down means it holds 3-phase (importing a little, bounded) or pauses until
  the cooldown clears — the accepted cost of not switching too often.
- **2 Modbus TCP connections max** — if EVCC or another master is still connected, HA's
  writes can be refused for lack of a slot. Run only HA against the charger.
- **Fuse protection is soft** — reactive at the 5 s guard cadence, and it fails *open*
  (does nothing) if P1 data is missing. It reduces nuisance trips; it is not the breaker.
- **Reallin meter** — this unit's per-phase currents and voltages read fine, but some
  aggregate power/energy registers can drop out. Solar surplus is built on
  `alfen_power_sum`; the `availability` guard on the surplus sensor stops brief dropouts
  from corrupting the loop, but chronic dropout is a Modbus/hardware issue to chase
  separately.

## The optional price layer

`optional_alfen_price.yaml` is included as a working example, not part of the reusable
engine — hence the name. It reacts to this install's cheap-window binary_sensors and
override/night input_booleans and simply selects **Fast** (or the configured fallback)
on `input_select.alfen_charge_mode`. To adapt it, swap in your own price entities; to
drop price control entirely, just don't install the file. The engine runs fine without
it.
