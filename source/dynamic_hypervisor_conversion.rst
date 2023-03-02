.. include:: vars.rst

=============================
Dynamic Hypervisor Conversion
=============================
This section describes procedures for changing the role of nodes in the cluster.
This includes converting a hypervisor to a bare metal compute node and vice versa.

.. note::
    It is recommended that this process is carried out inside of ``tmux`` as window one can be used for OpenStack client and a second window for Kayobe commands. The Openstack client can be setup following :doc:`working_with_openstack` and make sure to install the ironic client and acquire admin credentials. To setup a Kayobe environment please review :doc:`working_with_kayobe`.

Hypervisor conversion
=====================

This procedure will enable the conversion of baremetal nodes into hypervisors. This would be required if the current number of hypervisors is not meeting demand. The steps outlined below cover all required changes and commands within OpenStack and the Kayobe config.

OpenStack commands
------------------

#. Choose a node to convert. Check the node the node’s ``resource_class`` is equal to ``baremetal``. If not, pick another node::

    openstack baremetal node show <node> -f value -c resource_class

#. Check if there is a Nova instance currently running on the node you wish to convert::

    openstack baremetal node show <node> -f value -c instance_uuid

#. If an instance is present, check if it can be safely deleted, and delete if so. If not, pick another node::

    openstack server delete <instance_uuid>

#. Wait for the node to finish deprovisioning and to move though the process to an ``available`` provision state. You can check the state using ``baremetal node show``::

    openstack baremetal node show <node>

#. Set the node to enter maintenance::

    openstack baremetal node maintenance set <node> --reason "Converting to hypervisor"

#. Rename the baremetal node, change its name to conform to hypervisor naming standards (hb-hypervisor<ID>) and change resource class::

    openstack baremetal node set <node> --name <hypervisor> --resource-class hypervisor

#. Finally unset maintenance mode on the baremetal compute node::

    openstack baremetal node maintenance unset <hypervisor>

Kayobe commands
---------------

#. Rename the baremetal compute node by editing ``tools/hosts.csv``::

    sed -i -e 's/^<node>/<hypervisor>/' ~/deployment/src/kayobe-config/tools/hosts.csv

#. Install ``pandas`` using ``pip`` for use by ``gen-switch-interfaces.py``::

    pip3 install pandas

