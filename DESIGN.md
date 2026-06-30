# setup-zabbly-incus — design & build handoff

A GitHub Action that installs and configures **Incus** (from the **Zabbly** Debian/Ubuntu
packages) on a GitHub-hosted Ubuntu runner, configured so that **both system containers and
KVM-accelerated virtual machines work**, and so the incus bridge **coexists with the runner's
preinstalled Docker** instead of being broken by it.

This document is the complete spec for an agent to build the action. It captures the *why*, the
design decisions and their rationale, a full draft `action.yml`, the security posture, the test
plan, the scaffolding checklist, and the open decisions left to the human.

---

## 1. Why this exists (motivation — read this first)

The motivating project is **`dionysius/immich-deb`** (Debian packaging of Immich). Its systemd
units are hardened with sandbox directives. A real bug was found and confirmed this way:

- The `immich-machine-learning` unit had `ProcSubset=pid`, which mounts the service's `/proc` with
  `subset=pid` and **hides `/proc/cpuinfo`**. Onnxruntime (bundled in the ML stack) reads
  `/proc/cpuinfo` at import for CPU-feature detection and **aborts with SIGABRT** when it's missing.
- This is reproducible **only in a VM**. System containers do **not** reproduce it: the container
  manager injects a drop-in that forces `ProcSubset=all`/`ProtectProc=default` on every service, so
  the sandbox directive is silently neutralised and the bug is masked.

Conclusion: **to gate releases on systemd-sandbox correctness, the CI must boot the units in real
VMs, not containers.** That requires a reliable way to stand up incus VMs on GitHub-hosted runners.
No existing action does this well (see §7), hence this action.

The fidelity argument is architecture-independent: a VM runs a real guest kernel + real systemd, so
it applies sandbox directives faithfully regardless of CPU arch or KVM-vs-emulation. KVM only
affects *speed*. (But incus has **no** software-emulation fallback — see §4.4.)

---

## 2. Scope

**In scope:** a composite action that, on an `ubuntu-*` GitHub-hosted runner,
1. installs incus from the Zabbly apt repo (full VM support, newer than distro packages),
2. makes the incus bridge coexist with Docker's firewall rules,
3. enables/validates KVM so `incus launch --vm` works (fail fast & clearly when `/dev/kvm` is absent),
4. initialises incus and grants the runner user non-`sudo` access,
5. leaves the runner ready for `incus launch <image>` (container) and `incus launch <image> --vm`.

**Out of scope:** launching instances (the caller does that), remote incus servers (that's what the
nubificus/gntouts actions do — wrong model here), arm64 VM guarantees (see §9).

---

## 3. Naming & metadata

- **Convention:** `setup-<tool>` (`actions/setup-node`, `canonical/setup-lxd`, `maxwell-k/setup-incus`).
- **Chosen name:** `setup-zabbly-incus` (signals the Zabbly package source; disambiguates from
  `maxwell-k/setup-incus` and from distro incus). The idiomatic alternative `setup-incus` is also
  fine since the owner namespace already disambiguates — **decide before first publish** (§9).
- **Owner:** `dionysius`.
- **`action.yml` `name`:** `Setup Zabbly Incus`. Add `branding: { icon: 'box', color: 'blue' }` for
  the Marketplace.
- **License:** Apache-2.0 (matches both reference actions).

---

## 4. Design decisions (with rationale)

### 4.1 Install from Zabbly, via a pinned apt repo (not `curl … | sh`)

Zabbly is the upstream incus package source maintained by the incus maintainer (Stéphane Graber):
full VM support and newer than Ubuntu's archive. Repos:
- stable: `https://pkgs.zabbly.com/incus/stable`
- LTS:    `https://pkgs.zabbly.com/incus/lts-6.0` (track the current LTS line)
- key:    `https://pkgs.zabbly.com/key.asc`

Use the deb822 `.sources` format with `Signed-By` (mirrors `maxwell-k/setup-incus`), **not** the
`get/incus-stable | sh` convenience script — an embedded/pinned key is more auditable and matches
the hardened posture of `canonical/setup-lxd`.

**Key handling — decision:** embed the ASCII-armored Zabbly key in the repo as `zabbly.asc` and
point `Signed-By` at it (fully reproducible, no run-time trust). Add a CI job that re-downloads
`https://pkgs.zabbly.com/key.asc` and diffs it against the committed copy, so key rotation surfaces
as a failing check rather than a silent drift. (Simpler alternative: fetch the key at run time over
HTTPS like `nubificus/incus-remote-setup-action/src/install.js` does — less auditable; only choose
if maintaining the embedded key proves annoying.)

**Suites mapping:** Zabbly builds per distro codename. Derive the suite from the runner:
`sed -n 's/VERSION_CODENAME=/Suites: /p' /etc/os-release` (the maxwell-k trick).

