---
title: "BEAM: Nerves"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Nerves

*Nerves turns an Elixir/OTP release into a firmware image for embedded Linux boards. The device boots into your supervision tree and nothing else.*

---

## What it is

Nerves is a framework and toolchain for building embedded devices on
the BEAM. It uses Buildroot to produce a small, purpose-built Linux
system per hardware target (Raspberry Pi models, BeagleBone and others)
and packages your OTP release on top. The resulting firmware image
contains a kernel, a minimal root filesystem, the Erlang runtime and
your applications. There is no general-purpose distribution and no
package manager on the device; the root filesystem is read-only and
application data lives on a separate writable partition.

---

## How you use it

```
mix nerves.new sensor_node                  # generate a project
MIX_TARGET=rpi4 mix deps.get
MIX_TARGET=rpi4 mix firmware                # build the image for that board
MIX_TARGET=rpi4 mix firmware.burn           # write it to an SD card
MIX_TARGET=rpi4 mix upload                  # later: push new firmware over the network
```

`MIX_TARGET` selects the hardware system; with no target the project
builds for the host, which is where most logic is developed and tested.

Hardware access goes through ordinary libraries: the Circuits family
(`Circuits.GPIO`, `Circuits.I2C`, `Circuits.SPI`, `Circuits.UART`) exposes
buses and pins, and `nerves_runtime` exposes device-level concerns such
as firmware metadata, reboot and firmware validation. A sensor is
typically a GenServer that owns the bus handle and polls on a timer
sent to itself, so a bad read crashes and restarts one small process.

---

## Updates and rollback

Nerves firmware uses two slots (A/B). A new image is written to the
inactive slot and the device boots it tentatively. The application
confirms it with `Nerves.Runtime.validate_firmware/0`; if it never does,
the bootloader can revert to the previous slot. This is the same
"run it, then commit it" discipline as OTP's current/permanent releases
([HOT_CODE_UPGRADES](HOT_CODE_UPGRADES.md)), applied to a whole device.

## Pitfalls

- **Validate late.** Mark new firmware valid only after the parts that
  matter (network, mesh connection) are up, or rollback will never
  trigger when it should.
- **Keep writable state small and explicit.** Everything outside the
  data partition is replaced on update.
- **Test on real hardware early.** Timing, power and bus behaviour
  differ from host mocks.
- **Supervise the hardware edge narrowly** so a flaky peripheral
  restarts its own process, not the application
  ([SUPERVISION_TREES](SUPERVISION_TREES.md)).

## Relevance to the mesh

Edge devices that boot straight into an OTP release can run the same
code and supervision patterns as server nodes. Embedded is not a
separate world: it is the same [application](APPLICATIONS.md) structure
with a few hardware processes at the leaves.

## Sources

- *Build a Weather Station with Elixir and Nerves*, 1st edition, Alexander Koutmos, Bruce A. Tate and Frank Hunleth, Pragmatic Bookshelf, 2022. <https://pragprog.com/titles/passweather/build-a-weather-station-with-elixir-and-nerves/>
- Nerves documentation, Getting Started. <https://nerves.hexdocs.pm/getting-started.html>
- `nerves_runtime` documentation (firmware slots and validation). <https://nerves-runtime.hexdocs.pm/readme.html>
- `circuits_gpio` documentation. <https://circuits-gpio.hexdocs.pm/readme.html>
- Nerves Project. <https://nerves-project.org/>
