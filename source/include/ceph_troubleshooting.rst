Investigating a Failed Ceph Drive
---------------------------------

After deployment, when a drive fails it may cause OSD crashes in Ceph.
If Ceph detects crashed OSDs, it will go into `HEALTH_WARN` state.
Ceph can report details about failed OSDs by running:

.. code-block:: console

   ceph# ceph health detail

A failed OSD will also be reported as down by running:

.. code-block:: console

   ceph# ceph osd tree

Note the ID of the failed OSD.

The failed hardware device is logged by the Linux kernel:

.. code-block:: console

   storage-0# dmesg -T

Cross-reference the hardware device and OSD ID to ensure they match.
(Using `pvs` and `lvs` may help make this connection).

Removing a Failed Ceph Drive
----------------------------

If a drive is verified dead, stop and eject the osd (eg. `osd.4`)
from the cluster:

.. code-block:: console

   storage-0# systemctl stop ceph-osd@4.service
   storage-0# systemctl disable ceph-osd@4.service
   ceph# ceph osd out osd.4

.. ifconfig:: deployment['ceph_ansible']

   Before running Ceph-Ansible, also remove vestigial state directory
   from `/var/lib/ceph/osd` for the purged OSD, for example for OSD ID 4:

   .. code-block:: console

      storage-0# rm -rf /var/lib/ceph/osd/ceph-4

Remove Ceph OSD state for the old OSD, here OSD ID `4` (we will
backfill all the data when we reintroduce the drive).

.. code-block:: console

   ceph# ceph osd purge --yes-i-really-mean-it 4

Unset noout for osds when hardware maintenance has concluded - eg.
while waiting for the replacement disk:

.. code-block:: console

   ceph# ceph osd unset noout

Inspecting a Ceph Block Device for a VM
---------------------------------------

To find out what block devices are attached to a VM, go to the hypervisor that
it is running on (an admin-level user can see this from ``openstack server
show``).

On this hypervisor, enter the libvirt container:

.. code-block:: console
   :substitutions:

   |hypervisor_hostname|# docker exec -it nova_libvirt /bin/bash

Find the VM name using libvirt:

.. code-block:: console
   :substitutions:

   (nova-libvirt)[root@|hypervisor_hostname| /]# virsh list
    Id    Name                State
   ------------------------------------
    1     instance-00000001   running

Now inspect the properties of the VM using ``virsh dumpxml``:

.. code-block:: console
   :substitutions:

   (nova-libvirt)[root@|hypervisor_hostname| /]# virsh dumpxml instance-00000001 | grep rbd
         <source protocol='rbd' name='|nova_rbd_pool|/51206278-e797-4153-b720-8255381228da_disk'>

On a Ceph node, the RBD pool can be inspected and the volume extracted as a RAW
block image:

.. code-block:: console
   :substitutions:

   ceph# rbd ls |nova_rbd_pool|
   ceph# rbd export |nova_rbd_pool|/51206278-e797-4153-b720-8255381228da_disk blob.raw

The raw block device (blob.raw above) can be mounted using the loopback device.

Inspecting a QCOW Image using LibGuestFS
----------------------------------------

The virtual machine's root image can be inspected by installing
libguestfs-tools and using the guestfish command:

.. code-block:: console

   ceph# export LIBGUESTFS_BACKEND=direct
   ceph# guestfish -a blob.qcow
   ><fs> run
    100% [XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX] 00:00
   ><fs> list-filesystems
   /dev/sda1: ext4
   ><fs> mount /dev/sda1 /
   ><fs> ls /
   bin
   boot
   dev
   etc
   home
   lib
   lib64
   lost+found
   media
   mnt
   opt
   proc
   root
   run
   sbin
   srv
   sys
   tmp
   usr
   var
   ><fs> quit
