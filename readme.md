# Alfen Eve — native Modbus EV charging control for Home Assistant

Solar-surplus and dynamic-price control of an **Alfen Eve Pro-line (NG9xx)** EV charger over **native Modbus TCP**, driven entirely from Home Assistant. No custom integration — just the built-in `modbus` platform plus template sensors, a couple of scripts, and five automations (a Solar control loop, a keep-alive, a fuse guard, mode routing, and disconnect handling).

The charger runs in EMS/slave mode: Home Assistant is the sole brain. It writes a single current setpoint, switches between 1- and 3-phase, and keeps every phase under the fuse. Charging follows one of three modes selected from a dropdown.

---

## What it does

- **Solar** — charges from genuine solar surplus only. It always starts on 1-phase, modulates the car's current to track the surplus, and switches up to 3-phase once there's enough sustained surplus to run three phases (then back down as surplus fades). Start, stop and phase changes each wait out a configurable hold, and phase switches are further capped by a cooldown, so it stays calm on a cloudy day.
- **Fast** — 3-phase at maximum current (clamped by the car max and the fuse guard). Ignores surplus — will import from grid. Fast has no control loop of its own: the mode is applied once, and from then on the fuse guard owns every subsequent write (trim down, recover up, stop on sustained overload).
- **Off** — pauses the car.
- **Per-phase fuse protection** — an always-on guard that keeps every phase under your fuse rating by **adjusting** the charge current rather than cutting the car off, with configurable settle times on both directions to filter out P1 telemetry blips. It only stops outright if even 6 A no longer fits, and only after a sustained-overload timeout.
- **Survives restarts** — the selected mode and all tuning values persist across an HA restart. Fast re-applies on boot; Solar picks back up on the next control tick.
- Full metering, status and control surfaced as HA entities, plus a dashboard.

---

## How the Solar loop works

One automation (`alfen_solar`) runs every 20 s (and on entering Solar) and dispatches a single action per tick via a `choose`, so at most one setpoint write happens per tick — this is what keeps it from disturbing the car with rapid writes. It is **level-triggered**: each tick re-checks the current conditions rather than relying on a threshold-crossing edge, so it can't get stuck because a level was already crossed when a setting changed.

Durations ("surplus above X for N minutes") are measured from the `last_changed` of three threshold binary sensors (`alfen_surplus_over_start/up`, `alfen_surplus_under_down`). The branches, in priority order: start (if stopped and surplus has held), stop (if charging and surplus has fallen — resets to 1-phase as it stops), fuse-pause, phase-up 1→3, phase-down 3→1, then modulate. Phase switches use a pause→switch→resume script and are gated by a cooldown.

The modulate branch uses an **asymmetric deadband**, for the same reason the fuse guard does (see below): writing a *lower* current makes the car re-negotiate, writing a *higher* one does not. Raising the current still happens on a 1 A change; lowering it has to clear `alfen_solar_step_down_buffer` (suggested 2 A), so a flickering surplus doesn't force a re-negotiation on every 20 s tick. That buffer damps **surplus-driven** drops only — when the fuse headroom is what's capping the target, the deadband falls back to 1 A automatically and the car comes down immediately.

---

## Requirements

- Alfen Eve Pro-line / S-line on the **NG9xx** platform (this was built and tested on firmware **7.4.5**).
- The **Active Load Balancing** licence on the charger (paid unlock from Alfen). Without it the Modbus slave functionality can't be enabled.
- **ACE Service Installer** access (a service-level Alfen account) to configure the charger — see below. **Set the Modbus setpoint Validity time to 300 s** there (not the 60 s default) — this keeps charging alive across an HA restart and is EVCC's recommended value; details in the charger-configuration section.
- A **HomeWizard P1** meter (or any P1 meter) exposing **total** and **per-phase** active power. Grid import must read positive, export negative.
- Home Assistant with the built-in `modbus` integration, packages enabled, and the **apexcharts-card** HACS card for the dashboard graphs.

### Site assumptions to adapt

Two things are specific to the reference install and must be changed for another setup:

- **1-phase charging happens on L1.** The per-phase headroom and fuse logic treat L1 as the phase that carries the car in 1-phase mode. If your charger uses a different phase, adjust the L1 references.
- **P1 entity names** are `sensor.p1_meter_power` (total) and `sensor.p1_meter_power_phase_1/2/3`. Change these throughout if yours differ.

---

## Charger configuration (ACE Service Installer)

All of this is set in the **ACE Service Installer** app, connected to the charger, under **Load balancing**. The Eve Manager (end-user) app cannot set most of it.

**Active balancing:**

- Active Load Balancing — **on**
- Data Source — **Energy Management System** (this puts the charger in slave role)
- Safe current — the current the charger falls back to when HA stops writing for longer than the validity time. See note below.
- Allow single-/multiphase charging — **on** *(required, or the phase register does nothing)*
- Phase rotation — as wired (e.g. L1L2L3)

**TCP/IP EMS:**

- Mode — **Socket**
- Validity time — **300 s** *(recommended — see below; do not leave at the 60 s default)*

**Set the validity time to 300 s, not the 60 s default.** This is the window the charger keeps a written setpoint valid before falling back to Safe current. EVCC recommends 300 s for Alfen, and it buys two things: (1) an HA restart (which takes well under 5 min) no longer causes the charger to drop the setpoint mid-charge — charging continues across the reboot; and (2) generous margin against any write-timing jitter, so a delayed keep-alive can never trip the fallback. The keep-alive still rewrites every 30 s, now comfortably inside the window. There is no downside to 300 s for an EMS/slave setup.

On firmware 7.4.5 there are **no separate "Allow reading / Allow writing maximum currents" checkboxes** — enabling Active Load Balancing with the EMS data source and Socket mode is what permits Modbus writes. On some other firmware those toggles exist (sometimes only under *Advanced Settings*); if present, both must be on.

### Safe current

Safe current is the charger's own fallback for when HA goes silent (restart, network drop, the keep-alive dying). During normal operation your setpoint drives everything and safe current is dormant. Because HA is the sole controller, a fallback should ideally be conservative — but any sane value works. Note: while a car is connected and HA is *not* writing, the charger will charge at safe current on its own, outside all HA logic.

---

## Files

- **`alfen_modbus.yaml`**: The `modbus:` read layer — every register as a sensor. Placed in `configuration.yaml` via `modbus: !include alfen_modbus.yaml`.
- **`packages/alfen.yaml`**: The generic engine: helpers, template sensors, scripts, and the control + protection automations. Placed in the `packages/` directory.
- **`optional_alfen_price.yaml`**: **Site-specific, not reusable.** Dynamic-price/window wiring that only selects the charge mode. Placed in the `packages/` directory.
- **`alfen_dashboard.yaml`**: The sections-view dashboard with interactive help toggle. Placed in Lovelace dashboard config.

To enable packages in `configuration.yaml`, add:
`homeassistant:`
  `packages: !include_dir_named packages`

Set the charger's **IP** in `alfen_modbus.yaml` (`host:`). Give the charger a **static IP / DHCP reservation** — the modbus hub reads `host:` only at startup and doesn't follow a helper, so a moving IP silently breaks the connection.

The engine (`alfen_modbus.yaml`, `packages/alfen.yaml`, `alfen_dashboard.yaml`) is generic and portable. `optional_alfen_price.yaml` is an example of a site layer bolted on top; its only contract is **"set `input_select.alfen_charge_mode`, nothing else"** — it never writes current or phases directly, so it can't fight the engine.

---

## How it works

**Setpoint.** All current control is a single float32 write to register **1210** (the "Modbus Slave Max Current" setpoint). The `alfen_set_current` script packs the amps into two 16-bit words (big-endian IEEE-754, single multi-register write, as the charger requires) and records the value into `input_number.alfen_target_current`. `input_number.alfen_target_current` is therefore the single source of truth for "what did we last command", and every loop reads it rather than the charger's readback.

**Keep-alive.** The setpoint falls back to Safe current if it isn't refreshed within the configured validity time (set to 300 s — see charger config). `alfen_setpoint_renew` rewrites 1210 every 30 s whenever a car is connected — far inside the window. It is a **standalone** automation (not part of the control loop) so nothing can delay it — this is what keeps the charger off its safe-current fallback.

