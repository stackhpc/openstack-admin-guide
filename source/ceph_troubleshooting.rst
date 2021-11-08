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