### 4.2 Docker coexistence — surgical `DOCKER-USER` ACCEPT (not `nft flush ruleset`)

GitHub runners ship Docker, which sets the iptables `FORWARD` policy to DROP and installs a
`DOCKER-USER` chain. The incus bridge (`incusbr0`) traffic is dropped → instances get no network and
the **VM agent never comes up** (so `incus exec` hangs forever). This is *the* gotcha; it's why
naive setups quietly fall back to containers.

Two known fixes:
- **Blunt (`maxwell-k`):** purge Docker + `nft flush ruleset`. Works, but destroys Docker
  networking for the rest of the job — bad collateral for a general-purpose action.
- **Surgical (`canonical/setup-lxd`):** insert ACCEPT rules into `DOCKER-USER` for the bridge, in
  and out, with conntrack for return traffic. Docker keeps working. **Use this as the default.**

  ```bash
  if sudo iptables -nL DOCKER-USER >/dev/null 2>&1; then
    sudo iptables  -I DOCKER-USER -i "$bridge" -j ACCEPT
    sudo iptables  -I DOCKER-USER -o "$bridge" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
  fi
  # repeat with ip6tables
  ```

**⚠️ #1 validation item:** incus 6.x manages its own bridge firewalling via **nftables**, while
Docker on the runner uses **iptables (iptables-nft)**. The `DOCKER-USER` ACCEPT lives in the
iptables-nft ruleset; it should let bridge FORWARD traffic through regardless of incus's own nft
rules, but this interaction **must be confirmed empirically** in the integration test (launch a VM,
assert it gets a DHCP lease + outbound network). Offer a `firewall` input
(`docker-user` | `flush` | `none`) so users can fall back to the blunt flush if their job doesn't
need Docker.

### 4.3 KVM enablement & validation

- Add the standard udev rule so the `kvm` group has access (harmless even though incusd runs as
  root): `KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"`, then
  `udevadm control --reload-rules && udevadm trigger --name-match=kvm`.
- When `vm: true` (default), **assert `/dev/kvm` exists and fail with an actionable message if not**
  (because incus has no TCG fallback — see §4.4). Expose the result as an output `kvm-present`.

### 4.4 Incus has NO software-emulation fallback

Unlike raw QEMU (which can run guests under TCG without `/dev/kvm`, slowly), **incus refuses to
start a VM without `/dev/kvm`.** So KVM presence is a hard gate, not a speed knob. The action must
detect this and fail clearly rather than hang. (Per Oct-2025 community reports, `/dev/kvm` is
present on standard amd64 hosted runners; arm64 is uncertain — §9.)

### 4.5 Non-sudo access via group (mirror the references)

deb-incus uses the `incus-admin` socket group. To let the non-root runner user run `incus` without
`sudo`, repoint the socket group to one the runner user is already in (`adm`), like maxwell-k:

```bash
sudo sed -i "s/incus-admin/${group}/" /usr/lib/systemd/system/incus.service /usr/lib/systemd/system/incus.socket
sudo systemctl daemon-reload
sudo systemctl restart incus.socket
```

Expose a `group` input (default `adm`). `canonical/setup-lxd` does the equivalent with
`snap set lxd daemon.group`.

### 4.6 Init & CI speed

```bash
sudo incus admin waitready
# preseed if provided, else minimal
[ -z "$preseed" ] && sudo incus admin init --minimal || sudo incus admin init --preseed <<<"$preseed"
incus config set images.compression_algorithm none   # speeds up CI image handling (setup-lxd does this)
```

---

## 5. Inputs & outputs

Inputs (keep the surface close to `canonical/setup-lxd` where it makes sense):

| Input | Default | Purpose |
|---|---|---|
| `channel` | `stable` | Zabbly repo line: `stable` or `lts`. |
| `group` | `adm` | Socket group for non-sudo `incus` access. |
| `preseed` | `''` | `incus admin init` preseed YAML; empty → `--minimal`. |
| `bridges` | `incusbr0` | Comma-separated bridges to allow through the Docker firewall. |
| `vm` | `true` | Set up KVM and **require** `/dev/kvm`; fail fast if absent. |
| `firewall` | `docker-user` | Docker coexistence strategy: `docker-user` \| `flush` \| `none`. |

Outputs:

| Output | Purpose |
|---|---|
| `kvm-present` | `true`/`false` — whether `/dev/kvm` was found. |
| `incus-version` | Installed incus version. |

---

## 6. Draft `action.yml` (starting point — refine while implementing)