**Phase switching.** Writing `1` or `3` to register **1215** switches phases, but doing so live makes the car re-negotiate and can fault. `alfen_switch_phases` therefore does **pause → wait 10 s → write 1215 → wait 5 s → resume at min**. Both control loops refuse to run while that script is active, so nothing can write current into the pause window (see Known limitations). Expect a ~15 s charging gap on every phase change. Each switch stamps `input_datetime.alfen_last_phase_switch` (as a Unix timestamp) for the cooldown guard.

**Control loop.** `alfen_solar` runs every 20 s and on entering Solar, dispatching one action per tick via a `choose`. Modulation computes the target as `min(surplus / (phases × 230), max_current, headroom)`, floored at 6 A, and writes it when the change clears the asymmetric deadband (1 A up, `alfen_solar_step_down_buffer` down). Because it re-checks every tick (level-triggered) rather than firing on a threshold-crossing edge, it can't get stuck if a level was already crossed when a setting changed.

**Solar surplus.** `alfen_solar_surplus = charger_power − grid`. It is independent of the car's own draw (the car's power appears in both terms and cancels), so a running car doesn't hide the surplus. Three binary sensors flag it against the start / phase-up / phase-down thresholds; the loop measures how long each has held from its `last_changed`.

**Phase decisions.** Solar always starts on **1-phase**. It switches up to 3-phase only once surplus holds above `alfen_phase_up_threshold` for the phase hold, and drops back to 1-phase when surplus holds below `alfen_phase_down_threshold`. A **cooldown** (`alfen_phase_cooldown`) caps how often a switch can happen in either direction, and a full stop resets to 1-phase so a paused charger never sits on 3-phase.

**Anti-oscillation.** Configurable holds on start/stop (`alfen_solar_hold`) and on phase switches (`alfen_phase_hold`); a phase-switch cooldown (`alfen_phase_cooldown`); a scale-up cooldown after any guard action (`alfen_stop_cooldown`); asymmetric step buffers on the fuse guard (`alfen_step_down_buffer` / `alfen_step_up_buffer`) with settle times on each direction (`alfen_guard_down_settle` / `alfen_guard_up_settle`); and the asymmetric Solar modulate deadband (`alfen_solar_step_down_buffer`). Together these keep it from thrashing on a flickering cloudy day or bouncing on household loads.

**Why asymmetric.** On IEC 61851 Mode 3, *reducing* the offered current makes the car re-negotiate — a handshake that briefly interrupts the draw and, on some cars, can drop the session entirely. *Raising* it does not. Every loop in this package is therefore built to make down-writes rare and deliberate, while letting up-writes happen freely. This is the single most important design constraint in the engine; any tuning that makes the system step down more eagerly trades charge stability for responsiveness.

---

## Fuse protection

The charger's own load balancing is off (pure EMS), so keeping the main fuse safe is HA's job. It's a **soft** limit — reactive at the 20 s guard cadence, not a substitute for the physical breaker — but it prevents nuisance trips in normal operation, and rather than cutting the car off it **adjusts** the current to fit.

