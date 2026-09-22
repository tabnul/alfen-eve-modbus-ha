# Alfen Eve — EV charging control for Home Assistant

Control an **Alfen Eve Pro-line or S-line (NG9xx)** charger directly from Home Assistant over Modbus TCP. Charge from your own solar surplus, charge fast when you want to, and never trip your main fuse — without any custom integration or cloud service.

Home Assistant becomes the only brain: it tells the charger how many amps to offer and whether to use one or three phases. Everything is built from standard Home Assistant parts (the `modbus` integration, template sensors, scripts and automations), so you can read and change all of it.

---

## What it does

You pick one of three modes on the dashboard:

- **Off** — the car stays plugged in but doesn't charge.
- **Solar** — charges only from electricity your panels would otherwise send to the grid, following the sun up and down. You choose whether it may use one phase, three phases, or switch between them automatically. When the sun gets too weak it pauses, or — if you switch on **Keep charging at minimum** — it keeps going at the lowest current and adds the sun on top.
- **Fast** — charges as fast as the car and your fuses allow, on three phases, using grid power if needed.

In every mode:

- **Your fuse is protected.** The system watches how much the whole house uses on each phase and slows the car down when a phase gets close to your limit, then speeds back up when the load is gone. It only stops the car completely if even the minimum charging current no longer fits.
- **It survives restarts.** Your mode and settings are kept when Home Assistant restarts, and charging carries on through a normal restart.
- **Plugging in applies your mode straight away**, so the car never briefly charges at the charger's own default.
- **Important decisions are logged**, so you can see afterwards why charging slowed down, stopped or switched phases.

---

## Why it is careful about lowering the current

Many cars briefly stop and restart charging whenever the charger **lowers** the current. Raising it causes no interruption. Frequent restarts are annoying at best, and occasionally a car doesn't come back cleanly.

So the whole system is built to lower the current rarely and deliberately, and to raise it freely. That's why there are buffers and waiting times on the way down, and why Solar ignores small dips from passing clouds. Whether your car restarts on a lowered current depends on the car — some follow the change smoothly — but the defaults assume it does.

Switching between one and three phases is different: that always needs a short pause (about 15 seconds), on every car.

---

## What you need

- An **Alfen Eve Pro-line or S-line** on the **NG9xx** platform. Developed and tested on firmware **7.4.5**.
- The **Active Load Balancing** licence on the charger (a paid option from Alfen). Without it the charger can't be controlled over Modbus.
- Access to the **ACE Service Installer** app (an installer-level Alfen account) to change the charger settings below.
- A **P1 smart-meter reader** (for example HomeWizard) with four sensors: **total power** and **power per phase** (L1, L2, L3). Each must report in **watts (W)**, positive when you use power from the grid and negative when you send it back. You choose the sensors on the dashboard.
- Home Assistant with **packages** enabled, and the **apexcharts-card** (from HACS) for the dashboard chart.

### Something you may need to adapt

- **Single-phase charging is assumed to use L1.** If your charger is wired so that one-phase charging uses a different phase, change the L1 references in the headroom sensors.

---

## Charger settings (ACE Service Installer)

Connect to the charger with the ACE Service Installer app and open **Load balancing**. The regular end-user app can't change most of these.

**Active balancing**

- Active Load Balancing — **on**
- Data Source — **Energy Management System** (this lets Home Assistant take control)
- Allow single-/multiphase charging — **on** (otherwise phase switching does nothing)
- Phase rotation — as your installation is wired
- Safe current — see below

**TCP/IP EMS**

- Mode — **Socket**
- Validity time — **300 seconds** (not the default 60)

**Why 300 seconds.** Home Assistant has to repeat its instruction to the charger regularly; if it goes quiet for longer than the validity time, the charger falls back to its own *Safe current*. The system repeats it every 30 seconds, so 300 gives plenty of margin, and a normal Home Assistant restart (which takes well under five minutes) no longer interrupts charging. This is also the value evcc recommends for Alfen chargers.

**Safe current** is what the charger does on its own when Home Assistant is silent for longer than the validity time — for example during a long outage. If a car is plugged in, it will charge at this current, outside all of the logic here. If you'd rather it pause in that situation, set Safe current to **0** (or the lowest value the app accepts).

On firmware 7.4.5 there are no separate "Allow reading / Allow writing" checkboxes; the settings above are enough. Some other firmware versions have them (sometimes under *Advanced settings*); if yours does, turn both on.