> Pure composite shell. **Pass every input via `env:`, never inline `${{ inputs.* }}` inside `run:`**
> — this prevents template injection (see `canonical/setup-lxd` commit "Avoid template injection
> from input params"). Quote all expansions.

```yaml
name: 'Setup Zabbly Incus'
description: 'Install & configure Incus (Zabbly packages) on a runner, ready for containers and KVM VMs'
branding:
  icon: 'box'
  color: 'blue'
inputs:
  channel:  { default: 'stable',     description: 'Zabbly repo line: stable | lts' }
  group:    { default: 'adm',        description: 'Socket group for non-sudo incus access' }
  preseed:  { default: '',           description: 'incus admin init preseed YAML (empty = --minimal)' }
  bridges:  { default: 'incusbr0',   description: 'Comma-separated bridges to allow through Docker firewall' }
  vm:       { default: 'true',       description: 'Set up KVM and require /dev/kvm' }
  firewall: { default: 'docker-user',description: 'Docker coexistence: docker-user | flush | none' }
outputs:
  kvm-present:   { description: 'Whether /dev/kvm was found', value: '${{ steps.kvm.outputs.present }}' }
  incus-version: { description: 'Installed incus version',    value: '${{ steps.install.outputs.version }}' }
runs:
  using: composite
  steps:
    - name: KVM setup & gate
      id: kvm
      shell: bash
      env: { INPUT_VM: '${{ inputs.vm }}' }
      run: |
        set -euo pipefail
        echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules >/dev/null
        sudo udevadm control --reload-rules; sudo udevadm trigger --name-match=kvm || true
        if [ -e /dev/kvm ]; then echo "present=true" >>"$GITHUB_OUTPUT"; else
          echo "present=false" >>"$GITHUB_OUTPUT"
          if [ "$INPUT_VM" = "true" ]; then
            echo "::error::/dev/kvm is absent on this runner; incus has no emulation fallback so --vm will fail. Use an amd64 GitHub-hosted runner, or set vm: false for containers only."
            exit 1
          fi
        fi
    - name: Make incus bridge coexist with Docker
      shell: bash
      env: { INPUT_BRIDGES: '${{ inputs.bridges }}', INPUT_FIREWALL: '${{ inputs.firewall }}' }
      run: |
        set -euo pipefail
        case "$INPUT_FIREWALL" in
          none)  : ;;
          flush) sudo nft flush ruleset ;;
          docker-user)
            IFS=',' read -ra BR <<<"$INPUT_BRIDGES"
            for raw in "${BR[@]}"; do b=$(echo "$raw" | xargs)
              for ipt in iptables ip6tables; do
                if sudo "$ipt" -nL DOCKER-USER >/dev/null 2>&1; then
                  sudo "$ipt" -I DOCKER-USER -i "$b" -j ACCEPT
                  sudo "$ipt" -I DOCKER-USER -o "$b" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
                fi
              done
            done ;;
          *) echo "::error::unknown firewall mode: $INPUT_FIREWALL"; exit 1 ;;
        esac
    - name: Install incus from Zabbly
      id: install
      shell: bash
      env: { INPUT_CHANNEL: '${{ inputs.channel }}' }
      run: |
        set -euo pipefail
        case "$INPUT_CHANNEL" in stable) uri=https://pkgs.zabbly.com/incus/stable ;; lts) uri=https://pkgs.zabbly.com/incus/lts-6.0 ;; *) echo "::error::bad channel"; exit 1 ;; esac
        sudo install -d -m0755 /etc/apt/keyrings
        sudo cp "$GITHUB_ACTION_PATH/zabbly.asc" /etc/apt/keyrings/zabbly.asc   # embedded, pinned key
        { echo "Enabled: yes"; echo "Types: deb"; echo "URIs: $uri"; \
          sed -n 's/VERSION_CODENAME=/Suites: /p' /etc/os-release; \
          echo "Components: main"; echo "Architectures: $(dpkg --print-architecture)"; \
          echo "Signed-By: /etc/apt/keyrings/zabbly.asc"; } | sudo tee /etc/apt/sources.list.d/zabbly-incus.sources >/dev/null
        sudo apt-get update -qq
        sudo apt-get install -y -qq incus
        echo "version=$(incus --version)" >>"$GITHUB_OUTPUT"
    - name: Grant non-sudo access & init
      shell: bash
      env: { INPUT_GROUP: '${{ inputs.group }}', INPUT_PRESEED: '${{ inputs.preseed }}' }
      run: |
        set -euo pipefail
        sudo sed -i "s/incus-admin/${INPUT_GROUP}/" /usr/lib/systemd/system/incus.service /usr/lib/systemd/system/incus.socket
        sudo systemctl daemon-reload; sudo systemctl restart incus.socket
        sudo incus admin waitready
        if [ -z "$INPUT_PRESEED" ]; then sudo incus admin init --minimal; else sudo incus admin init --preseed <<<"$INPUT_PRESEED"; fi
        sudo incus config set images.compression_algorithm none || true
```

> Note: with the socket group repointed to `adm`, plain `incus` should work for the runner user, but
> the steps above use `sudo incus` during setup for robustness. Confirm group access timing in tests.

---

## 7. Reference implementations (already checked out locally — read these)

- `~/Projects/github.com/canonical/setup-lxd/action.yml` — **primary structural reference.** The
  surgical `DOCKER-USER` firewall, `env:`-based input hardening (no template injection), preseed vs
  `--auto`, `waitready`, `images.compression_algorithm none`. Composite, well-maintained (LXD team,
  zizmor-scanned). It installs the **snap**, which is exactly what we're replacing with Zabbly debs.
- `~/Projects/github.com/maxwell-k/setup-incus/action.yaml` — the Zabbly apt-source + embedded key,
  the codename→Suites `sed` trick, and the `incus-admin`→`adm` group sed-patch. Also shows the
  blunt Docker purge + `nft flush` (the `flush` fallback). Single-maintainer, 2★.
- `~/Projects/github.com/nubificus/incus-remote-setup-action/src/install.js` — the run-time key URL
  (`https://pkgs.zabbly.com/key.asc`) and deb822 `.sources` content (alternative key handling).
  NOTE: nubificus/gntouts actions target a **remote** incus server over SSH+TLS — wrong model here,
  reference only. (`gntouts/incus-remote-setup-action` even `console.log`s the trust token — don't
  copy its patterns.)
- `~/Projects/github.com/dionysius/immich-deb/.github/workflows/vm-probe.yml` — the **sandbox-fidelity
  probe** (boots an incus `--vm`, runs a `ProcSubset=pid` service, asserts `cpuinfo BLOCKED`). Fold
  this assertion into this action's integration test (§8) — it is the proof the action delivers
  VM-level systemd-directive testing.

---

## 8. Testing / CI for this repo

Create `.github/workflows/integration.yml`:
- Matrix over runner images: `ubuntu-22.04`, `ubuntu-24.04` (amd64).
- Steps: `actions/checkout` → `uses: ./` (the action) → then assert all of:
  1. **container** works: `incus launch images:debian/13/cloud c1 && incus exec c1 -- cloud-init status --wait`.
  2. **VM** works end-to-end: `incus launch images:debian/13/default v --vm`, wait for the agent,
     confirm it has **outbound network** (proves the Docker firewall fix), then run the
     **sandbox-fidelity probe**: a `ProcSubset=pid` service inside the VM must report
     `CPUINFO=BLOCKED`. (Reuse the immich-deb `vm-probe.yml` logic.)
  3. report `/dev/kvm` presence and VM boot time to the step summary.
- Add `.github/workflows/zizmor.yml` (GHA security scanner), `.github/dependabot.yml`, and run
  `actionlint`. Pin all third-party actions by full commit SHA.

Caveat to document in the README: `/dev/kvm` availability on hosted runners has historically varied;
if a runner lacks it the `vm: true` gate fails by design. arm64 hosted runners may not provide KVM
(§9).

---

## 9. Open decisions for the human

1. **Repo name:** `setup-zabbly-incus` (current) vs idiomatic `setup-incus`. Trivial to rename
   pre-publish.
2. **Default `channel`:** `stable` vs `lts`. (Draft defaults to `stable`.)
3. **Key handling:** embedded+CI-diff (recommended, drafted) vs run-time fetch.
4. **arm64:** decide whether to support/test arm64 VMs (GHA arm64 runner `/dev/kvm` is uncertain) or
   document "VMs are amd64-only; arm64 = containers only".
5. **Marketplace publish:** finalise `branding`, tag `v1`, add a moving major tag.

---

## 10. Scaffolding checklist

- [ ] `action.yml` (from §6, refined)
- [ ] `zabbly.asc` (embedded key; from `https://pkgs.zabbly.com/key.asc`)
- [ ] `README.md` (usage example: `uses:` + `incus launch … --vm`; inputs/outputs table; KVM caveat)
- [ ] `LICENSE` (Apache-2.0)
- [ ] `.github/workflows/integration.yml` (§8)
- [ ] `.github/workflows/zizmor.yml`, `.github/dependabot.yml`
- [ ] `.gitignore`
- [ ] First commit + `v1` tag once integration is green

---

## 11. How `immich-deb` will consume it

In `dionysius/immich-deb` `.github/workflows/packaging.yml`, the per-distro test job (amd64) becomes:

```yaml
- uses: dionysius/setup-zabbly-incus@v1
  with: { vm: 'true' }
- run: |
    incus launch images:debian/13/default t --vm -c limits.cpu=4 -c limits.memory=4GiB
    # wait for agent, push contrib/test/*, install the freshly-built .debs, boot-check the units
```

arm64 stays on the existing privileged-container boot check (sandbox-directive fidelity is covered by
the amd64 VMs, since that behaviour is architecture-independent). This action replaces the throwaway
`vm-probe.yml` once it's proven.