- **Per-phase headroom** (`alfen_headroom_l1/l2/l3`) = fuse rating − other house load on that phase, **clamped at the fuse rating**. `alfen_headroom_active` governs the current mode: L1 in 1-phase, tightest of the three in 3-phase. The clamp matters on a solar site: the P1 meter reports *net* phase power, so while exporting the "house load" term goes negative and the raw figure would read well above the fuse (40 A on a 24 A fuse was observed). That extra is real only while the sun holds, and the car's own draw has to fit the fuse without it, so it is never offered. The clamp can only ever lower the number.
- **Fuse guard** (`alfen_fuse_guard`, every 20 s, **all modes**) keeps every phase under `alfen_fuse_per_phase`. When household appliances turn on, it steps the target current down. In Fast mode it is also the *only* thing that writes current after the mode is applied.
- **Asymmetric step buffers** (`alfen_step_down_buffer` / `alfen_step_up_buffer`) decouple downward safety stepping from upward recovery. See the sizing note below — the down buffer is a safety figure, not a comfort one.
- **Settle times on both directions** (`alfen_guard_down_settle` / `alfen_guard_up_settle`) require the trim or recover condition to hold for a configurable window before acting. These are measured from dedicated binary sensors (`alfen_guard_should_trim` / `alfen_guard_can_recover`) whose `last_changed` only moves when the *condition* crosses — **not** from the headroom sensor, which updates on every P1 poll and would reset the timer forever, silently disabling the guard.
- **Cloud ride-through**: a headroom collapse trims to the 6 A floor and *holds* there rather than stopping. Only if headroom stays below 6 A for `alfen_critical_timeout` does it stop outright. One re-negotiation instead of a dropped session.
- **Scale-up cooldown** (`alfen_stop_cooldown`) blocks recovery for a period after **any** guard action — a trim as well as a stop, since both stamp `input_datetime.alfen_last_guard_stop`. Deliberately conservative: it's the thing that prevents up→down→up oscillation where every "down" costs a re-negotiation.

**Sizing `alfen_fuse_per_phase` and the down buffer together.** Set `alfen_fuse_per_phase` to the total per-phase current you'll allow (car + other household load on that phase), a little below your real fuse — e.g. **24 A under a 25 A fuse**. But note that the guard tolerates the car sitting up to `alfen_step_down_buffer` amps *above* that budget before it writes a lower value. With a 24 A budget under a 25 A fuse, a 1 A down buffer puts the worst case exactly at the fuse rating; a 2 A buffer puts it 1 A *over*. **Keep `alfen_step_down_buffer` at 1 A unless you have widened the gap between the budget and the real fuse.** The up buffer has no such constraint and can be larger.

---

## Logging

Safety-relevant decisions are written to the Home Assistant **logbook** via `logbook.log`, and the dashboard carries a logbook card showing the last 48 hours of them. The point is to be able to answer "why did the car stop / drop to 6 A at 14:20?" after the fact, without a trace or a debug log.

Every entry is anchored to `input_boolean.alfen_events` — a helper that exists **only** as a logbook anchor and is never toggled. That's what lets the dashboard card show engine events and nothing else; anchoring to `input_number.alfen_target_current` instead would drown the card in that entity's own automatic state-change entries.

**What is logged:**

| Event | Source |
| --- | --- |
| Fuse guard trims the current down | `alfen_fuse_guard` — includes the headroom reading, the settle time that elapsed, and the fuse budget |
| Fuse guard recovers upward | `alfen_fuse_guard` — Fast mode only |
| Sustained overload stop | `alfen_fuse_guard` — headroom stayed under the floor past `alfen_critical_timeout` |
| Solar start / stop | `alfen_solar` — with the surplus and the hold that was satisfied |
| Solar pause on fuse floor | `alfen_solar` — headroom below 6 A, minimum charging no longer fits |
| Phase switch 1↔3 | `alfen_switch_phases` — logged in the script, so the manual dashboard buttons are captured too |

**What is deliberately not logged:** the Solar modulate loop's routine surplus tracking. On a broken-cloud day that is tens of setpoint writes an hour, and logging them would bury every entry above. The setpoint trajectory is already visible as history on `input_number.alfen_target_current`.

This means the logbook does not distinguish a Solar step-down caused by the *fuse* from one caused by the *sun* — both are silent. That attribution was deliberately left out: doing it properly needs edge detection on a continuous condition, which costs an extra automation and an extra binary sensor for a diagnostic line. If you want it, `sensor.alfen_headroom_active` against `sensor.alfen_solar_surplus` on a history chart shows the same thing. The Solar pause at the 6 A floor — the case that actually stops charging — *is* logged.

Expect roughly 10–20 entries on an ordinary day.

---

## Metering — charging energy

`sensor.alfen_energy_consumed_total_kwh` is the total energy consumed by charging, in kWh (`device_class: energy`, `state_class: total_increasing`), so it can be added to the HA **Energy dashboard** and long-term statistics. It's derived from the charger's Wh delivered-energy register (÷1000).