---

## Installation

The package consists of these files:

| File | What it is | Where it goes |
| --- | --- | --- |
| `alfen_modbus.yaml` | Reads every charger value as a sensor | Include from `configuration.yaml`: `modbus: !include alfen_modbus.yaml` |
| `packages/alfen.yaml` | The control logic: settings, sensors, scripts, automations | Your `packages/` folder |
| `alfen_dashboard.yaml` | The dashboard, with optional help text | Paste into a dashboard's raw configuration editor |

If packages aren't enabled yet, add this to `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Put your charger's IP address in `alfen_modbus.yaml` (`host:`). **Give the charger a fixed IP address** (a DHCP reservation): Home Assistant only reads the address at startup, so if it changes, the connection silently stops working.

Want to charge on cheap electricity prices, a schedule, or anything else? See *Adding your own rules* below; nothing in the package itself needs changing.

A full restart of Home Assistant is needed after installing or updating, because the package adds new helpers and sensors.

---

## First start — test in this order

Doing it step by step means that if something fails, you know which part it is.

1. **Check the connection.** Load `alfen_modbus.yaml` and restart. The charger sensors should fill in: voltages around 230 V, a sensible temperature.
2. **Check that commands arrive.** Load the package and restart. In Developer Tools, run `script.alfen_set_current` with `amps: 6`. *Setpoint readback* should show 6 and *Setpoint accounted for* should show 1. If so, Home Assistant can control the charger.
3. **Set up your meter and fuse limit** before anything else. Check the four meter sensors under *Smart meter* (they start with the HomeWizard names; change them if yours differ) and make sure *Meter OK* is on. Until then charging is held at 6 A. Then set *Fuse per phase*. Wrong values here affect everything.
4. **Try the modes.** Try Fast, Solar and the Force 1-Phase / 3-Phase buttons. In Fast, switch on a big appliance and watch the car slow down, then speed up again when it's off. This is deliberately not instant: slowing down waits about 40 seconds, speeding up about a minute plus a 3-minute cooldown.
5. **Try Solar on a cloudy day.** Look at the history of *Commanded current*. Short clouds should be ignored and it should change in steady steps, not jump every 20 seconds. If it still follows every cloud, make the *Slow-down wait* longer.
6. **Check the event log** on the dashboard. A Force-phase button press is the quickest way to create an entry.

---

## Settings

Everything is adjustable from the dashboard; you never need to edit the code. Your values are kept across restarts. Each setting shows a suggested value, which is also used if a setting is ever left empty.

### Limits

| Setting | What it does | Suggested |
| --- | --- | --- |
| Max current | The most the car may draw per phase. Set to what your car can handle. | 16 A |
| Min current | The lowest current charging can run at. 6 A is the minimum every car and charger supports. | 6 A |
| Stop current | The value sent to pause the car. Anything below 6 A pauses charging. | 5 A |
| Fuse per phase | The most current the whole house (car plus everything else) may use on one phase. Set a little below your main fuse. | e.g. 24 A for a 25 A fuse |

### Smart meter

| Setting | What it does | Filled in on first start |
| --- | --- | --- |
| Total power sensor | Your meter's total power sensor. | `sensor.p1_meter_power` |
| L1 / L2 / L3 power sensor | Your meter's power sensor for each phase. | `sensor.p1_meter_power_phase_1` / `_2` / `_3` |
| Meter OK | Shows whether all four sensors are filled in, work and report watts. | — |

On first start the four fields are filled in with the names a HomeWizard P1 meter uses. If yours are different, change them on the dashboard; your values are kept across restarts. (Only empty fields are filled, so your own names are never replaced; an emptied field gets the HomeWizard name again at the next restart.) The logic only uses what is in the fields, with no hidden fallback. The sensors must report **watts**, positive for power taken from the grid and negative for power sent back. A sensor in kW, or a name that doesn't exist, keeps *Meter OK* off, and a warning shows on the dashboard.

**If the meter doesn't work** (*Meter OK* off for more than 30 seconds), the car is held at the minimum current (6 A) until it works again. Without meter data the fuse can't be protected, so this is the safe choice. Solar also stops making its own decisions until then, and the event log shows what happened. The 30 seconds ignore short hiccups and the meter still starting up after a Home Assistant restart.

