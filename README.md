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
New managed bridges carry a role-ownership label. A compatible pre-existing
unlabelled bridge may be used, but is deliberately not claimed retroactively.
Uninstall deletes a managed bridge only when its immutable ID carries the
matching role label; legacy, adopted, or otherwise unowned bridges are retained
and reported. Networks owned by a companion container are never qBittorrent
cleanup targets.

The explicit `container` mode instead joins another container's network
namespace. This is useful when a dedicated companion container owns networking,
for example a VPN gateway:

```yaml
qbittorrent_container_network_mode: container
qbittorrent_container_network_container: vpn-gateway
qbittorrent_container_network_container_contract_label: io.example.vpn.fail-closed-contract
qbittorrent_container_network_container_contract_label_value: v1
qbittorrent_container_network_container_systemd_service: vpn-gateway.service
qbittorrent_container_network_container_resolv_conf_path: /path/to/vpn-gateway/resolv.conf

# This is the Docker bridge attached to the namespace owner, not a network
# attached independently to qBittorrent.
qbittorrent_container_labels_traefik_docker_network: vpn-gateway
```

`qbittorrent_container_network_container` is a stable, exact Docker name used
only for discovery; it is not a container ID. A root-owned lifecycle helper
resolves that exact name to a full 64-hex container ID, verifies the matching
name, required owner-contract label, and healthy state by ID, repeats the
name-to-ID resolution immediately before creation, and creates qBittorrent with
`--network=container:<resolved-full-ID>`. Before starting qBittorrent, it again
proves the owner identity, contract label, and health and qBittorrent's exact
namespace binding. It does not fall back to an independently managed network.
The label is an integration contract, not sufficient evidence by itself: the
namespace-owner role must apply it only to a container whose firewall, DNS,
capability, and lifecycle settings implement that contract.

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
`.` or `..` path segments or control characters. They cannot shadow `/config`,
`/etc`, `/run`, `/proc`, `/sys`, `/dev`, or overlap the configured download
mount. Additional bind sources have the same canonical absolute-path
requirement and are resolved before rendering to reject current symlink
traversal; named-volume sources remain supported. Ordinary application-data
mounts remain supported.

The primary config and download sources receive the same fail-closed path
checks. They cannot point at `/run`, container-engine state or sockets, or
other obvious host-control trees. The download destination must not overlap
`/config`, `/etc`, `/run`, `/proc`, `/sys`, `/dev`, or the container root.
Container mode also rejects UID or GID `0`.

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

Sharing a network namespace also shares loopback and the namespace owner's
interfaces. qBittorrent can therefore reach listeners which the owner binds to
loopback, including the explicitly supplied DNS service and any health or
control listener. Approved reverse-proxy ingress reaches the shared namespace
through a bridge attached to the owner. The owner itself also needs narrowly
defined VPN-endpoint/bootstrap traffic. These are deliberate local or
owner-originated exceptions to “no Internet connectivity”; they must be
enumerated and protected by the owner contract. In particular, a compromised
qBittorrent process may attempt to abuse a bootstrap exception, so this design
does not claim universal non-exfiltration against a malicious application.

The lifecycle guarantee assumes qBittorrent is controlled by the rendered
systemd unit. Containers started, renamed, or otherwise managed directly with
Docker are outside this role's lifecycle boundary. Runtime acceptance tests
must include a negative direct-command case and confirm that such unmanaged
state is reported as outside the supported guarantee.

As a compatibility and command-targeting safeguard,
`qbittorrent_identifier` must not consist entirely of lowercase hexadecimal
characters. Docker accepts hexadecimal strings as container-ID prefixes, so
such a name cannot be used safely in every lifecycle command. This applies to
both install and uninstall and is a breaking restriction for an existing
custom identifier such as `deadbeef`: the role fails validation before stopping
or deleting anything. Rename or migrate such an installation explicitly before
using this version. A managed network name is subject to the same restriction
when role-driven network deletion is enabled.

### Switching modes and rollback

Configuration validation runs before installation or migration. If a requested
VPN configuration is invalid, the role makes no isolation change and the
currently deployed mode may remain running. This is a rejected target
configuration, not a VPN-runtime failure. The fail-closed transition begins
only after validation succeeds and the installation task reaches its migration
gate.

