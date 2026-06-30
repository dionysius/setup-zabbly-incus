# setup-zabbly-incus

A GitHub Action that installs and configures [Incus](https://linuxcontainers.org/incus/) from the
[Zabbly](https://github.com/zabbly/incus) packages on a GitHub-hosted Ubuntu runner, ready for both
system containers and KVM-accelerated virtual machines. It also makes the incus bridge coexist with
the runner's preinstalled Docker instead of being broken by it (see
[Prevent connectivity issues with Incus and Docker](https://linuxcontainers.org/incus/docs/main/howto/network_bridge_firewalld/#prevent-connectivity-issues-with-incus-and-docker)).

## Usage

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v6
      - uses: dionysius/setup-zabbly-incus@v1
      - name: Boot a VM
        run: |
          incus launch images:debian/13/default v --vm -c limits.cpu=4 -c limits.memory=4GiB
          for _ in $(seq 60); do incus exec v -- true 2>/dev/null && break; sleep 3; done
          incus exec v -- systemctl is-system-running --wait || true
```

The default `vm: auto` sets up KVM when present and otherwise carries on, so containers work the same
way (`incus launch images:debian/13/cloud c1`). Use `vm: false` to skip KVM setup on container-only
runs.

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `channel` | `stable` | Zabbly incus line: `stable`, `daily`, or an `lts-X.Y` line (e.g. `lts-7.0`). |
| `group` | `adm` | Socket group for non-sudo `incus` access. |
| `preseed` | `''` | `incus admin init` preseed YAML; empty → `--minimal`. |
| `bridges` | `incusbr0` | Comma-separated bridges to allow through the Docker firewall. |
| `vm` | `auto` | KVM mode: `auto` (set up KVM, don't require it), `true` (require `/dev/kvm`, fail if absent), or `false` (skip KVM). |
| `firewall` | `docker-user` | Docker coexistence strategy: `docker-user` \| `flush` \| `none`. |

## Outputs

| Output | Purpose |
|---|---|
| `kvm-present` | `true`/`false` — whether `/dev/kvm` was found. |
| `incus-version` | Installed incus version. |

## KVM

Incus has no software-emulation fallback — it will not start a VM without `/dev/kvm`. The default
`vm: auto` sets up KVM but continues without it; `vm: true` requires it and fails fast if absent;
`vm: false` skips KVM setup entirely.

> **Note:** GitHub's free Linux **arm64** hosted runners do not provide KVM, so VMs cannot run there.
> Use an amd64 runner for VMs, or `vm: false` for containers on arm64. See GitHub's community
> discussion [Nested virtualization for Linux ARM64 runners?](https://github.com/orgs/community/discussions/149673).

## Docker coexistence

GitHub runners ship Docker, whose firewall rules drop incus bridge traffic by default. The `firewall`
input controls this: `docker-user` (default) inserts targeted ACCEPT rules so Docker keeps working,
`flush` clears the nftables ruleset, and `none` leaves it to you.

## Security

Installs from a pinned deb822 apt source with the Zabbly key embedded as `zabbly.asc` (`Signed-By`).
Inputs are passed to the shell via `env:`, and third-party actions are pinned by commit SHA.
