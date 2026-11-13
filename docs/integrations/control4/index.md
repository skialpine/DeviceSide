# Control4

What DeviceSide's features do on Control4, their limits, and what DeviceSide changes in your setup. What applies
to every integration is in [integrations.md](../../integrations.md); which devices each feature is proven on is
on the [device compatibility page](../../device-compatibility.md); how the calls work, and what was measured, is
in [internals.md](internals.md).

Lights on the core `control4` integration: dimmers and Control4 light groups alike. DeviceSide borrows the
integration's own connection to the controller's Director, so it needs no sign-in of its own. A light it can't tie to
a Director item is refused (`not_addressable`). Every action here is flagged experimental: turn on the Experimental
features option to use it ([try it](#try-it-experimental-features)).

## Smooth dimming

`raise_brightness` and `lower_brightness` read the light's own level from the Director, then send one `RAMP_TO_LEVEL`
the light runs itself. A light already ramping, as when the direction or the limit changes with no `stop` between, is
halted first and moved from where it stopped; a repeat of the same call leaves its ramp running. `stop` sends
`SET_BRIGHTNESS_STOP`, which halts the ramp, and reads back where it halted. A full-range ramp takes at most 65.5 s,
so a longer `sweep_time` is clamped. Lowering to 0 % ends off. Raising a light that is off ramps it up from the
bottom. A Control4 light group takes one command, so its lights move together.

## Flash-free turn-on

`set_attributes` takes `color_temp_kelvin` alone. Home Assistant shows a Control4 light as brightness-only, so this is
the way to set its color temperature at all: the field doesn't appear in the action's form for these lights, but a
call in YAML that carries it is sent. The kelvin is clamped to the range the light itself reports, and the result then
carries `clamped_to`, the kelvin sent. A light that reports no color temperature is left `unchanged`
(`fields_dropped`), so a room that holds one still sets the others. A light that is on changes to it and is
`confirmed`, as Control4 reports it ([internals](internals.md)), or `sent` (`color_temp_not_reported`) if Control4
never shows it. A light that is off stays off and comes on at the new color temperature. Either way it lasts until
Control4 next applies the light's own On Color preset. While off, Control4 shows it for a little over a second and
then the old one again, so the result is `sent` (`stored_while_off`).

On a light that dims, `brightness`, `min_brightness` and `max_brightness` are refused (`fields_not_tested`). Control4
can store the level for a light's next on, but Home Assistant's own turn-on always sends a level, so a brightness
written while off would be lost to it. Colors are dropped (`fields_dropped`): no full-color Control4 light has been
tested.

## Get attributes

`get_attributes` reads the light from Control4 directly: `brightness` as Control4 holds it, which mid-ramp is up to
about a second old, where Home Assistant's own state is as of its last poll of Control4 (every five seconds by
default), and `color_temp_kelvin`, which Home Assistant doesn't show. Control4 holds a color temperature in tens of
kelvin, rounded down, so one set to 2203 reads 2200. A light that is off reports no brightness. Its
color temperature is the one Control4 shows: after a `set_attributes` on a lit light, that is the new one most of a
second later; after one while off, that is the new one for a little over
a second and then the old one again, until the light comes on.

## Try it: Experimental features

### Control4 lights

On a Control4 dimmer or light group, a test shows that the Director reports the light's level during a `RAMP_TO_LEVEL`
ramp, that `SET_BRIGHTNESS_STOP` halts the ramp where it is, whichever way it runs, and that the same command on a
light at rest changes nothing. One contributor saw all three on a light group of [Hue
downlights](#hue-white-ambiance-56-retrofit-downlight) (Control4 OS 4.3, 10 and 13 Nov 2026). Control4 documents none
of it, and no other driver has been tried. Home Assistant's own Control4 integration also reads a `LIGHT_LEVEL`
variable, which these lights don't report, so some drivers presumably report that one; DeviceSide assumes, untested,
that such a light reports no ramp target, and then `stop` on it reports `sent` (`halt_not_reported`), never
`confirmed`. One report per driver.

What the same test found, which the adapters rely on ([measurements](internals.md#tested-behavior)):

- **A ramp takes at most 65.5 s.** The Zigbee bulb driver gives its longest rate as 65535 ms (`brightness_rate_max` in
  the light's own capabilities variable, read from the Director); the light group's driver gives 86400000 ms. A call
  may reach either, so DeviceSide clamps to the shorter.
- **The Director shows a halt a moment late.** Usually within 0.2 s of the stop's reply; once in twenty, not within
  0.35 s. `stop` reads back for a little over a second before reporting `still_fading`: Control4 hadn't shown the halt
  yet, the driver ignored the stop, or a new move began meanwhile.
- **Mid-ramp, the level Control4 shows is up to a second old.** A ramp just begun still shows the level it left. So a
  raise or lower that finds the light ramping halts it and reads where it stopped, which costs that move 0.1 to 0.25
  s; if Control4 hasn't shown the halt by then, the move starts from the level shown. That covers a change of
  direction or of limit with no `stop` between, a ramp started from Control4, and a press within about a quarter of a
  second of a `stop`, while Control4 still shows the halted ramp. The one ramp left running is DeviceSide's own to the
  same end, as a remote that repeats while held asks for: nothing is sent, the ramp keeps its speed, and the call
  reports `sent`. A light that reports no ramp target always moves from the level shown: send `stop` before changing
  direction.
- **A color temperature set while off isn't shown for long.** The light stays off and comes on at it, but Control4
  shows it for a little over a second and then the old one again, so `set_attributes` reports `sent`, not `confirmed`,
  for a light that is off.
- **A light that has never reported reads as text.** Lights added to a project but not yet paired report `"0"`, as
  text, for their level; paired lights report numbers. DeviceSide reads either.

### Hue White Ambiance 5/6" Retrofit Downlight

The lights behind the hand tests above: `LTD010` (Philips 5996411U5), joined to Control4 over Zigbee on its
"Zigbee 3 Light" driver, three of them in a Control4 light group. Flagged experimental like every Control4 light, so
it isn't listed under Works.
