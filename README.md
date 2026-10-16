IoT-LAB platform overlay for Contiki-NG 5.1
===========================================

This repository is **not** a copy of Contiki-NG. It holds only the
[FIT IoT-LAB](https://www.iot-lab.info) platform layer, which lets an existing
Contiki-NG **5.1** tree build firmware for the testbed's nodes:

- `arch/platform/iotlab/`: board support (boot, clock/rtimer, AT86RF231 radio
  driver `radio-rf2xx.c`, LEDs, UART, watchdog, link address from `NODE_ID`)
- `arch/platform/iotlab/openlab/`: the STM32F103 hardware drivers from
  IoT-LAB's openlab project
- boards: `m3` (the one used and tested on this branch) and `a8-m3` (not tested
  on 5.1)

The OS (TSCH, Orchestra, RPL, your application) comes from your project's own
Contiki-NG 5.1 tree. The same application source then builds for Cooja
(`TARGET=cooja`) and for M3 (`TARGET=iotlab BOARD=m3`).

This is the default branch, `contiki5.1-m3`. `master` holds the earlier
port that built against a Contiki-NG 4.4 submodule.

## Use it in a project

```bash
# from your project root (which vendors Contiki-NG 5.1 under contiki-ng/)
git submodule add https://github.com/fdojurado/iot-lab-contiki-ng.git m3/iot-lab-contiki-ng
```

Build an application from your own Contiki-NG tree, pointing `ARCH_PATH` at
this overlay:

```bash
export PATH=/opt/gcc-arm-none-eabi-9-2019-q4-major/bin:$PATH
make -C contiki-ng/examples/<app> \
     ARCH_PATH=$PWD/m3/iot-lab-contiki-ng/arch \
     TARGET=iotlab BOARD=m3 \
     WERROR=0 RELSTR=release/v5.1 \
     DEFINES=NODE_ID=<id> \
     <target>
# image: contiki-ng/examples/<app>/build/iotlab/m3/<target>.iotlab
```

- **Toolchain: ARM GCC 9-2019-q4-major (9.2.1).** Newer GCC versions break the
  openlab HAL. For a reproducible build, pin it in Docker; see `Dockerfile.m3build`
  in tsch-adaptive-hopping.
- **`WERROR=0`**: the openlab HAL produces harmless GCC 9 warnings.
- **`RELSTR`**: a vendored Contiki-NG tree has no `.git`, so pass the version
  string explicitly.
- **`NODE_ID`** must reach the compiler as `-DNODE_ID` (via `DEFINES=` or
  `CFLAGS += -DNODE_ID=...`). Without it the link address falls back to the
  hardware UID, and schedulers that hash addresses (e.g. Orchestra) can put
  several nodes on the same cell.

## Project configuration (`project-conf.h`)

Settings that have mattered on M3. Put them under `#if CONTIKI_TARGET_IOTLAB`
so Cooja builds are unaffected:

| Define | Why |
|---|---|
| `LINKADDR_CONF_SIZE 2` | the short address is derived from `NODE_ID` |
| `IOTLAB_CONF_LINKADDR_LSB_FIRST 1` | optional; see below |
| `TSCH_CONF_AUTOSTART 0` | start TSCH explicitly after setting the coordinator; otherwise the root boots as a scanning joiner and association takes minutes |
| `TSCH_CONF_EB_PERIOD` / `TSCH_CONF_MAX_EB_PERIOD` `(4 * CLOCK_SECOND)` | faster association (default 16 s) |
| `IEEE802154_CONF_PANID`, `TSCH_PACKET_CONF_EACK_WITH_SRC_ADDR 1`, `WITH_SECURITY 0` | settings known to work on the testbed |

### Link-address byte order

With `LINKADDR_SIZE == 2`, the address comes from `NODE_ID`:

- default: **high byte first** (`u8[0] = id >> 8`, `u8[1] = id & 0xff`)
- `IOTLAB_CONF_LINKADDR_LSB_FIRST 1`: **low byte first** (`u8[0] = id & 0xff`).
  This matches what Cooja produces for 2-byte addresses. Use it when code
  hard-codes addresses as `{{id, 0x00}}` or hashes on `u8[0]`.

### Networking

SLIP (`slip.c`, `slip-arch.c`) is compiled only for IP builds. It is skipped for
`MAKE_NET = MAKE_NET_NULLNET` and `MAKE_NET_OTHER` (custom network layers),
which have no `uip_len`/`tcpip_input` to link against.

### TX power

```c
#if CONTIKI_TARGET_IOTLAB
#include "platform.h"
NETSTACK_RADIO.set_value(RADIO_PARAM_TXPOWER, PHY_POWER_0dBm); /* see phy.h */
#endif
```

## Projects using this branch

- tsch-adaptive-hopping (TORCH)
- sage-tsch (SAGE)

## Notes

- `tests/01-compile` still expects the old `contiki-ng` submodule layout and
  has not been updated for this branch.
- Running experiments (reserving nodes, flashing, serial capture) is outside
  the scope of this repository. Each project keeps that in its own scripts.

## Acknowledgements

Based on [iot-lab/iot-lab-contiki-ng](https://github.com/iot-lab/iot-lab-contiki-ng).
Thanks to Simon Duquennoy, without whom the original port would not exist.