### Fuse guard

The fuse guard checks every 20 seconds, in every mode — Solar included — so your fuse is protected the same way whatever mode you use.

| Setting | What it does | Suggested |
| --- | --- | --- |
| Step-down buffer | How far over the limit before it lowers the current. **Keep this small** — see below. | 1 A |
| Step-up buffer | How much spare room there must be before the current is raised again. Solar also never raises into this last bit of room. | 2 A |
| Scale-down settle time | How long the house must stay over the limit before it lowers the current. Ignores short spikes like a kettle. | 40 s |
| Scale-up settle time | How long there must be room again before it raises the current. | 60 s |
| Scale-up cooldown | After the guard lowered or stopped charging, nothing raises or restarts it for at least this long, in Solar too. Stops a switching appliance from bouncing the car up and down. | 3 min |
| Sustained overload timeout | If even 6 A doesn't fit, how long to wait before stopping completely. Short overloads are ridden out at 6 A. | 60 s |

**The step-down buffer is a safety margin, not a comfort setting.** The car can run this many amps over *Fuse per phase* before the guard reacts. With 24 A under a 25 A fuse, a 1 A buffer means the worst case is exactly your fuse rating; 2 A would go over it. Only raise it if you've left a bigger gap between *Fuse per phase* and your real fuse.

### Solar

These only apply in Solar mode, and changes take effect immediately.

| Setting | What it does | Suggested |
| --- | --- | --- |
| Solar phases | How many phases Solar may use: *Auto*, *1-phase only* or *3-phase only*. See below. | Auto |
| Keep charging at minimum | Off: pause when the sun is too weak. On: always charge at least at the minimum (about 1.4 kW on one phase, 4.1 kW on three), plus whatever the sun adds. Fewer stops and restarts, but it **also keeps charging at night, from the grid**, until the car is full or you change mode. Still stops if the fuse can't take the minimum. | Off |
| Start/stop hold | How long the sun must be strong enough (or too weak) before charging starts or pauses. | 3 min |
| Solar step-down buffer | How far the sun must drop before charging slows down. Small dips are ignored. | 2 A |
| Slow-down wait | How long the sun must stay lower before charging slows down. Clouds shorter than this are ignored completely. Longer is calmer but uses a bit more grid power. Speeding up never waits. | 60 s |
| Phase up at *(Auto only)* | Surplus needed before moving from one to three phases. Three phases need at least about 4.1 kW, so a higher value gives a margin. | 4800 W |
| Phase down at *(Auto only)* | Below this surplus it goes back to one phase. | 4140 W |
| Phase switch hold *(Auto only)* | How long the surplus must stay above or below those levels before switching. | 3 min |
| Phase switch cooldown | Minimum time between two phase switches, so it can't switch back and forth. | 12 min |

#### Which phase setting to choose

| Setting | Charging range* | Phase switches | Good for |
| --- | --- | --- | --- |
| Auto | 1.4 – 11 kW | Yes, when the sun changes enough | Most homes |
| 1-phase only | 1.4 – 3.7 kW | Never | Smaller solar installations, or cars that dislike restarts |
| 3-phase only | 4.1 – 11 kW | Never | Large solar installations |

\* With *Max current* at 16 A. Solar starts once there's enough sun for the minimum on the chosen phases: about 1.4 kW for one phase, 4.1 kW for three.

Every phase switch pauses charging for about 15 seconds, so the fixed choices are calmer. If you change the setting while the car is charging, or press a Force button in Solar mode, it moves back to the chosen phases, but not sooner than the *Phase switch cooldown* allows.

**If your car doesn't like frequent restarts**, the calmest Solar setup is *Solar phases* on **1-phase only** (or **3-phase only** with a large installation) and *Keep charging at minimum* **on**. That removes every start, stop and phase switch: the car simply keeps charging and follows the sun. Remember it then also charges at night, from the grid.

---

## Adding your own rules

The package decides *how* to charge; you decide *when*. Price-based charging, a timer, a button on your phone, a rule that charges fast when you're leaving early — all of these go in your own automations, outside the package.

**One rule:** your automations may only change the charge mode, `input_select.alfen_charge_mode` (`Off`, `Solar` or `Fast`). Never write current or phases yourself, and never call the package's scripts. That way the fuse protection and everything else keep working, and your rules can't conflict with the package.

