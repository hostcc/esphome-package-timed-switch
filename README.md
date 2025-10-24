# ESPHome configuration for timed switch

## Overview

`timed_switch` is an ESPHome configuration that provides a timed switch behavior.
It exposes a
user-facing template switch and a secondary "timed" switch which, when turned
on, runs a restartable script that turns a physical/internal switch on for a
configurable number of seconds and then turns it off.

This pattern is useful for things like a garage door opener, irrigation valve,
or any actuator where you want a single press to turn on the device for a
limited time and allow subsequent presses to extend/reset the timer.

## Features

- Restartable timeout script: repeated activations while the script is running
  will restart it (extend the running time).
- Separate template switch for triggering the timed behavior and for manual
  control.
- Configured using simple substitutions to allow reuse in multiple devices.

## Configuration

This component is designed to be included via substitutions (see the example
below). The component relies on a physical switch entity, which it
controls. The following substitutions are used in the component's
template:

- `timed_switch_max_delay` (integer, optional): maximum allowed timeout in
  seconds for the `turn_off_delay` number input, defaults to `900` seconds.
- `timed_switch_initial_delay` (integer, optional): initial value for the
  `turn_off_delay` number input (seconds), defaults to `60` seconds.
- `timed_switch_physical_switch_id` (string, required): the physical switch
  ID (internal/physical) which will actually be turned on/off by the
  component. This switch must exist in the device configuration.
- `timed_switch_name` (string, optional): human-friendly name used for
  entity names.
- `timed_switch_id` (string, required): base ID used to generate entity IDs
  (the component creates `${timed_switch_id}_timed`, `${timed_switch_id}`,
  etc.).
- `timed_switch_additional_timed_condition` (boolean, optional): an extra
  lambda-style condition that can be injected to gate starting the timed
  script, defaults to `true`.

## Entities created

- Number: `${timed_switch_id}_turn_off_delay` — seconds the physical switch
  will remain ON when triggered. This number entity is created and managed by
  the component itself; you do not need to define it in your configuration.
- Script: `timed_switch_script_${timed_switch_id}` — a restartable script that
  turns the physical switch on, delays for `turn_off_delay` seconds, then turns
  it off. This script is part of the component and is created automatically.
- Switch (physical): `${timed_switch_physical_switch_id}` — a reference to the physical switch ID. This switch must be defined elsewhere in your configuration; the component does not create or extend it.
- Switch (template): `${timed_switch_id}_timed` — a switch that starts the
  timed script when turned on (subject to guards). Turning this off will
  directly turn the physical switch off.
- Switch (template): `${timed_switch_id}` — the user-facing switch which will
  stop the running script and turn the physical switch on when manually
  toggled on; turning it off simply turns the physical switch off.

## How it works

1. When `${timed_switch_id}_timed` is turned on, the component checks three
   conditions in order:
   - If the physical switch is already ON: no action is taken (prevents
     redundant starts).
   - If `timed_switch_additional_timed_condition` evaluates to false: the
     timed start is skipped.
   - If the script is already running: the script is restarted (the timer is
     reset/extended).
2. If checks pass, the restartable script `timed_switch_script_${timed_switch_id}`
   is executed. The script turns the physical switch on, waits for the
   configured number of seconds (from the component-created `number` entity),
   then turns the physical switch off.
3. The primary template switch (`${timed_switch_id}`) provides manual control
   over the physical switch and will stop the running timeout script when
   manually turned on (ensuring explicit manual control overrides the timed
   behavior).

## Example usage

Below is a compact example showing typical substitutions. You do NOT need to
define the supporting `number` or `script` entities yourself — the component
creates and manages them. The only entity you must provide in your device
configuration is the physical `switch` referenced by
`timed_switch_physical_switch_id`.

```yaml
substitutions:
  timed_switch_max_delay: '300'
  timed_switch_initial_delay: '60'
  timed_switch_name: 'Tap'
  timed_switch_id: 'tap'
  # Set to your physical switch ID
  timed_switch_physical_switch_id: 'garden_valve'

packages:
  timed_switch: github://hostcc/esphome-package-timed-switch/main.yaml@main

switch:
  - id: garden_valve
    internal: true
    ...
```

Usage note: the component creates `${timed_switch_id}_timed` and
`${timed_switch_id}` template switches. Turning `${timed_switch_id}_timed` ON
will start or restart the timeout script (the duration is read from the
component-created `${timed_switch_id}_turn_off_delay` number). Turning
`${timed_switch_id}` ON gives manual control and stops the running timeout
script.

## Notes

- The component expects the physical switch ID referenced by
  `timed_switch_physical_switch_id` to exist. In many templates the physical
  switch is added as an `!extend` so the component can mark it internal.
- The restartable script uses `mode: restart` so each execution resets the
  delay — this allows using the timed switch with motion sensors or repeated
  triggers to extend the active period.
