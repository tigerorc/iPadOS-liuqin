# Touch: the IOHIDDevice port plan (2026-11-04)

Status: **plan + scope measured, nothing ported yet.** Read this with
`HID_LAST_MILE.md`, which contains the probe results this plan is built on.

## Where the port actually is

The lock screen is up (`NOTES.md`, 2026-11-04 19:13 milestone), so the kernel
runs iPadOS 16.6.1 userland. What does not work yet is **input**:

| symptom | root cause |
| --- | --- |
| touch does nothing | userland opens `IOHIDDevice` (`newUserClient type 0x484944` = `kIOHIDLibUserClientConnectManager`), and this tree has **no `IOHIDDevice.cpp` / `IOHIDLibUserClient.cpp`** — only `IOHIDDeviceKeys.h` and `IOHIDDeviceTypes.h`. The `IOHIDEventService` path (`LiuqinHID.cpp`) is a dead end for the digitizer: backboardd never opens it. |
| power/volume keys do nothing, though the log shows the presses | `liuqin_keys_thread` only `printf`s; `liuqin_hid_touch()` / `gLiuqinButtons` dispatch only when the HID service objects exist, and nothing userland-side ever binds them. Same missing half. |
| the logging client gets zero `device client sel` calls (touch56) | userland gives up *before* its first external method, so its pre-flight check is in the device's registry properties — most likely `ReportDescriptor` / usages / transport. |

## Scope measured against apple-oss-distributions/IOHIDFamily-1915.140.3

Cloned at tag `IOHIDFamily-1915.140.3` (the version `HID_LAST_MILE.md` names,
the closest published source to this root's userland). 13 files, **319 KB**:

```
IOHIDDevice.cpp                 98,243   the device object
IOHIDLibUserClient.cpp          80,686   the 'HID' client the userland opened
IOHIDElementPrivate.cpp         75,430   element tree from the report descriptor
IOHIDDeviceElementContainer.cpp  2,100
IOHIDDescriptorParserPrivate.h   8,837
IOHIDFamilyPrivate.h             5,499
IOHIDEventSource.h               1,593
IOHIDDebug.h                     1,584
IOHIDInterface.h                13,221
OSStackRetain.h                  1,731
IOHIDLibUserClient.h            14,589
IOHIDElementPrivate.h           14,268
IOHIDDeviceElementContainer.h    1,228
```

Plus the public `IOHIDDevice.h` from the same tag.

**The good news, measured, not guessed**: the macOS-only surface is small.

| file | macOS-only references |
| --- | --- |
| `IOHIDLibUserClient.cpp` | 1 (`#include <IOKit/hidsystem/IOHIDUsageTables.h>` — a path we already have as `IOHIDUsageTables.h`) |
| `IOHIDElementPrivate.cpp` | **0** |
| `IOHIDDevice.cpp` | ~18 total: `IOHIDSystemActivityTickle` ×2, `IOHIKeyboard` ×5, `IOHIPointing` ×5, `kIOHIDSystemDeviceSeizeRequestMessage` ×4, `NX_HARDWARE_TICKLE` ×1, `NX_NULLEVENT` ×1 |

So the port is: three real source files, a handful of stubs, and no
`IOHIDSystem`/`IOHIKeyboard` machinery — iOS does not have that layer either,
which is why the references are all in seize/tickle paths that a digitizer on
iOS never takes.

## Plan

1. **Stub layer first** (`iokit/Kernel/arm/hid/LiuqinHIDStubs.h`): the macOS
   names above as no-ops, plus whatever `IOHIDDevice.cpp` pulls in transitively
   (`IOKit/hidsystem/IOHIDShared.h`). Keep every stub behind
   `ARM64_BOARD_CONFIG_LIUQIN` so a future real port can delete them cleanly.
2. **Vendor the four sources** into `iokit/Kernel/arm/hid/` under Apple's
   licence headers (they are APSL — keep the headers intact) and add them to
   `iokit/conf/files.arm64`.
3. **Publish the digitizer as a real `IOHIDDevice` subclass** — the same
   `LiuqinHID.cpp` touch path, but as an `IOHIDDevice` with
   `PrimaryUsagePage 0xff00` / `PrimaryUsage 0xf` and a **multitouch report
   descriptor**. This is what makes backboardd's `IOHIDDevice` match resolve.
4. **Shape the report descriptor from the userland's own traffic**, not from
   guesswork: bring up `IOHIDLibUserClient` with the real selectors, log
   `getReportDescriptor` / `createQueue` / `addElementToQueue` / `startQueue`
   and the first `getReport`, then match. `HID_LAST_MILE.md` already has the
   logging-client probe (`0203`) that names each selector — extend it from
   "return unsupported" to "answer, then log".
5. **Keys through the same device**: `IOHIDDevice` also carries the
   Consumer/GenericDesktop usages `liuqin_keys_thread` already maps
   (`kHIDUsage_Csmr_Power`, volume up/down), so one port fixes both symptoms.

## Acceptance

- `LIUQIN HID: ... newUserClient type 0x484944` followed by real
  `device client sel N (<selector name>)` lines instead of none.
- `getReportDescriptor` returns the multitouch descriptor and the userland
  creates a queue on it.
- A finger on the panel produces a contact frame the lock screen reacts to
  (swipe unlocks), and power/volume change the lock screen state.

## Honest boundaries

- Nothing here has been run on hardware yet; the scope above is from source
  inspection of the tagged Apple tree plus the probe results in
  `HID_LAST_MILE.md`.
- `IOHIDDevice.cpp` may pull in more headers than the direct grep shows
  (transitive includes are not measured). Budget one build iteration for that.
- The report descriptor is the real unknown: the userland parses vendor
  multitouch reports itself, so a descriptor that parses but does not match its
  expectations will produce a device that opens and then does nothing.
