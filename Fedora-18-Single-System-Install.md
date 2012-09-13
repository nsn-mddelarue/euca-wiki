## Upgrade Fedora 17 to Fedora 18 (optional)

This is only necessary if you don't have a working Fedora 18 install.  It is recommended that you install Fedora 18.

1. kickstart fedora 17
1. switch selinux to permissive mode
1. reboot
1. /sbin/fixfiles -F restore (should use autorelabel, but reboots were failing)
1. yum update yum
1. rpm --import https://fedoraproject.org/static/DE7F38BD.txt
1. yum clean all
1. yum --releasever=18 --disableplugin=presto distro-sync
1. Review http://fedoraproject.org/wiki/Upgrading_Fedora_using_yum#Fedora_17_-.3E_Fedora_18 to ensure that there aren't other known issues to be worked around.

## Network Configuration

**NOTE: This configuration may vary by system, depending on your overall network configuration.  If em1 is your only/default interface, it is likely that NetworkManager will drop your connection as soon as ifcfg-em1 changes.**

1. Configure the bridge (so that system mode will work).
```
cat <<EOF >/etc/sysconfig/network-scripts/ifcfg-br0
DEVICE=br0
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=dhcp
TYPE=Bridge
DELAY=0
PERSISTENT_DHCLIENT=yes
EOF

cat <<EOF >/etc/sysconfig/network-scripts/ifcfg-em1
DEVICE=em1
ONBOOT=yes
NM_CONTROLLED=no
# BOOTPROTO=dhcp
TYPE=Ethernet
BRIDGE=br0
EOF
```
1. Fix SELinux context:
```
chcon -u system_u /etc/sysconfig/network-scripts/ifcfg-*
```
1. Enable legacy network setup and remove NetworkManager.
```
systemctl enable network.service
yum erase NetworkManager NetworkManager-glib
reboot
```

You could get by without rebooting here, but I find it useful to make sure that the network comes up properly after a reboot before proceeding with the rest of the configuration.  If you do not want to reboot, you need to do this before removing NetworkManager:
```
systemctl stop NetworkManager; systemctl start network
```

## Kernel Parameter Configuration

