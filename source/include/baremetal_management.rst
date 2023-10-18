.. _ironic-node-lifecycle:

Ironic node life cycle
----------------------

The deployment process is documented in the `Ironic User Guide <https://docs.openstack.org/ironic/yoga/user/index.html>`__.
The |project_name| OpenStack deployment uses the
`direct deploy method <https://docs.openstack.org/ironic/yoga/user/index.html#example-1-pxe-boot-and-direct-deploy-process>`__.

The Ironic state machine can be found `here <https://docs.openstack.org/ironic/latest/user/states.html>`__. The rest of
this documentation refers to these states and assumes that you have familiarity.

High level overview of state transitions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following section attempts to describe the state transitions for various Ironic operations at a high level.
It focuses on trying to describe the steps where dynamic switch reconfiguration is triggered.
For a more detailed overview, refer to the :ref:`ironic-node-lifecycle` section.

Provisioning
~~~~~~~~~~~~

Provisioning starts when an instance is created in Nova using a bare metal flavor.

- Node starts in the available state (available)
- User provisions an instance (deploying)
- Ironic will switch the node onto the provisioning network (deploying)
- Ironic will power on the node and will await a callback (wait-callback)
- Ironic will image the node with an operating system using the image provided at creation (deploying)
- Ironic switches the node onto the tenant network(s) via neutron (deploying)
- Transition node to active state (active)

.. _baremetal-management-deprovisioning:

Deprovisioning
~~~~~~~~~~~~~~

Deprovisioning starts when an instance created in Nova using a bare metal flavor is destroyed.

.. ifconfig:: deployment['ironic_automated_cleaning']

   Automated cleaning is enabled, and occurs when nodes are deprovisioned.

   - Node starts in active state (active)
   - User deletes instance (deleting)
   - Ironic will remove the node from any tenant network(s) (deleting)
   - Ironic will switch the node onto the cleaning network (deleting)
   - Ironic will power on the node and will await a callback (clean-wait)
   - Node boots into Ironic Python Agent and issues callback, Ironic starts cleaning (cleaning)
   - Ironic removes node from cleaning network (cleaning)
   - Node transitions to available (available)

.. ifconfig:: not deployment['ironic_automated_cleaning']

   Automated cleaning is currently disabled.

   - Node starts in active state (active)
   - User deletes instance (deleting)
   - Ironic will remove the node from any tenant network(s) (deleting)
   - Node transitions to available (available)

Cleaning
~~~~~~~~

Manual cleaning is not part of the regular state transitions when using Nova, however nodes can be manually cleaned by administrators.

- Node starts in the manageable state (manageable)
- User triggers cleaning with API (cleaning)
- Ironic will switch the node onto the cleaning network (cleaning)
- Ironic will power on the node and will await a callback (clean-wait)
- Node boots into Ironic Python Agent and issues callback, Ironic starts cleaning (cleaning)
- Ironic removes node from cleaning network (cleaning)
- Node transitions back to the manageable state (manageable)

.. ifconfig:: deployment['ironic_automated_cleaning']

   See :ref:`baremetal-management-deprovisioning` for information about
   automated cleaning.

Rescuing
~~~~~~~~

Feature not used. The required rescue network is not currently configured.

Baremetal networking
--------------------

Baremetal networking with the Neutron Networking Generic Switch ML2 driver requires a combination of static and dynamic switch configuration.

.. _static-switch-config:

Static switch configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. ifconfig:: deployment['kayobe_manages_physical_network']

   Static physical network configuration is managed via Kayobe.

   - A Virtual Routing and Forwarding (VRF) instance has been configured on the spine switches that routes between the workload
     provisioning and internal API networks. This allows Ironic Python Agent to post data back to Ironic and Ironic Inspector
     (Ironic and Inspector listen on the internal API network, whilst the node PXE boots on the workload provisioning network).

   - This VRF is defined by the Kayobe switch configuration in |kayobe_config_source_url|.

   - Controllers can reach the iDRACs via the workload out-of-band network.
     This is so Ironic can perform power control actions, firmware updates, etc.

   - Controllers are attached to the following layer 2 networks:

     * Overcloud provisioning (VLAN 1074, access)
     * Workload out-of-band (VLAN 1075)
     * Public (VLAN 16)
     * Internal API (VLAN 1079)
     * Ceph Storage (VLAN 1071)
     * Tunnel (VLAN 1077)
     * Workload provisioning network (VLAN 1069)
     * Any VLAN networks defined in OpenStack. These are needed for DHCP services and routers to function correctly. An example of such a network is the "habrok slurm management" network (VLAN 1078).

   - The management (1G) network is not accessible to bare metal nodes, it is used only for iDRACs.

   - Some initial switch configuration is required before the networking generic switch Neutron driver can take over the management of a port group.
     This is defined by the Kayobe switch configuration in |kayobe_config_source_url|.
     For example:

     .. code-block:: yaml

        "Ethernet 1/1/1":
          description: hb-node026
          config: "{{ switch_interface_config_baremetal }}"

     This configuration is be applied to each switch in the leaf pair.

     **NOTE**: You only need to do this if Ironic isn't aware of the node.

   Configuration with kayobe
   ^^^^^^^^^^^^^^^^^^^^^^^^^

   Kayobe can be used to apply the :ref:`static-switch-config`.

   - Upstream documentation can be found `here <https://docs.openstack.org/kayobe/latest/configuration/reference/physical-network.html>`__.
   - Kayobe does all the switch configuration that isn't :ref:`dynamically updated using Ironic <dynamic-switch-configuration>`.
   - Optionally switches the node onto the provisioning network (when using ``--enable-discovery``)

     + NOTE: This is a dangerous operation as it can wipe out the dynamic VLAN configuration applied by neutron/ironic.
       You should only run this when initially enrolling a node. It is possible to use the ``interface-description-limit``. For example:

       .. code-block::

         kayobe physical network configure --interface-description-limit <description> --group leaf-switches --display --enable-discovery

       In this example, ``--display`` is used to preview the switch configuration without applying it.

   - Configuration is done using a combination of ``group_vars`` and ``host_vars``

     * The bulk of the configuration is done with templates in the `group_vars for the different switches groups <https://github.com/rug-cit-hpc/rug-kayobe-config/tree/rug/yoga/etc/kayobe/environments/habrok/inventory/group_vars>`__. This allows us to share
       configuration templates across all switches.
     * Each switch has host variables defined in `host_vars
       <https://github.com/rug-cit-hpc/rug-kayobe-config/tree/rug/yoga/etc/kayobe/environments/habrok/inventory/group_vars>`_
       that provide configuration specific to the switch.

.. ifconfig:: not deployment['kayobe_manages_physical_network']

   Static physical network configuration is not managed via Kayobe.

.. _dynamic-switch-configuration:

Dynamic switch configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ironic dynamically configures the switches using the Neutron `Networking Generic Switch <https://docs.openstack.org/networking-generic-switch/latest/>`_ ML2 driver.

- Used to toggle the baremetal nodes onto different networks

  + Can use any VLAN network defined in OpenStack, providing that the VLAN has been trunked to the controllers
    as this is required for DHCP to function.
  + See :ref:`ironic-node-lifecycle`. This attempts to illustrate when any switch reconfigurations happen.

- Only configures VLAN membership of the switch interfaces or port groups. To prevent conflicts with the static switch configuration,
  the convention used is: after the node is in service in Ironic, VLAN membership should not be manually adjusted and
  should be left to be controlled by ironic i.e *don't* use ``--enable-discovery`` without a limit when configuring the
  switches with kayobe.
- Ironic is configured to use the neutron networking driver.

.. _ngs-commands:

Commands that NGS will execute
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Networking Generic Switch is mainly concerned with toggling the ports onto different VLANs. It
cannot fully configure the switch.

NGS manages the high speed network, but not the management network.

- Switching the port onto the high speed network

  .. code-block:: shell

     interface ethernet 1/1/1
     switchport mode access
     switchport access vlan 1234
     exit

- Unplugging from the provisioning network

  .. code-block:: shell

     interface ethernet 1/1/1
     no switchport access vlan
     exit

NGS will save the configuration after each reconfiguration (by default).

Ports managed by NGS
^^^^^^^^^^^^^^^^^^^^

The command below extracts a list of port UUID, node UUID and switch port information.

.. code-block:: bash

   admin# openstack baremetal port list --field uuid --field node_uuid --field local_link_connection --format value

NGS will manage VLAN membership for ports when the ``local_link_connection`` fields match one of the switches in ``ml2_conf.ini``.
The rest of the switch configuration is static.
The switch configuration that NGS will apply to these ports is detailed in :ref:`dynamic-switch-configuration`.

.. _ironic-node-discovery:

Ironic node discovery
---------------------

Discovery is the process of PXE booting the nodes into the Ironic Python Agent (IPA) ramdisk. This ramdisk will collect hardware and networking configuration from the node in a process known as introspection. This data is used to populate the baremetal node object in Ironic. The series of steps you need to take to enrol a new node is as follows:

- Ensure that all necessary `inspection rules <https://github.com/rug-cit-hpc/rug-kayobe-config/blob/rug/yoga/etc/kayobe/inspector.yml>`_ are defined.

- Configure credentials on the |bmc|. These are needed for Ironic to be able to perform power control actions.

- Controllers should have network connectivity with the target |bmc|.

.. ifconfig:: deployment['kayobe_manages_physical_network']

   - Add any additional switch configuration to kayobe config.
     The minimal switch configuration that kayobe needs to know about is described in :ref:`tor-switch-configuration`.

- Apply any :ref:`static switch configration <static-switch-config>`. This performs the initial
  setup of the switchports that is needed before Ironic can take over. The static configuration
  will not be modified by Ironic, so it should be safe to reapply at any point. See :ref:`ngs-commands`
  for details about the switch configuation that Networking Generic Switch will apply.

.. ifconfig:: deployment['kayobe_manages_physical_network']

   - Put the node onto the provisioning network using the ``--enable-discovery`` flag. See :ref:`static-switch-config`.

     * This is only necessary to initially discover the node. Once the node is in registered in Ironic,
       it will take over control of the the VLAN membership. See :ref:`dynamic-switch-configuration`.

     * This provides ethernet connectivity with the controllers over the `workload provisioning` network

.. ifconfig:: not deployment['kayobe_manages_physical_network']

   - Put the node onto the provisioning network.

- Add node to the `kayobe inventory <https://github.com/rug-cit-hpc/rug-kayobe-config/blob/rug/yoga/etc/kayobe/environments/habrok/inventory/hosts>`_.

- PXE boot the node.

  * Go to the node's |bmc| web interface to trigger discovery by network booting:

      - Start up a Virtual Console
      - Boot -> PXE
      - Power -> Power On System (or Reset System if already powered on)
      - To debug, view the HTML5 console as the node boots

.. _tor-switch-configuration:

Top of Rack (ToR) switch configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Networking Generic Switch must be aware of the Top-of-Rack switch connected to the new node.
Switches managed by NGS are configured in ``ml2_conf.ini``.
This file is generated based on the ``kolla_neutron_ml2_generic_switch_hosts``
and ``kolla_neutron_ml2_generic_switch_extra`` variables in `neutron.yml
<https://github.com/rug-cit-hpc/rug-kayobe-config/blob/rug/yoga/etc/kayobe/environments/habrok/neutron.yml>`_.

After adding switches to the NGS configuration, Neutron must be redeployed.

Considerations when booting baremetal compared to VMs
------------------------------------------------------

- You can only use networks of type: vlan
- Without using trunk ports, it is only possible to directly attach one network to each port or port group of an instance.

  * To access other networks you can use routers
  * You can still attach floating IPs

- Instances take much longer to provision (expect at least 15 mins)
- When booting an instance use one of the flavors that maps to a baremetal node via the RESOURCE_CLASS configured on the flavor.

Deploy templates/traits
-----------------------

Ironic `deploy templates
<https://docs.openstack.org/ironic/latest/admin/node-deployment.html>`__ is a
feature that allows for dynamically configuring bare metal nodes at deployment
time, based on metadata provided by the Nova flavor.
This `OpenStack summit presentation <https://www.youtube.com/watch?v=DrQcTljx_eM>`__ explains how it works.

The Bateleur system uses deploy templates to enable or disable hyperthreading.
Bare metal compute nodes have hyperthreading disabled, while hypervisors have hyperthreading enabled.

This relies on two traits, ``CUSTOM_HYPERTHREADING_ENABLED`` and ``CUSTOM_HYPERTHREADING_DISABLED``, as well as two corresponding deploy templates.
Bare metal nodes are marked as supporting both traits via an `inspection rule <https://github.com/rug-cit-hpc/rug-kayobe-config/blob/rug/yoga/etc/kayobe/inspector.yml>`__.
Baremetal Nova flavors are marked as `requiring <https://github.com/rug-cit-hpc/rug-config/blob/b56559574f96997facd9e8133c708efef3277178/etc/openstack-config/openstack-config.yml#L498>`__ the ``CUSTOM_HYPERTHREADING_DISABLED`` trait, while the hypervisor flavor is marked as `requiring <https://github.com/rug-cit-hpc/rug-config/blob/b56559574f96997facd9e8133c708efef3277178/etc/openstack-config/openstack-config.yml#L550>`__ the ``CUSTOM_HYPERTHREADING_ENABLED`` trait.
A deploy template is `registered in Ironic <https://github.com/rug-cit-hpc/rug-config/blob/b56559574f96997facd9e8133c708efef3277178/etc/openstack-config/openstack-config.yml#L724>`__ for each of these traits.
