---
summary: User asks about bridges, VLANs, firewall rules, IP addressing, or network troubleshooting
---

# Network Configuration

## When to use

The user asks about bridges, VLANs, firewall rules, IP addressing, or network troubleshooting.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Instructions

`proxman` covers Proxmox network introspection and modification through typed JSON commands. Discover argv via `proxman --help`. **Network configuration changes can be disruptive — always warn the user and require explicit confirmation.**

Typical shapes:

- `proxman network list --node <name> --json` — node bridges, bonds, VLANs.
- `proxman vm config --vmid <id> --json` / `proxman lxc config --vmid <id> --json` — per-guest `net*` fields.
- `proxman firewall rules --scope cluster|node|vm|lxc --json` — firewall rule lookup (see `--help` for the actual flag names).

### Common bridge setup (reference, for `/etc/network/interfaces` on the Proxmox host)

```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.x/24
    gateway 192.168.1.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

VLAN-aware bridge:

```
auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

If `proxman` does not expose a particular host-level network edit, fall back to `ssh root@<node> 'sed/...'` using your agent-managed SSH key — but only after you've confirmed the change with the user, written down the rollback step, and have a recent snapshot of `/etc/network/interfaces`.

### Safety

- Network changes can drop your own SSH session mid-edit. Use `ifreload -a` rather than `systemctl restart networking` when possible (preserves running connections).
- Always present a rollback plan: "If this breaks, the operator runs `<command>` on console to restore."
