```
#doas ee /usr/local/etc/pkg/repos/FreeBSD.conf
#FreeBSD: { enabled: yes }

doas pkg update
# For Cloud Support
# pkg install qemu-tools
doas pkg install vm-bhyve bhyve-firmware [optional[grub2-bhyve]] 

# Configure VM Repository
doas zfs create zroot/vm
doas sysrc vm_enable="YES"
doas sysrc vm_dir="zfs:zroot/vm"
doas vm init

# Configure network and templates
doas cp /usr/local/share/uefi-firmware/BHYVE_BHF_UEFI.fd /zroot/vm/.config/BHYVE_UEFI.fd
doas cp /usr/local/share/examples/vm-bhyve/* /zroot/vm/.templates/
doas vm switch create public
doas vm switch add public igb1

# Modify the firwall
doas ee /etc/pf.conf.d/bhf

## Add this
# VM passthrough
pass in quick on vm-public all
pass out quick on vm-public all

# Reload the rules, might also need to reload after reboot if rules are in wrong spot
doas pfctl -f /etc/pf.conf.d/bhf 


# Download the ISO
doas vm iso https://releases.ubuntu.com/jammy/ubuntu-22.04.2-live-server-amd64.iso
#doas vm img https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Create the VM
doas vm create ubuntu-server
doas cp /usr/local/share/uefi-firmware/BHYVE_BHF_UEFI_VARS.fd /zroot/vm/ubuntu-server/uefi-vars.fd
#doas vm create -i jammy-server-cloudimg-amd64.img ubuntu-server
doas ee /zroot/vm/ubuntu-server/ubuntu-server.conf

Line 296

# Overwrite everything inside the config file with below
loader="uefi-custom"
uefi_vars="yes"
bhyveload_args="fwcfg=qemu"
graphics="yes"
graphics_port="5999"
graphics_listen="0.0.0.0"
graphics_res="1600x900"
graphics_wait="auto"
xhci_mouse="yes"
cpu="2"
memory="6G"
network0_type="virtio-net"
network0_switch="public"
disk0_type="virtio-blk"
disk0_name="disk0.img"
disk0_size="30G"

# Install the VM
doas vm install -f ubuntu-server ubuntu-22.04.2-live-server-amd64.iso

# Remote in with VNC, or whatever. After install you can start the machine with 
doas vm start ubuntu-server

# You can have it auto-start on boot with
doas sysrc vm_list="ubuntu-server"

```

Test is out with the BHF Bootrom by overwriting the vanilla Bhyve-firmware files. Eventually vm-bhyve will allow you to specify a bootrom, but to revert you can just reinstall Bhyve-firmware.

```
doas cp /usr/local/share/uefi-firmware/BHYVE_BHF_UEFI_VARS.fd /zroot/vm/ubuntu-server/uefi-vars.fd

doas cp /usr/local/share/uefi-firmware/BHYVE_BHF_UEFI.fd /zroot/vm/.config/BHYVE_UEFI.fd
```





Old way

```
doas mkdir $HOME/vms
doas mkdir $HOME/vms/ubuntu-server

doas fetch -o $HOME/ubuntu-server-installer.iso https://releases.ubuntu.com/jammy/ubuntu-22.04.2-live-server-amd64.iso

doas zfs create -V 30G -o volmode=dev zroot/ubuntu-server-vm_disk0

doas ee /etc/rc.conf

cloned_interfaces="bridge0"
ifconfig_bridge0="inet 192.168.1.5 netmask 255.255.255.0"

doas ee /etc/pf.conf.d/bhf

pass in quick on bridge0
pass in quick proto tcp to port 5900
pass in quick proto tcp to port 48898

doas pfctl -f /etc/pf.conf.d/bhf

#Copy script over

doas sh ubuntu-server-vm.sh config
doas sh ubuntu-server-vm.sh run –-install

```

```
#!/bin/sh

#This script has been modified from an example
vm_name="ubuntu-server-vm"
vm_dir="/usr/home/Administrator/vms/ubuntu-server"
iso_cd0="/usr/home/Administrator/ubuntu-server.iso"

vm_bridge0="bridge0"
vnc_port="5900"
nic_port="igb1"
efi_vars="${vm_dir}/EFI_VARS.fd"
vm::init(){
	kldload vmm
}

err(){
    echo "Error: ${1}" 1>&2
    exit 1
}

vm::config(){
	
	if [ ! -f "${efi_vars}" ]; then
                cp /usr/local/share/uefi-firmware/BHYVE_BHF_UEFI_VARS.fd "${vm_dir}/EFI_VARS.fd"
        fi
		
	_tap0="$(ifconfig tap create)"
	sysctl net.link.tap.up_on_open=1 #enable the tap interface
	sysctl net.link.bridge.pfil_member=0 #enable communication through the firewall to members of the bridge interface

	# bridge device is created via cloned interfaces in rc.conf
	#this line issues the command to add the tap0 and em1 interfaces as members of bridge0.
	#This will allow communication on the em1 interface to be sent to the tap0 interface where the vm will be able to receive it
	ifconfig bridge0 addm tap0 addm ${nic_port}

	
	
}

vm::run(){

	while true; do
	
		if [ "${install_os}" == "true" ]; then #if the caller has indeicated that this is an installation run.
                	_installerdisk="-s 10:0,ahci-cd,${iso_cd0}"
        	else
                	_installerdisk=""
        	fi

		bhyve -c 2 -m 4G \
		-A -H \
		-l bootrom,/usr/local/share/uefi-firmware/BHYVE_BHF_UEFI.fd \
		-s 0:0,hostbridge \
		-s 1:0,virtio-blk,/dev/zvol/zroot/ubuntu-server-vm_disk0 \
		-s 2:0,virtio-net,tap0 \
		${_installerdisk} \
		-s 29:0,fbuf,tcp=0.0.0.0:"${vnc_port}",w=1024,h=768,wait \
		-s 30:0,xhci,tablet \
		-s 31:0,lpc \
		ubuntu-server-vm

		_rc=$?
	
	# see man 8 bhyve for definition of bhyve return codes
		if [ ${_rc} -ne 0 ]; then
			break
		fi
		
		if [ "${install_os}" == "true" ]; then		
			install_os="false"
			_installerdisk=""
		fi 
		
	done
	
}

vm::destroy(){
	bhyvectl --vm=${vm_name} --destroy
	sleep 1
}

vm::cleanup(){

	if [ -f bhyve.pid ]; then
		rm bhyve.pid
	fi

	if [ -n "$(ifconfig tap0)"  ]; then
		ifconfig bridge0 deletem tap0 deletem ${nic_port}
		ifconfig tap0 destroy 
	fi

}

#Main script instructions
if [ $(id -u) -ne 0 ]; then
    err "${0##*/} must be run as root!"
fi


install_os="false"

for _arg in $@; do
	case ${_arg} in
		--install)
			install_os="true"
			break
		;;
	esac
done

if [ $# -lt 1 ]; then
    err "No Commands Provided"
fi

case $1 in
	config)
		vm::config
		;;
	run)
		vm::init
		vm::run
		;;
	destroy)
		vm::destroy
		vm::cleanup
		;;
esac
```

