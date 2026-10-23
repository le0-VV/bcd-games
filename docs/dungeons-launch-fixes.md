# Minecraft Dungeons II launch fixes

These changes address launch failures observed with BCD 0.1.592 on iOS.
The tested sequence reached the main menu and later entered online gameplay.
It does not establish reliable online sessions. Error 0015 and a subsequent
access violation during return to the menu remain unresolved.

## Runtime corrections

- Wine's iOS DOS-path resolver recognises complete GLOBALROOT wrappers and
  resolves the inner DOS path. The original string offsets, options and
  native-device fallback are retained. This lets the game's metadata file open.
- The Mach exception repeat guard includes the saved guest RIP when a bounded,
  fault-safe probe identifies the shared FEX HLT thunk. Different guest
  instructions no longer count as one stationary fault. Unknown and native
  faults retain the existing identity. The 2,000-repeat limit remains in force.
- The page-fault loop guard applies to EXC_BAD_ACCESS. Instruction traps no
  longer count towards its memory-fault limits. Real memory-fault termination
  and the existing rate-based guard option remain intact.
- With the existing `pool-split = 1`, a misplaced second JIT allocation is
  released and retried up to three times. Each attempt rechecks current free
  ranges. Bounds, the executable window, temporary reservations and release
  failures are checked. The existing single-region fallback is retained.

These correctness changes are scoped to the iOS code paths. They do not add
new global compatibility switches. Pool placement retries are used only when
split-pool allocation is already requested.

## Optional log reduction

`env.MADEIRA_RECOVERY_LOG_SAMPLE = 1` enables Test 5's recovery-log sampling.
The default is off. Configuration is read before guest threads start; signal
handlers only read the cached value. The filter keeps recognised routine
instruction-recovery records at bounded intervals. Access violations, loader
errors, unknown faults and mixed or partial writes are retained. Sampling
changes logging only. It does not suppress exception delivery or relax guards.
The existing RTCS cap is independent.

The investigation-only SRW-history and socket-metadata probes are excluded.
No FEX source, Wine submodule pin or compiled Windows runtime DLL is changed.
The GLOBALROOT correction is applied as an idempotent native Wine build overlay.

## Per-game configuration used for the successful launch

```ini
env.MADEIRA_X64_IMAGE_NOCOPY = 1
pool-split = 1
quiet-log = 1
env.MADEIRA_RECOVERY_LOG_SAMPLE = 1
env.MADEIRA_EC_HOOK_TRACE = 0
env.MADEIRA_LOCK_ORPHAN = 0
env.MADEIRA_VCRT_ARM64_KEY = 14.44.35211.0
env.MADEIRA_OPTIN_DLLS = twinapi.appcore
```

Keep this complete configuration when reproducing the successful launch.
The source fixes alone do not enable these per-game options. In particular,
retain copy omission, split-pool allocation, orphan-lock mode `0`, the ARM64
VC runtime marker and the `twinapi.appcore` opt-in. The successful tests used
all five. We did not test removing each setting after launch succeeded, so
this is the supported test configuration rather than proof that each option
is necessary on every device.

Current upstream also supports `env.MADEIRA_VCRT_ARM64_KEY = 1`. It publishes
at least `14.44.35211.0` and uses the session's VC++ runtime version when that
is newer. An explicit version remains supported. The block above preserves
the value used in the phone tests; the automatic value was checked against
upstream source and the sanitised existing host test, without a new phone test.

These options already exist, except for the recovery-log sampling gate.
Copy omission avoids the redundant pure-x64 image copies that exhausted the
pool in previous runs. Quiet logging avoids the observed ten-second debug
output waits. Recovery-log sampling only reduces log volume; it is optional
for launch. MetalFX scaling is a display preference and is omitted here.

Test 5 enabled sampling unconditionally. The proposed setting enables the
same sampling logic and is covered by host tests, including the disabled
path. This new configuration gate has not been tested on the phone.

## Validation and remaining limits

The host checks compile the actual C helpers with AddressSanitizer and
UndefinedBehaviorSanitizer. Fixtures reproduce the old false terminations,
verify guest progress and native/unreadable fallbacks, and preserve real
memory-fault limits. A Swift fixture runs the production placement prefix
through races, bounds, release failures and cleanup. Logging checks cover
normal errors, context collisions, mixed writes and the opt-in/default paths.

Run the focused checks from the repository root:

```sh
python3 tests/host/check-globalroot.py
python3 tests/host/check-guest-fault-progress.py
python3 tests/host/check-page-fault-guard.py
python3 tests/host/check-pool-placement-retry.py
python3 tests/host/check-recovery-log.py
python3 tests/host/check-fault-guard-rate.py
python3 tests/host/check-rtcs-cap.py
python3 tests/host/check-x64-image-nocopy.py
python3 tests/host/check-config-catalog.py
```

A configuration follow-up also ran `check-vcrt-arm64-key.py`. Its source
checks and the fixture with ASan/UBSan passed. The unsanitised fixture
aborted on the host with a stack-buffer-overflow report; that separate
failure remains unresolved. No VCRT runtime code is changed here.

Initialise the pinned source submodules before running the path and catalog
checks. The native build applies the GLOBALROOT overlay; the path regression
also tests it directly against the pinned source.

The report deliberately separates launch compatibility from session stability.
A direct gameplay-UDP routing test still disconnected while the messaging
connection remained proxied. The available evidence does not clear either
the remaining network paths or the runtime as possible causes.
