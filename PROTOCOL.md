# Service authentication and execution boundary

The sequence below describes the ILCE-7CM2 2.01 observations. The authentication framing follows [Sony-PMCA-RE's Senser implementation](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/sony.py); the two-mode sequence is also visible in its [USB service command](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/commands/usb.py).

## USB session and authentication

The host claims interface 0 with bulk IN and OUT endpoints. It first talks to the Mass Storage identity `054C:0E8A`. After the first authentication, the camera reconnects as `054C:0336`; the host opens that new device and authenticates again. The service requests are sent only after the second authentication.

| Phase | Exchange | Meaning |
| --- | --- | --- |
| Start | Vendor control OUT: `bmRequestType=0x43`, request `1`, value `0x37FF`, index `0xD7AA` | Begin Senser. On this camera the control status may stall while authentication remains available. |
| Challenge, repeated three times | Host sends logical command `1`; camera returns result `2` or `8` with challenge data | Result `2` selects the camera-compatible SHA-1 branch; `8` selects SHA-256. |
| Proof, once per challenge | Host sends logical command `3`, a one-byte marker `1`, and the calculated digest; camera returns result `4` | Confirms that round. |
| Completion | Host sends logical command `5`; camera returns result `6` and success byte `1` | Completes this device identity's authentication. |
| Stop | Vendor control OUT: `bmRequestType=0x43`, request `1`, value `0xC800`, index `0x2855` | Ends the Senser session and initiates return to normal USB mode. |

Each authentication message has a 516-byte big-endian frame: a 16-bit complemented command, a 16-bit salt, and 512 data bytes. The reply result is decoded as `((~cmd) & 0xFFFF) - salt`. The digest input depends on the currently enumerated device:

- On `0E8A`, hash the first four challenge bytes.
- On `0336`, hash the 512-byte challenge followed by the round's entry in the matching SHA-1 or SHA-256 [key table](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/constants.py).
- The result-`2` branch uses the SHA-1 length behavior implemented by Senser, rather than an interchangeable standard SHA-1 call. The compatibility function is in the linked [upstream implementation](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/crypto.py).

Successful completion on `0E8A` is followed by USB re-enumeration, so the `0336` exchange uses a fresh device handle and sequence state. A Start request or the new USB ID by itself is not evidence that authentication finished. After the second exchange, the host sends Senser packets with a 12-byte little-endian header (`size:32`, `pFunc:16`, `sequence:16`, then four 8-bit fields). The packet sequence starts at `1` and increments for each request. Function `0x10` reads ProductInfo; the investigated Jiritsu requests use function `0x40`. ProductInfo returned 88 bytes in the observed session.

## How a file operation becomes code execution

Jiritsu selects a file path before reading or writing it. In the investigated firmware, selectors containing `/tmp/jiritsu/../..` were resolved beyond the intended Jiritsu directory. The research used this path behavior to place a shared object in the temporary Work area and influence the loader's preload configuration. A later read through the same service file interface found the object path in `/etc/ld.so.preload`.

The trigger was a Jiritsu DEBUG request. It started a root child running a fixed shell command. The child's dynamic loader consulted `/etc/ld.so.preload`, loaded the Work-area shared object, and ran its initialization function before the fixed command. The worker wrote a versioned result record and removed the preload entry. That record and a separate readback of the preload path establish execution and cleanup; an accepted USB request alone does not.

The observed facts are the accepted path selectors, the `/etc/ld.so.preload` readback, the worker result, and the child context. The camera's internal mapping between the selected `/tmp` path and the observed `/etc` path has not been uniquely identified. The security boundary is clear: service-controlled file content became input to a privileged process's dynamic loader.
