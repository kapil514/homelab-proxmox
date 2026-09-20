Proxmox Custom LXC Template Builder

This repository contains the Standard Operating Procedure (SOP) for creating, sanitizing, and publishing custom Linux LXC images (e.g., Rocky Linux 10) on Proxmox VE.

By following this guide, you can create pre-configured templates with essential software and SSH keys that appear natively in the Proxmox 'Create CT' Web GUI menu across all nodes.

Prerequisites

Root access to a Proxmox VE node.

A public RSA SSH key ready for injection.

Basic familiarity with Proxmox CLI (pct and vzdump commands).

Phase 1: Build the Base Container

Create a standard LXC container (e.g., Rocky Linux 10) using the Proxmox UI or CLI. Assign a unique ID, such as 100. Do not start the container immediately after creation.

Start the container from the host CLI to begin configuration:

pct start 100


Update packages and install necessary software (OpenSSH, vim, git, etc.):

pct exec 100 -- dnf update -y
pct exec 100 -- dnf install -y openssh-server vim git


Enable the SSH daemon to ensure connectivity upon deployment:

pct exec 100 -- systemctl enable sshd


Phase 2: Inject SSH Key & Sanitize

Create the SSH configuration directory for the root user:

pct exec 100 -- mkdir -p /root/.ssh


Add your public SSH key to the authorized keys file (replace the placeholder string with your actual public key):

pct exec 100 -- bash -c "echo 'ssh-rsa AAAAB3NzaC1... user@host' > /root/.ssh/authorized_keys"


Set secure permissions for the SSH directory and key file:

pct exec 100 -- chmod 700 /root/.ssh
pct exec 100 -- chmod 600 /root/.ssh/authorized_keys


Sanitize the container to strip unique identifiers. This is critical to prevent network and SSH conflicts when the template is cloned into multiple new containers:

pct exec 100 -- dnf clean all
pct exec 100 -- rm -f /etc/ssh/ssh_host_*
pct exec 100 -- truncate -s 0 /etc/machine-id


Phase 3: Package the Template

Stop the base container to ensure file system consistency before packaging:

pct stop 100


Generate a highly compressed backup archive using the vzdump utility:

vzdump 100 --compress zstd


Note: The default output path is typically /var/lib/vz/dump/vzdump-lxc-100-*.tar.zst.

Phase 4: Publish to the 'Create CT' Menu

Move and rename the generated archive to the Proxmox template cache directory. Using a clean, descriptive name helps identify the template easily in the GUI:

mv /var/lib/vz/dump/vzdump-lxc-100-*.tar.zst /var/lib/vz/template/cache/custom-rocky10-ssh-amd64.tar.zst


Deploy to other nodes (Optional): To make the template available across a cluster, use scp to copy the file to the same template cache directory on other Proxmox nodes.

scp /var/lib/vz/template/cache/custom-rocky10-ssh-amd64.tar.zst root@<TARGET_NODE_IP>:/var/lib/vz/template/cache/


Verification: Open the Proxmox Web Interface, click Create CT, navigate to the Template tab, and verify that custom-rocky10-ssh-amd64.tar.zst is available in the selection menu.
