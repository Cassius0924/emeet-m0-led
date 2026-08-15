# eMeet M0 LED control in Home Assistant

Two `template select` entities backed by a `shell_command` that calls
[`emeet-led`](../emeet-led) inside the HA container.

```yaml
# configuration.yaml
shell_command:
  # NOTE: HA runs commands with templates via shlex (no shell), so wrap in
  # bash -c when you need redirects/pipes.
  emeet_led_set: "bash -c 'python3 /config/scripts/emeet-led \"{{ ring }}\" \"{{ button }}\"'"

template:
  - select:
      - name: "eMeet M0 Ring"
        unique_id: emeet_m0_ring
        icon: mdi:led-strip-variant
        options: "{{ ['off', 'red', 'blue', 'green'] }}"
        select_option:
          action: shell_command.emeet_led_set
          data:
            ring: "{{ option }}"
            # green ring forces the button green on hardware; unknown/edge
            # states fall back to 'off' so the script never gets bad args
            button: "{{ 'off' if option == 'green' or states('select.emeet_m0_jie_ting_jian_deng') not in ['off', 'red'] else states('select.emeet_m0_jie_ting_jian_deng') }}"

      - name: "eMeet M0 Button"
        unique_id: emeet_m0_button
        icon: mdi:phone-outline
        # red button is only valid when the ring is not green
        options: "{{ ['off'] if is_state('select.emeet_m0_yuan_huan_deng', 'green') else ['off', 'red'] }}"
        select_option:
          action: shell_command.emeet_led_set
          data:
            ring: "{{ states('select.emeet_m0_yuan_huan_deng') if states('select.emeet_m0_yuan_huan_deng') in ['off', 'red', 'blue', 'green'] else 'off' }}"
            button: "{{ option }}"
```

## Setup

1. Copy `emeet-led` into the HA config directory, e.g. `/config/scripts/emeet-led`
   (inside the container it needs rw access to `/dev/hidraw0` — the HA
   container must be privileged or have the device mapped).
2. Add the config above to `configuration.yaml`, restart HA.
3. Entities (IDs depend on your registry — rename preserves old IDs):
   - `select.emeet_m0_yuan_huan_deng` — ring: off / red / blue / green
   - `select.emeet_m0_jie_ting_jian_deng` — button: off / red

## Notes / pitfalls

- HA runs templated `shell_command`s with `asyncio.create_subprocess_exec`
  + `shlex.split` — **no shell**. Use `bash -c '...'` if you need `>>`, `|`, etc.
- The green ring (report 3) forces the button LED green on the hardware, and
  green-button combos with red/blue rings are unsupported — the config above
  prevents invalid combinations from being sent.
- Template `select` options must be a Jinja-rendered **list**
  (`"{{ ['off', 'red'] }}"`), even though the YAML schema wants a string.
- Renaming a template entity keeps its old entity_id (registry), so query
  `/api/states` after setup to learn the actual IDs.
