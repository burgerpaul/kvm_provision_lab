kvm_provision_lab
=================

With this role you can deploy a lab environment of Linux VMs to a KVM/QEMU hypervisor.

For the hypervisor host I have tested but other/newer releases might work too:

 - Debian 11
 - Debian 12
 - Debian 13
 - RHEL 9
 - Fedora 37

If you are capable of reading German you'll find a tutorial on my blog at: https://www.my-it-brain.de/wordpress/labor-umgebung-mit-ansible-in-kvm-erstellen/.

Requirements
------------

This role needs the `community.libvirt.virt` module on the Ansible Control Node. On the remote hypervisor host the role automatically installs the required packages (`libguestfs-tools` and `python3-libvirt` on Debian/Ubuntu, `libguestfs` and `python3-libvirt` on RedHat/Fedora). The commands `virt-customize`, `virt-sysprep`, `virt-sparsify`, `qemu-img` and `virsh` must be available on the remote host (they are typically provided by `libguestfs-tools`/`libguestfs`).

You can use the command `ansible-galaxy collection install community.libvirt community.general` to install the required collections to your Ansible Control Node.

Role Variables
--------------

The following code block shows all variables with their default values. You need to specify the directory where your base disk image templates are stored using `libvirt_template_dir`, and the directory where guest disk images will be created using `libvirt_pool_dir`. With `vm_root_pass` you set the password for the root user and `ssh_key` adds an SSH public key to the authorized_keys file.

The VMs are defined as a dictionary under the `guests` key as shown below.

```yaml
libvirt_pool_dir: "/var/lib/libvirt/images"
libvirt_template_dir: "/srv/vmtemplates"
vm_root_pass: "123456"
ssh_key: "/path/to/ssh-pub-key"

guests:
  test:
    vm_ram_mb: 512
    vm_vcpus: 1
    vm_iftype: network
    vm_net: default
    os_type: rhel9
    file_type: qcow2
    base_image_name: rhel9-template
    vm_template: "rhel9-template"
    second_hdd: false
    second_hdd_size: ""
  test2:
    vm_ram_mb: 1024
    vm_vcpus: 2
    vm_iftype: network
    vm_net: default
    os_type: debian12
    file_type: qcow2
    base_image_name: debian12-template
    vm_template: "debian12-template"
    second_hdd: true
    second_hdd_size: "10G"
    dns: true
    dhcp_host: true
```

With the variables from the code block above you would create the two VMs _test_ and _test2_ with the parameters specified in the dictionary. The core guest variables are explained in the table below.

| Variable | Description |
| --- | --- |
| `guests` | Name of the dictionary specifying the parameters for guest domains (VMs). |
| `test` | This key is used as name for the domain (VM) to create. |
| `vm_ram_mb` | Guest memory in MB. |
| `vm_vcpus` | Count of vCPUs for the domain. |
| `vm_iftype` | Type of network interface to use. Possible values are `network` (default) or `bridge`. |
| `vm_net` | Network or bridge name. Must already exist in your environment. |
| `os_type` | Type of the guest domain, e.g. `rhel9`, `debian12` or `debian13`. Must match an XML template in `templates/`. |
| `file_type` | Disk image format. Defaults to `qcow2`. You usually don't have to change this. |
| `base_image_name` | The name of the qcow2 image (without extension) located in `libvirt_template_dir` that is used as the source to copy from. |
| `vm_template` | Name of the XML template to use (without the `.xml.j2` suffix). Bundled templates: `debian11-template`, `debian12-template`, `debian13-template`, `rhel9-template`. |
| `second_hdd` | Boolean to specify whether a second disk image should be created and attached. |
| `second_hdd_size` | Size of the second disk image. Sizes can be specified in `k`/`K` (kilobyte), `M` (megabyte), `G` (gigabyte) or `T` (terabyte). |
| `dns` | Optional boolean. When set to `true`, an A and PTR record are created for this VM via `nsupdate`. |
| `dns_name` | Optional string. Override the DNS hostname for this VM (defaults to the VM name / dictionary key). |
| `dhcp_host` | Optional boolean. When set to `true`, a static DHCP host entry is added to the ISC DHCP server config for this VM. |

