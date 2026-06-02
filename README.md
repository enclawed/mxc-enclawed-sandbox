# mxc-enclawed-sandbox

Integration repo for running [Enclawed](https://github.com/enclawed/enclawed-oss) inside [Microsoft eXecution Container (MXC)](https://github.com/microsoft/mxc) — JSON policy configs + bootstrap glue.

MXC launches processes directly under one of its OS-native containment backends (Bubblewrap / LXC on Linux, ProcessContainer / Windows Sandbox on Windows, Seatbelt on macOS) with a JSON-defined filesystem + network + UI policy. This repo ships the four ready-to-use configs that drop the Enclawed runtime into each backend, with the network policy locked to Enclawed's two default backends (Anthropic + local Ollama) and the filesystem narrowed to the install path + the keyring-state directory.

Companion to [enclawed/openshell-enclawed-sandbox](https://github.com/enclawed/openshell-enclawed-sandbox), which provides the equivalent integration on the NVIDIA OpenShell side.

## What's Enclawed?

A classification-gated AI agent gateway with an MCP-attested transport layer ([arXiv:2605.24248](https://arxiv.org/abs/2605.24248)). Composes admission control + tool-level authorization + a hash-chained audit log around standard MCP servers. Bundled apps: `secretary` (Gmail / CalDAV / CardDAV automation) and `codex` (hardened coding agent).

## Repo contents

| File | Role |
|---|---|
| `configs/linux-bubblewrap.json` | Linux default backend. Read-only `/opt/enclawed`, read-write keyring state, network locked to Anthropic + loopback Ollama. |
| `configs/linux-lxc.json` | Same policy under MXC's LXC backend (heavier isolation; needs the `lxc` toolset). |
| `configs/windows-processcontainer.json` | Windows 11 24H2+ default backend. Paths use `%USERPROFILE%` / `%ProgramFiles%`. |
| `configs/macos-seatbelt.json` | macOS Seatbelt (schema 0.6.0-alpha+; needs MXC's `experimental` mode on the macOS schema branch). |
| `LICENSE` | MIT (matches MXC upstream and enclawed-oss core). |

## Quick start

Install MXC and Enclawed once:

```bash
# Linux (Bubblewrap default)
sudo apt install bubblewrap                            # MXC's Linux default backend
git clone https://github.com/enclawed/enclawed-oss ~/.enclawed/enclawed-oss
cd ~/.enclawed/enclawed-oss && pnpm install --frozen-lockfile
# follow https://github.com/microsoft/mxc#building for the MXC CLI
```

Run Enclawed under MXC:

```bash
# Linux
mxc run --config configs/linux-bubblewrap.json

# Windows (from PowerShell, 24H2+ build 26100+)
mxc run --config configs\windows-processcontainer.json

# macOS (schema 0.6.0-alpha+, experimental)
mxc run --config configs/macos-seatbelt.json --experimental
```

The four configs all carry the same intent:

- **Process** — launches `enclawed secretary` (the Gmail/calendar daemon). Swap `secretary` for `codex` to host the coding-agent surface instead.
- **Filesystem** — read-only on the source tree, read-write only on the keyring state directory and a sandbox-scoped `/tmp` slice.
- **Network** — `defaultPolicy: block`, allow-list locked to `api.anthropic.com` and `127.0.0.1` (the two backends Enclawed ships with by default). Add other LLM provider hosts if your install targets a different backend.

## License

MIT. See `LICENSE`.
