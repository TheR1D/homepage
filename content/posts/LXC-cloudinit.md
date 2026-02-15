+++
date = '2026-02-15T12:00:00+01:00'
draft = false
title = 'Cloud-Init for Proxmox LXC Containers'
description = "A practical guide to enabling cloud-init in Proxmox LXC containers and automating provisioning with Terraform."
tags = ["proxmox", "terraform", "cloud-init", "lxc", "homelab"]
image = "images/posts/proxmox-lxc-cloudinit.png"
+++

Proxmox VE has excellent cloud-init support for virtual machines, but **LXC containers are a different story**. There's no native cloud-init integration for containers, which makes automated provisioning challenging. While working on my [homelab](https://github.com/TheR1D/homelab) infrastructure, I found a workaround: pre-baking cloud-init configuration directly into LXC templates.

This approach lets you use the same cloud-init workflows you'd use with VMs-user creation, SSH keys, package installation, custom scripts but for containers.

## How It Works

The solution is straightforward. LXC templates from [Linux Containers](https://images.linuxcontainers.org/images/) are just compressed tarballs containing a root filesystem. Many of these images come with cloud-init pre-installed. We can:

1. Download a cloud-ready template
2. Inject our cloud-init configuration files
3. Remove the disable flag (if present)
4. Recompress and use as a new template

Cloud-init uses the [NoCloud datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html) when it finds seed files at boot. We need two files inside the container's filesystem:

- `/var/lib/cloud/seed/nocloud/user-data` - Cloud-init configuration (users, packages, scripts)
- `/var/lib/cloud/seed/nocloud/meta-data` - Instance metadata (hostname, instance-id)

Additionally, we must ensure `/etc/cloud/cloud-init.disabled` doesn't exist, as this file prevents cloud-init from running.

## The Repacking Script

I wrote a bash script [lxcc.sh](https://gist.github.com/TheR1D/88a57f90f55331364bf6fb2bc962e00c) to automate the template repacking process:

```bash
#!/usr/bin/env bash
set -e

if [ "$#" -ne 4 ]; then
  echo "Usage: $0 <input.tar.xz> <user-data> <meta-data> <output.tar.xz>"
  exit 1
fi

INPUT="$1"
USER_DATA="$2"
META_DATA="$3"
OUTPUT="$4"

# Select tar implementation (GNU tar required)
if [[ "$(uname)" == "Darwin" ]]; then
  TAR="gtar"
else
  TAR="tar"
fi

if ! $TAR --version 2>/dev/null | grep -q "GNU tar"; then
  echo "GNU tar required. On macOS: brew install gnu-tar"
  exit 1
fi

TMP_TAR="$(mktemp).tar"

# Decompress
xz -dc "$INPUT" > "$TMP_TAR"

# Remove cloud-init.disabled if present
if $TAR -tf "$TMP_TAR" | grep -q '^./etc/cloud/cloud-init.disabled$'; then
  $TAR --delete -f "$TMP_TAR" ./etc/cloud/cloud-init.disabled
fi

# Inject user-data and meta-data
$TAR --owner=0 --group=0 \
  --transform='s|^.*$|var/lib/cloud/seed/nocloud/user-data|' \
  -rf "$TMP_TAR" "$USER_DATA"

$TAR --owner=0 --group=0 \
  --transform='s|^.*$|var/lib/cloud/seed/nocloud/meta-data|' \
  -rf "$TMP_TAR" "$META_DATA"

# Recompress and cleanup
xz -z -c "$TMP_TAR" > "$OUTPUT"
rm -f "$TMP_TAR"

echo "Created: $OUTPUT"
```

> **Note:** On macOS, you'll need GNU tar (`brew install gnu-tar`) since BSD tar doesn't support the `--transform` and `--delete` options.

### Usage

```bash
./lxcc.sh ubuntu-noble-cloud.tar.xz user-data.yml meta-data.yml output.tar.xz
```

**Arguments:**
- `input.tar.xz` - Original template from linuxcontainers.org
- `user-data` - Your cloud-init configuration
- `meta-data` - Instance metadata
- `output.tar.xz` - Repacked template ready for Proxmox

## Example Cloud-Init Configuration

Here's a practical `user-data` example:

```yaml
#cloud-config
hostname: my-container
timezone: Europe/London

# Security
ssh_pwauth: false
disable_root: true

users:
  - name: admin
    groups: [sudo]
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-key-here

packages:
  - openssh-server
  - curl
  - htop
  - vim

# Configure firewall
runcmd:
  - ufw allow from 192.168.1.0/24 to any port 22 proto tcp
  - ufw --force enable
```

And a minimal `meta-data`:

```yaml
instance-id: my-container
local-hostname: my-container
```

After repacking, import the template to Proxmox and create containers from it. Cloud-init runs automatically on first boot.

## Terraform Integration

You can automate template preparation with Terraform using `local-exec` provisioners:

```hcl
variable "container_name" {
  default = "web-server"
}

locals {
  template_dir = "${path.root}/.terraform/lxc-templates"
}

resource "local_file" "user_data" {
  filename = "${local.template_dir}/${var.container_name}-user-data"
  content  = <<-EOT
    #cloud-config
    hostname: ${var.container_name}
    users:
      - name: admin
        groups: [sudo]
        shell: /bin/bash
        lock_passwd: true
  EOT
}

resource "local_file" "meta_data" {
  filename = "${local.template_dir}/${var.container_name}-meta-data"
  content  = "instance-id: ${var.container_name}\nlocal-hostname: ${var.container_name}\n"
}

resource "terraform_data" "repack_template" {
  triggers_replace = [
    local_file.user_data.content,
  ]

  provisioner "local-exec" {
    command = <<-EOT
      mkdir -p "${local.template_dir}"

      # Download base template if not cached
      BASE_TEMPLATE="${local.template_dir}/ubuntu-noble-cloud.tar.xz"
      if [ ! -f "$BASE_TEMPLATE" ]; then
        curl -fSL -o "$BASE_TEMPLATE" \
          "https://images.linuxcontainers.org/images/ubuntu/noble/amd64/cloud/20260212_07%3A42/rootfs.tar.xz"
      fi

      # Repack with cloud-init configuration
      ./lxcc.sh "$BASE_TEMPLATE" \
        "${local_file.user_data.filename}" \
        "${local_file.meta_data.filename}" \
        "${local.template_dir}/${var.container_name}-cloudinit.tar.xz"
    EOT
  }

  depends_on = [local_file.user_data, local_file.meta_data]
}

# Create LXC container from repacked template
resource "proxmox_lxc" "web_server" {
  # ...
  template = "${local.template_dir}/${var.container_name}-cloudinit.tar.xz"
  # ...
}
```
The full implementation is available in my [homelab repository](https://github.com/TheR1D/homelab). Feel free to adapt it for your own infrastructure.
