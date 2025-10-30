# ESPHome Package: Timed Switch

## Overview

What could be simpler than turning a switch on for a limited time with a single
press in ESPHome? However, things get more complicated when you want to add an
override — for example, turning on yard lights upon motion detection for a set
period while still allowing manual control to keep them on until you turn them
off.

The `timed_switch` package solves this problem by adding timed behavior with
override capability to an ESPHome configuration.

For an existing switch, it exposes two user-facing switches:
* One for triggering "timed" operation with a configurable delay.
* Another for manual control, allowing you to turn the switch on or off without
  interference from the timed operation.

This package works with any existing `switch` entity in ESPHome, not just
lights. The example above is just one of many possible use cases.

## How It Works

1. When `${timed_switch_id}_timed` is turned on, the package checks three
   conditions in order:
   * If the timed operation is already running, it is restarted, extending the
     timer. In other words, turning on the timed switch again while the timed
     operation is in progress resets the timer, prolonging the ON duration.
   * If the physical switch is already ON, no action is taken, implementing the
     override capability.
   * If an additional condition (`${timed_switch_additional_timed_condition}`)
     evaluates to `false`, the timed operation is skipped. This can be used to
     prevent timed operation based on external conditions (e.g., a light sensor
     detecting daylight).
2. If those checks pass, the physical switch is turned on, followed by a wait
   for the number of seconds configured (via `${timed_switch_id}_turn_off_delay`
   control), and then turned off.
3. The override switch (`${timed_switch_id}`) provides manual control over the
   physical switch and stops the running timed operation when manually turned
   on.

## Configuration

This package is designed to be configured via substitutions. The following
substitutions are available:

* `timed_switch_delay_name` (string, optional): Name for the turn-off delay
  number entity. If not provided, defaults to `${timed_switch_name} turn off delay`.
* `timed_switch_max_delay` (integer, optional, default: 900): Maximum allowed
  timeout in seconds for the turn-off delay. Use this to cap how long the timed
  operation can run.
* `timed_switch_initial_delay` (integer, optional, default: 60): Initial value
  (in seconds) for the turn-off delay.
* `timed_switch_delay_step` (integer, optional, default: 1): Step size (in
  seconds) for adjusting the turn-off delay.
* `timed_switch_delay_internal` (boolean, optional, default: false): If set to
  `true`, the turn-off delay number entity will be marked as `internal`, hiding
  it from Home Assistant UI.
* `timed_switch_delay_restore` (boolean, optional, default: true): If set to `false`,
  the turn-off delay will reset to `${timed_switch_initial_delay}` on each restart.
* `timed_switch_physical_switch_id` (string, required): ID of the physical
  `switch` entity that the component will control. This must match an existing
  `switch` ID in your device configuration.
* `timed_switch_name` (string, required): Name for the timed switch, used as a
  prefix for other names.
* `timed_switch_id` (string, required): ID for the timed switch, used as a
  prefix for other IDs.
* `timed_switch_internal` (boolean, optional, default: false): If set to `true`,
  the override switch (ID: `${timed_switch_id}`) will be marked as `internal`, hiding it from Home Assistant UI.
* `timed_switch_restore` (string, optional, default: `RESTORE_DEFAULT_OFF`): Controls
  the restore behavior of the override switch, see
  [ESPHome documentation](https://esphome.io/components/switch/index.html#restore-mode)
  for possible values.
* `timed_switch_timed_name` (string, optional): Name for the timed operation switch.
  If not provided, defaults to `${timed_switch_name} timed`.
* `timed_switch_timed_internal` (boolean, optional, default: false): If set to `true`,
  the timed operation switch (ID: `${timed_switch_id}_timed`) will be marked as `internal`, hiding it from Home Assistant UI.
* `timed_switch_additional_timed_condition` (lambda/boolean, optional, default:
  true): An optional C/C++ expression evaluated before starting the timed
  script. If it evaluates to `false`, the timed start will be skipped.
  Example usage: `timed_switch_additional_timed_condition: "id(my_sensor).state"`

**NOTE**: It is recommended to set the physical switch, referred to in
`${timed_switch_physical_switch_id}`, as `internal: true` to avoid exposing it
directly in Home Assistant, which could interfere with the timed switch
behavior.

## User-Visible Entities

* **Number** (ID: `${timed_switch_id}_turn_off_delay`): The number of seconds
  the physical switch will remain ON when triggered.
* **Switch** (ID: `${timed_switch_id}_timed`): A switch that starts the timed
  operation when turned on. Turning this off will directly turn the physical
  switch off.
* **Switch** (ID: `${timed_switch_id}`): Directly controls the physical switch
  and blocks the timed operation. Any subsequent triggers to the timed switch
  above will be ignored while this switch is ON. Turning it off simply turns
  the physical switch off.

## Home Assistant Actions

The package provides the following Home Assistant actions to integrate with
automations:

* **`turn_on_timed_${timed_switch_id}`**

  This action starts the timed switch behavior programmatically. It accepts an
  optional parameter `delay_sec` to override the default turn-off delay:
  * If `delay_sec` is a non-zero value, it specifies the duration (in seconds)
    for which the physical switch will remain ON.
  * If `delay_sec` is zero or omitted, the value of the
    `${timed_switch_id}_turn_off_delay` number entity is used.

### Example Automation

Below is an example of how to use the `turn_on_timed_${timed_switch_id}` action
in a Home Assistant automation:

```yaml
automation:
  - alias: "Turn on timed switch with custom delay"
    trigger:
      - platform: state
        entity_id: binary_sensor.motion_sensor
        to: "on"
    action:
      - service: esphome.turn_on_timed_light
        data:
          delay_sec: 120  # Override the default delay with 120 seconds
```

In this example, the `turn_on_timed_light` action is triggered when motion is
detected, and the physical switch remains ON for 120 seconds.

## Example Usage

Below is an example showing a typical configuration for a single switch.

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

Multiple switches can also be handled:

```yaml
substitutions:
  timed_switch_max_delay: '300'
  timed_switch_initial_delay: '60'

packages:
  timed_switch:
    url: github://hostcc/esphome-package-timed-switch/
    ref: main
    files:
      - path: main.yaml
        vars:
          timed_switch_name: 'Tap 1'
          timed_switch_id: 'tap_1'
          timed_switch_physical_switch_id: 'garden_valve_1'
      - path: main.yaml
        vars:
          timed_switch_name: 'Tap 2'
          timed_switch_id: 'tap_2'
          timed_switch_physical_switch_id: 'garden_valve_2'

switch:
  - id: garden_valve_1
    internal: true
    ...
  - id: garden_valve_2
    internal: true
    ...
```
