# kasm-vm-template

Builds the Proxmox VMs [Kasm Workspaces](https://www.kasmweb.com/) needs, in
two separate ways:

- **`kasm_templates`: agent templates.** Docker hosts, converted to Proxmox
  templates, for Kasm's Proxmox autoscale provider to clone.
- **`kasm_servers`: Linux desktop servers.** Full desktop VMs for Kasm "server"
  connections (RDP or VNC). They are **not** templates and are not autoscaled:
  the role builds and provisions each one and leaves it **running**, and you add
  it to Kasm by hand (Infrastructure > Servers).

It runs in three phases (`kasm_template_phase`: `build`, `provision`,
`finalize`), so the playbook calls it three times; see `kasm-template.yml` and
`kasm-server.yml` in [homelab-ansible](https://github.com/dpisano/homelab-ansible)
for working examples, including the `kasm_build` group the provision phase
targets.

## What it does

1. **On the Proxmox node** (`[pve]`): downloads the cloud image, creates the
   VM, grows the disk and boots it with a cloud-init snippet that installs
   `qemu-guest-agent` and authorizes your SSH key. The VM's address comes from
   the guest agent.
2. **Over SSH**, jumping through the node: upgrades the OS, then installs
   - agent template: Docker (Docker's repo on Debian/Ubuntu, `docker` on Arch),
     for Kasm's Proxmox autoscale provider. Optional
     `kasm_template_prepull_images`.
   - server: XFCE + xrdp (Debian/Ubuntu), or the Omarchy / Pop!_OS desktop with
     wayvnc, as below.
3. **Back on the node**:
   - **template:** cleans the guest (package cache, cloud-init state,
     machine-id, SSH host keys), shuts it down, drops the build-time user-data,
     and runs `qm template`. Clones use Proxmox's normal cloud-init settings
     (user, SSH key, network), so set those on the clone or in Kasm's
     autoscale config.
   - **server:** nothing is cleaned or converted: the VM keeps its machine-id,
     SSH host keys and cloud-init user-data and keeps running. For the ISO
     installs (Omarchy, Pop!_OS) the install media is detached and the VM is
     restarted once, so it boots from its disk. The run ends by printing the
     address and the connection settings to give Kasm.

## Servers

```yaml
kasm_servers:
  - { name: kasm-desktop-ubuntu-2404, vmid: 9110, os: ubuntu-2404, memory: 8192, ip: 192.168.1.50/24, gateway: 192.168.1.1 }
  - { name: kasm-desktop-omarchy, vmid: 9111, os: omarchy, memory: 8192, disk: 60 }
```

Keys are the template keys (`name`, `vmid`, `os`, `cores`, `memory`, `disk`,
`storage`, `bridge`, `vlan`, `pool`), plus `ip`, `gateway` and `dns` for a
static address on a cloud-image server. `kind` is always `desktop`.

- **Address.** Kasm connects to a server by address, so it must not change.
  Either set `ip` (cloud-image OSes), or give the VM a DHCP reservation by MAC;
  Omarchy's MAC is `BC:24:11:` plus the VMID as three hex bytes
  (VMID 9111 is `BC:24:11:00:23:97`).
  Omarchy and Pop!_OS take DHCP only.
- **Login.** On a cloud-image server the role creates the RDP login Kasm uses:
  user `kasm_server_login_user` (default `kasm`) with
  `kasm_server_login_password` (required; hashed on the controller and set only
  when the account is created, so change it later with `passwd` on the VM). It
  is a plain user without sudo; the cloud image's own user (`ubuntu`, `debian`)
  with your SSH key is the admin login. Omarchy and Pop!_OS log in with their own
  user and the wayvnc password.
- **Adding it to Kasm.** Infrastructure > Servers > Add: IP/Hostname = the
  address, Connection Type RDP and port 3389 (xrdp), or VNC and port 5900
  (wayvnc), Static Credentials with the login above. See the guide in the
  homelab repo for every field.
- **Re-running** is safe: a finished server is left alone (and its address is
  printed again); an unfinished one is provisioned again. `-e
  kasm_template_rebuild=true` **destroys** the VM at that VMID and builds it
  again.

## Supported OSes (`kasm_template_os`)

| key | image | agent template | desktop server |
|---|---|---|---|
| `ubuntu-2404`, `ubuntu-2204` | Ubuntu cloud image | yes | yes |
| `debian-12`, `debian-13` | Debian genericcloud | yes | yes |
| `arch` | Arch Linux cloud image | yes | no (xrdp is AUR-only) |
| `omarchy` | Omarchy ISO, unattended | no | yes (wayvnc) |
| `popos` | Pop!_OS ISO, installed by hand | no | yes (xrdp, or wayvnc for COSMIC) |

A `kasm_templates` entry can only be `kind: agent`; desktops go in `kasm_servers`.

To add an OS, add an entry to `kasm_template_os` with its image URL, default
user, and a command that installs the guest agent. If it needs a new package
manager, add `base-`, `docker-` and `clean-<family>.yml` task files.

## Omarchy

Omarchy ships only as an ISO (no cloud image, no install-on-Arch script), so
`os: omarchy` uses a different build: the node attaches the ISO plus a second
drive labeled `cidata` holding the installer's config, and Omarchy installs
itself unattended
([manual](https://github.com/basecamp/omarchy/blob/master/manual/51-unattended-installs.md)).
Then the role installs `qemu-guest-agent`, `cloud-init` and `wayvnc`; a server is left running and a template gets Proxmox's cloud-init drive and is converted.

Before the first run:

1. Do one interactive Omarchy install in a throwaway VM with a 40+ GB `scsi0`
   disk, encryption **off**, and copy `/root/user_configuration.json` to
   `files/omarchy/user_configuration.json` next to the playbook. The role refuses a config
   with `disk_encryption` (a LUKS prompt blocks every unattended boot).
2. Set `kasm_omarchy_version` to a release tag, plus
   `kasm_omarchy_password_hash` (`openssl passwd -6`) and
   `kasm_omarchy_vnc_password` from your secret store.

Things to know:

- The build VM's address is found by scanning the bridge's subnet for its MAC
  (installs `nmap` on the node); give the server `build_ip` to skip that.
  Port 22 answering is taken to mean the install finished.
- Omarchy's installer leaves the login without passwordless sudo, so set
  `kasm_omarchy_password` (the plaintext behind `kasm_omarchy_password_hash`);
  the role gives it to sudo while provisioning, and stops with a clear message
  if sudo still does not work.
- Omarchy is not upgraded during the build (use `omarchy-update`). Networking
  stays with NetworkManager (DHCP), so reserve its address by MAC. (As a
  template, set `kasm_omarchy_cloudinit_network: true` to let cloud-init apply
  a clone's static ipconfig instead; untested against NetworkManager.)
- Kasm connects as a **VNC server** to `kasm_omarchy_vnc_port` (5900) with the
  wayvnc username/password. wayvnc is started by a systemd user unit tied to
  the graphical session, so it needs Omarchy's autologin.
- Hyprland needs GPU acceleration or software rendering. The VM uses
  `--vga virtio`; set `vga: virtio-gl` on the template for virgl if the node
  has the GL libraries.

## Pop!_OS

System76 ships Pop!_OS only as an ISO, with no cloud image and no documented
unattended install, so `os: popos` has a manual step. It is `kind: desktop`
only, and the role runs twice:

1. Set `kasm_popos_iso_url` and `kasm_popos_iso_sha256` (link and SHA256 from
   [pop.system76.com](https://pop.system76.com)) and `kasm_popos_username`,
   then run. The role downloads and verifies the ISO, creates the VM with it
   attached (in `pool` if you set one) and starts it. It builds nothing else
   for that server yet.
2. In the VM's Proxmox console, install Pop!_OS with the user
   `kasm_popos_username` (disk encryption off), then in the new system run
   `sudo apt install -y qemu-guest-agent && sudo reboot`.
3. Run again. The role finds the VM, uses the guest agent (as root) to install
   sshd, authorize `kasm_template_ssh_public_key` and allow passwordless sudo
   for that user, then provisions it like the other desktops: remote access
   (below) and a clean-up. The server is left running (the install ISO is
   detached and the VM restarted once).

Things to know:

- **Pop!_OS 22.04** (GNOME-based) is reached over RDP with xrdp and a
  `gnome-session` login (`kasm_popos_session_command`).
- **Pop!_OS 24.04** uses COSMIC, which is Wayland-only and has no xrdp session,
  so it gets wayvnc and Kasm connects as a **VNC server** to
  `kasm_popos_vnc_port` (5900) with `kasm_popos_vnc_user` /
  `kasm_popos_vnc_password` (required). This is best effort: it needs COSMIC's
  compositor to allow screen capture and input injection, which is unconfirmed.
  Turn on automatic login in COSMIC's settings during the install so the
  session (and wayvnc, started by an autostart entry) is up after a boot.
  `kasm_popos_remote` forces `xrdp` or `wayvnc`; the default `auto` picks
  wayvnc when `cosmic-session` is installed.
- A VM that exists but is not yet finished is taken as the install to
  continue with (see "Resuming"). `-e kasm_template_rebuild=true` destroys it
  and starts over.
- The passwordless-sudo file stays on the server, as on the cloud images.
- It keeps NetworkManager's DHCP, so reserve its address by MAC.
- Untested on a real node: the xrdp login to Pop's GNOME session, and wayvnc on COSMIC.

Qubes OS is not supported: it is a Xen-based OS with its own installer, no
cloud image and no cloud-init, so it can't be cloned and provisioned this way.

## Resuming

A run can be repeated, and it carries on from where an earlier one stopped. For
each template or server the role looks at the VM at its VMID:

| VM at the VMID | What the next run does |
|---|---|
| none | builds it |
| already a template | skips it (an error for a server) |
| tagged `kasm-provisioned`, not yet a template | template: shuts it down, fixes the drives, converts it. Server: detaches leftover install media (restarting it), makes sure it is running, prints its address |
| tagged `kasm` only (provisioning did not finish) | starts it if needed, finds its address and provisions it again |
| no `kasm` tag (not made by this role) | stops with an error |

The `kasm-provisioned` tag (`kasm_template_provisioned_tag`) is set on the VM as
the last step of provisioning, so it only exists once everything succeeded.
Finalizing checks each step against the VM's config, so it also resumes after
failing partway (a drive already detached, the VM already shut down). VMs that
did not finish provisioning are left alone by the finalize phase instead of
being converted. Provisioning again is safe for the cloud-image templates; for
Omarchy, a run that stopped during the install itself should be rebuilt. VMs
built before this tag existed are treated as unfinished and provisioned again.

## Variables

See `defaults/main.yml`. The VMs themselves are the `kasm_templates` and
`kasm_servers` lists in the playbook.

Set `kasm_template_no_log: true` to hide the output of tasks that handle the
SSH key / cloud-init data (default `false`).

## Notes

- Needs the private key matching `kasm_template_ssh_public_key` (default
  `users[0].ssh_key` from your inventory's `group_vars/all.yml`) on the controller.
- `-e kasm_template_rebuild=true` destroys whatever is at the VMIDs (a
  template, a server, a half-built VM, or a VM this role did not create) and
  builds it again. A plain run never does that; see "Resuming" below.
- The Kasm install and the Proxmox provider config in Kasm are out of scope.