1. These kernel parameters are required for postgresql to function properly.  This was addressed in one way in [EUCA-2034](https://eucalyptus.atlassian.net/browse/EUCA-2034), but that solution is not acceptable in Fedora, so it must be done manually.
```
cat <<EOF >/etc/sysctl.d/eucalyptus-cloud
kernel.shmmax = 536870912
kernel.sem = 250 32000 32 32768
EOF
sysctl -p /etc/sysctl.d/eucalyptus-cloud
```

## Eucalyptus Package Installation

1. edit /etc/yum.repos.d/fedora-updates-testing.repo (For [FEDORA-2012-11983](https://admin.fedoraproject.org/updates/FEDORA-2012-11983/)):
   * set `enabled=1`
   * set `includepkgs=mule*,*wso2*`
1. Add eucalyptus repo. This will not be necessary once eucalyptus is accepted into Fedora.
```
cat <<EOF >/etc/yum.repos.d/eucalyptus.repo
[euca]
name=euca
baseurl=http://downloads.eucalyptus.com/devel/packages/fedora-18/x86_64/
gpgcheck=0
enabled=1
EOF
yum install eucalyptus-cloud eucalyptus-sc eucalyptus-walrus eucalyptus-cc eucalyptus-nc
```

## Other System Modifications

* Link make an unversioned libmod_rampart.so (I'm unsure whether this is a eucalyptus or wso2 issues):
```
ln -s libmod_rampart.so /usr/lib64/wso2-axis2/modules/rampart/libmod_rampart.so.0
```

## Cloud Initialization

```
su eucalyptus -c "euca_conf --initialize"
systemctl start eucalyptus-cloud
```

The first startup can take a few minutes while the databases are loaded.  You can `tail -f /var/log/eucalyptus/cloud-exhaust.log` and watch for the "Mule ESB and Integration Platform" box to appear, which is a good indication that the web services stack is fully loaded.

## Component Registration

* Get credentials - Note that if you don't wait long enough, you may get an error or a zero-length file here.  If so, remove the empty file and try again.
```
euca_conf --get-credentials admin.zip
unzip admin.zip
source ./eucarc
```

* Verify that the java-based services are all in ENABLED state (except for the Dns service, which will likely be "broken" due to conflict with dnsmasq):
```
euca-describe-services
```
* Register and start the remaining components
```
euca_conf --register-walrus -C walrus -P walrus -H <host-ip-address>
euca_conf --register-sc -C SC00 -P PARTI00 -H <host-ip-address>
euca_conf --register-cluster -C CC00 -P PARTI00 -H <host-ip-address>
systemctl start eucalyptus-cc
euca_conf --register-nodes <host-ip-address>
systemctl start eucalyptus-nc
```

## Verify compute resources

At this point, if all services are working properly, you should be able to see available resources:
```
euca-describe-availability-zones verbose
```

If you do not, it is likely that either the CC or NC has not started properly.  Here are a couple of known issues:

* If you do not reboot the system or reload the kvm modules after installing the qemu packages, then /dev/kvm may have the wrong permissions.  You can fix this by reloading the modules:
```
rmmod kvm_intel
rmmod kvm
modprobe kvm_intel
```
* There is an issue with libvirt in F18 currently:  https://bugzilla.redhat.com/show_bug.cgi?id=852549
Until this is fixed, you may need to manually download and install a new libvirt from koji.  Here is what I did:
```
mkdir /tmp/libvirt
cd /tmp/libvirt
for x in $( rpm -qa --qf "%{NAME} " '*libvirt*' ); do 
  wget -nd http://kojipkgs.fedoraproject.org//packages/libvirt/0.10.1/2.fc18/x86_64/${x}-0.10.1-2.fc18.x86_64.rpm;
done
yum install *.rpm
systemctl restart libvirtd
```

## Image Registration

```
wget -nd http://s3.amazonaws.com/fedora17-201209050306/initramfs-3.5.3-1.fc17.x86_64.img
wget -nd http://s3.amazonaws.com/fedora17-201209050306/ks-fedora17-201209050306.img.gz
wget -nd http://s3.amazonaws.com/fedora17-201209050306/vmlinuz-3.5.3-1.fc17.x86_64
euca-bundle-image  --kernel true -i vmlinuz-3.5.3-1.fc17.x86_64
euca-upload-bundle -m /tmp/vmlinuz-3.5.3-1.fc17.x86_64.manifest.xml -b fedora17
EKI_ID=$( euca-register fedora17/vmlinuz-3.5.3-1.fc17.x86_64.manifest.xml )
euca-bundle-image --ramdisk true -i initramfs-3.5.3-1.fc17.x86_64.img 
euca-upload-bundle -m /tmp/initramfs-3.5.3-1.fc17.x86_64.img.manifest.xml -b fedora17
ERI_ID=$( euca-register fedora17/initramfs-3.5.3-1.fc17.x86_64.img.manifest.xml )
gunzip ks-fedora17-201209050306.img.gz; 
euca-bundle-image --kernel $EKI_ID --ramdisk $ERI_ID -i ks-fedora17-201209050306.img
euca-upload-bundle -m /tmp/ks-fedora17-201209050306.img.manifest.xml -b fedora17
EMI_ID=$( euca-register fedora17/ks-fedora17-201209050306.img.manifest.xml )
```

## Test Instance Launch

```
euca-add-keypair foobar > foobar.priv
chmod 600 foobar.priv
euca-run-instances -k foobar -t m1.large $EMI_ID
```

Note: The instance boot time is _really_ slow, partly due to SELinux file relabelling.  Also, since I'm testing system mode here, cloud-init fails, so you have to log in as root rather than ec2-user.  It is possible to set up metadata service access for system mode, but I haven't done the iptables rules for that yet.