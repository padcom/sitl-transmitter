# sitl-transmitter

A pass-through virtual transmitter for [ArduPilot SITL](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html).
It reads one USB gamepad or radio through [pico-gamepad](https://github.com/padcom/pico-gamepad) and writes its channels, as a
receiver would, to SITL's UDP RC input on `127.0.0.1`. SITL keeps the last values, so the vehicle behaves as if a transmitter were
switched on; the vehicle's own `RCMAP_*`, `RCn_REVERSED`, `RCn_DZ` and `RCn_MIN/TRIM/MAX` do all the role mapping, inversion and
dead zones. **No mapping, no calibration, no profile.** Linux only (pico-gamepad's only backend), SITL only. It sends to
`127.0.0.1` unless you give `--host <ip|name>` (for a SITL that lives in a virtual machine: SITL binds its RC input on `0.0.0.0`);
any other host is an explicit choice, and the program says so ("sending to <host>:<port>: for ArduPilot SITL only").

**Status: the core (T1).** The program still prints only its version. The core is there and tested without hardware: `channels.pi`
(the pad's axes and buttons to the 16 pulses), `packet.pi` (the 32 bytes SITL takes) and `gate.pi` (the start gate and the run
states: nothing is sent until the sticks have been at rest for 10 reads, a stalled tick or a lost pad gives one neutral frame and
the gate again, the yaw is held to 80 % while the throttle is at the bottom). The command line, the sender and the loop follow
(story T2); the plan is `shared/decisions/sitl-transmitter.md` of the team's repository. This README grows with them (the SITL launch line, the first run, the start gate in words, the TX16S notes, and the
warning that a killed process leaves SITL's last values).

## Build and test

It expects `pico-gamepad` next to this folder (`project.json`'s `workspace` says `../pico-gamepad`), and `pico` 0.5.1 or later on
the `PATH`:

```
pico build main.pi -o sitl-transmitter
pico run main.pi -- --version
pico fmt --check . && pico lint . && pico test .
```

The tests need no hardware: the pad is pico-gamepad's fake system. CI runs them on Linux (x64, arm64), macOS and Windows with a
literal `PICO_VERSION` and a checkout of `padcom/pico-gamepad` next to this repository.