### DNS and DHCP variables

The following variables control the optional DNS and DHCP update functionality (task file `dnsupdate.yml`). They are not required unless you set `dns: true` or `dhcp_host: true` on at least one guest.

| Variable | Default | Description |
| --- | --- | --- |
| `dns_zone` | `example.local` | DNS zone name for A and PTR records. |
| `dns_server` | `127.0.0.1` | Address of the DNS server used for `nsupdate` and reverse-lookup probes. |
| `dns_ttl` | `3600` | TTL (in seconds) for DNS records created via `nsupdate`. |
| `dns_tsig_key_name` | _(omit)_ | TSIG key name for authenticated `nsupdate`. Leave unset for unauthenticated updates. |
| `dns_tsig_key_secret` | _(omit)_ | TSIG key secret (base64-encoded) for authenticated `nsupdate`. |
| `dns_tsig_key_algorithm` | `hmac-sha512` | TSIG key algorithm, e.g. `hmac-sha256` or `hmac-sha512`. |
| `network_ip_ranges` | _(undefined)_ | Dictionary mapping libvirt network names to IP ranges. Required for automatic IP assignment. See example below. |
| `dhcp_config_path` | `/etc/dhcp/dhcpd.conf` | Path to the ISC DHCP server configuration file on `dhcp_server_host`. |
| `dhcp_leases_file` | `/var/lib/dhcp/dhcpd.leases` | Path to the ISC DHCP leases file used to detect already-used IPs. |
| `dhcp_server_host` | _(required if `dhcp_host` used)_ | Hostname or IP of the host running the ISC DHCP server. Config changes are delegated to this host. |
| `dhcp_service_name` | `isc-dhcp-server` | Name of the DHCP systemd service to restart after config changes. |

Example `network_ip_ranges` configuration:

```yaml
network_ip_ranges:
  default:
    start: 192.168.122.100
    end: 192.168.122.200
  intern:
    start: 10.0.1.50
    end: 10.0.1.150
```

The role probes the configured DNS server for PTR records to find the first available IP in the range, ensuring no two VMs are assigned the same address even across multiple playbook runs.

Tags
----

| Tag | Description |
| --- | --- |
| `createvm` | Runs all tasks related to provisioning VMs (copy images, configure, define domains, start VMs). |
| `dnsconfig` | Runs only the DNS and DHCP update tasks. Can be combined with `createvm`. |

Dependencies
------------

- `community.libvirt`
- `community.general`

Example Playbook
----------------

```yaml
- hosts: kvm_hosts
  vars:
    libvirt_template_dir: "/srv/vmtemplates"
    vm_root_pass: "S3cr3t!"
    ssh_key: "~/.ssh/id_ed25519.pub"
    dns_zone: "lab.example.com"
    dns_server: "192.168.122.1"
    network_ip_ranges:
      default:
        start: 192.168.122.100
        end: 192.168.122.200
    dhcp_server_host: "192.168.122.1"
    guests:
      webserver:
        vm_ram_mb: 2048
        vm_vcpus: 2
        vm_iftype: network
        vm_net: default
        os_type: rhel9
        file_type: qcow2
        base_image_name: rhel9-template
        vm_template: "rhel9-template"
        second_hdd: false
        second_hdd_size: ""
        dns: true
        dhcp_host: true
  roles:
    - kvm_provision_lab
```

Contribution
------------

In case you need additional information on how to use this role, please raise an issue with your question at: https://github.com/Tronde/kvm_provision_lab/issues
Pull requests improving this README as well as the role are appreciated.

License
-------

GPL-2.0-or-later

Author Information
------------------

Author: Tronde