#. Update the switch port configuration using ``tools/gen-switch-interfaces.py``::

    ~/deployment/src/kayobe-config/tools/gen-switch-interfaces.py ~/deployment/src/kayobe-config/tools/hosts.csv \
    --out-path ${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/inventory/host_vars

#. Comment out the baremetal compute node entry in the ``baremetal-standard`` group by editing ``etc/kayobe/environments/habrok/inventory/hosts``::

    sed -e '/<node>/ s/^#*/#/' -i ~/deployment/src/kayobe-config/etc/kayobe/environments/habrok/inventory/hosts

#. Add the hypervisor to the ``compute-standard`` group by editing ``etc/kayobe/environments/habrok/inventory/hosts``::

    [compute-standard]
    <hypervisor> ipmi_address=<hypervisor_ipmi_address>

#. Check the configuration with ``git diff --name-only``. You should see similar output the example output provided below::

    etc/kayobe/environments/habrok/inventory/host_vars/hb-sw-leaf01-h04.yml
    etc/kayobe/environments/habrok/inventory/host_vars/hb-sw-leaf02-h04.yml
    etc/kayobe/environments/habrok/inventory/host_vars/hb-sw-mgmt01-h04/switch-config.yml
    etc/kayobe/environments/habrok/inventory/hosts
    etc/kayobe/environments/habrok/network-allocation.yml
    tools/hosts.csv

#. Reconfigure leaf switch ports::

    kayobe physical network configure --group leaf-switches

OpenStack commands
------------------

#. Get the Ironic node UUID::

    openstack baremetal node show <hypervisor> -f value -c uuid

#. Create Nova instance for the hypervisor::

    openstack server create <hypervisor> \
        --flavor hypervisor --image overcloud-ubuntu-focal \
        --network provision-overcloud --key-name admin \
        --availability-zone nova::<ironic node UUID> \
        --os-compute-api-version 2.74

#. Query the instance until it has been assigned an IP address on the ``provision-overcloud`` network and the ``status`` is ``ACTIVE``::

    openstack server show <hypervisor>

Kayobe commands
---------------

#. Add the IP address to the ``provision_oc_ips`` dict by editing ``~/deployment/src/kayobe-config/etc/kayobe/environments/habrok/network-allocation.yml``::

    provision_oc_ips:
      <other hosts>
      <hypervisor>: <ip_address>

#. Reconfigure the leaf switch ports to apply trunked VLANs::

    kayobe physical network configure --group leaf-switches

#. Perform a host configuration::

    kayobe overcloud host configure -l compute

#. Deploy overcloud services onto the hypervisor::

    kayobe overcloud service deploy -kl compute

#. Commit the changes to the ``kayobe-config`` repository and open a pull request::

    git add -u
    git commit -m "feat: convert baremetal to hypervisor"

Baremetal conversion
====================

This procedure will convert hypervisors back into baremetal instances effectively reverting the procedure outlined above. This would be required if the demand for hypervisors has dropped allowing for fewer hypervisors needed. The steps outlined below cover all required changes and commands within OpenStack and the Kayobe config.

Kayobe commands
---------------

#. Disable nova compute services on the hypervisor you are converting into a baremetal node::

    kayobe playbook run ${KAYOBE_CONFIG_PATH}/ansible/nova-compute-disable.yml --limit <hypervisor>

#. Drain the hypervisor of all virtual machines::

    kayobe playbook run ${KAYOBE_CONFIG_PATH}/ansible/nova-compute-drain.yml --limit <hypervisor>

#. Delete hypervisor from inventory hosts::

    sed -e '/<hypervisor>/d' -i ${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/inventory/hosts

#. Add the baremetal entry into the inventory hosts file by uncommenting the line containing it::

    sed -i '/<node>/s/^#//' ${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/inventory/hosts

#. Remove all IP allocations for the hypervisor from ``network-allocation.yml``::

    sed -e '/<hypervisor>/d' -i  ${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/network-allocation.yml

#. Rename the hypervisor node by editing ``tools/hosts.csv``::

    sed -i -e 's/^<hypervisor>/<node>/' ~/deployment/src/kayobe-config/tools/hosts.csv

#. Update the switch port configuration using ``tools/gen-switch-interfaces.py``::

    ~/deployment/src/kayobe-config/tools/gen-switch-interfaces.py \
        ~/deployment/src/kayobe-config/tools/hosts.csv \
        --out-path ${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/inventory/host_vars

#. Reconfigure leaf switch ports::

    kayobe physical network configure --group leaf-switches

#. Check the configuration with ``git diff --name-only``. You should see similar output the example output provided below::

    etc/kayobe/environments/habrok/inventory/host_vars/hb-sw-leaf01-h04.yml
    etc/kayobe/environments/habrok/inventory/host_vars/hb-sw-leaf02-h04.yml
    etc/kayobe/environments/habrok/inventory/host_vars/hb-sw-mgmt01-h04/switch-config.yml
    etc/kayobe/environments/habrok/inventory/hosts
    etc/kayobe/environments/habrok/network-allocation.yml
    tools/hosts.csv

#. Commit the changes to the ``kayobe-config`` repository and open a pull request::

    git add -u
    git commit -m "feat: convert hypervisor to baremetal"

OpenStack commands
------------------

#. Remove the OpenStack compute service from the disabled and drained hypervisor::

    openstack compute service delete <hypervisor>

#. Get the ``instance_uuid`` of the instance running on the hypervisor::

    openstack baremetal node show <hypervisor> -f value -c instance_uuid

#. Delete the hypervisor instance running on the hypervisor::

    openstack server delete <instance_uuid>

#. Wait for the node to finish deprovisioning and to move though the process to an ``available`` provision state. You can check the state using ``baremetal node show``::

    watch -n 10 -d=permanent openstack baremetal node show <hypervisor> \
        -f yaml -c provision_state -c provision_updated_at

#. Find the OVN controller agent and OVN Metadata agent associated with the hypervisor, make note of the UUID of the OVN Metadata agent::

    openstack network agent list --fit-width

#. Delete both the OVN controller agent and OVN Metadata agent associated with the hypervisor

    openstack network agent delete <hypervisor>
    openstack network agent delete <metadata-uuid>

#. Set the node to enter maintenance::

    openstack baremetal node maintenance set <hypervisor> --reason "Converting to baremetal"

#. Rename the hypervisor node, change its name to conform to baremetal node naming standards and change resource class::

    openstack baremetal node set <hypervisor> --name <node> --resource-class baremetal

#. Finally unset maintenance mode on the baremetal compute node::

    openstack baremetal node maintenance unset <node>
