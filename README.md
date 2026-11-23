# sitl-transmitter

A pass-through virtual transmitter for [ArduPilot SITL](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html).
It reads one USB gamepad or radio through [pico-gamepad](https://github.com/padcom/pico-gamepad) and writes its channels, as a
receiver would, to SITL's UDP RC input. SITL keeps the last values for ever (its simulated receiver re-sends them at 50 Hz, with no
timeout), so the vehicle behaves as if a transmitter were switched on; **the vehicle's own `RCMAP_*`, `RCn_REVERSED`, `RCn_DZ` and
`RCn_MIN/TRIM/MAX` do all the role mapping, inversion and dead zones.** There is **no mapping, no calibration and no profile** here.

**What it is not:** it cannot fly the app's built-in simulator (that has no RC input), it is Linux only (pico-gamepad's only backend),
and it knows nothing of the vehicle: not its mode, not whether it is armed or on the ground.

## Build

It needs `pico` 0.10.1 or later on the `PATH`; `pico` fetches the two dependencies, [pico-gamepad](https://github.com/padcom/pico-gamepad)
and [pico-commander](https://github.com/padcom/pico-commander) (which reads the command line), at the tags `project.json` pins:

```
pico build main.pi -o sitl-transmitter
```

Without building: `pico run main.pi -- <options>`.

## First run

```
sitl-transmitter --list                    # what pads there are (a radio shows up as soon as it is plugged in, no stick needed)
sitl-transmitter --pad 1209:4f54           # the TX16S, by id (an index of the list, or a part of its name, also do)
```

On Linux the desktop has to give your user access to the device (a udev rule with `uaccess`, which every desktop's logind applies to
a joystick that is plugged in). If it has not, the program says so and exits with 3 after two seconds of trying; it never asks you to
join the `input` group.

It then says what it found (the pad, its axes and buttons, which RC channels they go to, where it sends) and waits for **Enter**
(`--go` skips it; it is required when stdin is not a terminal). Then it waits for **the start gate**.

### The start gate

Nothing is sent until, for 10 reads in a row, **roll, pitch and yaw are within 15 % of the middle and the throttle is at its rest**
(`--throttle-rest centre`, the default: within 15 % of the middle; `--throttle-rest low`: at the bottom). It tells you once a second
which stick is off and where it reads ("roll is not at the middle: +0.72"), and waits, with no timeout. This is what stops a stuck
stick, a trigger that rests at -1 or the wrong pad from reaching the vehicle when you press Enter. The four sticks are the RC channels
of `--sticks` (default `1,2,3,4`: set it to the vehicle's `RCMAP_ROLL,PITCH,THROTTLE,YAW`), and each must be an **axis** of the pad: a
stick that lands on a button, or on nothing, ends the program with 4 and names it.

After any loss (a pad that was unplugged, or a tick that came too late: more than 100 ms, or three periods at `--rate` under 30) **one neutral frame** is sent (roll, pitch and yaw
in the middle, the throttle in the middle or at the bottom by its rest, every other channel 0 = untouched) and the gate runs again. A
pad that comes back is found again by **its own id** (never another pad), and Enter is not asked again.

### When nothing moves

The once-a-second status line says what the program waits for: `ready: nothing is sent until you press Enter (or start with --go)`,
or `gate: waiting for rest, 3/10 reads right (throttle is not in the middle: -1.00)` (a throttle that rests at the bottom wants
`--throttle-rest low`), each with the four sticks' live values, and `no packet is sent yet` until the gate has passed. `--verbose` adds
`packets: N sent, the last one CH1..CH8 ...` (or `none yet (<state>)`) once a second, even with `--quiet`. If packets are sent and
SITL does not react, look at SITL:
- **The port.** It is whatever SITL really listens on: check with `ss -ulpn | grep arducopter` (the UDP socket on `0.0.0.0:<port>`) and give
  that to `--port`. `--instance n` (5501 + 10 n) is right only where SITL derives its RC port from `-I n` (the source does, unless
  `--rc-in-port` comes later on its command line, or the build is older); verify with `ss`.
- **`SIM_RC_FAIL`.** 1 ignores the input altogether (no pulses), 2 forces neutral sticks with the throttle at 950, 3 sets the protocol's
  failsafe flag; it has to be 0. `RC_PROTOCOLS` must include UDP (the default).
- SITL prints "Malformed SITL RC input" on its console for a packet that is not 16 or 32 bytes.

### The channels

Pad axis *i* goes to RC channel *i* + 1 + `--offset`, then the buttons to the channels after them, while there are channels left (up to
16); what does not fit is dropped (the banner says how many). A **TX16S** has 8 axes and 24 buttons: CH1..CH8 are its axes, CH9..CH16
its first 8 buttons, the other 16 buttons are dropped. An axis is `round(1500 + 500 v)` µs, 1000 to 2000 from the pad's declared
range, with **no dead zone, no curve and no inversion** (the kernel's own `flat` is not applied); a button is 1000 released and 2000
pressed; a channel the pad does not drive is sent as **0 = keep what you have**. The packet is 16 × `uint16` little-endian, 32 bytes.

EdgeTX mixes the channels of a TX16S: set the vehicle's `RCMAP_*` and `RCn_REVERSED` if its order differs from the vehicle's, and
calibrate the vehicle's `RCn_MIN/MAX` in the app's Setup > Radio (which now reads the real stream).

### The yaw hold

Unless `--rudder-arm` is given, while the throttle stick is at the bottom (value ≤ -0.8) the yaw is held to ±80 % (1100..1900 µs), so
that the pad cannot make the vehicle's stick arm/disarm gesture (full yaw with the throttle down). It looks at the throttle's *value*,
so it assumes the throttle is not reversed on the vehicle. **Note:** a vehicle with a raised `RC3_MIN` or `RC3_DZ` reads "throttle at
the bottom" a little above the stick's real bottom, so the gesture is possible in a sliver of about 2 % of the travel just above the
hold: do not rely on it where that matters.

### Ending

`q` + Enter, the end of input at a terminal, `--exit-after <s>`, and **Ctrl-C / `kill` (SIGINT, SIGTERM, SIGHUP) on Linux** end it
with **the neutral frame sent twice, 20 ms apart**. (Pico cannot catch a signal, so the program blocks them and looks for one every
20 ms; `signals.linux.test.pi` is the proof that a blocked signal sent to the process does not end it. The look happens between ticks, so while
the program waits in a name lookup (`--host`) or in the two-second retry of a pad that the desktop has not yet given access to, **Ctrl-C does
nothing until that wait is over**; nothing has been sent by then.) **A run that never passed the
gate owes no neutral frame**: it has touched nothing in SITL, so it sends nothing at the end. `kill -9`, a crash or a power cut
leave SITL with the last values (a stuck stick): the vehicle keeps flying on them. Do not rely on it for anything else.

## SITL

The program sends to **`127.0.0.1`** unless you give **`--host <ip|name>`**. The destination is resolved once at start (an IPv4 address
or a name; a broadcast, multicast or `0.0.0.0` address is refused, and a host that cannot be resolved ends it with 5), and the banner
says "sending to <host>:<port>: for ArduPilot SITL only". The packet is SITL's RC-input format and nothing else: a real autopilot has
no such UDP input, so this cannot fly one; but it is the host you name that decides where it goes, so name it on purpose.

**The port is whatever SITL listens on**: check with `ss -ulpn | grep arducopter` and use `--port` for it. By its source SITL
listens on **UDP 5501 + 10 × instance** (`-I n`) when it derives the RC port from `-I` (a later `--rc-in-port PORT` on its command line wins;
`uds:PATH` is a Unix datagram socket and not for this program), but a build or command that does not do so listens on 5501 whatever `-I` says. It binds the port on `0.0.0.0`, not on loopback only. The team's harness starts it as

```
arducopter -w --model quad --speedup 1 -I 0 --sim-address=127.0.0.1 [--defaults <file>]
```

and then this program needs no port option for instance 0 (5501); where SITL derives its port from `-I`, `--instance 2` is 5521 and
`--instance 1000` is 15501 (any instance up to 6003, port 65531); otherwise name the port `ss` shows with `--port`.
**`sim_vehicle.py` starts MAVProxy, which feeds the same RC input itself**: do not use MAVProxy's `rc` together with this program, and do
not turn on the app's "Allow remote control" at the same time: a second source fights over the same channels.

### SITL in a virtual machine

Run SITL in the VM, the transmitter on the host (where the radio is plugged in):

1. In the VM start SITL as above. It listens on every interface of the VM, so nothing needs changing there.
2. The VM's network must be reachable from the host (a bridged or host-only adapter, not NAT without a port forward) and its
   **firewall must let UDP 5501 + 10 × instance in**. Find the VM's address (`ip addr` in it).
3. On the host: `sitl-transmitter --pad 1209:4f54 --host 192.168.x.y` (with `--instance n` or `--port p` for another port).

UDP has no reply, so a wrong address or a closed firewall looks like a transmitter that does nothing: check that SITL's console says
that it has RC input (or watch the channels in the app's Setup > Radio).

## Options

The command line is read by pico-commander, so it behaves as a commander.js program does: `--flag=value` works too, a misspelt
option is answered with "(Did you mean …?)", and `-h`/`--help` shows the help (as wide as the terminal).

```
--list                      list the pads and exit
--pad <n|id|text>           the list index, the exact id, or a part of the name (default: the only pad)
--instance <n>              SITL instance (-I n), 0..6003: port = 5501 + 10 n, only where SITL derives its RC port from -I (verify with ss -ulpn)
--port <p>                  UDP port 1024..65535; wins over --instance (default 5501)
--host <ip|name>            where SITL runs (default 127.0.0.1)
--rate <hz>                 send rate 10..100 (default 50)
--offset <n>                channel offset 0..8 (default 0)
--sticks <r,p,t,y>          the RC channels of roll, pitch, throttle, yaw (default 1,2,3,4; four different, 1..16)
--throttle-rest <rest>      centre or low: where the throttle rests (default centre)
--rudder-arm                lift the yaw hold
--go                        no "press Enter"
--fake-pad <file>           read the pad from a script instead of a device
--exit-after <s>            end after s seconds
--quiet                     no status line
--verbose                   once a second, the first 8 channels of the last packet and how many were sent
--version, -h/--help
```

Exit codes: 0 normal end; 2 usage error; 3 no pad or no access to it; 4 the start was refused (no terminal and no `--go`, or a stick
on a button); 5 the UDP socket could not be made or the host could not be resolved.

### A pad without a pad (`--fake-pad`)

For trying it, and for tests: a script of JSON Lines, each `{"t": <ms since start>, "axes": [<-1..1 or null>...], "buttons": [<0|1 or
null>...]}` (null is "unchanged", the last state holds, `#` starts a comment); see `examples/hover.jsonl`:

```
sitl-transmitter --fake-pad examples/hover.jsonl --go --exit-after 10
```

## Tests

`pico fmt --check . && pico lint . && pico test .`. No hardware is needed: the pad is pico-gamepad's fake system or a script, the
network is the loopback address with a receiver of the tests' own, and the loop runs on a clock that only moves when it waits. CI runs
them on Linux (x64, arm64), macOS and Windows with a literal `PICO_VERSION` (`signals.linux.pi`, the one Linux-only file, is replaced off Linux by `signals.pi`).

## License

MIT, see `LICENSE`.
