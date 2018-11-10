# _A GUIDE TO INSTALLING CUKCOO WITH A WEB INTERFACE ON UBUNTU 18.04 LTS_
UPDATED NOVEMBER 8 2018
# `Have fun, the power of malware compels you !` 

# `HOST MACHINE REQUIREMENTS AND CUCKOO INSTALL`        

# `Python Libraries`
```
sudo apt-get install python python-pip python-dev libffi-dev libssl-dev
sudo apt-get install python-virtualenv python-setuptools
sudo apt-get install libjpeg-dev zlib1g-dev swig
```

# `MongoDB`
```
sudo apt-get install mongodb
```

# `PostgreSQL`
```
sudo apt-get install postgresql libpq-dev
```

# `Virtual Box`
```
sudo apt-get update
sudo apt-get install virtualbox
```

# `TCPdump`
```
sudo apt-get install tcpdump apparmor-utils
sudo aa-disable /usr/sbin/tcpdump
```
# SET `TCPdump Privs`
```
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
```
# Verify `Cap Privs`
```
getcap /usr/sbin/tcpdump
should display - /usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip
```

# `guacd`
```
sudo apt install libguac-client-rdp0 libguac-client-vnc0 libguac-client-ssh0 guacd
```

#### `NEW USER CREATION IF NEEDED` ####
# New User (cuckoo)
#`sudo adduser cuckoo`

# Add to vboxusers
#`sudo usermod -a -G vboxusers cuckoo`

# Install `cuckoo`
```
#  NOTE: You will need to install with "sudo pip --proxy=http://xxxxxxxx -U pip setuptools" if behind a proxy.
#  NOTE: You will need to install with "sudo pip --proxy=http://xxxxxxxx -U pip cuckoo" if behind a proxy.
```
```
sudo pip install -U pip setuptools
sudo pip install -U cuckoo (may have broken deps from #1) - Run #1 again if needed.
```
# `NETWORKING SETUP`

# `Create Virtual Interface`
If the hostonly interface vboxnet0 does not exist already.
```
VBoxManage hostonlyif create
```
# Configure vboxnet0
```
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
```
# CONFIGURE ROUTING BETWEEN HOST AND GUEST
```
sudo iptables -A FORWARD -o enp0s3 -i vboxnet0 -s 192.168.56.0/24 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A POSTROUTING -t nat -j MASQUERADE
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
exit
```
## RDP INSTALL FOR VM GUEST INSTALLATION (THIS WILL BE REMOVED AFTER INSTALL)

# Install xrdp
```
sudo apt install xrpd
```
# Install Desktop Environment (xfce)
```
sudo apt install xfce4 slim
sudo service slim start
```

# ******** RDP into Server to Continue Installation *********
`Using RDP client, connect to the cuckoo server using your server creds and continue installation`

# GUEST(VM) Installation
# https://cuckoo.sh/docs/installation/guest/index.html
```
Once you are in the xfce4 desktop, open virtualbox and create/install your windows image
NOTE: You may want different operating systems and different versions for your cuckoo
analysis. In this install guide we are using a Windows 7 32bit SP1 Image.
```
```
```
# `WINDOWS VM REQUIREMENTS`
```
```
# WINDOWS OR OS IMAGE OF CHOICE
# Windows 7 32 SP1
#
AS THE WIN VM HAS NO INTERNET ACCESS YOU WILL NEED TO CREATE A SHARED FOLDER TO INSTALL
THE REQUIRED FILES. THIS REQUIRES AN INSTALL OF GUEST ADDITIONS WHICH WILL BE REMOVED AFTER 
INSTALL. SOME MALWARE MAY DETECT IT'S IN A SANDBOXED ENVIRONMENT WITH GUEST EDITIONS INSTALLED.
#
# INSTALL PYTHON 2.7 ON WINDOWS
http://www.python.org/getit/

# INSTALL PYTHON PILLOW - USED FOR SCREENSHOTS
https://python-pillow.org/

# INSTALL 'PDF READERS, BROWSERS ETC FOR TESTING'
`Adobe Reader, Adobe Flash Player, Java, Outlook Express, Office 2016`

# MAKE SURE YOU `OPEN APPLICATIONS AFTER INSTALL` TO GET RID OF THE "FIRST-RUN" POP-UPS
```
```
# `WINDOWS VBOX INSTALL` 
```
```
# Create a windows virtal machine in virtualbox named cuckoo1
If you are running multiple instances, name the VM accordingly. ie; `cuckoo1, cuckoo2, cuckoo3`
```
Ensure it is set to "_Host-Only_" Networking on vboxnet0 - You do not ever want to expose this
Windows VM to the internet ever. It has `_NO FIREWALL/UPDATES/AV_ -> _HOST-ONLY NETWORKING !!!!!_`
```
You may need to create a host-only network in host-network manager in virtualbox

# `WINDOWS VM SETUP`
# Windows Install Configuration
```
USERNAME: cuckoo
PASSWORD: cuckoo (machine will running from snapshot so doesn't matter)
Uncheck auto activate and click next. Do not enter key
Home Network
```
# DISABLE WINDOWS FEATURES
DISABLE `WINDOWS FIREWALL, UAC & AUTOMATIC UPDATES`

