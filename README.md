# VM Ansible Role

## Description
This is a simple role to speed up provisioning VMs for netboot.

By default the qemu:///session URI is used and each VM is given a name prefixed with `cow_`.

## Create a Playbook to use the role. It can be as simple as
```
- hosts: localhost
  gather_facts: false
  roles:
    - role: "{{ playbook_dir }}/roles/vm"
```

### Example usage:
- `ansible-playbook ~/Documents/ansible/vm.yml`
- `ansible-playbook ~/Documents/ansible/vm.yml -e distribution=rhel`
- `ansible-playbook ~/Documents/ansible/vm.yml -e distribution=centos`
- `ansible-playbook ~/Documents/ansible/vm.yml -e distribution=rhel -e vm_name=rhel-sever -e memory=32 -e cpus=2`

The only difference between distributions is the MAC Address prefix used in order to facilitate secure booting different distributions. Please look at [defaults/main.yml](roles/vm/defaults/main.yml) for configurable options. These can be overridden in your playbook to set your preferred defaults and further overridden as extra vars when running the playbook for individual VMs.

## A note about net booting with UEFI Secure Boot enabled
The procedure for net booting Fedora, RHEL, and CentOS is to boot `shimx64.efi` which loads `grubx64.efi` with your accompanying `grub.cfg` to start the installation. Unfortunately the signature for RHEL, CentOS, and Fedora are all different and so require different shims. In order to facilitate this I use different pools in dhcpd to load the different shims.

`shimx64.efi` can be obtained from the shim-x64 package of whichever distribution you want to install. To extract it without installing use rpm2cpio and cpio. For example
```
mkdir shim
cd shim
cp $shim-x64.rpm ./
rpm2cpio $shim-x64.rpm > shim.cpio
cpio -ivd < shim.cpio
```

With all files in place for booting your tree might look like so.
```
$ tree 
.
├── centos
│   ├── c7
│   │   ├── initrd.img
│   │   └── vmlinuz
│   ├── c8-stream
│   │   ├── initrd.img
│   │   └── vmlinuz
│   ├── c9-stream
│   │   ├── initrd.img
│   │   └── vmlinuz
│   ├── grub.cfg
│   ├── grubx64.efi
│   └── shimx64.efi
├── fedora
│   ├── f36
│   │   ├── initrd.img
│   │   └── vmlinuz
│   ├── f37
│   │   ├── initrd.img
│   │   └── vmlinuz
│   ├── grub.cfg
│   ├── grubx64.efi
│   └── shimx64.efi
└── rhel
    ├── grub.cfg
    ├── grubx64.efi
    ├── r7
    │   ├── initrd.img
    │   └── vmlinuz
    ├── r8
    │   ├── initrd.img
    │   └── vmlinuz
    ├── r9
    │   ├── initrd.img
    │   └── vmlinuz
    └── shimx64.efi
```

A dhcpd.conf snippet implementing different dhcp pools might look like below. Note this example is using HTTP boot instead of PXE to eliminate the need for a tftp server. The downside of using HTTP instead of PXE is that the OVMF UEFI default is IPv4 PXE -> IPv6 PXE -> IPv4 HTTP -> IPv6 HTTP, so you either want to hit escape at the UEFI boot menu and manually select IPv4 HTTP or be patient while it gives up and falls through the other options.
```
class "525401" {
	match if substring (hardware, 1, 3) = 52:54:01;
}
class "525402" {
	match if substring (hardware, 1, 3) = 52:54:02;
}
class "525400" {
	match if substring (hardware, 1, 3) = 52:54:00;
}
subnet 192.168.15.1.0 netmask 255.255.255.0 {
	pool {
		deny members of "525401";
		deny members of "525402";

		range 192.168.15.1.200 192.168.15.1.210;
	}

	pool {
		deny members of "525400";
		deny members of "525402";
		ignore bootp;

		option custom-lan-0-1 "HTTPClient";
		if substring (option vendor-class-identifier, 0, 10) = "HTTPClient" {
			filename "http://linux.example.com/tftpboot/centos/shimx64.efi";
		}

		range 192.168.15.1.211 192.168.15.1.220;
	}

	pool {
		deny members of "525400";
		deny members of "525401";

		option custom-lan-1-1 "HTTPClient";
		if substring (option vendor-class-identifier, 0, 10) = "HTTPClient" {
			filename "http://linux.example.com/tftpboot/rhel/shimx64.efi";
		}

		range 192.168.15.1.221 192.168.15.1.230;
	}

	option routers 192.168.15.1.1;
	option domain-name-servers 192.168.15.1.1;

	option custom-lan-1 "HTTPClient";
	if substring (option vendor-class-identifier, 0, 10) = "HTTPClient" {
		filename "http://linux.example.com/tftpboot/fedora/shimx64.efi";
	}

}
```

An example grub.cfg entry for RHEL install media copied from an ISO to server might look like this:
```
function load_video {
	insmod efi_gop
	insmod efi_uga
	insmod video_bochs
	insmod video_cirrus
	insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio

menuentry 'Exit this grub' {
        exit
}

menuentry 'Install RHEL 9 64-bit'  --class fedora --class gnu-linux --class gnu --class os {
	linuxefi r9/vmlinuz ip=dhcp inst.stage2=https://linux.example.com/rhel/9/x86_64/ inst.repo=https://linux.example.com/rhel/9/x86_64/BaseOS inst.ks=http://linux.example.com/rhel/install.ks
	initrdefi r9/initrd.img
}
```

And a basic kickstart:
```
clearpart --initlabel --all
zerombr
part /boot --fstype=xfs --size=512
part /boot/efi --fstype=efi --size=512
part pv.01 --size=1 --grow
volgroup rhel pv.01
logvol swap --vgname=rhel --size=8102 --name=swap
logvol / --vgname=rhel --size=1 --grow --fstype=xfs --name=/

keyboard us
lang en_US.UTF-8
timezone --utc America/New_York
reboot

%packages
@standard
%end
```
