# ─────────────────────────────────────────────────────────────────────────────
#  Copilot guidance – “cloudkey-ansible” (revised)
# ─────────────────────────────────────────────────────────────────────────────
#
#  Goal
#  ────
#  Turn a UniFi CloudKey-Gen2Plus (Debian 8 Jessie, root/password) into a
#  clean Debian 12 Bookworm host with:
#    • SSH key–only access (root + user “red”)
#    • SSD (/dev/sda) formatted, mounted at /home
#    • user “red” on the SSD, public-key, zsh later
#    • in-place dist-upgrades: 8→9→10→11→12 (reboot & wait each step)
#    • chezmoi init for “red” after all upgrades
#
#  Repository layout
#  ─────────────────
#  .
#  ├── inventory
#  ├── site.yml
#  ├── playbooks/files/
#  │   └── root_id_ed25519{,.pub}
#  └── roles/
#      ├── bootstrap/           ← now only handles Jessie→Stretch→Buster
#      │   └── tasks/main.yml
#      ├── ssh_hardening/
#      ├── storage/
#      ├── users/
#      ├── upgrade/             ← Bullseye→Bookworm only
#      └── chezmoi/
#
#  Play order (site.yml)
#  =====================
#  
#  ● Phase 1: bootstrap → get Python 3.7+ on the remote
#  
#    - name: Bootstrap OS to Buster (raw Jessie→Stretch→Buster)
#      hosts: cloudkey
#      gather_facts: false
#      tasks:
#        - name: Point at Jessie → Stretch repos & puppy-grade apt.conf
#          raw: >-
#            cat > /etc/apt/apt.conf.d/99archive <<'EOF'
#            Acquire { Check-Valid-Until "false"; AllowInsecureRepositories "true"; };
#            EOF
#            cat > /etc/apt/sources.list <<'EOF'
#            deb http://archive.debian.org/debian           stretch        main contrib non-free
#            deb http://archive.debian.org/debian-security  stretch/updates main contrib non-free
#            EOF
#        - name: dist-upgrade to Stretch
#          raw: >-
#            apt-get update -o Acquire::Check-Valid-Until=false
#            DEBIAN_FRONTEND=noninteractive \
#            apt-get dist-upgrade -y \
#              -o Dpkg::Options::="--force-confdef" \
#              -o Dpkg::Options::="--force-confold"
#        - name: Reboot and wait
#          reboot:
#            reboot_timeout: 300
#        - name: Point at Buster & pull in Python 3 + sshd
#          raw: >-
#            cat > /etc/apt/sources.list <<'EOF'
#            deb http://archive.debian.org/debian           buster         main contrib non-free
#            deb http://archive.debian.org/debian-security  buster/updates main contrib non-free
#            EOF
#        - name: apt-get install Python 3 & openssh-server
#          raw: >-
#            apt-get update -o Acquire::Check-Valid-Until=false
#            DEBIAN_FRONTEND=noninteractive \
#            apt-get install -y --allow-unauthenticated \
#              python3 python3-apt openssh-server
#        - name: Set up Ansible’s interpreter to Python 3
#          set_fact:
#            ansible_python_interpreter: /usr/bin/python3
#
#  ● Phase 2: normal Ansible with f-strings
#
#    - name: ssh_hardening       # now we have Python 3 on the remote
#      hosts: cloudkey
#      gather_facts: yes
#      roles:
#        - ssh_hardening
#        - storage
#        - users
#        - { role: upgrade, tags: upgrade }
#        - { role: chezmoi, tags: chezmoi }
#
#  Key points
#  ──────────
#  • Phase 1 uses only raw/dist-upgrade + reboot to reach an OS with Python 3.7+  
#  • After reboot into Buster, we set `ansible_python_interpreter: /usr/bin/python3`  
#  • Phase 2 can then use all the normal `apt:`, `file:`, `authorized_key:` modules  
#  • The `upgrade/` role in Phase 2 only needs to handle Bullseye→Bookworm
#  • Reboots in upgrade stages should use `reboot:` + `wait_for_connection` (300 s)
#  • SSH keypair for root is still generated in bootstrap (Phase 2) and pushed  
#  • User “red” is created + SSH key imported in the `users/` role
#  • Storage is handled in `storage/`, etc.
# ─────────────────────────────────────────────────────────────────────────────