Both `managed` (direct bridge) to `container` and `container` to `managed`
transitions use a persistent systemd quiesce. Before stopping anything, the
role places this guard at
`/etc/systemd/system/<identifier>.service.d/zzzz-mash-fail-closed-quiesce.conf`,
reloads systemd, and proves the effective unit has `RefuseManualStart=yes`,
`Restart=no`, an inert start command, and name-free stop commands. It then
stops the unit, removes only the captured and authorized full container ID, and
proves that both the ID and stable name are absent. A later same-name
replacement is never substituted as a cleanup target; it is retained and makes
convergence fail.

Authorization normally comes from the fixed role-ownership label. To migrate
an older unlabelled role container, the role also has a narrow semantic legacy
check covering the image repository, UID/GID, automatic removal, dropped
capabilities, non-privileged execution, non-default networking, and the
configured config/download mounts. This check compares the old container with
the requested settings. Consequently, make the network-mode transition the
only lifecycle-significant change on the first migration run: do not
simultaneously change UID/GID, image repository, data/download source paths, or
the download destination. An image tag change within the same repository is
accepted. An unclassifiable legacy container is retained and fails safely;
after a successful migration, the ownership label removes this one-change
limitation.

The final target unit is rendered while the quiesce remains installed. The role
then removes the guard, reloads systemd, and verifies the effective
mode-specific start and stop lifecycle exactly. If release or verification
fails normally, it restores and re-verifies the guard. A later task failure
after completed quiescing therefore leaves qBittorrent stopped and absent, with
config and downloads untouched. A later run recognizes and adopts the reserved
role-owned guard, including when the main unit was already removed, and retries
the requested transition. After abrupt controller or host interruption, rerun
the role and verify the effective unit state rather than inferring which
individual lifecycle operation completed.

On a fresh `container`-mode install with no prior unit there is nothing to
quiesce. The rendered unit's `After=`/`BindsTo=` relationship and ID-validating
health gate defer qBittorrent creation until the owner exists and is healthy.

Uninstall performs the same independent control-plane checks: it applies the
systemd quiesce, removes only a captured full ID authorized by the ownership
label or semantic legacy check, proves the unit inactive and both the captured
ID and stable name absent, and only then removes the unit and root-owned helper.
A later replacement is never a cleanup target. Config, downloads, and any
unlabelled or foreign managed network retain the role's non-destructive
behavior.

To roll back, restore `qbittorrent_container_network_mode: managed`, remove the
namespace-owner setting and bound owner service, restore any desired managed
network or port settings, and run the role again. Once the validated transition
gate is reached, the protected container is quiesced and removed; a failure
does not leave it running in the old namespace. It remains Internet-isolated
and unavailable behind the persistent guard until a retry completes. The
managed bridge is prepared before the final unit is released. Starting that
unit is an intentional downgrade of the role's VPN enforcement, because
qBittorrent then regains independently managed network connectivity.

### Diagnostics during fail-closed states

There are two intentionally different diagnostic states:

- If the namespace owner remains running while its tunnel is down, the owner's
  firewall must block Internet egress. qBittorrent may remain running so its
  local or approved reverse-proxy diagnostics can remain available.
- If the owner service stops, or a transition/uninstall is interrupted,
  qBittorrent is stopped and its container is absent. Its Web UI is therefore
  unavailable by design. Diagnose through the qBittorrent and owner systemd
  status/journal, the effective unit and drop-in paths, and read-only Docker
  inspection. Do not remove the reserved quiesce file manually as a recovery
  shortcut; correct the target configuration or unit problem and rerun the
  role so release is verified transactionally.

## Limitations

Docker does not permit assigning qBittorrent its usual stable container hostname
in `container` network mode. Treat unclean shutdown and stale Qt lock-file
recovery as a runtime acceptance gate for this mode: the selected namespace
owner lifecycle must prove recovery, or an explicit lock-file mitigation must
be designed, before production use.

The role-local `container_network` Molecule scenario defines exact owner
identity and contract-label checks, immutable-ID cleanup, systemd stop coupling,
owner recreation, direct↔container transitions, persistent-quiesce retry, and
refusal to delete an untracked same-name replacement. Its owner is synthetic:
it does not establish a VPN tunnel or firewall and therefore does not test VPN
enforcement. External lifecycle tests must still prove owner absence, unhealthy
and stopped owner, tunnel failure with the owner still running, same-host peer
access, and an actual VPN-backed rollback before this mode is treated as
production-ready. They must also exercise the documented direct-command
boundary rather than treating an unmanaged container as role-controlled state.

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
