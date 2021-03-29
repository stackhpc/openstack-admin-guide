.. include:: vars.rst

=======================
Full Shutdown Procedure
=======================

In case a full shutdown of the system is required, we advise to use the
following order:

* Perform a graceful shutdown of all virtual machine instances
* Stop Ceph (if applicable)
* Put all nodes into maintenance mode in Bifrost
* Shut down compute nodes
* Shut down monitoring node
* Shut down network nodes (if separate from controllers)
* Shut down controllers
* Shut down Ceph nodes (if applicable)
* Shut down seed VM
* Shut down Ansible control host

Virtual Machines shutdown
-------------------------

Contact Openstack users to stop their virtual machines gracefully,
If that is not possible shut down VMs using openstack CLI as admin user:

.. code-block:: bash

   for i in `openstack server list --all-projects -c ID -f value` ; \
   do openstack server stop $i ; done


.. ifconfig:: deployment['ceph_managed']

   Stop Ceph
   ---------
   Procedure based on `Red Hat documentation <https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/administration_guide/understanding-process-management-for-ceph#powering-down-and-rebooting-a-red-hat-ceph-storage-cluster_admin>`__ 

   - Stop the Ceph clients from using any Ceph resources (RBD, RADOS Gateway, CephFS)
   - Check if cluster is in healthy state

   .. code-block:: bash

      ceph status 

   - Stop CephFS (if applicable)

   Stop CephFS cluster by reducing the number of ranks to 1, setting the cluster_down flag, and then failing the last rank.

   .. code-block:: bash

      ceph fs set FS_NAME max_mds 1
      ceph mds deactivate FS_NAME:1 # rank 2 of 2
      ceph status # wait for rank 1 to finish stopping
      ceph fs set FS_NAME cluster_down true
      ceph mds fail FS_NAME:0

   Setting the cluster_down flag prevents standbys from taking over the failed rank.

   - Set the noout, norecover, norebalance, nobackfill, nodown and pause flags.

   .. code-block:: bash

      ceph osd set noout
      ceph osd set norecover
      ceph osd set norebalance
      ceph osd set nobackfill
      ceph osd set nodown
      ceph osd set pause

   - Shut down the OSD nodes one by one:

   .. code-block:: bash

      systemctl stop ceph-osd.target

   - Shut down the monitor/manager nodes one by one:

   .. code-block:: bash

      systemctl stop ceph.target

Set Bifrost maintenance mode
----------------------------

Set maintenance mode in bifrost to prevent nodes from automatically
powering back on

.. code-block:: bash

   bifrost# for i in `openstack --os-cloud bifrost baremetal node list -c UUID -f value` ; \
   do openstack --os-cloud bifrost baremetal node maintenance set --reason full-shutdown $i ; done

Shut down nodes
---------------

Shut down nodes one at a time gracefully using:

.. code-block:: bash

   systemctl poweroff

Shut down the seed VM
---------------------

Shut down seed vm on ansible control host gracefully using:

.. code-block:: bash
   :substitutions:

   ssh stack@|seed_name| sudo systemctl poweroff
   virsh shutdown |seed_name|

.. _full-power-on:

Full Power on Procedure
-----------------------

* Start ansible control host and seed vm
* Remove nodes from maintenance mode in bifrost
* Recover MariaDB cluster
* Start Ceph (if applicable)
* Check that all docker containers are running
* Check Kibana for any messages with log level ERROR or equivalent

Start Ansible Control Host
--------------------------

The Ansible control host is not enrolled in Bifrost and will have to be powered
on manually.

Start Seed VM
-------------

The seed VM (and any other service VM) should start automatically when the seed
hypervisor is powered on. If it does not, it can be started with:

.. code-block:: bash

   virsh start seed-0

Unset Bifrost maintenance mode
------------------------------

Unsetting maintenance mode in bifrost should automatically power on the nodes

.. code-block:: bash

   bifrost# for i in `openstack --os-cloud bifrost baremetal node list -c UUID -f value` ; \
   do openstack --os-cloud bifrost baremetal node maintenance unset $i ; done

Recover MariaDB cluster
-----------------------

If all of the servers were shut down at the same time, it is necessary to run a
script to recover the database once they have all started up. This can be done
with the following command:

.. code-block:: bash

   kayobe# kayobe overcloud database recover

.. ifconfig:: deployment['ceph_managed']

   Start Ceph
   ----------
   Procedure based on `Red Hat documentation <https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/administration_guide/understanding-process-management-for-ceph#powering-down-and-rebooting-a-red-hat-ceph-storage-cluster_admin>`__

   - Start monitor/manager nodes:

   .. code-block:: bash

      systemctl start ceph.target

   - Start the OSD nodes:

   .. code-block:: bash

      systemctl start ceph-osd.target

   - Wait for all the nodes to come up

   - Unset the noout, norecover, norebalance, nobackfill, nodown and pause flags

   .. code-block:: bash

      ceph osd unset noout
      ceph osd unset norecover
      ceph osd unset norebalance
      ceph osd unset nobackfill
      ceph osd unset nodown
      ceph osd unset pause

   - Start CephFS (if applicable)

   CephFS cluster must be brought back up by setting the cluster_down flag to false

   .. code-block:: bash

      ceph fs set FS_NAME cluster_down false

   - Verify ceph cluster status

   .. code-block:: bash

      ceph status
