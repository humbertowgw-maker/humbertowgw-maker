# PhysicalKey: building hardware authentication that actually resists cloning

**[Live product](https://physicalkey.whitegwireless.com) · [Source](https://github.com/humbertowgw-maker/physicalkey-core) · [Back to profile](../README.md)**

PhysicalKey is a hardware authentication system: an ESP32 key fob, a native iOS app, and
a backend that requires three independent proofs before it grants access — phone
biometric, physical device, and cryptographic challenge-response. The idea is simple to
state and hard to build correctly: no single copyable secret should be enough to
impersonate a user. This is the story of the parts that didn't go as planned, because
those are the parts that actually show the engineering.

## Debugging a silicon-level dead end

The firmware started on Arduino-ESP32, the default choice for ESP32 projects. It never
ran on this hardware — not "BLE didn't work," nothing did. Even a completely empty
`setup(){}` / `loop(){}` crashed identically on first call: a double exception inside the
Xtensa register-window-spill routine.

Rather than assume a bad board or a bad build, I ran the elimination systematically:

- 2 different physical ESP32 boards — same crash
- Two Arduino-ESP32 core versions (2.0.17 and 3.3.11) — same crash
- Every flash mode/frequency and CPU frequency combination, including the conservative
  corner cases — same crash
- A completely fresh, isolated toolchain install, untouched by anything else on the
  machine — same crash, ruling out local corruption

Then the control: Espressif's own ESP-IDF `hello_world` example, on the exact same board.
Clean boot, correct chip identification, zero crashes, repeated cycles. That isolated the
fault to the Arduino core's startup path specifically — not the chip, the board, the
cables, or the toolchain. It matched a known, unresolved upstream report
([espressif/arduino-esp32#8349](https://github.com/espressif/arduino-esp32/issues/8349)):
this chip revision (ESP32-D0WD-V3) appears to be genuinely incompatible with the
Arduino-ESP32 core's boot sequence, closed unfixed.

Given ESP-IDF was proven to work on the exact chip, the firmware was ported directly to
ESP-IDF/NimBLE instead of continuing to chase a dead framework. Same GATT UUIDs, same
wire protocol, same vendored Ed25519 signing library — the phone app needed zero changes.

## Flash encryption doesn't mean what you'd assume

Once the firmware ran, the next question was: where does the device's private signing
key actually live? Enabling ESP32 flash encryption sounds like it should answer that. It
doesn't — the NVS partition (exactly where the key gets stored via `nvs_set_blob()`) is
specifically excluded from flash encryption's coverage. That's documented ESP-IDF
behavior, not a bug, and easy to miss if you stop at "flash encryption: on."

Actually protecting the key needs the separate NVS Encryption feature. The clean way to
key that — deriving from the chip's HMAC peripheral — isn't available on this hardware
either: the original ESP32 has no HMAC peripheral (S2/S3/C3-and-newer only). The only
scheme this chip supports stores the NVS encryption keys in a dedicated partition that's
*itself* protected by flash encryption — so flash encryption ends up required as a
dependency of NVS encryption on this chip, not because the whole flash needed protecting
for its own sake.

Getting there took a custom partition table, moving the bootloader's partition-table
offset after it outgrew the default budget (a common flash-encryption gotcha), and a full
chip erase on every already-flashed board, since old plaintext NVS entries aren't in the
format NVS Encryption expects. All three physical boards now run with flash + NVS
encryption verified in their boot logs, not just assumed from a Kconfig flag.

## Closing the relay-attack gap

BLE's default "Just Works" pairing has a real weakness: it authenticates that *some*
device is nearby, not that it's the *right* device. An attacker relaying the pairing
handshake from a distance can complete it just as easily as the legitimate owner
standing next to the fob.

The fix was per-unit Passkey Entry pairing: since this board has no display, a random
passkey is generated once on first boot, logged, and written on the unit's physical
label at provisioning time. NimBLE's `DISP_ONLY` I/O capability plus MITM protection then
requires the phone's owner to type that number in during pairing — something only
available by holding the physical unit. Each board still permanently bonds to the first
phone that pairs with it; every other peer's connection attempt is rejected outright.

## Proving the crypto, not trusting the logs

"It compiled" and "it ran" are different claims, and "bytes came back" and
"cryptographically valid" are further apart still. The verification chain for the
device's Ed25519 identity went through all of them separately: a Python `bleak` BLE
client connected to the real board, read the 44-byte public key and device ID, wrote a
challenge, and received a 64-byte signature over notify — then that signature was
checked independently with Node's `crypto.verify()`, off the device entirely. Only after
that passed did the real iOS app get paired against the physical board for the actual
phone ↔ device ↔ backend flow — the first genuine end-to-end success, not a simulated
stand-in.

## Where it stands

Phase 1 — Git/API authentication, the device's Ed25519 identity, the BLE MITM passkey
fix, and flash + NVS encryption — is live and verified on all three physical boards.

One piece is still genuinely in progress: migrating the *phone's* identity from a
software Ed25519 key to a Secure-Enclave-resident P-256 key (hardware-backed, gated by
biometric enrollment, never leaves the co-processor). The migration code is written and
unit-tested, but real-hardware verification hit
`Error Domain=com.apple.CryptoTokenKit Code=3 "Corrupted object ID detected"` when
generating the Secure Enclave key on a physical iPhone, and that's still unresolved. It's
listed here deliberately, unfinished — the alternative is a portfolio that only shows
the parts that already worked, which isn't a more honest picture of the actual
engineering.

---

**[← Back to profile](../README.md)**
