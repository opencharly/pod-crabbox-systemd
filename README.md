# pod-crabbox-systemd

The **systemd-init CachyOS lease image** for the Crabbox `containerd` provider —
the base [`pod-crabbox`](https://github.com/opencharly/pod-crabbox) lease contract
with systemd as PID 1.

## What it provides

Every lease contract property of the base image is inherited unchanged:

| Property | Value |
|---|---|
| Base | CachyOS (`cachyos.cachyos`) via `pod-crabbox:v2026.307.1828` |
| SSH | `sshd` on **2222**, public-key only, user `crabbox` |
| Sudo | `crabbox` has passwordless sudo |
| Inputs | read-only `/crabbox`: `authorized_keys`, `lease.env`, `bootstrap.sh` |
| Readiness | `/usr/local/libexec/crabbox/ready` — polled over SSH until exit 0 |

This variant changes the init and adds what systemd-as-PID-1 needs:

| Property | Value |
|---|---|
| PID 1 | `/sbin/init` → `/usr/lib/systemd/systemd` |
| Units enabled | `sshd.service`, `crabbox-bootstrap.service` |
| Masked | `serial-getty@hvc0.service` (the getty hangs a krun boot on `dev-hvc0.device`) |
| sysctl | `/etc/sysctl.d/50-pid-max.conf` → `kernel.pid_max=32768` |
| Shutdown | `/usr/lib/systemd/system-shutdown/krun-poweroff` — a poweroff/halt becomes `systemctl reboot -ff`, the clean krun stop |
| Runner deps | `icu krb5 libunwind lttng-ust` (the Actions runner's .NET runtime deps) |

## How to use it

The image is run by the Crabbox `containerd` provider. Under crun/runc it needs the
runtime's systemd mode, and under krun the guest's PID-1 handoff:

```sh
# crun / runc
--systemd=always
# krun
-e KRUN_INIT_PID1=1 --ulimit nofile=524288:524288
```

The lease inputs are the same read-only `/crabbox` directory the base image uses;
`crabbox-bootstrap.service` applies them at boot, and the consumer runs the
readiness hook over SSH afterwards.

## Layout

- `charly.yml` — the `pod-crabbox-systemd` image (base import + systemd entrypoint).
- `candy/systemd-init/` — the units, mask, sysctl drop-in, shutdown hook and
  runner dependencies this variant adds.
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `.github/workflows/image.yml` — builds on `pull_request` and publishes at the
  merge tag, so the image tag equals the git tag.
- `CHANGELOG/` — per-CalVer history.

## Related

- [`opencharly/pod-crabbox`](https://github.com/opencharly/pod-crabbox) — the lease
  base image this one imports.
- [`CONTAINERD-IMAGES.md`](https://github.com/opencharly/opencharly/blob/main/CONTAINERD-IMAGES.md)
  §2 — the lease-image contract.
