# setup-zabbly-incus

A GitHub Action that installs and configures [Incus](https://linuxcontainers.org/incus/) from the
[Zabbly](https://github.com/zabbly/incus) Debian/Ubuntu packages on a GitHub-hosted Ubuntu runner —
set up so that **both system containers and KVM-accelerated virtual machines work**, and so the
incus bridge **coexists with the runner's preinstalled Docker** instead of being broken by it.

Why VMs and not just containers: a VM runs a real guest kernel + real systemd, so it applies
systemd **sandbox directives faithfully**. An incus *system container* injects a drop-in that forces
`ProcSubset=all` / `ProtectProc=default`, silently neutralising those directives — which masks real
bugs. To gate releases on systemd-sandbox correctness, CI must boot the units in real VMs.

## Usage

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v6
      - uses: dionysius/setup-zabbly-incus@v1
        with:
          vm: 'true'
      - name: Boot a VM
        run: |
          incus launch images:debian/13/default v --vm -c limits.cpu=4 -c limits.memory=4GiB
          for _ in $(seq 60); do incus exec v -- true 2>/dev/null && break; sleep 3; done
          incus exec v -- systemctl is-system-running --wait || true
```

Containers only (no KVM required):

```yaml
      - uses: dionysius/setup-zabbly-incus@v1
        with:
          vm: 'false'
      - run: incus launch images:debian/13/cloud c1
```

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `channel` | `stable` | Zabbly repo line: `stable` or `lts` (LTS 6.0 line). |
| `group` | `adm` | Socket group for non-sudo `incus` access. Must be a group the runner user is already in. |
| `preseed` | `''` | `incus admin init` preseed YAML; empty → `--minimal`. |
| `bridges` | `incusbr0` | Comma-separated bridges to allow through the Docker firewall. |
| `vm` | `true` | Set up KVM and **require** `/dev/kvm`; fail fast if absent. |
| `firewall` | `docker-user` | Docker coexistence strategy: `docker-user` \| `flush` \| `none`. |

## Outputs

| Output | Purpose |
|---|---|
| `kvm-present` | `true`/`false` — whether `/dev/kvm` was found. |
| `incus-version` | Installed incus version. |

## KVM / architecture caveat

Incus has **no software-emulation (TCG) fallback** — it refuses to start a VM without `/dev/kvm`.
So `/dev/kvm` is a hard gate, not a speed knob. With `vm: true` (the default) the action fails fast
and clearly if `/dev/kvm` is absent. Standard **amd64** GitHub-hosted runners provide it; **arm64**
hosted runners may not — there, run containers with `vm: false`. Sandbox-directive fidelity is
architecture-independent, so amd64 VMs cover it for all arches.

## Docker coexistence

GitHub runners ship Docker, whose iptables `FORWARD` DROP policy and `DOCKER-USER` chain drop
`incusbr0` traffic — instances get no network and the VM agent never comes up. The default
`firewall: docker-user` inserts surgical ACCEPT rules for the bridge (Docker keeps working). Use
`firewall: flush` to blunt-flush the nftables ruleset if your job doesn't need Docker, or
`firewall: none` to manage it yourself.

## Security

- Installs from a **pinned deb822 apt source** with the Zabbly key embedded as `zabbly.asc`
  (`Signed-By`), not a `curl … | sh` convenience script. A scheduled CI job diffs the embedded key
  against upstream so rotation surfaces as a failing check.
- All inputs are passed to shell via `env:` (never inlined into `run:`) to prevent template
  injection. Third-party actions are pinned by commit SHA; `zizmor` scans the workflows.

## License

[Apache-2.0](LICENSE).