# NETWORK SETUP
Use `ONLY STATIC IP` addresses for your guest, as Cuckoo doesn’t support DHCP and using it will break your setup.
The recommended setup is using a Host-Only networking layout with proper forwarding.
In this case 192.168.56.1 is the gateway , DNS 0.0.0.0

# INSTALL CUCKOO AGENT ON WINDOWS DESKTOP
In the ``$CWD/agent/`` directory you will find the agent.py file. Copy this file to the Guest operating system
and run it. The Agent will launch a small API server.

# SAVING THE IMAGE SNAPSHOT
Now you should be ready to save the virtual machine to a snapshot state.
Before doing this make sure you rebooted it softly and that it’s currently running, with Cuckoo’s agent running and with Windows fully booted.
# TAKE SNAPSHOT OF VIRTUAL MACHINE
```
VBoxManage snapshot "<Name of VM>" take "<Name of snapshot>" --pause
```
# ONCE SNAPSHOT IS COMPLETED, YOU CAN POWER OFF MACHINE AND RESTORE IT
```
VBoxManage controlvm "<Name of VM>" poweroff
```
```
VBoxManage snapshot "<Name of VM>" restorecurrent
```
# YOUR WINDOWS VM IS CONFIGURED
```
cuckoo --cwd /data/cuckoo/.cuckoo -d
```
# THIS SNAPSHOT CAN BE CLONED FOR MULTIPLE CUCKOO INSTANCES - JUST NUMBER THE VBOX ACCORDINGLY: cuckoo1, cuckoo2 , etc

# REMEMBER TO REMOVE GUEST ADDITIONS AFTER INSTALL IS COMPLETE - SOME MALWARE LOOKS FOR GUEST ADDITIONS TO DETECT SANDBOX


# `STARTING CUCKOO FOR THE FIRST TIME`

# FIRST RUN
```
cuckoo -d (this will generate default directories)
You must then edit a few cuckoo config files. Most of these should already be set up.
Confirm this in the following files:
All Cuckoo configuration files are located in the ~/.cuckoo/conf directory.

cuckoo community (updates cuckoo)
==================================
_cuckoo.conf_
[cuckoo]
memory_dump = yes
machinery = virtualbox

[resultserver]
ip = 192.168.56.1
==================================
_auxillary.conf_
[sniffer]
enabled = yes
==================================
_virtualbox.conf_
[virtualbox]
mode = gui
machines = cuckoo1

[cuckoo1]
label = cuckoo1
platform = windows
ip = 192.168.56.101
snapshot = Snapshot1
==================================
_processing.conf_
[memory]
enabled = yes
==================================
_memory.conf_ (for volatility profile)
[basic]
guest_profile = “Win7SP1x86”
==================================
_reporting.conf_
[singlefile]
Enable creation of report.html?
enabled = yes
﻿
[mongodb]
enabled = yes
=================================

...............,´¯­­`,
.........,´¯`,....­­/
....../¯/.../..../
..../../.../..../­­..,-----,
../../.../....//­­´...........`.
./../.../..../­­......../´¯\....\
('.('..('....('....­­...|.....'._.'
.\.................­­...`\.../´...)
...\...............­­......V...../
.....\.............­­............/
.......`•............­­.......•´
..........|.........­­........|
```
# Basic Configuration of cuckoo is complete.

```
# (************ADD TO RUN AT BOOT************)
vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
cuckoo --cwd /data/cuckoo/.cuckoo -d
cuckoo --cwd /data/cuckoo/.cuckoo web runserver <IP>:<PORT>
```
# IMPORTANT: ENSURE VBOXNET0 IS UP (ifconfig) OR RUN:
```
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
```
# NOW START CUCKOO
```
cuckoo -d (starts cuckoo with diagnostics -d)
cuckoo --cwd /data/cuckoo/.cuckoo -d (Run cuckoo in a CWD other than home. In this case data)
cuckoo --cwd /data/cuckoo/.cuckoo web runserver <IP>:<PORT>
```

# CUCKOO RUN OPTIONS

# In order to start the web interface, you can simply run the following command from the web/ directory:
```
cuckoo web  (starts on localhost:8000 by default)
cuckoo --cwd /data/cuckoo/.cuckoo web runserver x.x.x.x:xxxx
```
# If you want to configure the web interface as listening for any IP on a specified port, you can start it with the following command (replace PORT with the desired port number):
```
cuckoo web runserver 0.0.0.0:PORT
```
# Or directly without the runserver part as follows while also specifying the host to listen on:
```
cuckoo web -H 0
```
## YOU SHOULD NOW BE READY TO USE CUCKOO 
This covers the very basic installation and configuration of Cuckoo. You will need to set it up according your requirements and enable the appropriate analyzers and their configurations. 

# COMING SOON: WEB DEPLOYMENT OF CUCKOO WITH HTTPS AND AUTHENTICATION
