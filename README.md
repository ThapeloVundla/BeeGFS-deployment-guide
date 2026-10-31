# BeeGFS 7.4.5 Cluster Deployment Guide (Rocky Linux 9)

This repository contains the end-to-end procedure and Ansible automation to deploy a BeeGFS parallel file system across a Rocky Linux 9 HPC cluster.

## 1. Cluster Architecture & Storage Layout

| Node Hostname | Role | BeeGFS Daemons | Mount Point |
|---|---|---|---|
| head | Management, Metadata, Storage & Client | beegfs-mgmtd, beegfs-meta, beegfs-storage, beegfs-helperd, beegfs-client | /mnt/beegfs |
| com1 | Compute Node (Client) | beegfs-helperd, beegfs-client | /mnt/beegfs |
| com2 | Compute Node (Client) | beegfs-helperd, beegfs-client | /mnt/beegfs |

Storage Partition: On single-drive training hardware, metadata and storage targets are hosted on `/dev/mapper/rl-home` (`/home`, 384GB available) rather than `/` (68GB available) to prevent root filesystem exhaustion.

### Ansible Inventory (`/etc/ansible/hosts`)

```ini
[headnode]
head ansible_connection=local

[compute]
com1
com2
```

## 2. Phase 1: Kernel & Header Alignment (Pre-Flight)

The `beegfs-client` service compiles a custom kernel module (`beegfs.ko`) dynamically on startup. If DNF installs a newer `kernel-devel` package than the kernel currently running in RAM, client compilation will fail with `Linux kernel build directory not found`.

Align the kernel and headers across all nodes before installing BeeGFS:

Install the latest kernel and matching development headers across the cluster:

```bash
ansible all -b -K -m shell -a "dnf update -y kernel && dnf install -y kernel-devel elfutils-libelf-devel"
```

Reboot the compute nodes via Ansible:

```bash
ansible compute -b -K -m reboot
```

(Note: Running reboot against all will fail on head because Ansible blocks rebooting the local control node mid-execution.)

Reboot the headnode manually:

```bash
sudo reboot
```

Reconnect and verify version alignment across all nodes:

```bash
ansible all -m shell -a "uname -r && rpm -q kernel-devel"
```

Ensure the running `uname -r` output matches the installed `kernel-devel` version string.

## 3. Phase 2: Headnode Server Deployment

Run these steps on `head` to configure the Management, Metadata, and Storage backend services.

### Step 1: Add the Verified BeeGFS 7.4.5 Repository

Do not use the generic `beegfs_7.4` URL, which returns a `302 Found` / `404 Not Found` HTML page and breaks DNF. Use the explicit 7.4.5 release path with `-L` to follow redirects:

```bash
sudo curl -L -o /etc/yum.repos.d/beegfs.repo https://www.beegfs.io/release/beegfs_7.4.5/dists/beegfs-rhel9.repo
```

Verify the file contains valid INI configuration (`[beegfs]`) and not HTML tags:

```bash
cat /etc/yum.repos.d/beegfs.repo
```

### Step 2: Install Server Daemons

```bash
sudo dnf install -y beegfs-mgmtd beegfs-meta beegfs-storage beegfs-utils
```

### Step 3: Initialize Server Directories

Initialize the management, metadata (`-s 1`), and storage (`-s 1 -i 101`) directories pointing to the management host (`-m head`):

```bash
sudo /opt/beegfs/sbin/beegfs-setup-mgmtd -p /home/beegfs/mgmt
sudo /opt/beegfs/sbin/beegfs-setup-meta -p /home/beegfs/meta -s 1 -m head
sudo /opt/beegfs/sbin/beegfs-setup-storage -p /home/beegfs/storage -s 1 -i 101 -m head
```

Competition Multi-Drive Note: When deploying across multiple physical drives (e.g., 8x 300GB drives mounted at `/data/disk1` through `/data/disk8`), run `beegfs-setup-storage` once per drive with a unique target ID (`-i 101`, `-i 102`, etc.) to stripe data across all disks.

### Step 4: Configure Connection Authentication

BeeGFS 7.4+ enforces cryptographic authentication and will immediately crash with `Unrecoverable error: No connAuthFile configured` if started without a shared secret key.

