# Eurobtec

Reverse-engineering, documenting, and re-tooling the **Eurobtec ROB 3** — a
6-axis industrial robot whose controller is built around an **Intel 8031**
(MCS-51) running from an 8 KB EPROM — and the microcontroller-simulation tooling
that makes that work verifiable.

The goal across these repos is to recover and faithfully document the *original*
firmware behavior (verified against the ROM, a simulator, and the hardware) and
to provide clean, modern interfaces on top of it — a Python protocol library, a
ROS 2 driver, and reusable ucSim tooling.

## Projects

### ROB3 robot

| Repo | What it is |
| :--- | :--------- |
| [**rob3**](https://github.com/eurobtec/rob3) | ROB3 firmware reverse-engineering: an assembling 1:1 annotated 8031 disassembly (byte-identical to the EPROM), a two-layer golden-byte + behavioral ucSim test rig, hardware reference docs, and Arduino bench bring-up rigs. The source of truth for the protocol and hardware. |
| [**rob3_py**](https://github.com/eurobtec/rob3_py) | Pure-Python library for the ROB3 RS-232 low-level protocol: the wire-protocol codec, serial transport, joint↔count calibration, and a high-level client. ROS-independent; verified against the real ROM in ucSim. |
| [**rob3_ros2_driver**](https://github.com/eurobtec/rob3_ros2_driver) | ROS 2 driver for the ROB3 (JointState / FollowJointTrajectory / JointJog teleop / services), built on top of `rob3_py`, with URDF, launch, and a Docker + RViz/noVNC setup. |
| [**rob3_ucsim**](https://github.com/eurobtec/rob3_ucsim) | ROB3-specific ucSim simulation harness: a `UCSimEngine` (subclass of `pyucsim.UCSimEngine`) that knows this firmware's memory landmarks and `cl_hw` peripherals, plus a motor/pot `Plant` model. Drives the ROB3 ROM and reads back its state; pairs with `rob3_py`. |
| [**tbps_compiler**](https://github.com/eurobtec/tbps_compiler) | Compiler, disassembler, source-level debugger, and native-8051 backend for the **Teach Box Programming System (TBPS)** — the robot's teach-pendant program language. Emits the exact bytes the firmware interprets from SRAM; every opcode is `[SIM]`-verified against the ROM in ucSim (direct load, RS-232 `0x81` upload/`0x80` readback, native==interpreter equivalence). |

### ucSim tooling

| Repo | What it is |
| :--- | :--------- |
| [**pyucsim**](https://github.com/eurobtec/pyucsim) | A dependency-free Python client for Daniel Drotos' µCsim microcontroller simulator — drives a ucSim binary as a persistent subprocess (load firmware, reset/run/step, inspect memory/registers, breakpoints, cl_hw plugins). |
| [**ucsim-mcp**](https://github.com/eurobtec/ucsim-mcp) | A Model Context Protocol (MCP) server for µCsim — a thin adapter over `pyucsim` that lets AI models interact with, debug, and execute code on virtual hardware targets. |
| [**ucsim**](https://github.com/eurobtec/ucsim) | Fork of the upstream µCsim simulator (`s51` / `ucsim_51` and the rest of the family), with the patches used by the ROB3 work. |

## How it fits together

```
             ┌─────────────┐   drives/verifies    ┌──────────────┐
    rob3  ───┤  8031 ROM   ├─────────────────────►│    ucsim     │ (fork)
 (firmware)  │  + docs     │                      └──────┬───────┘
             └──────┬──────┘                             │ subprocess
                    │ protocol spec                      ▼
                    ▼                              ┌──────────────┐
              ┌───────────┐   verified against     │   pyucsim    │
              │  rob3_py  │◄───── the real ROM ────┤  (Py client) │
              │ (protocol │        in ucSim        └──┬────────┬──┘
              │  library) │                           │        │ subclassed by
              └─────┬─────┘              used by ──────┘        ▼
                    │ used by          ┌──────────────┐  ┌───────────────┐
                    │                  │  ucsim-mcp   │  │  rob3_ucsim   │
                    │                  │ (MCP server) │  │ (ROB3 harness │
                    │                  └──────────────┘  │  + Plant)     │
                    ▼                                     └───────────────┘
           ┌──────────────────┐
           │ rob3_ros2_driver │
           │   (ROS 2 node)   │
           └────────┬─────────┘
                    │ RS-232  (same bytes to the robot or to ucSim)
                    ▼
          ╔═══════════════════╗
          ║  ROB 3 robot      ║   ← the real 6-axis arm
          ║  (8031 controller)║     (or ucSim standing in for it)
          ╚═══════════════════╝
```

`rob3_ucsim` subclasses `pyucsim` to drive *this* ROM with ROB3 knowledge, and
`rob3_ros2_driver` builds on `rob3_py`. Because `rob3_py` speaks the *same*
RS-232 bytes to the physical robot and to the ROM running in ucSim, the whole
stack can be developed and tested without the hardware, then pointed at the real
**ROB 3** arm unchanged.

`tbps_compiler` targets the robot's other input path — the Teach Box program
store: it compiles TBPS source to the bytes the firmware's stored-program
interpreter runs, and verifies them against the ROM via `rob3_ucsim` (direct
SRAM load and the RS-232 `0x81` upload path).

## Provenance convention

Firmware claims in these repos are tagged with how they were established, so
verified facts stay separate from hypotheses:

- **[BYTE]** — verified from the ROM bytes (byte-exact / golden match)
- **[SIM]** — verified by running the ROM in ucSim and observing state
- **[HW]** — confirmed against a hardware doc or an Arduino bench bring-up
- **[INFER]** — hypothesis, not yet proven (never stated as fact)

## License

Each repository carries its own license (MIT for the ROB3 projects; `ucsim` keeps
its upstream license). See the individual repos.
