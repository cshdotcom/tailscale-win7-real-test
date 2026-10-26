# Tailscale Windows 7 Real Test (QEMU)

Real-machine test of `cshdotcom/tailscale-win7-builds` v1.0.0 builds on an actual
Windows 7 SP1 x64 virtual machine (QEMU, software emulation, no KVM).

## Test scope

1. PE header verification (SubsystemVersion / OS version = 6.1)
2. Windows 7 SP1 x64 boot inside QEMU TCG
3. `tailscaled-win7-legacy.exe` daemon launch on Win7
4. `tailscale-win7-legacy.exe` CLI on Win7 (`version`, `up`, `status`)
5. Tailscale login flow: control-plane reachability + login URL generation from Win7

## Artifacts under test

- `tailscale-win7-legacy.exe` (25,686,016 bytes)
- `tailscaled-win7-legacy.exe` (37,849,600 bytes)
- `test-login-win7.bat` (1,625 bytes)

Source: https://github.com/cshdotcom/tailscale-win7-builds/releases/tag/v1.0.0

Test evidence (logs, screenshots) will be committed after the run.

Tag: `v1.0.0-win7-test`
