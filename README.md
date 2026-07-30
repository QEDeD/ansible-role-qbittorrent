<!--
SPDX-FileCopyrightText: 2025 spatterlight
SPDX-FileCopyrightText: 2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# qBittorrent Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [qBittorrent](https://www.qbittorrent.org/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Container network modes

The default `managed` network mode preserves the role's traditional behavior:
the role creates a dedicated Docker bridge, assigns a stable container hostname,
publishes configured ports, and connects configured additional networks.

The explicit `container` mode instead joins another container's network
namespace. This is useful when a dedicated companion container owns networking,
for example a VPN gateway:

```yaml
qbittorrent_container_network_mode: container
qbittorrent_container_network_container: vpn-gateway
qbittorrent_container_network_container_systemd_service: vpn-gateway.service
qbittorrent_container_network_container_resolv_conf_path: /path/to/vpn-gateway/resolv.conf

# This is the Docker bridge attached to the namespace owner, not a network
# attached independently to qBittorrent.
qbittorrent_container_labels_traefik_docker_network: vpn-gateway
```

`qbittorrent_container_network_container` is a stable, exact Docker name used
only for discovery; it is not a container ID. A root-owned lifecycle helper
resolves that exact name to a full 64-hex container ID, verifies the matching
name and healthy state by ID, repeats the name-to-ID resolution immediately
before creation, and creates qBittorrent with
`--network=container:<resolved-full-ID>`. Before starting qBittorrent, it again
proves the owner identity and health and qBittorrent's exact namespace binding.
It does not fall back to an independently managed network.

Docker forbids or makes unsafe several independent container settings while
sharing a network namespace. In `container` mode, validation therefore requires
the following settings to be empty or disabled:

- `qbittorrent_container_hostname`
- `qbittorrent_container_network`
- `qbittorrent_container_network_deletion_enabled`
- `qbittorrent_container_http_host_bind_port`
- `qbittorrent_container_torrenting_host_bind_port`
- `qbittorrent_container_additional_networks`
- `qbittorrent_container_extra_arguments`

The defaults for hostname, managed network, and network deletion adjust
automatically when the mode changes. Labels and additional volumes remain
available, allowing a reverse proxy to discover the service. Container mode
requires `qbittorrent_container_network_container_resolv_conf_path`, an
absolute path to resolver configuration supplied by the namespace owner. Point
it at the owner's VPN-scoped resolver configuration, not the host's general
resolver. The helper mounts this file read-only at `/etc/resolv.conf`;
additional volumes cannot replace that destination. This makes the resolver
selection explicit and prevents Docker from supplying an independent resolver.
The helper requires the source to exist as a regular, non-symlink file
immediately before container creation. The file may be absent while the
namespace owner is not running, so the role does not require it during
installation.
The qBittorrent container still drops all capabilities before adding only its
existing file-permission capabilities; it does not receive `NET_ADMIN` or
`NET_RAW`.

When Traefik labels are enabled, explicitly set
`qbittorrent_container_labels_traefik_docker_network` to a network attached to
the namespace owner. Traefik follows the container network-mode relationship to
reach qBittorrent through that owner network. Container mode rejects additional
mounts whose source or destination directly exposes a container-engine control
socket, the host root, `/proc`, `/sys`, or `/dev`; broad control-bearing parent
mounts are also rejected. The resolver source has the same source-path
restrictions. Additional mount destinations must be absolute and cannot contain
`.` or `..` path segments. Additional bind sources have the same canonical
absolute-path requirement; named-volume sources remain supported. Ordinary
application-data mounts remain supported. Static validation cannot resolve host
symlinks for general additional mounts, so operators must still ensure allowed
bind sources do not resolve to host-control paths.

The namespace-owner systemd service is mandatory in `container` mode and is
automatically added to qBittorrent's bound service list. Each entry becomes both
`BindsTo=` and `After=`, so qBittorrent starts after the namespace owner and is
stopped if that owner disappears.

