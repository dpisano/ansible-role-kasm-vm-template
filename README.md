# kasm-vm-template

Builds Proxmox VM templates for [Kasm Workspaces](https://www.kasmweb.com/)
from upstream cloud images. It runs in three phases (`kasm_template_phase`:
`build`, `provision`, `finalize`), so the playbook calls it three times; see
`kasm-template.yml` in [homelab-ansible](https://github.com/dpisano/homelab-ansible)
for a working example, including the `kasm_templates` list and the
`kasm_build` group the provision phase targets.

## What it does

1. **On the Proxmox node** (`[pve]`): downloads the cloud image, creates the
   VM, grows the disk and boots it with a cloud-init snippet that installs
   `qemu-guest-agent` and authorizes your SSH key. The VM's address comes from
   the guest agent.
2. **Over SSH**, jumping through the node: upgrades the OS, then installs
   - `kind: agent`: Docker (Docker's repo on Debian/Ubuntu, `docker` on Arch),
     for Kasm's Proxmox autoscale provider. Optional
     `kasm_template_prepull_images`.
   - `kind: desktop`: XFCE + xrdp, for Kasm "server" (RDP) connections.
3. **Back on the node**: cleans the guest (package cache, cloud-init state,
   machine-id, SSH host keys), shuts it down, drops the build-time user-data,
   and runs `qm template`.

Clones use Proxmox's normal cloud-init settings (user, SSH key, network), so
set those on the clone or in Kasm's autoscale config.

## Supported OSes (`kasm_template_os`)

| key | image | agent | desktop |
|---|---|---|---|
| `ubuntu-2404`, `ubuntu-2204` | Ubuntu cloud image | yes | yes |
| `debian-12`, `debian-13` | Debian genericcloud | yes | yes |
| `arch` | Arch Linux cloud image | yes | no (xrdp is AUR-only) |
| `omarchy` | Omarchy ISO, unattended | no | yes (wayvnc) |
| `popos` | Pop!_OS ISO, installed by hand | no | yes (xrdp) |

To add an OS, add an entry to `kasm_template_os` with its image URL, default
user, and a command that installs the guest agent. If it needs a new package
manager, add `base-`, `docker-` and `clean-<family>.yml` task files.

## Omarchy

Omarchy ships only as an ISO (no cloud image, no install-on-Arch script), so
`os: omarchy` uses a different build: the node attaches the ISO plus a second
drive labeled `cidata` holding the installer's config, and Omarchy installs
itself unattended
([manual](https://github.com/basecamp/omarchy/blob/master/manual/51-unattended-installs.md)).
Then the role installs `qemu-guest-agent`, `cloud-init` and `wayvnc`, and templates the VM with Proxmox's cloud-init drive attached.

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
  (installs `nmap` on the node); give the template `build_ip` to skip that.
  Port 22 answering is taken to mean the install finished.
- Omarchy is not upgraded during the build (use `omarchy-update`). Clones
  get their hostname, user and SSH key from cloud-init, but share the
  template's wayvnc certificate. Networking stays with NetworkManager (DHCP);
  set `kasm_omarchy_cloudinit_network: true` to let cloud-init apply static
  ipconfig instead (untested against NetworkManager).
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
   for that template yet.
2. In the VM's Proxmox console, install Pop!_OS with the user
   `kasm_popos_username` (disk encryption off), then in the new system run
   `sudo apt install -y qemu-guest-agent && sudo reboot`.
3. Run again. The role finds the VM, uses the guest agent (as root) to install
   sshd, authorize `kasm_template_ssh_public_key` and allow passwordless sudo
   for that user, then provisions it like the other desktops: xrdp with a
   `gnome-session` login, cloud-init for clones, a clean-up, and `qm template`.

Things to know:

- Use the GNOME-based **Pop!_OS 22.04**. COSMIC (24.04+) is Wayland-only and
  has no xrdp session; the role stops with a message if `gnome-session` is
  missing (set `kasm_popos_session_command` for another X11 session).
- A VM that exists but is not yet a template is taken as the install to
  continue with. `-e kasm_template_rebuild=true` destroys it and starts over.
- The passwordless-sudo file stays in the template, as on the cloud images.
- Clones keep NetworkManager's DHCP; `kasm_popos_cloudinit_network: true` lets
  cloud-init apply a clone's static ipconfig instead (untested).
- Untested on a real node: the xrdp login to Pop's GNOME session.

Qubes OS is not supported: it is a Xen-based OS with its own installer, no
cloud image and no cloud-init, so it can't be cloned and provisioned this way.

## Variables

See `defaults/main.yml`. The templates themselves are the `kasm_templates` list
in the playbook (`kasm_templates`).

Set `kasm_template_no_log: true` to hide the output of tasks that handle the
SSH key / cloud-init data (default `false`).

## Notes

- Needs the private key matching `kasm_template_ssh_public_key` (default
  `users[0].ssh_key` from your inventory's `group_vars/all.yml`) on the controller.
- A leftover half-built VM at a VMID aborts the run; destroy it or pass
  `-e kasm_template_rebuild=true`, which destroys any VM at those VMIDs.
- The Kasm install and the Proxmox provider config in Kasm are out of scope.
