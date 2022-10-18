Making a Ceph-Ansible Checkout
==============================

Invoking Ceph-Ansible
=====================

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

   kayobe$ kayobe overcloud host configure -t lvm -kt none -l storage-0 -kl storage-0

Before running Ceph-Ansible, also remove vestigial state directory
from ``/var/lib/ceph/osd`` for the purged OSD

Reapply Ceph-Asnible in the usual manner.

.. note::

   Ceph-Ansible runs can fail to complete if there are background activities
   such as backfilling underway when the Ceph-Ansible playbook is invoked.
