# PXEboot_config
Preboot Execution Environment Config

This recipe describes how to set up a PXEboot server on Debian. We need the
actual PXE server (tftp-hpa), a DHCP server (dnsmasq), and NFS. The client root
system and directories will be stored locally and then exported over the LAN via
NFS. We refer to every machine that connects to the server as a client.  Begin
by installing:

```
server$ sudo apt install tftpd-hpa pxelinux syslinux dnsmasq nfs-kernel-server
```

# Basic client install config

Install a basic system on the server in /srv.
```
server$ sudo mkdir /srv/client
server$ sudo debootstrap --arch amd64 stable /srv/config
```
Configure it using the server as a template.
```
server$ sudo cp /etc/apt/sources.list /srv/client/etc/apt
server$ sudo cp /etc/network/interfaces /srv/client/etc/network
server$ sudo chroot /srv/client
server$ echo "client" > /etc/hostname
server$ apt install console-setup console-data keyboard-configuration locales locales-all tzdata ssh sudo net-tools nfs-common ufw
server$ dpkg-reconfigure console-setup console-data keyboard-configuration locales tzdata
server$ adduser 'user'
server$ usermod -aG sudo 'user'
```
Then we configure /etc/fstab on the client as follows.
```
###### /etc/fstab
/dev/nfs                        /            nfs   auto,nofail,noatime,nolock,hard,intr 0 0
proc                            /proc        proc  defaults 0 0
tmpfs                           /tmp         tmpfs nosuid,nodev 0 0
tmpfs                           /var/tmp     tmpfs defaults 0 0
tmpfs                           /var/log     tmpfs defaults 0 0
tmpfs                           /var/spool   tmpfs defaults 0 0
ip.addr.of.host:/lib/modules    /lib/modules nfs   auto,nofail,noatime,nolock,hard,intr 0 0
```
/lib/modules will another one of the directories that we will export via NFS
from the main server as RO. This is so client(s) can get access to Kernel
modules. The client will also share the Kernel and /boot files of the main
server - again exported over NFS (later).

# Configure PXE boot

The basic config can be left alone as follows.
```
###### /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure"
```
We will have the following directory structure under /srv/tftp.
```
server$ sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp
server$ sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp
server$ sudo mkdir /srv/tftp/pxelinux.cfg
server$ sudo mkdir /srv/tftp/boot
```
On the server we mount bind the server's /boot directory to /srv/tftp/boot
```
server$ sudo mount -o bind /boot /srv/tftp/boot
```
We can also set it in the server's /etc/fstab for convenience:
```
###### /etc/fstab
/boot  /srv/tftp/boot  none  defaults,bind   0 0
```
In /srv/tftp/pxelinux.cfg we will have the following in a file named “default”.
The filename may alternatively also correspond to the MAC address of the client
AA-BB-CC-DD-EE-FF prefixed by 01-. So the PXE boot config file could also be
called 01-AA-BB-CC-DD-EE-FF. This useful if there are multiple clients that will
each need to boot with different configurations (such as their own individual
root system).
```
###### /srv/tftp/pxelinux.cfg/default
DEFAULT linux
LABEL linux
KERNEL boot/vmlinuz-amd64
APPEND rw initrd=boot/initrd.img root=/dev/nfs nfsroot="ip.addr.of.host:/srv/client net.ifnames=0 biosdevname=0
PROMPT 0
TIMEOUT 0
```
Finally we also symlink the appropriate vmlinuz-*-*amd64 to vmlinuz-amd64 and
initrd.img-amd64 to initrd.img in the /boot directory.

# Configure Dnsmasq
```
###### /etc/dnsmasq.d/client.conf
###### DNSMASQ config for local PXE server
dhcp-boot=pxelinux.0
interface=eth0               # Use interface
listen-address=10.0.0.10     # Explicitly specify the address to listen on
bind-interfaces              # Bind to the interface to make sure we aren't sending things elsewher
server=208.67.222.222        # Forward DNS requests to OpenDNS
server=208.67.220.220        # Forward DNS requests to OpenDNS
domain-needed                # Don't forward short names
bogus-priv                   # Never forward addresses in the non-routed address spaces.
dhcp-range=10.0.0.20,10.0.0.20,12h
```
The client is connected directly to the eth0 interface of the server.
```
server$ sudo ip ad add 10.0.0.10/24 dev eth0
```
# Configure NFS

We set the following NFS exports.
```
###### /etc/exports
/srv/client    10.0.0.0/24(rw,async,nohide,crossmnt,no_subtree_check,no_root_squash)
/lib/modules   10.0.0.0/24(ro,async,nohide,crossmnt,no_subtree_check,no_root_squash)
```
# Configure Firewall

We need to make sure port 69 on the server is open for TFTP. We need to make
sure that ports 111, 2049, 33333 are open for NFS. In Debian we need to fix the
rpcmountd port to 33333 because NFS will change it everytime.
```
###### /etc/default/nfs-kernel-server
RPCMOUNTDOPTS="--port 33333"
```
```
server$ sudo ufw allow from <LAN> to any port 111
server$ sudo ufw allow from <LAN> to any port 2049
server$ sudo ufw allow from <LAN> to any port 33333
server$ sudo ufw allow from <LAN> to any port 69 proto tcp
server$ sudo ufw allow from <LAN> to any port 69 proto udp
server$ sudo ufw enable && sudo ufw reload
```
On the sever you will want to redirect gateway traffic to eth0 so that the
client has access to it.
