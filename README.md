# TC BSD Helpful Notes

## Changing the login shell

### Remembering is hard and typing is error prone!

To reduce your mental overhead so that you can concentrate on the important stuff, you should make use of more advanced shell applications which support at least autofill, reverse search and spelling correction. In Addition, colorized output can help you to quickly filter for relevant information.

Consequently, lets try to install and configure a different shell program as our default.

```Bash
doas pkg install zsh git
```
```Bash
cat /etc/shells
```
```Bash
chsh -s /usr/local/bin/zsh Administrator
```
```Bash
touch ~/.zshrc
```
```Bash
doas reboot
```

Install oh-my-zsh for QoL shell improvements
```Bash
curl -o install-oh-my-zsh.sh -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh
```
```Bash
ls -l
```
```Bash
sh install-oh-my-zsh.sh
```
To use, press TAB to get an autocomplete feature. If you press TAB again, it will display all subdirectories. The TAB key also runs through a spell/path check.


## Ubuntu Hypervisor

### Preparing the system

Make sure the Bhyve package is installed
```Bash
doas pkg update
doas pkg install uefi-edk2-bhyve-bhf
```

Let's set up some directories for the VM files to be stored. These will be in the Administrator's user directory.

```Bash
mkdir ~/vms
mkdir ~/vms/ubuntu
```

Now we should download the latest ISO for Ubuntu

```Bash
fetch -o ~/vms/ubuntu/ubuntu-installer.iso https://releases.ubuntu.com/24.04.1/ubuntu-24.04.1-desktop-amd64.iso
```
**_NOTE:_**  If using a USB stick, you'll need to mount the drive and then copy the iso to the ~/vms/ubuntu/ directory.

Now we need to create a empty virtual disk for the VM to use. Be sure to check the size of disk needed for the OS; here we use 18G.
```Bash
doas zfs create -V 18G -o volmode=dev "zroot/ubuntu-vm_disk0"
```

Next we need to create a bridge between the guest vm machine and the host computer. 

Get the name of the interface
```Bash
ifconfig
```
Look for the interface that you are using for SSH if you want to use the same network. In my case, the interface is 'igb1'.

Modify the rc.conf file
```Bash
doas ee /etc/rc.conf
```
Add the following line to the file, but select an IP address on your subnet. This will be the address of the VM.
```
cloned_interfaces="bridge0"
ifconfig_bridge0="inet 192.168.40.187 netmask 255.255.255.0"
```
Press esc, and save changes.
**_NOTE:_**  If you have issues with networking down the line. Try manually typing out the settings in the ssh connection instead of copy/past. Sometimes Windows Powershell will copy characters incorrectly over ssh.

Now we need to set up firewall rules
```Bash
doas ee /etc/pf.conf.d/bhf
```
Add the following below the allow ADS secure
```
# allow ADS unsecure
pass in quick proto tcp to port 48898

# allow VM
pass in quick on bridge0
pass in quick proto tcp to port 5900
```
Exit and save changes

Reboot the system
```Bash
doas reboot now
```

### Installing and configuring Ubuntu

Modify the variables inside the included ubuntu-vm.sh file. It's important to look around and get a feel for how the script works. Customers will need to make their own custom OS script most likely.

If you need to copy the ubuntu-vm.sh from your system to the remote. Use the SCP command in powershell.

**_NOTE:_** Pay attention to line 48. ```bhyve -c 2 -m 6G``` specifies 2 CPU core and 6G of memory.

```Shell
 scp "C:\Ubuntu Quick Start\ubuntu-vm.sh" Administrator@192.168.40.183:~/vms/ubuntu
```
Make sure you  are in the script directory location
```Bash
cd ~/vms/ubuntu
```
Configure the VM
```Bash
doas sh ubuntu-vm.sh config
```
Boot the VM and install
```Bash
doas sh ubuntu-vm.sh run --install
```
With this ssh window open, open a TightVNC or UltraVNC client application and connect to the IP address you selected for the machine.

Follow the installation process over VNC as you would normally on a bare metal machine. Ubuntu Desktop might take a while to boot, and you might need to select (Safe Graphics) boot to installer.

Once the installation is complete, you can simply launch the system when you want using the command below.
```Bash
doas sh ubuntu-vm.sh run
```

**_NOTE:_** You can stop the VM at any time by pressing Ctrl+C in the SSH session

If you get the feedback "vm_reinit: Device busy" you might need to destroy the session
```Bash
doas sh ubuntu-vm.sh destroy
doas sh ubuntu-vm.sh cleanup
```
```Bash
doas sh ubuntu-vm.sh config
doas sh ubuntu-vm.sh run
```

## Install PyAds for Ubuntu

Install the ADS C++ Driver
```Bash
# clone the repository
git clone https://github.com/Beckhoff/ADS.git
# change into root of the cloned repository
cd ADS
# configure meson to build the library into "build" dir
meson setup build
# let ninja build the library
ninja -C build
```

Install PyAds
```Bash
sudo apt install python3-pip
pip install pyads
```
Depending on your Linux system rules, you might need something like this instead
```Bash
pip install pyads --break-system-packages
```

Sample Program
```Python
import pyads
CLIENT_NETID = '1.2.3.4.1.1'
CLIENT_IP = '192.168.10.187
TARGET_IP = '192.168.10.197'
TARGET_NETID = '192.168.10.197.1.1'
TARGET_USERNAME = 'Administrator'
TARGET_PASSWORD = '1'
ROUTE_NAME = 'RouteToMyPC'

# Create a Route if not already generated
pyads.open_port()
pyads.set_local_address(CLIENT_NETID)
pyads.add_route_to_plc(CLIENT_NETID, CLIENT_IP, TARGET_IP, TARGET_USERNAME, TARGET_PASSWORD, route_name=ROUTE_NAME)
pyads.close_port()

plc = pyads.Connection(TARGET_NETID, pyads.PORT_TC3PLC1, TARGET_IP)
plc.open()

# check the connection state
plc.read_state()

# read int value by name
print(f'PLC Value: {plc.read_by_name("GVL.int_val")}')

plc.close()

```


## Install ROS

Follow the guide here to install ROS into the Ubuntu Hypervisor

[https://docs.ros.org/en/rolling/Installation/Ubuntu-Install-Debs.html](https://docs.ros.org/en/rolling/Installation/Ubuntu-Install-Debs.html)