This is **AC energy fed from the charger to the car** — i.e. what you drew and paid for. The Alfen is an AC charger, so this is the only meaningful energy figure it can report; the AC→DC conversion losses (~10–15%) happen inside the car's onboard charger, downstream of any meter the charger or HA can see, so "energy actually stored in the battery" is not obtainable here — only the car's own telemetry knows that. For cost and consumption tracking, the AC figure is the correct one. (The charger's separate `energy_consumed` register is dead on this meter and is not used.)

---

## Tunable settings (helpers)

All tuning is via `input_number` / `input_select` / `input_boolean` helpers — no YAML logic edits. None have a fixed `initial:`, so **your values persist across restarts**; each shows a suggested starting value on the dashboard, and the templates fall back to that suggestion if a helper is ever left blank.

- **`alfen_max_current`**: Car's charge-rate ceiling per phase (suggested: 16 A).
- **`alfen_min_current`**: The 6 A IEC floor; never charges below this (suggested: 6 A).
- **`alfen_stop_current`**: The "off" value written to pause, below 6 A (suggested: 5 A).
- **`alfen_fuse_per_phase`**: Total per-phase current cap for car + house (suggested: 24 A).

**Fuse guard tuning**

- **`alfen_step_down_buffer`**: Minimum drop before the guard writes a lower current. **Safety-critical — see the sizing note under Fuse protection** (suggested: 1 A).
- **`alfen_step_up_buffer`**: Minimum headroom above the current setpoint before recovering upward (suggested: 2 A).
- **`alfen_guard_down_settle`**: Seconds the trim condition must hold before stepping down (suggested: 40 s).
- **`alfen_guard_up_settle`**: Seconds the recover condition must hold before stepping up (suggested: 60 s).
- **`alfen_stop_cooldown`**: Minutes after any guard trim or stop before recovery may ramp up (suggested: 3 min).
- **`alfen_critical_timeout`**: Seconds headroom must stay below 6 A before a full stop. Shorter = stops sooner on real overload; longer = rides through more clouds (suggested: 60 s).

**Solar tuning**

- **`alfen_phase_up_threshold`**: Surplus in Watts needed to switch from 1→3 phase (suggested: 4800 W).
- **`alfen_phase_down_threshold`**: Surplus in Watts needed to switch from 3→1 phase (suggested: 4140 W).
- **`alfen_phase_cooldown`**: Minutes blocking any further phase switch (suggested: 12 min).
- **`alfen_solar_hold`**: Minutes surplus must hold before start/stop (suggested: 3 min).
- **`alfen_phase_hold`**: Minutes surplus must hold before a phase switch (suggested: 3 min).
- **`alfen_solar_step_down_buffer`**: How far the surplus target must fall below the commanded current before Solar writes a lower value. Purely anti-churn, not safety — headroom-driven drops bypass it (suggested: 2 A).

**Other**

- **`alfen_dashboard_show_help`**: Boolean to toggle dashboard explanation cards on/off (suggested: Off).
- **`alfen_events`**: Boolean used **only** as the logbook anchor for engine safety events — see Logging. Never toggle it; nothing reads its state.
- **`alfen_charge_mode`**: Off / Solar / Fast.
- **`alfen_price_fallback`**: Mode entered when a price window closes (suggested: Solar).

Two `input_datetime` helpers hold timestamps for the cooldowns and are written by the engine, not by you: `alfen_last_phase_switch` and `alfen_last_guard_stop`. Both are stamped as Unix timestamps via `now().timestamp()` and read back with `state_attr(..., 'timestamp')` — never parsed from the state string, which is naive local time and will read as UTC.

---

## Register reference

Slave **200** = station; slave **1** = socket. All values big-endian, no word swap.

- **1100–1101**: Station Active Max Current, hard ceiling (float32, Read-only)
- **1102–1103**: Board temperature (float32, Read-only)
- **1201–1205**: Mode 3 state, IEC 61851 A/B/C/D/E/F (string, Read-only)
- **1206–1207**: Actual Applied Max Current (float32, Read-only)
- **1208–1209**: Setpoint remaining valid time (uint32, Read-only)
- **1210–1211**: **Modbus Slave Max Current setpoint** (float32, Read/Write)
- **1212–1213**: Active Load Balancing safe current (float32, Read-only)
- **1214**: Setpoint accounted for, 1 = charger is using it (uint16, Read-only)
- **1215**: **Charge using 1 or 3 phases** (uint16, Read/Write)
- **306–336**: Per-phase voltage, current, power factor, frequency (float32, Read-only)
- **374 / 390**: Real energy delivered / consumed sum (float64, Read-only)

`sensor.alfen_setpoint_accounted_for` (1214) is the definitive "are my writes landing" check: **1** means the charger is using your setpoint.

---

## Bring-up / testing order

Do this in order so a failure points at one layer:

1. Set the real charger IP in `alfen_modbus.yaml`, load it, restart HA. Confirm sensors populate — voltages near 230 V, temperature sane.
2. Load `packages/alfen.yaml`. In Developer Tools call `script.alfen_set_current` with `amps: 6` and watch `sensor.alfen_setpoint_readback` → 6 and `sensor.alfen_setpoint_accounted_for` → 1. That validates the whole write path.
3. **Set `alfen_fuse_per_phase` to your real safe rating** before anything else — a wrong (too-low) value clamps or trips everything.
4. Try **Solar** and **Fast**, and the Force-phase buttons; in Fast, add a household load on a phase and confirm the fuse guard trims the car and recovers when it clears. Remember the trim waits out `alfen_guard_down_settle` (40 s) and the recovery waits out both `alfen_guard_up_settle` (60 s) and `alfen_stop_cooldown` (3 min) — nothing is wrong if it doesn't react instantly.
5. In **Solar** on a broken-cloud day, watch `input_number.alfen_target_current` in the history graph. It should step down in meaningful jumps, not sawtooth every 20 s. If it still chases the surplus, raise `alfen_solar_step_down_buffer` from 2 A to 3 A.
6. Load the dashboard; confirm the **Safety Event Log** card fills in as you exercise steps 4–5 (a phase switch via the Force buttons is the quickest way to produce an entry). Wire the price layer last.

---

## Known limitations & gotchas

- **Reallin meter (post-2021 units)** only expose a subset of measurement registers over Modbus; several aggregate registers read NaN / never populate. On this build, `alfen_current_sum` and `alfen_energy_consumed_sum` are dead and are not used. Voltages, per-phase current, charger power, and **energy delivered** all report fine — and energy delivered is what matters (see Metering below). Verify `sensor.alfen_power_sum` reports before relying on Solar.
- **Integer-amp control** — the setpoint steps in whole amps, so ~230 W (1-phase) or ~690 W (3-phase) of granularity per step is unavoidable. Inherent to amp-stepped charging, not a bug.
- **Solar has holds by design** — surplus must hold above the start threshold for the start/stop hold before charging begins (and below it before stopping), and above the phase-up threshold for the phase hold before going 3-phase. On a marginal day it pauses more and switches less; that's the trade-off for not thrashing. All holds are tunable.
- **Guard cadence vs. meter update rate.** `alfen_fuse_guard` ticks every 20 s (`seconds: "/20"`), matched to a P1 meter that updates roughly every 10 s. **Do not set the tick faster than your meter updates** — the guard would act repeatedly on the same stale reading. If your meter updates every second and you want faster reaction to household loads, lower the trigger to `/5` and shorten `alfen_guard_down_settle` to match; the settle time, not the tick, is what filters transient drops.
- **Settle times must never be measured from a continuously-updating sensor.** If a settle check is written against `states.sensor.alfen_headroom_active.last_changed`, the timer resets on every P1 poll and the condition can never mature — the guard goes permanently silent while appearing correctly configured. This actually happened during development: a 15 s settle time left a 15.2 A setpoint standing for 11 minutes against 8.3 A of headroom. The dedicated `alfen_guard_should_trim` / `alfen_guard_can_recover` binary sensors exist precisely to give the timers a `last_changed` that only moves on a condition crossing.
- **Nothing that calls `alfen_switch_phases` may be triggered by the Mode 3 state.** The switch script pauses the car for 10 s, which itself changes Mode 3 (C2 → B1/B2). `alfen_charge_mode_apply` used to trigger on `sensor.alfen_mode3_state` with `mode: restart`, so that pause re-triggered the automation, which cancelled its own in-flight run — and since the script is called as a service it is a *child* of that run, so it died at the delay and register 1215 was never written. Fast mode would log a phase switch and silently stay on 1 phase. Solar was never affected, because `alfen_solar` calls the same script and is not triggered by mode3. Disconnect handling is therefore split into `alfen_disconnect_pause`, which only ever writes stop current and so is safe to restart.
- **Nothing may write current during a phase switch.** `alfen_switch_phases` drops the setpoint to `stop_current` and holds it for 10 s so the contactor changes under no load. Both control loops (`alfen_fuse_guard` and `alfen_solar`) carry a `script.alfen_switch_phases` is `off` condition for this reason. Without it the fuse guard sees the deliberately-low setpoint, concludes headroom has recovered, and writes max current back *into* the pause — the phase register is then written while the car is drawing full current and the charger silently refuses to switch, leaving you stuck on the old phase count. This was observed live: a switch logged at 15:42:18 and a guard recovery to 16 A at 15:42:20. The keep-alive (`alfen_setpoint_renew`) is deliberately *not* gated — it rewrites whatever the current target is, which during a pause is the pause value, and it must keep running to hold off the validity timeout.
- **Automations edited in the HA UI override the package.** An automation from `packages/alfen.yaml` that you then open and save in Settings → Automations is copied into `automations.yaml`, and that copy wins from then on — edits to the package file will appear to do nothing. Delete it from Settings → Automations to hand control back to the package.
- **After an HA restart**, the selected mode and all tuning values are restored (no helper has a fixed `initial:`, so HA restores the last value). Fast re-applies on boot; Solar resumes on the next 20 s control tick if there's surplus. *On the very first reload after removing the `initial:` values, the helpers may be blank — set them once from the dashboard (suggested values are shown inline) and they persist thereafter.*
- **Autonomous charging during an HA outage** — while HA is *down* longer than the setpoint validity window (300 s), the charger falls back to **Safe current** and charges on its own, outside all HA logic. The 300 s window means a normal HA restart rides through without a fallback; only a genuine multi-minute outage triggers it. Nothing in HA can prevent it while HA is down; the only fix is to set **Safe current to 0** (or the lowest accepted) in the Service Installer, so the fallback pauses instead of charging.
- **Phase switches cost ~15 s** of charging each (the pause-switch-resume), and are capped to at most one per `alfen_phase_cooldown` in either direction.
- **2 Modbus TCP connections max** — if EVCC or another master is still connected, HA's writes can be refused for lack of a slot. Run only HA against the charger.
- **Fuse protection is soft** — reactive at the 20 s guard cadence, and it fails *open* (does nothing) if P1 data is missing. It reduces nuisance trips; it is not the breaker.
- **The car is a second controller.** Some EVs refuse to charge at low currents, drop the pilot (Mode 3 → B1/E/F), or won't hold 3-phase without enough power — behaviour that can look like a control bug but is car-side. Mode 3 = C means it's actually drawing; A/E/F or B1 flapping points at the cable or car, not the integration.

---

## Disclaimer & Disclaimer of Liability

> **USE AT YOUR OWN RISK.**  
> This software interacts directly with high-voltage electrical hardware (EV charger) and main household electrical infrastructure. It is provided strictly as-is for educational and personal automation purposes. 
> 
> - The author(s) assume **no responsibility or liability** for any damage to your EV charger, electrical vehicle, main fuses, household wiring, software installation, or property.
> - Always verify that your physical safety devices (fuses, circuit breakers, residual current devices) are properly installed by a qualified electrician. Soft automation safety limits (such as the software fuse guard in this package) are reactive and are **not a substitute for hardware circuit protection**.

---

## License

This project is licensed under the **MIT License** — free to use, modify, distribute, and integrate without restriction.

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.