Example — charge fast while a cheap-price period is active, and go back to Solar afterwards. Replace `binary_sensor.cheap_power` with your own sensor (for example from a dynamic-price integration):

```yaml
automation:
  - id: my_cheap_power_charging
    alias: Charge fast when power is cheap
    triggers:
      - trigger: state
        entity_id: binary_sensor.cheap_power
        to: ["on", "off"]
    actions:
      - service: input_select.select_option
        target:
          entity_id: input_select.alfen_charge_mode
        data:
          option: "{{ 'Fast' if trigger.to_state.state == 'on' else 'Solar' }}"
```

Put it in `automations.yaml` or in a package file of your own, not in `packages/alfen.yaml`, so updating the package never overwrites it.

## The event log

The dashboard shows a 48-hour log of every decision that changed what the car may draw:

- the fuse guard slowing down, speeding back up, or stopping
- Solar starting or pausing
- the car being unplugged or reporting a fault
- meter data missing, and the car being held at minimum
- every phase switch, including the Force buttons

Normal following of the sun is **not** logged — on a cloudy day that's dozens of small changes an hour and would drown everything else. The history of *Commanded current* shows that instead. Expect roughly 10–20 entries on a normal day.

---

## Charging energy

`sensor.alfen_energy_consumed_total_kwh` is the total energy delivered to the car, in kWh. It can be added to the Home Assistant **Energy dashboard**.

This is the electricity that went from the charger into the car — what you paid for. The car loses roughly 10–15% converting it for the battery, but that happens inside the car where neither the charger nor Home Assistant can measure it.

---

## Good to know

- **Fuse protection is a software safety net, not a replacement for your fuses.** It reacts within about a minute. If the smart-meter data is missing it can't see the fuse at all, and holds the car at 6 A until the meter is back. It prevents nuisance trips; it is not a circuit breaker.
- **Current changes in whole amps**, so each step is about 230 W on one phase or 690 W on three. That's how charging works, not a limitation of this package.
- **Solar waits on purpose.** It needs the sun to hold for a few minutes before it starts, stops or switches phases. On a changeable day it pauses more and switches less; that's the trade-off for not constantly stopping the car. All waiting times are adjustable.
- **A phase switch pauses charging for about 15 seconds**, and happens at most once per *Phase switch cooldown*.
- **The charger accepts at most two Modbus connections.** If evcc or another controller is still connected, Home Assistant's commands can be refused. Use only one controller.
- **During a long Home Assistant outage** (longer than the 300-second validity time) the charger falls back to its own Safe current and charges on its own. See *Safe current* above.
- **If the car reports a fault** (or is unplugged) for more than 10 seconds, charging is paused. In Fast mode it resumes automatically after the fuse guard's *Scale-up settle time*; in Solar through its normal start.
- **The car has a say too.** Some cars refuse very low currents, drop out when power changes, or won't use three phases without enough power. That can look like a problem with this package but is the car. *EV status* "Charging" means it's really drawing power; repeated switching between unplugged, error or "connected" states usually points to the cable or the car.
- **Some units don't report every value.** Chargers with certain meters (for example the Reallin meter on newer units) leave a few total values empty. This package doesn't use those; everything it relies on — voltages, currents per phase, charger power and delivered energy — is reported by all units. Check that *Charger draw* shows a value before relying on Solar.
- **Editing an automation in the Home Assistant UI takes it over.** If you open one of this package's automations under Settings → Automations and save it, Home Assistant stores a copy, and that copy wins from then on: changes to the package file seem to do nothing. Delete it there to hand control back to the package.
- **First start after an upgrade:** if some settings show up empty, fill them in once from the dashboard (suggested values are shown). They're kept from then on.

---

## Technical details

This section is for anyone changing the code.

### Structure

