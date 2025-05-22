# ─────────────────────────────────────────────────────────────────────────────
#  Copilot guidance – “cloudkey-ansible”
# ─────────────────────────────────────────────────────────────────────────────
#
#  Goal
#  ────
#  Build an Ansible project that turns a modified and repurposed UniFi CloudKey-Gen 2 Plus
#  (Debian 8 Jessie, root-only, password auth) into a clean Debian 12 Bookworm
#  host with:
#    • proper SSH key-based access (root + user “red”)
#    • SSD (/dev/sda) formatted, mounted at /home
#    • user “red” created with home on the SSD, supplied public-key, zsh later
#    • in-place distro upgrades: 8→9→10→11→12 (reboot & wait each step)
#    • chezmoi init for user red (run as root with sudo) **after** upgrade
#
#  Repository layout to GENERATE
#  ─────────────────────────────
#  .
#  ├── inventory                       # static inventory with CloudKey IP/FQDN - Single host
#  ├── site.yml                        # master play applying roles in order
#  ├── playbooks/
#  │   └── files/
#  │       └── root_id_ed25519         # generated once by bootstrap
#  │       └── root_id_ed25519.pub
#  └── roles/
#      ├── bootstrap/
#      │   ├── tasks/main.yml          # install python3 & openssh-server +
#      │   │                           # generate 4096-bit ed25519 keypair on
#      │   │                           # the *control* host, copy to root’s
#      │   │                           # ~/.ssh/, set 600 perms
#      │   └── files/                  # (keypair lives in playbooks/files)
#      ├── ssh_hardening/
#      │   ├── tasks/main.yml          # deploy sshd_config, handler restart
#      │   └── templates/sshd_config.j2
#      ├── storage/
#      │   └── tasks/main.yml          # single GPT part on /dev/sda,
#      │                               # mkfs.ext4, fstab entry → /home
#      ├── users/
#      │   └── tasks/main.yml          # create user red, group wheel/sudo,
#      │                               # home=/home/red, authorized_keys
#      ├── upgrade/
#      │   ├── tasks/main.yml          # include per-release tasks
#      │   ├── tasks/stage-stretch.yml # 9  (Jessie → Stretch)
#      │   ├── tasks/stage-buster.yml  # 10
#      │   ├── tasks/stage-bullseye.yml# 11
#      │   └── tasks/stage-bookworm.yml# 12  ← stop here
#      │   └── templates/
#      │       └── sources-{{ release }}.list.j2
#      └── chezmoi/
#          └── tasks/main.yml          # become: red; run_once; chezmoi init
#
#  Play order (site.yml)
#  =====================
#    - bootstrap           # python + ssh; generates new root keypair
#    - ssh_hardening       # push config; uses root key
#    - storage             # prep SSD
#    - users               # user red; create, home on SSD ; import existing ssh key
#    - upgrade             # 8→…→12 ; tag=upgrade
#    - chezmoi             # tag=chezmoi
#
#  Key behavioural rules
#  ─────────────────────
#  • All tasks idempotent.
#  • Reboots: use `reboot:` module + `wait_for_connection` (300 s timeout).
#  • sshd_config:
#      PasswordAuthentication no
#      PermitRootLogin prohibit-password
#      AllowUsers root red
#  • After bootstrap, subsequent plays must connect with the new root key
#    (`ansible_ssh_private_key_file=playbooks/files/root_id_ed25519`).
#  • Storage: single GPT partition on /dev/sda, ext4, mount at /home.
#  • Users: create user red, group wheel/sudo, home=/home/red, authorized_keys should NOT use the key that was generated for root, but import an existing key.
#  • Upgrades: render sources.list template for each release, then
#      apt-get update && apt-get -y upgrade && apt-get -y dist-upgrade
#      reboot
#  • Stop after Bookworm; do NOT attempt usrmerge / Debian testing.
#  • chezmoi init: run as root with sudo, but on behalf of/for the user red.
#
#  Thanks!  Use the structure above exactly.
#  External references:
#  - https://xdaforums.com/t/unifi-cloud-key-gen-2-plus.4664639/
#  - https://colincogle.name/blog/unifi-cloud-key-rescue/
#  - https://fullduplextech.com/turn-unifi-cloud-key-gen-2-into-a-headless-linux-server/
#  - https://raw.githubusercontent.com/jmewing/uckp-gen2/main/reinstall.sh
# ─────────────────────────────────────────────────────────────────────────────