.. include:: vars.rst

=============================
Hardware Inventory Management
=============================

At its lowest level, hardware inventory is managed in the Bifrost service (see :ref:`accessing-the-bifrost-service`).

Reconfiguring Control Plane Hardware
------------------------------------

If a server's hardware or firmware configuration is changed, it should be
re-inspected in Bifrost before it is redeployed into service. A single server
can be reinspected like this (for a host named |hypervisor_hostname|):

.. code-block:: console
   :substitutions:

   kayobe# kayobe overcloud hardware inspect --limit |hypervisor_hostname|

.. _enrolling-new-hypervisors:

Enrolling New Hypervisors
-------------------------

New hypervisors can be added to the Bifrost inventory by using its discovery
capabilities. Assuming that new hypervisors have IPMI enabled and are
configured to network boot on the provisioning network, the following commands
will instruct them to PXE boot. The nodes will boot on the Ironic Python Agent
kernel and ramdisk, which is configured to extract hardware information and
send it to Bifrost. Note that IPMI credentials can be found in the encrypted
file located at ``${KAYOBE_CONFIG_PATH}/secrets.yml``.

.. code-block:: console
   :substitutions:

   bifrost# ipmitool -I lanplus -U |ipmi_username| -H |hypervisor_hostname|-ipmi chassis bootdev pxe

If node is are off, power them on:

.. code-block:: console
   :substitutions:

   bifrost# ipmitool -I lanplus -U |ipmi_username| -H |hypervisor_hostname|-ipmi power on

If nodes is on, reset them:

.. code-block:: console
   :substitutions:

   bifrost# ipmitool -I lanplus -U |ipmi_username| -H |hypervisor_hostname|-ipmi power reset

Once node have booted and have completed introspection, they should be visible
in Bifrost:

.. code-block:: console
   :substitutions:

   bifrost-7.2.1]# openstack baremetal node list --provision-state enroll
   +--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+
   | UUID                                 | Name                  | Instance UUID | Power State | Provisioning State | Maintenance |
   +--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+
   | da0c61af-b411-41b9-8909-df2509f2059b | |hypervisor_hostname| | None          | power off   | enroll             | False       |
   +--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+

After editing ``${KAYOBE_CONFIG_PATH}/overcloud.yml`` to add these new hosts to
the correct groups, import them in Kayobe's inventory with:

.. code-block:: console

   kayobe# kayobe overcloud inventory discover

We can then provision and configure them:

.. code-block:: console
   :substitutions:

   kayobe# kayobe overcloud provision --limit |hypervisor_hostname|
   kayobe# kayobe overcloud host configure --limit |hypervisor_hostname| --kolla-limit |hypervisor_hostname|
   kayobe# kayobe overcloud service deploy --limit |hypervisor_hostname| --kolla-limit |hypervisor_hostname|

Replacing a Failing Hypervisor
------------------------------

To replace a failing hypervisor, proceed as follows:

* :ref:`Disable the hypervisor to avoid scheduling any new instance on it <taking-a-hypervisor-out-of-service>`
* :ref:`Evacuate all instances <evacuating-all-instances>`
* :ref:`Set the node to maintenance mode in Bifrost <set-bifrost-maintenance-mode>`
* Physically fix or replace the node
* It may be necessary to reinspect the node if hardware was changed (this will require deprovisioning and reprovisioning)
* If the node was replaced or reprovisioned, follow :ref:`enrolling-new-hypervisors`

.. _evacuating-all-instances:

Evacuating all instances
------------------------

.. code-block:: console
   :substitutions:

   admin# nova host-evacuate-live |hypervisor_hostname|

You should now check the status of all the instances that were running on that
hypervisor. They should all show the status RUNNING. This can be verified with:

.. code-block:: console

   admin# openstack server show <instance uuid>

Troubleshooting
+++++++++++++++

Servers that have been shut down
********************************

If there are any instances that are SHUTOFF they won’t be migrated, but you can
use ``nova host-servers-migrate`` for them once the live migration is finished.

Also if a VM does heavy memory access, it may take ages to migrate (Nova tries
to incrementally increase the expected downtime, but is quite conservative).
You can use ``nova live-migration-force-complete <instance_uuid>
<migration_id>`` to trigger the final move.

You get the migration ID via ``nova server-migration-list <instance_uuid>``.

For more details see:
http://www.danplanet.com/blog/2016/03/03/evacuate-in-nova-one-command-to-confuse-us-all/

Flavors have changed
********************

If the size of the flavors has changed, some instances will also fail to
migrate as the process needs manual confirmation. You can do this with:

.. code-block:: console

   openstack # openstack server resize confirm <instance-uuid>

The symptom to look out for is that the server is showing a status of ``VERIFY
RESIZE`` as shown in this snippet of ``openstack server show <instance-uuid>``:

.. code-block:: console

   | status | VERIFY_RESIZE |

.. _set-bifrost-maintenance-mode:

Set maintenance mode on a node in Bifrost
+++++++++++++++++++++++++++++++++++++++++

For example, to put |hypervisor_hostname| into maintenance:

.. code-block:: console
   :substitutions:

   seed# sudo docker exec -it bifrost_deploy /bin/bash
   (bifrost-deploy)[root@seed bifrost-base]# OS_CLOUD=bifrost openstack baremetal node maintenance set |hypervisor_hostname|

.. ifconfig:: deployment['ceph_managed']

   .. include:: hardware_inventory_management_ceph.rst

.. _unset-bifrost-maintenance-mode:

Unset maintenance mode on a node in Bifrost
+++++++++++++++++++++++++++++++++++++++++++

For example, to take |hypervisor_hostname| out of maintenance:

.. code-block:: console
   :substitutions:

   seed# sudo docker exec -it bifrost_deploy /bin/bash
   (bifrost-deploy)[root@seed bifrost-base]# OS_CLOUD=bifrost openstack baremetal node maintenance unset |hypervisor_hostname|

.. ifconfig:: deployment['ceph_managed']

   .. include:: hardware_inventory_management_ceph.rst
