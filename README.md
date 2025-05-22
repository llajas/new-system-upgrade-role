# CloudKey Ansible Playbook

This project automates the conversion of a repurposed UniFi CloudKey-Gen 2 Plus (Debian 8 Jessie, root-only, password auth) into a clean Debian 12 Bookworm host with:

- **Proper SSH key–based access** for both root and a user (default: `red`)
- **SSD setup**: formats `/dev/sda` with a single GPT partition and mounts it at `/home`
- **User creation**: Creates a user (configured via the `target_user` variable) with home on the SSD and imports existing SSH keys.
- **In-place distro upgrades**: sequentially upgrading through Debian releases (8 → 9 → 10 → 11 → 12) with reboots and wait for connection
- **chezmoi initialization**: runs as root (with sudo) on behalf of the user for post-upgrade configuration  
  (Uses a provided GitHub username to pull configuration)

## Repository Layout

```
.
├── inventory                       # Static inventory with CloudKey IP/FQDN and group variables
├── site.yml                        # Master play applying roles in order
├── playbooks/
│   └── files/
│       ├── root_id_ed25519         # Generated once by bootstrap
│       ├── root_id_ed25519.pub
│       └── *(other key files if needed)*
└── roles/
    ├── bootstrap/                  # Install python3, openssh-server, and generate root keypair
    ├── ssh_hardening/              # Deploy sshd_config (- PasswordAuthentication no, etc.)
    ├── storage/                    # Prepare/format SSD, set up /home mount point
    ├── users/                      # Create user (default: red) and install existing SSH key files
    ├── upgrade/                    # Perform sequential distro upgrades with templates & reboots
    └── chezmoi/                    # Initialize chezmoi for the target user (runs once)
```

## Requirements

- **Ansible 2.9+** (or newer)
- A *control host* running Ansible; the playbook is designed for use on Windows (via command prompt or PowerShell) in your workspace located at `d:\new-system-upgrade-role`
- A CloudKey device that meets the target requirements

## Configuration

Edit the `inventory` file to adjust target host and variables. Example:

```ini
[cloudkey]
goldenrod.johto.lajas.tech

[cloudkey:vars]
target_user=red
public_key_file=playbooks/files/red_id_ed25519.pub
private_key_file=playbooks/files/red_id_ed25519
ssh_key_suffix=id_ed25519
github_username=llajas
```

- **target_user**: The non-root user to be created (default: red)
- **public_key_file** and **private_key_file**: File paths to the SSH key files to be imported for the user
- **ssh_key_suffix**: Suffix for the private key file name (supports key types like RSA or Ed25519)
- **github_username**: GitHub username for chezmoi initialization

## Running the Playbook

### Before SSH Hardening

For the initial plays, password authentication is used. Supply the `ansible_password` via inventory or extra-vars. Example from the command prompt:

```cmd
ansible-playbook -i inventory site.yml --extra-vars "ansible_password=YourRootPassword"
```

### After SSH Hardening

Once the bootstrap and SSH hardening plays complete, subsequent tasks (storage, users, upgrade, chezmoi) pivot to key–based authentication using the generated root key (`playbooks/files/root_id_ed25519`).

The play order in `site.yml` is as follows:

1. **bootstrap** – Install required packages and generate root keypair
2. **ssh_hardening** – Deploy custom `sshd_config` (disabling password login)
3. **storage** – Partition, format, and mount SSD at `/home`
4. **users** – Create `red` (or specified user) on the SSD and install SSH key files
5. **upgrade** – Perform sequential distro upgrades (debian/stretch, buster, bullseye, bookworm) with reboots
6. **chezmoi** – Initialize chezmoi for the target user using GitHub repository configuration

## Logging & Idempotence

- All tasks have been written to be *idempotent*.
- Reboots are handled by the `reboot` module and waited for a 300-second timeout.
- The playbook uses DRY principles by templating configurations and looping over release stages.

## External References

- [UniFi CloudKey Gen 2 Plus on XDA Forums](https://xdaforums.com/t/unifi-cloud-key-gen-2-plus.4664639/)
- [Unifi CloudKey Rescue on Colin Cogle’s Blog](https://colincogle.name/blog/unifi-cloud-key-rescue/)
- [Headless Linux Server for UniFi CloudKey Gen2](https://fullduplextech.com/turn-unifi-cloud-key-gen-2-into-a-headless-linux-server/)
- [Reinstall Script on GitHub](https://raw.githubusercontent.com/jmewing/uckp-gen2/main/reinstall.sh)

## Notes

- Ensure that your public and private key files exist at the paths defined in your inventory.
- You may override any variable via extra-vars or by using group_vars/host_vars for further customization.
- The `chezmoi` task executes just once and checks for an existing configuration to avoid reinitialization.

Happy automating!