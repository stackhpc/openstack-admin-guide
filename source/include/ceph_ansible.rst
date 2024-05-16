Making a Ceph-Ansible Checkout
==============================

Invoking Ceph-Ansible
=====================

Removing a Failed Ceph Drive
============================

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

Replacing a Failed Ceph Drive
=============================

Once an OSD has been identified as having a hardware failure,
the affected drive will need to be replaced.

.. note::

   Hot-swapping a failed device will change the device enumeration
   and this could confuse the device addressing in Kayobe LVM
   configuration.

   In kayobe-config, use ``/dev/disk/by-path`` device references to
   avoid this issue.

   Alternatively, always reboot a server when swapping drives.

If rebooting a Ceph node, first set ``noout`` to prevent excess data
movement:

.. code-block:: console

   ceph# ceph osd set noout

Apply LVM configuration using Kayobe for the replaced device (here on ``storage-0``):

.. code-block:: console

   kayobe$ kayobe overcloud host configure -t lvm -l storage-0

Before running Ceph-Ansible, also remove vestigial state directory
from ``/var/lib/ceph/osd`` for the purged OSD

Reapply Ceph-Asnible in the usual manner.

.. note::

   Ceph-Ansible runs can fail to complete if there are background activities
   such as backfilling underway when the Ceph-Ansible playbook is invoked.
