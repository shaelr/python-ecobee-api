# python-ecobee-api (fork)

A fork of [nkgilley/python-ecobee-api](https://github.com/nkgilley/python-ecobee-api)
0.4.1 — the Python client for the ecobee API. Hosted at
[shaelr/python-ecobee-api](https://github.com/shaelr/python-ecobee-api).

## Why this is a separate repo, not vendored

The sibling project **[ha-ecobee](https://github.com/shaelr/ha-ecobee)** (a
HACS-installable Home Assistant custom integration) depends on this package.
It's kept as its own repo, referenced from `ha-ecobee`'s `manifest.json` via
a pinned git URL (`python-ecobee-api @ git+https://github.com/shaelr/python-ecobee-api.git@vX.Y.Z`),
rather than being copied into that repo. This is deliberate: it keeps this
free to be PR'd upstream to `nkgilley/python-ecobee-api` independently of
the HA-specific integration work.

If you're adding a new `Ecobee` client method for the HA integration to use,
it goes here, gets its own version bump in `setup.py` + git tag + GitHub
release, and then `ha-ecobee`'s `manifest.json` requirement gets bumped to
point at the new tag. See that repo's `CLAUDE.md` for the full release
sequence HACS requires.

**Only push/tag/release when the user explicitly asks.** Implement and
commit locally by default.

## Two categories of setter method — this distinction matters

`Ecobee`'s setter methods in `pyecobee/__init__.py` fall into two groups
with meaningfully different behavior, which the HA integration's UI-update
logic depends on:

1. **Pure POST, no local mutation**: most of the original methods —
   `set_hold_temp`, `set_hvac_mode`, `set_fan_mode`, `set_climate_hold`,
   `resume_program`, etc. These just build a request body from scratch and
   POST it. They never touch `self.thermostats[index]`.
2. **Read-patch-POST, local mutation before the network call**:
   `update_climate_sensors` (pre-existing) and the three added this session
   — `set_climate_temperatures`, `set_climate_fan_mode`,
   `set_schedule_slots`. These read `self.thermostats[index]["program"]`,
   mutate the relevant piece of it in place (a climate's temps/fan, or a
   schedule day's slots), *then* POST the whole patched `program` back. The
   mutation happens on the shared in-memory dict before the network request
   even starts.

If you add a new setter, decide up front which category it belongs to and
follow the existing pattern for that category — category 2 needs the
"pop `currentClimateRef`, mutate, POST `{"thermostat": {"program": programs}}`"
shape that `update_climate_sensors`/`set_climate_temperatures`/etc. use.
This distinction is why the HA integration can safely display category-2
writes instantly with zero network round trip, but category-1 writes need
to force an actual re-poll to see the new value — see `ha-ecobee`'s
`CLAUDE.md` for the full explanation and the exact bug history that came
from getting this mixed up (calendar.py briefly did an unnecessary re-fetch
after a category-2 write, which raced ecobee's own eventual consistency).

## New methods added this session (v0.5.0)

- `set_climate_temperatures(index, climate_ref, heat_temp=None, cool_temp=None)`
- `set_climate_fan_mode(index, climate_ref, fan_mode)`
- `set_schedule_slots(index, day_index, start_slot, end_slot, climate_ref)`
- `InvalidClimateError` (in `errors.py`) — raised by all three above when
  `climate_ref` doesn't match anything in the thermostat's program.

`_find_climate(programs, climate_ref)` is the shared lookup helper for all
three.

## No live test environment

No real ecobee thermostat is available while developing this. New methods
are verified with `python3 -m py_compile` plus standalone scripts that mock
`Ecobee._request_with_refresh` and assert the constructed request body —
see the git history around the v0.5.0 commit for the pattern used. Every
method added here still needs real-account verification once the HA
integration side is actually installed and tested.