This dependency only covers qBittorrent's side of the lifecycle. Configure the
owner service with the reverse `Wants=qbittorrent.service` relationship (usually
through that role's wanted-services list), so qBittorrent is started again and
recreated after the owner recovers. Without that reverse edge, `BindsTo=` stops
qBittorrent safely but does not restart it when the owner returns.

Changing the configured owner name or service changes the rendered qBittorrent
unit and causes the normal service-manager restart path. Recreating an owner
container under the same name does not change the unit text. The `BindsTo=`
stop and reverse `Wants=` start must recreate qBittorrent so Docker resolves the
replacement owner's new container ID. This recovery path must be proven by
runtime lifecycle tests.

Docker health gates creation and startup; it is not a continuous service-stop
signal. If the owner stays active while its tunnel or health fails, ongoing
network isolation is the namespace owner's firewall or VPN kill switch. That
boundary is intentional so the qBittorrent process and local diagnostics can
remain available without Internet access.

The lifecycle guarantee assumes qBittorrent is controlled by the rendered
systemd unit. Containers started, renamed, or otherwise managed directly with
Docker are outside this role's lifecycle boundary. Runtime acceptance tests
must include a negative direct-command case and confirm that such unmanaged
state is reported as outside the supported guarantee.

### Switching modes and rollback

When switching an existing installation from `managed` to `container`, the role
uses exact-name Docker queries to resolve the owner and qBittorrent to full IDs,
then inspects only those immutable IDs. Unless qBittorrent's current network
mode is `container:<resolved-owner-ID>`, the role attempts both systemd stop and
direct Docker removal. The originally resolved full ID and any container which
subsequently owns the stable qBittorrent name are separate cleanup targets. The
role proves systemd inactivity, the original ID absent or stopped, and the
stable name absent or stopped before changing support files or pulling an
image. Ambiguous Docker or systemd results fail rather than count as isolation.
If the target owner does not exist yet, the role first isolates qBittorrent and
then continues rendering the unit; this supports fresh installs where
containers are started only in the later service-manager phase. The rendered unit's
`After=`/`BindsTo=` relationship and ID-validating health gate defer qBittorrent
creation until the owner exists and is healthy. A later playbook failure
therefore leaves qBittorrent stopped instead of leaving the old instance with
direct network access. Config and downloads remain on their existing
bind-mounted paths.

Uninstall performs the same independent control-plane checks: it attempts
systemd stop, directly removes both the captured original full ID and any
container now owning the stable name, proves the unit inactive and both Docker
identities absent, and only then removes the unit and root-owned helper. Config
and downloads retain the role's existing non-destructive behavior.

To roll back, restore `qbittorrent_container_network_mode: managed`, remove the
namespace-owner setting and bound owner service, restore any desired managed
network or port settings, and run the role again. The managed bridge is created
before the rendered unit is restarted. Until that successful restart, an
already-protected qBittorrent container remains on its previous namespace.
This rollback is an intentional downgrade of the role's VPN enforcement:
accept explicitly that qBittorrent will regain independently managed network
connectivity before performing it.

## Limitations

Docker does not permit assigning qBittorrent its usual stable container hostname
in `container` network mode. Treat unclean shutdown and stale Qt lock-file
recovery as a runtime acceptance gate for this mode: the selected namespace
owner lifecycle must prove recovery, or an explicit lock-file mitigation must
be designed, before production use.

The role-local Molecule scenario does not yet exercise `container` mode. This is
a readiness blocker, not evidence supplied by the managed-mode scenario.
External lifecycle tests must prove owner absence, unhealthy and stopped owner,
tunnel failure with the owner still running, owner recreation under the same
name, qBittorrent restart, same-host peer access, and managed-to-container
migration before this mode is treated as production-ready. They must also
exercise the documented direct-command boundary rather than treating an
unmanaged container as role-controlled state.

In particular, the owner-still-running tunnel-loss test must prove the owner's
firewall remains fail-closed; this qBittorrent role cannot establish that
property on the namespace owner's behalf.

This role configures qBittorrent with security in mind by doing the following:

1. Running the container as a non-root user
2. Making the filesystem read-only
3. Dropping most capabilities

Unfortunately, due to upstream requirements, some admissions had to be made:

1. Several capabilities related to permissions are added to the container
   - SETUID
   - SETGID
   - CHOWN
   - FOWNER
   - DAC_OVERRIDE
2. A `tmpfs` volume is mounted with `exec` permissions

You can read more about these upstream requirements in the documentation:

1. <https://docs.linuxserver.io/misc/non-root/>
2. <https://docs.linuxserver.io/misc/read-only/>

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

Refer to [this page](./molecule/README.md) for details about how to utilize it.