| Part | Role |
| --- | --- |
| `script.alfen_set_current` | The only way current is changed. Writes register 1210 (float32, two 16-bit words, big-endian) and records the value in `input_number.alfen_target_current`, which every loop treats as "what we last commanded". See *Why five automations and not one* below. |
| `script.alfen_switch_phases` | Pause (stop current) → wait 10 s → write register 1215 → wait 5 s → resume at min current. Stamps `input_datetime.alfen_last_phase_switch`. |
| `alfen_setpoint_renew` | Keep-alive. Every 30 s while a car is connected, asks the write script to resend the current value. Kept separate so nothing can delay it. |
| `alfen_fuse_guard` | Every 20 s, all modes. The only thing that lowers or stops charging for the fuse, in every mode, so fuse behaviour is identical in Solar and Fast. Also raises again in Fast; in Fast it is the only thing that writes after the mode is applied. |
| `alfen_solar` | Every 20 s in Solar. One `choose`, so at most one action per tick: start, stop, correct the phases (fixed phase setting), phase up / down (Auto only), follow the sun down (after the slow-down wait) or follow the sun up. |
| `alfen_charge_mode_apply` | Applies the mode on mode change, Home Assistant start, and plug-in. |
| `alfen_disconnect_pause` | Pauses when the car is unplugged or faulted (Mode 3 A/E/F for 10 s). |

**Meter data** comes in through `sensor.alfen_grid_power` and `sensor.alfen_grid_l1/l2/l3`, which read whichever meter sensors are set in `input_text.alfen_meter_*`. On Home Assistant start, empty fields are filled with the HomeWizard names (never overwriting a value); beyond that there is no fallback, so the logic only uses what is in the fields. They're only available when the source exists, is a number and is in W. Everything else reads these four, never the meter directly. `binary_sensor.alfen_meter_ok` is on when all four are available; after 30 s off, the fuse guard holds at min current and the Solar loop stands still. Mode routing in Fast sets min current instead of max without meter data, except right after a Home Assistant start, when it leaves the current alone so a slow-loading meter can't cause a needless drop.

The control loops are **level-triggered**: they re-check the current situation on every tick instead of reacting to a value crossing a threshold, so they can't miss a change. Durations ("surplus has been above X for N minutes") are measured from the `last_changed` of dedicated binary sensors.

**Solar surplus** is `charger power − net grid power`. The car's own draw appears in both terms and cancels out, so a charging car doesn't hide the surplus. The follow-the-sun target (`sensor.alfen_solar_target`) is `surplus ÷ (phases × 230)`, between min and max current, and deliberately **not** limited by fuse headroom.

**How Solar and the fuse guard share the work.** The guard is the only part that lowers or stops for the fuse, in every mode, with its own buffers, settle times and overload timeout. Solar only follows the sun:

- **Down** when `binary_sensor.alfen_solar_should_lower` (the current is at least the Solar step-down buffer above the target) has been on for the slow-down wait. It lowers to the target, or to the fuse room if that's even lower, so a sun drop and a house load never cost two separate restarts.
- **Up** straight away (raising causes no restart), but never into the last *Step-up buffer* amps of fuse room, and not during the guard's *Scale-up cooldown*. Starts also wait for that cooldown.

That removes the difference that used to exist: Solar reacted to a kettle within 20 seconds, while Fast waited for the guard's settle time.

**Solar phases** (`input_select.alfen_solar_phases`) sets the phase count Solar starts on and, in the fixed choices, always uses. The start threshold (`binary_sensor.alfen_surplus_over_start`) follows it: min current × 230 V on one phase, × 3 on three. The start check also uses the headroom of the phases it will start on — the tightest of all three for *3-phase only* — so it never switches to three phases only to find one of them full. Any value other than the two fixed choices is treated as *Auto*.

**Headroom** per phase is `fuse per phase − house load on that phase`, capped at *Fuse per phase*. The cap matters when exporting: the smart meter reports *net* power, so the house load can come out negative and the raw headroom would exceed the fuse. That extra only exists while the sun holds, so it's never offered to the car. In one-phase mode L1 governs; in three-phase mode the tightest phase does.

### Why five automations and not one

The five automations run independently, so two of them can want to change the current at the same moment: the fuse guard and the Solar loop both tick every 20 s, and the keep-alive every 30 s. Merging them all into one automation that does everything strictly one step at a time would rule that out by design, but it would also make one very long automation that is harder to read and change.

Instead, every write goes through one place, `script.alfen_set_current`, and that script makes simultaneous writes harmless:

- **One at a time, in order.** It runs *queued*: a write that arrives while another is busy waits its turn instead of being dropped.
- **Never an old value.** The keep-alive doesn't pass a number; the script reads the current value at the moment it writes. So the keep-alive can't put back a value that the fuse guard has just lowered.
- **Nothing during a phase switch.** While `script.alfen_switch_phases` runs, the script refuses every write except the switch's own. This is checked at the moment of writing, not only when an automation starts.

