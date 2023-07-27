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

   bifrost# baremetal node list --provision-state enroll
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

To deprovision an existing hypervisor, run:

.. code-block:: console
   :substitutions:

   kayobe# kayobe overcloud deprovision --limit |hypervisor_hostname|

.. warning::

   Always use ``--limit`` with ``kayobe overcloud deprovision`` on a production
   system. Running this command without a limit will deprovision all overcloud
   hosts.

.. _evacuating-all-instances:

Evacuating all instances
------------------------

.. code-block:: console
   :substitutions:

   admin# nova host-evacuate-live |hypervisor_hostname|

You should now check the status of all the instances that were running on that
hypervisor. They should all show the status ACTIVE. This can be verified with:

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

   seed# docker exec -it bifrost_deploy /bin/bash
   (bifrost-deploy)[root@seed bifrost-base]# OS_CLOUD=bifrost baremetal node maintenance set |hypervisor_hostname|

.. _unset-bifrost-maintenance-mode:

Unset maintenance mode on a node in Bifrost
+++++++++++++++++++++++++++++++++++++++++++

For example, to take |hypervisor_hostname| out of maintenance:

.. code-block:: console
   :substitutions:

   seed# docker exec -it bifrost_deploy /bin/bash
   (bifrost-deploy)[root@seed bifrost-base]# OS_CLOUD=bifrost baremetal node maintenance unset |hypervisor_hostname|

Detect hardware differences with cardiff
----------------------------------------

Hardware information captured during the Ironic introspection process can be
analysed to detect hardware differences, such as mismatches in firmware
versions or missing storage devices. The cardiff tool can be used for this
purpose. It was developed as part of the `Python hardware package
<https://pypi.org/project/hardware/>`__, but was removed from release 0.25. The
`mungetout utility <https://github.com/stackhpc/mungetout/>`__ can be used to
convert Ironic introspection data into a format that can be fed to cardiff.

The following steps are used to install cardiff and mungetout:

.. code-block:: console
   :substitutions:

   kayobe# virtualenv |base_path|/venvs/cardiff
   kayobe# source |base_path|/venvs/cardiff/bin/activate
   kayobe# pip install -U pip
   kayobe# pip install git+https://github.com/stackhpc/mungetout.git@feature/kayobe-introspection-save
   kayobe# pip install 'hardware==0.24'

Extract introspection data from Bifrost with Kayobe. JSON files will be created
into ``${KAYOBE_CONFIG_PATH}/overcloud-introspection-data``:

.. code-block:: console
   :substitutions:

   kayobe# source |base_path|/venvs/kayobe/bin/activate
   kayobe# source |base_path|/src/kayobe-config/kayobe-env
   kayobe# kayobe overcloud introspection data save

The cardiff utility can only work if the ``extra-hardware`` collector was used,
which populates a ``data`` key in each node JSON file. Remove any that are
missing this key:

.. code-block:: console
   :substitutions:

   kayobe# for file in |base_path|/src/kayobe-config/overcloud-introspection-data/*; do if [[ $(jq .data $file) == 'null' ]]; then rm $file; fi; done

Cardiff identifies each unique system by its serial number. However, some
high-density multi-node systems may report the same serial number for multiple
systems (this has been seen on Supermicro hardware). The following script will
replace the serial number used by Cardiff by the node name captured by LLDP on
the first network interface. If this node name is missing, it will append a
short UUID string to the end of the serial number.

.. code-block:: python

   import json
   import sys
   import uuid

   with open(sys.argv[1], "r+") as f:
       node = json.loads(f.read())

       serial = node["inventory"]["system_vendor"]["serial_number"]
       try:
           new_serial = node["all_interfaces"]["eth0"]["lldp_processed"]["switch_port_description"]
       except KeyError:
           new_serial = serial + "-" + str(uuid.uuid4())[:8]

       new_data = []
       for e in node["data"]:
           if e[0] == "system" and e[1] == "product" and e[2] == "serial":
               new_data.append(["system", "product", "serial", new_serial])
           else:
               new_data.append(e)
       node["data"] = new_data

       f.seek(0)
       f.write(json.dumps(node))
       f.truncate()

Apply this Python script on all generated JSON files:

.. code-block:: console
   :substitutions:

   kayobe# for file in ~/src/kayobe-config/overcloud-introspection-data/*; do python update-serial.py $file; done

Convert files into the format supported by cardiff:

.. code-block:: console
   :substitutions:

   source |base_path|/venvs/cardiff/bin/activate
   mkdir -p |base_path|/cardiff-workspace
   rm -rf |base_path|/cardiff-workspace/extra*
   cd |base_path|/cardiff-workspace/
   m2-extract |base_path|/src/kayobe-config/overcloud-introspection-data/*.json

.. note::

   The ``m2-extract`` utility needs to work in an empty folder. Delete the
   ``extra-hardware``, ``extra-hardware-filtered`` and ``extra-hardware-json``
   folders before executing it again.

We are now ready to compare node hardware. The following command will compare
all known nodes, which may include multiple generations of hardware. Replace
``*.eval`` by a stricter globbing expression or by a list of files to compare a
smaller group.

.. code-block:: console

   hardware-cardiff -I ipmi -p 'extra-hardware/*.eval'

Since the output can be verbose, it is recommended to pipe it to a terminal
pager or redirect it to a file. Cardiff will display groups of identical nodes
based on various hardware characteristics, such as system model, BIOS version,
CPU or network interface information, or benchmark results gathered by the
``extra-hardware`` collector during the initial introspection process.

.. ifconfig:: deployment['ceph_managed']

   .. include:: hardware_inventory_management_ceph.rst
