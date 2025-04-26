# Ansible Role: virt-install

[![Build Status](https://travis-ci.org/your_username/ansible-role-virt-install.svg?branch=master)](https://travis-ci.org/your_username/ansible-role-virt-install)

An Ansible role to create KVM virtual machines on RHEL-like systems using `virt-install`.

Supports two primary creation methods:

1.  **qcow2**: Creates a VM by cloning and resizing a base qcow2 image (like official RHEL/Rocky cloud images) and optionally configuring it using cloud-init.
2.  **kickstart**: Creates a VM by booting from an ISO and performing an automated installation using a customizable Kickstart template.

## Requirements

*   KVM host running a RHEL-like distribution (RHEL 9, Rocky Linux 9, CentOS Stream 9+).
*   `ansible-core` installed on the control node.
*   Required packages on the KVM host (will be installed by the role): `qemu-kvm`, `libvirt-daemon-kvm`, `libvirt-client`, `virt-install`, `genisoimage`.
*   For RHEL hosts/guests requiring subscription during setup, valid Red Hat subscription credentials/activation keys are needed.

## Role Variables

See `defaults/main.yml` for a comprehensive list of variables and their default values.

**Key Variables:**

*   `vm_creation_method`: (`qcow2` | `kickstart`) - The method to use for VM creation. Default: `qcow2`.
*   `vm_name`: Name of the virtual machine.
*   `vm_memory_mb`: RAM for the VM in MiB.
*   `vm_vcpus`: Number of virtual CPUs.
*   `vm_os_variant`: OS variant hint for libvirt (e.g., `rhel9.0`, `rocky9.0`). Use `osinfo-query os` on the KVM host.
*   `vm_network`: Libvirt network to connect the VM to (e.g., `default`). This value is passed to the `virt-install --network` argument.
*   `vm_autostart`: (`true` | `false`) - Whether the VM should start automatically with the host.

**DNS Configuration:**

*   `vm_dns_enable`: (`true` | `false`) - Whether to register the VM's hostname in libvirt's internal DNS. Default: `true`.
*   `vm_dns_domain`: The domain suffix for VM hostnames (e.g., `virt.local`). Default: `virt.local`.
*   `vm_dns_hostname`: The hostname to register (without domain). Default: `{{ vm_name }}`.
*   `vm_dns_fqdn`: The fully qualified domain name. Default: `{{ vm_dns_hostname }}.{{ vm_dns_domain }}`.

**VM State Management:**

*   `vm_state`: Desired state of the VM (`running`, `shutdown`, `destroyed`, `undefined`). Default: `running`.
    *   `running`: Ensures the VM is created and started.
    *   `shutdown`: Ensures the VM exists but is powered off (graceful shutdown).
    *   `destroyed`: Ensures the VM is powered off (forced) and its configuration is removed (`undefine`). Disk images are **deleted**.
    *   `undefined`: Same as `destroyed`.

**Disk Configuration:**

*   `vm_disk_path`: Directory on the KVM host to store all VM disk images.
*   `vm_image_download_dir`: Subdirectory within `vm_disk_path` to store downloaded base images/ISOs.
*   `vm_disk_size_gb`: Size of the primary OS disk.
*   `vm_disks`: (Optional) List of dictionaries to define additional data disks. Each dictionary can have:
    *   `name`: A short name used for the disk filename (e.g., `data`, `logs`). (Required)
    *   `size_gb`: Size in GiB. (Required)
    *   `format`: Disk format (e.g., `qcow2`, `raw`). Default: `qcow2`. (Optional)
    *   `bus`: Disk bus type (e.g., `virtio`, `sata`, `scsi`, `ide`, `usb`). Default: `virtio`. (Optional)

**QCOW2 Specific Variables:**

*   `vm_qcow2_base_image_path`: **Required**. Absolute path to the base qcow2 image on the KVM host.
*   `vm_cloud_init_user_data`: Dictionary for cloud-init `user-data`.
*   `vm_cloud_init_meta_data`: Dictionary for cloud-init `meta-data`.
*   `vm_cloud_init_network_config`: Dictionary for cloud-init `network-config`.

**Kickstart Specific Variables:**

*   `vm_iso_path`: **Required**. Absolute path or URL to the installation ISO.
*   `vm_kickstart_template`: Path to the Jinja2 kickstart template within the role (Default: `kickstart.ks.j2`).
*   `vm_kickstart_vars`: Dictionary of variables passed to the kickstart template.
    *   `hostname`: VM hostname.
    *   `root_password_hash`: Hashed root password.
    *   `timezone`, `locale`, `keyboard`: System settings.
    *   `partitions`: (Optional) A list of dictionaries defining the desired partition layout. If empty or undefined, a default scheme (`/boot`, `swap`, `/`) is used based on `default_*_partition_size_mb` variables. Each dictionary in the list should have:
        *   `mount`: Mount point (e.g., `/`, `/boot`, `/var`) or `swap` for a swap partition. (Required)
        *   `size_mb`: Size in Megabytes. (Required)
        *   `fstype`: Filesystem type (e.g., `xfs`, `ext4`, `swap`). Defaults to `xfs` (or `swap` if mount is `swap`). (Optional)
        *   `grow`: Boolean (`true`/`false`). If true, the partition will use remaining disk space. Defaults to `false`. (Optional)
        *   `options`: String of mount options (e.g., `nodev`). (Optional)
    *   `default_boot_partition_size_mb`, `default_swap_partition_size_mb`, `default_root_partition_size_mb`: Used only if `partitions` list is empty.
    *   `users`: List of user dictionaries to create (name, groups, password_hash, shell).
    *   `packages`: List of packages/groups to install.
    *   `ks_rhel_subscription_*`: RHEL subscription details for guest registration during `%post`.

**RHEL Subscription Variables:**

*   `rhel_subscription_org`: Org ID for `subscription-manager`.
*   `rhel_subscription_activation_key`: Activation key for `subscription-manager`.
*   `rhel_subscription_pool_id`: Optional pool ID if not covered by the activation key.
*   `vm_kickstart_vars.ks_rhel_subscription_*`: Corresponding variables passed to the kickstart `%post` script for guest registration.

**Advanced:**

*   `virt_install_extra_args`: String of additional arguments to pass directly to `virt-install`.

## Dependencies

None.

## Tags

This role uses tags to allow running specific parts:

*   `kvm_host_setup`: Runs tasks related to installing KVM packages, ensuring `libvirtd` is running, and handling RHEL host subscription.
*   `create_vm`: Runs tasks related to creating the virtual machine (using either qcow2 or kickstart method).

You can run only the host setup using `ansible-playbook playbook.yml --tags kvm_host_setup`.
You can run only the VM creation using `ansible-playbook playbook.yml --tags create_vm`.

## Example Playbook

**Creating a Single VM:**

```yaml
---
- hosts: kvm_hosts
  become: true
  vars:
    # Common VM settings
    vm_name: my-webserver-01
    vm_memory_mb: 4096
    vm_vcpus: 2
    vm_os_variant: rocky9.0
    vm_disk_size_gb: 50
    vm_network: my_custom_network # Specify the desired libvirt network here

    # QCOW2 specific settings
    vm_creation_method: qcow2
    vm_qcow2_base_image_path: /var/lib/libvirt/images/Rocky-9-GenericCloud-latest.x86_64.qcow2
    vm_cloud_init_user_data:
      users:
        - name: ansible
          sudo: ALL=(ALL) NOPASSWD:ALL
          groups: wheel
          ssh_authorized_keys:
            - ssh-rsa AAAA...
      packages:
        - nginx
        - firewalld
      runcmd:
        - [ systemctl, enable, --now, nginx ]
        - [ firewall-cmd, --permanent, --add-service=http ]
        - [ firewall-cmd, --reload ]

  roles:
    - role: ansible-role-virt-install
      # Optionally use tags:
      # tags: [ create_vm ] # Assumes host setup was done previously
```

**Creating Multiple VMs using a Loop:**

```yaml
---
- hosts: kvm_hosts
  become: true
  vars:
    vms_to_create:
      - name: app-vm-01
        memory: 2048
        vcpus: 2
        os_variant: rocky9.0
        base_image: /var/lib/libvirt/images/Rocky-9-GenericCloud-latest.x86_64.qcow2
        disk_size: 30
        cloud_init_packages: [ 'python3', 'pip' ]
      - name: app-vm-02
        memory: 2048
        vcpus: 2
        os_variant: rocky9.0
        base_image: /var/lib/libvirt/images/Rocky-9-GenericCloud-latest.x86_64.qcow2
        disk_size: 30
        cloud_init_packages: [ 'python3', 'pip' ]
      - name: db-vm-01
        memory: 8192
        vcpus: 4
        os_variant: rhel9.0
        creation_method: kickstart
        iso_path: /var/isos/RHEL-9.0-x86_64-dvd.iso
        disk_size: 100
        kickstart_vars:
          hostname: db-vm-01.example.com
          root_password_hash: $6$yoursecurehash$...
          packages: [ '@server-product-environment', 'postgresql-server' ]
          # Example custom partitioning
          partitions:
            - mount: /boot
              size_mb: 512
            - mount: swap
              size_mb: 4096
            - mount: /var/lib/pgsql
              size_mb: 51200 # 50GB
              fstype: xfs
            - mount: /
              size_mb: 1024 # Min size
              grow: true

  tasks:
    - name: Create multiple VMs
      ansible.builtin.include_role:
        name: ansible-role-virt-install
      vars:
        vm_creation_method: "{{ item.creation_method | default('qcow2') }}"
        vm_name: "{{ item.name }}"
        vm_memory_mb: "{{ item.memory }}"
        vm_vcpus: "{{ item.vcpus }}"
        vm_os_variant: "{{ item.os_variant }}"
        vm_disk_size_gb: "{{ item.disk_size }}"
        # QCOW2 specific
        vm_qcow2_base_image_path: "{{ item.base_image | default(omit) }}"
        vm_cloud_init_user_data:
          packages: "{{ item.cloud_init_packages | default(omit) }}"
        # Kickstart specific
        vm_iso_path: "{{ item.iso_path | default(omit) }}"
        vm_kickstart_vars: "{{ item.kickstart_vars | default({}) }}"
      loop: "{{ vms_to_create }}"
      tags: [ create_vm ] # Apply tag to the loop task
```

**Example with Additional Disks and State Management:**

```yaml
---
- hosts: kvm_hosts
  become: true
  vars:
    vms_to_manage:
      - name: web-server-prod
        state: running # Ensure it's running
        memory: 2048
        vcpus: 2
        os_variant: rocky9.0
        base_image: rocky-9-qcow2
        disk_size: 40
        disks:
          - name: webdata
            size_gb: 100
            bus: virtio
      - name: old-test-server
        state: undefined # Ensure it's destroyed and disks removed

  tasks:
    - name: Manage multiple VMs
      ansible.builtin.include_role:
        name: ansible-role-virt-install
      vars:
        vm_state: "{{ item.state | default('running') }}"
        vm_name: "{{ item.name }}"
        vm_memory_mb: "{{ item.memory | default(omit) }}"
        vm_vcpus: "{{ item.vcpus | default(omit) }}"
        vm_os_variant: "{{ item.os_variant | default(omit) }}"
        vm_disk_size_gb: "{{ item.disk_size | default(omit) }}"
        vm_disks: "{{ item.disks | default([]) }}"
        # QCOW2 specific
        vm_creation_method: "{{ item.creation_method | default('qcow2') }}"
        vm_qcow2_base_image_path: "{{ item.base_image | default(omit) }}"
        # Kickstart specific
        vm_iso_path: "{{ item.iso_path | default(omit) }}"
        vm_kickstart_vars: "{{ item.kickstart_vars | default({}) }}"
      loop: "{{ vms_to_manage }}"
      tags: [ create_vm ] # Tag covers creation and state management
```

**Example with DNS Configuration:**

```yaml
---
- hosts: kvm_hosts
  become: true
  vars:
    # DNS configuration at the host level
    vm_dns_domain: "lab.example.com"  # Custom domain for all VMs

    vms_to_create:
      - name: db01
        memory: 4096
        vcpus: 2
        os_variant: rocky9.0
        base_image: rocky-9-qcow2
        disk_size: 60
        # Static IP for DNS registration
        network_config_method: static
        network_ip: "192.168.122.10/24"
        network_gateway: "192.168.122.1"
        # Custom hostname (otherwise defaults to name)
        dns_hostname: "database01" # Will be registered as database01.lab.example.com

      - name: app01
        memory: 2048
        vcpus: 2
        os_variant: rocky9.0
        base_image: rocky-9-qcow2
        disk_size: 40
        # Uses default DHCP - IP will be detected from VM lease
        # Custom DNS settings per VM
        dns_enable: true
        dns_domain: "apps.example.com" # Override host-level domain
        # Will be registered as app01.apps.example.com

  tasks:
    - name: Create VMs with DNS registration
      ansible.builtin.include_role:
        name: ansible-role-virt-install
      vars:
        vm_name: "{{ item.name }}"
        vm_memory_mb: "{{ item.memory }}"
        vm_vcpus: "{{ item.vcpus }}"
        vm_os_variant: "{{ item.os_variant }}"
        vm_disk_size_gb: "{{ item.disk_size }}"
        vm_creation_method: "{{ item.creation_method | default('qcow2') }}"
        vm_qcow2_base_image_path: "{{ item.base_image }}"

        # Network configuration
        vm_network_config_method: "{{ item.network_config_method | default('dhcp') }}"
        vm_network_ip: "{{ item.network_ip | default('') }}"
        vm_network_gateway: "{{ item.network_gateway | default('') }}"

        # DNS configuration
        vm_dns_enable: "{{ item.dns_enable | default(true) }}"
        vm_dns_hostname: "{{ item.dns_hostname | default(item.name) }}"
        vm_dns_domain: "{{ item.dns_domain | default(vm_dns_domain) }}"
      loop: "{{ vms_to_create }}"
```

# You might run host setup separately first:
# ansible-playbook playbook.yml --tags kvm_host_setup
# Then create VMs:
# ansible-playbook playbook.yml --tags create_vm

## License

MIT (or your chosen license)

## Author Information

Your Name / Your Company