Generate the 128-byte shared secret key:

```bash
sudo dd if=/dev/urandom of=/etc/beegfs/connauthfile bs=128 count=1
```

Set temporary read permissions (`444`):

(Temporarily allow the non-root `cput` user to read the file so Ansible can copy it to the compute nodes in Phase 3 without a `Permission denied` error.)

```bash
sudo chmod 444 /etc/beegfs/connauthfile
```

Inject the key path into all server configuration files:

```bash
sudo sed -i 's|^connAuthFile.*|connAuthFile = /etc/beegfs/connauthfile|g' /etc/beegfs/beegfs-*.conf
```

### Step 5: Start and Verify Server Services

```bash
sudo systemctl enable --now beegfs-mgmtd beegfs-meta beegfs-storage
systemctl status beegfs-mgmtd beegfs-meta beegfs-storage
```

## 4. Phase 3: Cluster-Wide Client Deployment (Ansible)

Both `beegfs-client` and `beegfs-helperd` require the connection authentication key in their respective configuration files (`beegfs-client.conf` and `beegfs-helperd.conf`). Omitting `beegfs-helperd.conf` causes the helper daemon to fail and aborts the `/mnt/beegfs` mount with `mount(2) system call failed: Operation canceled`.

### Step 1: Create the Client Playbook

Create `deploy_beegfs_clients.yml` on `head`:

```bash
nano deploy_beegfs_clients.yml
```

Paste the following configuration:

```yaml
---
- name: Mount BeeGFS Across Cluster
  hosts: all
  become: yes
  tasks:
    - name: Distribute Repository
      copy:
        src: /etc/yum.repos.d/beegfs.repo
        dest: /etc/yum.repos.d/beegfs.repo

    - name: Install Client and Build Tools
      dnf:
        name:
          - kernel-devel
          - elfutils-libelf-devel
          - beegfs-client
          - beegfs-helperd
        state: present

    - name: Distribute Security Key
      copy:
        src: /etc/beegfs/connauthfile
        dest: /etc/beegfs/connauthfile
        owner: root
        group: root
        mode: '0400'

    - name: Initialize Client Pointing to Headnode
      command: /opt/beegfs/sbin/beegfs-setup-client -m head
      ignore_errors: yes

    - name: Inject Security Key into Client and Helper Configs
      lineinfile:
        path: "{{ item }}"
        regexp: '^connAuthFile'
        line: 'connAuthFile = /etc/beegfs/connauthfile'
      loop:
        - /etc/beegfs/beegfs-client.conf
        - /etc/beegfs/beegfs-helperd.conf

    - name: Enable and Restart Client Services
      systemd:
        name: "{{ item }}"
        state: restarted
        enabled: yes
      loop:
        - beegfs-helperd
        - beegfs-client
```

### Step 2: Execute the Playbook and Lock the Key

Run the playbook using the default `/etc/ansible/hosts` inventory:

```bash
ansible-playbook deploy_beegfs_clients.yml -K
```

Lock down the key permissions on `head` once the playbook finishes (the playbook's `mode: '0400'` parameter already locks it on all target nodes):

```bash
sudo chmod 400 /etc/beegfs/connauthfile
```

## 5. Phase 4: Verification & Diagnostics

Verify the global mount across all nodes:

```bash
ansible all -m shell -a "df -h | grep beegfs"
```

Expected output on `head`, `com1`, and `com2`:

```text
beegfs_nodev         387G  2.8G  384G   1% /mnt/beegfs
```

Verify registered nodes and storage targets:

```bash
beegfs-ctl --listnodes --nodetype=meta --nicdetails
beegfs-ctl --listnodes --nodetype=storage --nicdetails
beegfs-ctl --listnodes --nodetype=client --nicdetails
beegfs-ctl --listtargets --nodetype=storage --state
```

Test shared read/write permissions for your cluster user:

```bash
sudo chown -R cput:cput /mnt/beegfs
touch /mnt/beegfs/cluster_test.txt
ansible compute -m shell -a "ls -l /mnt/beegfs/cluster_test.txt"
```
