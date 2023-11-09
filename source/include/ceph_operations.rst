

Replacing drive
---------------

See upstream documentation:
https://docs.ceph.com/en/quincy/cephadm/services/osd/#replacing-an-osd

In case where disk holding DB and/or WAL fails, it is necessary to recreate
(using replacement procedure above) all OSDs that are associated with this
disk - usually NVMe drive. The following single command is sufficient to
identify which OSDs are tied to which physical disks:

.. code-block:: console

   ceph# ceph device ls

Host maintenance
----------------

https://docs.ceph.com/en/quincy/cephadm/host-management/#maintenance-mode

Upgrading
---------

https://docs.ceph.com/en/quincy/cephadm/upgrade/
