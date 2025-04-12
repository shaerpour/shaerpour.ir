---
title: "Deploy Minio Cluster with Ansible"
date: "2025-04-12"
description: "Fully initializing a minio multinode cluster with Ansible."
toc: true
draft: false
tags: ["Ansible", "Minio"]
---

## Introduction
In this post, we'll demonstrate how to deploy a scalable, multinode MinIO cluster using Ansible, creating a robust object storage solution suitable for businesses of any size.

First, let's take a closer look at these concepts:

## MinIO
MinIO is an open-source, high-performance object storage server that is fully compatible with the Amazon S3 (Simple Storage Service) API.

It is designed to handle unstructured data like photos, videos, backups, and logs. MinIO is highly scalable and secure, making it suitable for both personal use and enterprise-level applications that require reliable object storage.

## Ansible
Ansible is an open-source automation tool used to configure systems, manage deployments, and automate IT tasks like application installation, updates, and cloud provisioning.

It simplifies the automation of repetitive processes, enabling system administrators and DevOps teams to efficiently manage infrastructure at scale without complex setups.

## Requirements

{{< callout type="custom" emoji="📌" title="Note" text="Currently, this role only supports Debian-based systems. Contributions to make it more flexible and dynamic are greatly appreciated." style="background-color: transparent; border: 3px solid #00a7a7;" >}}

{{< callout type="alert" text="The deployment must be performed on servers without any Minio service installed, as it may cause configuration and data loss!" >}}

For our demonstration, we need at least 2 servers with 2 disks each, and Ansible already installed.

### Servers
I’ve already provisioned two Debian Bookworm servers in [Hetzner](https://console.hetzner.cloud/), each with 3 additional disks.

### Install Ansible
Installing ansible is easy. You can install it with your distro's package manager or creating a python virtual environment.

I prefer the second optoin since every python project can have different dependencies and versions.

```bash
# Install virtual environment module for python
sudo apt update && sudo apt install python3-venv

# Create and activate a venv called ansible
python3 -m venv ansible
source ansible/bin/activate

# Install Ansible in this environment
pip install --upgrade ansible
```

## Setup Cluster
Now it's time to roll up our sleeves and dive in. I wrote an Ansible Role that is available via this [Link](https://github.com/shaerpour/playbook.d/tree/main/minio_multinode/).
```bash
# Clone the project
git clone https://github.com/shaerpour/playbook.d.git && cd playbook.d/minio_multinode
```

In next step, edit **inventory.yml** file and add your servers to the list.
```bash
vim inventory.yml
```
```yaml
---
all:
  hosts:
    minio-1:
      ansible_host: ""
    minio-2:
      ansible_host: ""
```

{{< callout type="warning" text="Update roles/minio_multinode/vars/main.yml to avoid using default settings (Minio credentials and 2 disks per node)." >}}

```bash
vim roles/minio_multinode/vars/main.yml
```
```yaml
---
minio_multinode_root_user: "ahsp"                           # Default: minioadmin
minio_multinode_root_password: "BwHVuS6j4U03K6hzXfnXVMSp1"  # Default: minioadmin
minio_disk_count: 3                                         # Default: 2
```
As you can see, I have 3 disks per node and custom credentials for my cluster. Leave them comment if you want them with default values.

{{< callout type="tip" text="Minimum length for username is 3 and password is 8." >}}

```bash
# Run the main playbook to start initializing minio cluster.
# add --ask-become-pass if you are running playbook with
# non-privileged users.
ansible-playbook -i inventory.yml main.yml -u debian -K
```
Congrats! As the image below shows, We successfully initialized a minio cluster.

![Run Playbook Status](https://blog.shaerpour.ir/images/ansible/minio_multinode/run-playbook-status.png)

Open up your browser and paste one of your server's IP on port 9001. Your minio console will show up.

![Minio Login](https://blog.shaerpour.ir/images/ansible/minio_multinode/minio-login.png)

You can log in with the credentials you have configured and enjoy!

![Minio Panel](https://blog.shaerpour.ir/images/ansible/minio_multinode/minio-panel.png)
