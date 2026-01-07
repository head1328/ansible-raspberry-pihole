# Ansible Role: Raspberry - Pihole

An Ansible role that manages [Pi-hole](https://pi-hole.net) on Raspberry Pi OS (Debian 13 / Trixie) using containers.

## Description

This Ansible role deploys **Pi-hole** as a container using **Podman Quadlets** and
exposes the web interface securely via a **Caddy reverse proxy** with **internal TLS**.

The setup is intended for **internal networks** (e.g. FRITZ!Box environments) and
does **not** rely on public port exposure or Let's Encrypt.

## Architecture

```
Browser
  │
  ▼
https://pihole.fritz.box:8443
  │
  ▼
Caddy (internal CA, TLS termination, :443)
  │
  ▼
http://pihole:80   (Podman network)
  │
  ▼
Pi-hole Admin UI

DNS:
Clients ──▶ Host:53
           (iptables REDIRECT)
           Host:5353 ──▶ Pi-hole:53
```

## Key Characteristics

- Podman + systemd **Quadlets**
- **Automatic container updates** via Podman auto-update (configurable)
- No host networking
- No DHCP (DNS only)
- Pi-hole web UI is **not exposed** on the host
- HTTPS via **Caddy `tls internal`**
- No Let's Encrypt, no public ACME challenges
- Minimal container privileges (no `NET_ADMIN`)
- Lifecycle managed via **systemd handlers**
- Host selection via inventory groups (`hosts: piholes`)
- IPv4 and IPv6 supported
- Staggered update scheduling for high availability

## Requirements

- Podman with Quadlet support
- systemd
- Ansible collections:
  ```
  containers.podman
  community.general
  ```

## Dependencies

This role depends on `head1328.podman` which:
- Installs Podman
- Configures rootless Podman for a dedicated user
- Sets up user namespaces (subuid/subgid)
- Enables systemd user lingering
- Configures resource delegation for containers

The dependency is automatically installed when using `ansible-galaxy` with `requirements.yml`.

## Usage

### Installation

Install the role and its dependencies using Ansible Galaxy:

```bash
ansible-galaxy role install head1328.pihole
```

Or use a `requirements.yml`:

```yaml
---
roles:
  - name: head1328.pihole
```

Then install with:

```bash
ansible-galaxy install -r requirements.yml
```

### Playbook Example

Assign hosts to the `piholes` inventory group and include the role in a play:

```yaml
- name: Deploy Pi-hole
  hosts: piholes
  become: true
  roles:
    - head1328.pihole
```

The `head1328.podman` dependency will be automatically applied before this role.

The role itself does **not** contain any host group logic.

## DNS Upstreams (Freifunk München)

This role is configured to use the **Freifunk München (FFMUC) DNS resolvers**
as upstream DNS servers.

These resolvers are operated by the Freifunk München community and provide
privacy-friendly, non-commercial DNS resolution.

Reference:
https://ffmuc.net/wiki/knb:dns

Example configuration:

```yaml
pihole_dns_upstreams: "5.1.66.255;185.150.99.255"
```

## DNS Port Handling (iptables)

Pi-hole listens on port **53 inside the container**, but the container exposes DNS
on **host port 5353**.

To allow clients to continue using standard DNS port **53**, the role configures
iptables **NAT PREROUTING REDIRECT** rules:

- TCP 53 -> 5353 (IPv4)
- UDP 53 -> 5353 (IPv4)
- TCP 53 -> 5353 (IPv6)
- UDP 53 -> 5353 (IPv6)

The rules are applied on the interface defined by:

```yaml
pihole_in_interface
```

Firewall rules are automatically persisted using **iptables-persistent**:

- `/etc/iptables/rules.v4` - IPv4 rules
- `/etc/iptables/rules.v6` - IPv6 rules

Rules are only saved when changes are detected, ensuring idempotent behavior.

## Variables

Default values are defined in:

```
roles/pihole/defaults/main.yml
```

Important variables include:

```yaml
pihole_fqdn: pihole.fritz.box
pihole_timezone: UTC
pihole_dns_upstreams: "5.1.66.255;185.150.99.255"
pihole_in_interface: eth0
pihole_dns_port: 5353
pihole_https_port: 8443

# Auto-update settings
pihole_autoupdate_enabled: true
pihole_autoupdate_time: "03:00"
pihole_autoupdate_random_delay: 900
```

Sensitive values (e.g. `pihole_password`) should be overridden via:

```
group_vars/piholes.yml
```

## Automatic Updates

The role configures **Podman auto-update** to automatically pull and deploy new
container images from the registry.

### Configuration

Auto-updates are **enabled by default** and run via a systemd timer.

Key variables:

- `pihole_autoupdate_enabled` - Enable/disable auto-updates (default: `true`)
- `pihole_autoupdate_time` - When to run updates (systemd OnCalendar format, default: `"03:00"`)
- `pihole_autoupdate_random_delay` - Random delay in seconds to avoid exact simultaneous updates (default: `900` = 15 minutes)

### Staggered Updates for Multiple Pi-holes

When running multiple Pi-hole instances, configure **different update times** to
ensure high availability:

```yaml
# inventories/hq/host_vars/hq-pihole-primary.yml
pihole_autoupdate_time: "03:00"
pihole_autoupdate_random_delay: 900

# inventories/hq/host_vars/hq-pihole-secondary.yml
pihole_autoupdate_time: "04:30"
pihole_autoupdate_random_delay: 900
```

This ensures:
- Primary updates around 3:00 AM (±15 minutes)
- Secondary updates around 4:30 AM (±15 minutes)
- DNS service remains available during updates

### Disable Auto-updates

To disable auto-updates for a specific host:

```yaml
# inventories/hq/host_vars/hq-pihole-test.yml
pihole_autoupdate_enabled: false
```

### Manual Updates

Check for updates manually:

```bash
# As podman user
podman auto-update --dry-run

# Apply updates
podman auto-update
```

View timer status:

```bash
# As podman user
systemctl --user status podman-auto-update.timer
systemctl --user list-timers
```

## TLS Trust (One-time Setup)

Caddy uses an **internal Certificate Authority**.

After the first deployment, export the CA certificate:

```bash
podman cp caddy:/data/caddy/pki/authorities/local/root.crt ./caddy-root-ca.crt
```

Trust this certificate on all client systems to avoid browser warnings.

## Notes

- This role intentionally avoids Let's Encrypt.
- External devices (e.g. FRITZ!Box) may forward custom ports
  (e.g. `8443 -> 443`) to the host.
- The role assumes Podman networking for container-to-container
  communication (Caddy -> Pi-hole).
- iptables is used for DNS port redirection; nftables is not covered.

## Testing

This role uses [Molecule](https://molecule.readthedocs.io/) for testing with Podman.

### Prerequisites

- `podman` e.g. via homebrew
- `python3` e.g. via homebrew
- `ansible` `molecule` and `molecule-plugins[podman]`, e.g. `pip install molecule "molecule-plugins[podman]"`

### Test Scenarios

- **default**: Tests the complete role including the `head1328.podman` dependency

### Running Tests Locally

```bash
# Run full test suite (create, converge, idempotence, verify, destroy)
make test

# Run all test scenarios
make test-all

# Development workflow
make converge  # Apply the role
make verify    # Run verification tests
make destroy   # Clean up test instances
make login     # Login to test instance

# Code quality
make lint      # Run ansible-lint
```

### CI/CD

This role uses [Woodpecker CI](https://woodpecker-ci.org/) for automated code quality checks on Codeberg. Linting with `ansible-lint` runs automatically on push and pull requests.

## License

AGPL-3.0-or-later
