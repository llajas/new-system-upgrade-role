# ─────────────────────────────────────────────────────────────────────────────
#  Copilot guidance – “cloudkey-ansible” (revised)
# ─────────────────────────────────────────────────────────────────────────────
#
#  Goal
#  ────
#  Turn a UniFi CloudKey-Gen2Plus (Debian 8 Jessie, root/password) into a
#  clean Debian 12 Bookworm headless host with:
#    • in-place dist-upgrades: 8→9→10→11→12 (reboot & wait each step - 8-11 done via raw commands, then python 3.9 is installed on 11 before upgrading to 12)    
#    • SSH key–only access (root + a custom user)
#    • SSD (/dev/sda) formatted, mounted at /home
#    • custom user on the SSD, public-key, zsh later
#    • chezmoi init for custom user after all upgrades
#
#  Repository layout
#  ─────────────────
#  ─────────────────
#  .
#  ├── inventory
#  ├── README.md
#  ├── site.yml
#  ├── ssh_keys
#  │   ├── root_ed25519
#  │   └── root_ed25519.pub
#  └── roles/
#      ├── bootstrap/           ← handles Jessie→Stretch→Buster then pivots from raw requests to using a python interpreter as ansible needs python 3.7 which isn’t available until we’re on the buster release
#      │   ├── handlers/
#      │   │   └── main.yml
#      │   └── tasks/
#      │       └── main.yml
#      ├── chezmoi/
#      │   └── tasks/
#      │       └── main.yml
#      ├── ssh_hardening/
#      │   ├── handlers/
#      │   │   └── main.yml
#      │   ├── tasks/
#      │   │   └── main.yml
#      │   └── templates/
#      │       └── sshd_config.j2
#      ├── storage/
#      │   └── tasks/
#      │       └── main.yml
#      └── users/
#          └── tasks/
#              └── main.yml
#
#  Play order (site.yml)
#  =====================
#
#  ● Pre-Flight: Check if key-auth already works on the remote host and skip bootstrap if key is already installed.
#
#  ● Phase 1: bootstrap → Update the remote OS until we can get Python 3.9+ on the remote
#  
#    Start with a bootstrap role that upgrades the OS to Bullseye (raw Jessie→Stretch→Buster→Bullseye)
#    using raw Ansible due to lack of python with ansible-core support.
#    Once Python 3.9 (and OpenSSH) is available from Bullseye, install it and then pivot to the Ansible Python interpreter
#    to leverage the normal Ansible modules for the rest of the OS upgrades and the remainder of the playbook.
#
#    Then proceed with ssh_hardening role. We check for existing SSH keys,
#    create the root SSH keypair on the control node if it doesn't exist,
#    and push the public key to the remote host, adding it to the authorized_keys file for root.
#    A hardened sshd_config is then pushed to the remote host.
#
#  ● Phase 2: Once ssh key auth works, we pivot to using the root ssh private key for further actions.
#    We start by installing parted and provisioning the SSD with LVM after turning off any swap on the device, clearing it and creating a filesystem on it, one large partition.
#    Then we mount it at /home.
#
#    Next, we create a new custom user with an appropriate home directory on the SSD
#    We then add the existing public key on the control node to the authorized_keys file for that user.
#    We also set the default shell to zsh and then pull chezmoi down to the remote host.
#    Since our new user is not root, we need to use the become directive to run the Chezmoi tasks as root on behalf of the custom user.
#    We run 'chezmoi init' for the custom user as root, which will create the .local/share/chezmoi directory in the user's home directory.
#
#  Key points
#  ──────────
#  • Phase 1 uses only raw/dist-upgrade + reboot to reach an OS with Python 3.9+
#  • After reboot into Bullseye, we set `ansible_python_interpreter: /usr/bin/python3`
#  • The rest of Phase1 as well as Phase 2 can then use all the normal `apt:`, `file:`, `authorized_key:` modules
#  • Custom user is created + SSH key imported in the `users/` role
#  • Storage is handled in `storage/`, etc.
# ─────────────────────────────────────────────────────────────────────────────
