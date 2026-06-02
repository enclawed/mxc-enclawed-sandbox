# mxc-enclawed-sandbox

Integration repo for running [Enclawed](https://github.com/enclawed/enclawed-oss) inside [Microsoft eXecution Container (MXC)](https://github.com/microsoft/mxc) — JSON policy configs.

MXC launches processes directly under one of its OS-native containment backends (Bubblewrap / LXC on Linux, ProcessContainer / Windows Sandbox on Windows, Seatbelt on macOS) with a JSON-defined filesystem + network policy. This repo ships the four ready-to-use configs that drop the Enclawed runtime into each backend, with the filesystem narrowed to the install path + the keyring-state directory and the network locked to Enclawed's two default backends (Anthropic + local Ollama).

Companion to [enclawed/openshell-enclawed-sandbox](https://github.com/enclawed/openshell-enclawed-sandbox), which provides the equivalent integration on the NVIDIA OpenShell side.

## What's Enclawed?

A classification-gated AI agent gateway with an MCP-attested transport layer ([arXiv:2605.24248](https://arxiv.org/abs/2605.24248)). Composes admission control + tool-level authorization + a hash-chained audit log around standard MCP servers. Bundled apps: `secretary` (Gmail / CalDAV / CardDAV automation) and `codex` (hardened coding agent).

## Repo contents

| File | Role |
|---|---|
| `configs/linux-bubblewrap.json` | Linux default backend. ro=`/opt/enclawed`, rw=`/var/lib/enclawed` + `/tmp`, network locked to Anthropic + loopback Ollama. **Live-validated against microsoft/mxc HEAD + bubblewrap 0.9.0 (see `mxc-validation.txt`).** |
| `configs/linux-lxc.json` | Same intent under MXC's LXC backend (heavier isolation; needs the `lxc` toolset). Shape mirrors the bubblewrap config which was live-validated; LXC itself was not live-tested on this host. |
| `configs/windows-processcontainer.json` | Windows 11 24H2+ default backend. Paths use `%USERPROFILE%` / `%ProgramFiles%`. Schema-validated; not live-tested in WSL2. |
| `configs/macos-seatbelt.json` | macOS Seatbelt (schema 0.6.0-alpha+; needs MXC's `experimental` mode). Schema-validated; not live-tested in WSL2. |
| `mxc-validation.txt` | Captured transcript from the live Bubblewrap run that emitted `enclawed 1.0.1 (7d25e4a)` from inside an MXC-managed sandbox. |
| `LICENSE` | MIT (matches both MXC upstream and enclawed-oss core). |

## How operators run this

MXC's user-facing API is the **Node SDK** (`@microsoft/mxc-sdk`), not a `mxc` CLI. The configs in this repo are JSON payloads passed to `spawnSandboxFromConfig()`. Minimal driver:

```javascript
import fs from 'node:fs';
import { spawnSandboxFromConfig } from '@microsoft/mxc-sdk';

const cfg = JSON.parse(fs.readFileSync('configs/linux-bubblewrap.json', 'utf8'));
const child = spawnSandboxFromConfig(cfg, { usePty: false });
child.stdout?.pipe(process.stdout);
child.stderr?.pipe(process.stderr);
child.on('close', (code) => process.exit(code ?? 1));
```

## One-time host setup (Linux)

The Linux Bubblewrap config references two paths that must exist on the host before MXC will bind-mount them:

```bash
sudo apt install bubblewrap                    # MXC's Linux default backend
sudo mkdir -p /opt/enclawed /var/lib/enclawed
sudo chown $USER:$USER /var/lib/enclawed       # operator writes audit log + keyring state here

git clone https://github.com/enclawed/enclawed-oss /opt/enclawed
cd /opt/enclawed && pnpm install --frozen-lockfile
```

The same shape applies on Windows (`%ProgramFiles%\Enclawed` ro, `%USERPROFILE%\.enclawed` rw) and macOS (`/usr/local/enclawed` ro, `$HOME/.enclawed` rw). Adjust the readonly/readwrite path arrays in the corresponding config if your install lays out differently.

## Network policy caveat

The bundled configs ship `network.defaultPolicy: "block"` plus `enforcementMode: "firewall"` with an allow-list. Two known frictions:

1. **`firewall` mode uses iptables and requires root.** Running unprivileged fails at startup with `iptables ... Permission denied (you must be root)`.
2. **Firewall mode currently rejects IPv6-resolved hosts.** Any `allowedHost` whose DNS returns AAAA records (e.g. `api.anthropic.com`) crashes MXC at policy-install time because it calls `iptables` (IPv4 only) for both A and AAAA results. Tracked as [microsoft/mxc#479](https://github.com/microsoft/mxc/issues/479). Until that lands:
    - Switch `enforcementMode` to `proxy` and route outbound HTTP through an MXC-launched HTTP proxy. No iptables, no IPv6 issue, no root.
    - Or drop the firewall (`defaultPolicy: "allow"`) and rely entirely on Enclawed's own internal egress allowlist (`enforceAllowlists=true` in the framework policy). Defensible because Enclawed already enforces its egress allowlist at the MCP boundary; the MXC firewall is defense in depth, not the only line.
    - Or restrict `allowedHosts` to IPv4-only literals (`140.82.114.*` style) — clunky but unblocks the iptables path.

## LXC host setup

The LXC config requires unprivileged-LXC host configuration before MXC can dispatch into it (otherwise `lxc-create` fails with `No uid mapping for container root`). One-time operator setup:

```bash
sudo apt install lxc lxc-templates uidmap
mkdir -p ~/.config/lxc && cp /etc/lxc/default.conf ~/.config/lxc/
# Add subuid/subgid mappings per the LXC docs, then:
echo "$USER veth lxcbr0 10" | sudo tee -a /etc/lxc/lxc-usernet
```

After that, MXC will spawn LXC containers from the bundled config without further intervention.

## Live validation status

| Platform | Backend | Status |
|---|---|---|
| Linux | Bubblewrap (no firewall) | Live e2e: `enclawed 1.0.1 (7d25e4a)` printed from inside the sandbox. See `mxc-validation.txt`. |
| Linux | Bubblewrap + firewall | Config-shape accepted under sudo; blocked by [microsoft/mxc#479](https://github.com/microsoft/mxc/issues/479) (iptables + IPv6). Workarounds documented above. |
| Linux | LXC | Config accepted by MXC's LXC dispatch; remaining failure is unprivileged-LXC host setup. See `mxc-validation.txt`. |
| Windows | ProcessContainer | Schema-validated; not live-tested in WSL2 (requires Windows 11 24H2+). |
| macOS | Seatbelt | Schema-validated; not live-tested in WSL2 (requires macOS). |

## License

MIT. See `LICENSE`.