`script.alfen_set_phases` has the same protections. Keep it that way: **never write the charger's registers directly from an automation — always go through these two scripts.**

### Rules that must not be broken

Each of these caused a real failure when it was broken. They're easy to break by accident because the result often looks like it works.

1. **Never compare the raw Mode 3 register.** The charger sends the state as a 10-byte text value, so "unplugged" arrives as `A` plus nine padding characters and never equals `'A'`. Any "is a car connected?" check against `sensor.alfen_mode3_state` is always true. Always use the cleaned `sensor.alfen_mode3`. You can see the padding in Developer Tools → Template: `{{ states('sensor.alfen_mode3_state') | length }}` returns 10.

2. **Never write current while a phase switch is running.** The switch script lowers the current and holds it for 10 seconds so the phases change with no load. A loop that sees that low value will think there's room and raise it again — into the middle of the pause. The charger then won't switch and stays on the old number of phases. The write script refuses any write during a switch except the switch's own (see above), and the fuse guard and Solar loop also don't start while a switch runs.

3. **Nothing that switches phases may be triggered by Mode 3 changes — except plugging in.** A phase switch itself changes Mode 3 (charging → connected → charging). An automation with `mode: restart` that both triggers on Mode 3 and calls the switch script will restart itself in the middle of the switch; because the script is called as a service it is part of that run and gets cancelled too. `alfen_charge_mode_apply` therefore only triggers on Mode 3 leaving `A` (plug-in), which a phase switch never causes. Everything else Mode-3-related lives in `alfen_disconnect_pause`, which never switches phases.

4. **Never measure a waiting time from a sensor that changes constantly.** If a settle time is measured from the `last_changed` of the headroom sensor, the timer restarts on every meter update and never completes, so the guard never acts, although it looks correctly configured. Measure from a binary sensor that only changes when the condition itself flips (`alfen_guard_should_trim`, `alfen_guard_can_recover`, `alfen_solar_should_lower`).

5. **Never read time from an `input_datetime` state string.** It's local time without a time zone and gets read as UTC, which shifts cooldowns by hours. Store with `now().timestamp()` and read with `state_attr(..., 'timestamp')`.

6. **Don't make the fuse guard tick faster than the smart meter updates.** It would act several times on the same outdated reading. A 20-second tick suits a meter that updates about every 10 seconds. With a faster meter you can lower it, together with the scale-down settle time.

### Logging

Every log entry is attached to `input_boolean.alfen_events`, a helper that is never switched and exists only for this. That's what lets the dashboard's log card show these events and nothing else. The one decision that is deliberately not logged is Solar following the surplus.

### Register reference

Slave **200** = station, slave **1** = socket. All values big-endian, no word swap.

| Register | Content | Type | Access |
| --- | --- | --- | --- |
| 1100–1101 | Station active max current (hard ceiling) | float32 | read |
| 1102–1103 | Board temperature | float32 | read |
| 1201–1205 | Mode 3 state A–F (padded text — see rule 1) | string | read |
| 1206–1207 | Actual applied max current | float32 | read |
| 1208–1209 | Setpoint remaining valid time | uint32 | read |
| **1210–1211** | **Max current setpoint** | float32 | **read/write** |
| 1212–1213 | Safe current | float32 | read |
| 1214 | Setpoint accounted for (1 = charger is using it) | uint16 | read |
| **1215** | **Number of phases, 1 or 3** | uint16 | **read/write** |
| 306–336 | Voltage, current, power factor, frequency per phase | float32 | read |
| 374 / 390 | Energy delivered / consumed | float64 | read |

`sensor.alfen_setpoint_accounted_for` (1214) is the quickest check that commands are arriving: **1** means the charger is using Home Assistant's value.

---

## Disclaimer

> **USE AT YOUR OWN RISK.**
> This software controls high-voltage electrical equipment (an EV charger) and interacts with your household electrical installation. It is provided as-is, for personal automation and educational purposes.
>
> - The authors accept **no responsibility or liability** for any damage to your charger, vehicle, fuses, wiring, software or property.
> - Make sure your physical protection (fuses, circuit breakers, residual-current devices) is installed correctly by a qualified electrician. Software limits such as the fuse guard in this package react after the fact and are **not a substitute for hardware protection**.

---

## License

This project is licensed under the **MIT License** — free to use, modify, distribute, and integrate without restriction.

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.