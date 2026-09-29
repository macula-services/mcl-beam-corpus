---
title: "BEAM: Nerves"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Nerves

*The BEAM on bare metal: Nerves packages an OTP release into a bootable firmware image. Elixir from the sensor to the cloud, one language.*

---

## What Nerves is

Nerves builds a **firmware image** — a complete, minimal Linux system
whose only application is your OTP release. `mix firmware` produces
an image you flash to a device; the device boots directly into your
Elixir application.

| Piece | What it provides |
|-------|------------------|
| **Nerves core** | The minimal root filesystem: kernel + busybox + the BEAM |
| **`mix firmware`** | The release, baked into a bootable image |
| **`mix nerves.burn`** | Flash the image to an SD card |
| **Nerves.Runtime** | The firmware's interfaces: networking, file system, reboot |

The result: a device that is **one OTP release from boot to
application** — no distro, no package manager, nothing on the device
you did not put there.

---

## Why it matters on the BEAM

- **One language, whole stack.** Sensors, business logic, and the
  cloud API are all Elixir — no C for the device and Elixir for the
  server.
- **The fault-tolerance model comes along.** Supervision on a
  weather station: a crashed sensor process restarts, the system
  reports, the fleet sees it.
- **The release is the whole system.** Firmware updates are release
  updates; rollback is booting the previous firmware slot (A/B
  partitions).

---

## The shape of a Nerves app

```
mix new weather --sup           # the app
MIX_TARGET=rpi4 mix firmware    # the image, per target hardware
mix nerves.burn                 # to SD card
```

Hardware interaction is Elixir through ports and NIFs — Circuits.GPIO
for pins, Circuits.I2C/SPI for buses, `:gen_server` processes holding
the device state. The sensor becomes a supervised process; the
reading loop is a `handle_info` timer.

## Rules of thumb

- **The device is a release, treat it like one.** Reproducible
  builds, signed images, A/B slots — the deploy discipline of the
  server applies to the firmware.
- **Supervise the hardware paths.** A flaky sensor is a crashed
  process, not a bricked device — design the tree so hardware
  failures restart small.
- **Minimize the writable state.** Nerves filesystems are often
  read-only; state goes to a data partition, not the root image.
- **Test on the target early.** `MIX_TARGET` changes the world —
  mock only what you must, validate on hardware what you can.

## Why it matters for the mesh

The mesh's edge — stations, sensors, weather hardware — is exactly
Nerves territory: a device that boots straight into an OTP release
speaking the mesh protocol is a first-class node, not a peripheral.
The BEAM corpus note: embedded is not a different world, it is the
same supervision tree with a GPIO attached.
