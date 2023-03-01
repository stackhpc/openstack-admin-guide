EVPN VXLAN Fabric details
=========================

The Habrok system is composed of 8 racks, 5 of which contain a leaf
pair. There are 2 spines. Each rack has a management switch, and these
hang off the spines with a layer 2 trunk.

.. figure:: _static/habrok-network-design.png
   :alt: Habrok network design

The configuration follows the "VXLAN BGP EVPN — Multiple AS topology with
asymmetric IRB" example in the `Dell OS10 EVPN guide
<https://dl.dell.com/content/manual50032153-vxlan-and-bgp-evpn-configuration-guide-for-dell-smartfabric-os10-release-10-5-4-0.pdf>`__,
with a few exceptions:

-  Added a missing increased MTU to the vlan 4000 (inter-switch link)
   interface.
-  Increased fabric MTUs to 9216.
-  In general Habrok does not use Virtual Link Trunking (VLT aka MLAG)
   interfaces - each compute node has a separate link to each of the
   leaf switches in a pair.

   -  The gotcha here is that since leaf switch pairs share a VXLAN VTEP
      IP, a packet may need to traverse the inter-switch link if it gets
      decapsulated on the "wrong" leaf switch.

-  The spines are configured as VTEPs, to allow trunking VLANs down to
   the management switches and up to the core switch.

.. figure:: _static/dell-multiple-as-topology.png
   :alt: Dell OS10 multiple AS topology

Configuration
-------------

The switch configuration is managed by Kayobe, and lives in the
following files:

.. code:: bash

   etc/kayobe/environments/habrok/inventory/
     group_vars/
       leaf-rack-7
       leaf-switches
       mgmt-switches
       spine-leaf-switches
       spine-switches
       switches
     host_vars/
       hb-sw-*/
         switch-config.yml
       hb-sw-*.yml
     switches

The switch configuration is applied via the following command:

.. code:: bash

   kayobe physical network configure --group <group>

The main groups are: ``leaf-switches``, ``spine-switches``,
``mgmt-switches``. There is also a ``spine-leaf-switches`` group for
convenience.

Generally the various ASNs and IPs are automatically calculated based on
the switch layer (leaf or spine), rack number, leaf pair index, etc. The
following commands are useful for dumping the actual allocations.

ASN:

.. code:: bash

   kayobe configuration dump -l spine-leaf-switches --var-name switch_bgp_asn
   {
       "hb-sw-leaf01-h01": "65001",
       "hb-sw-leaf01-h03": "65003",
       "hb-sw-leaf01-h04": "65004",
       "hb-sw-leaf01-h05": "65005",
       "hb-sw-leaf01-h07": "65007",
       "hb-sw-leaf02-h01": "65001",
       "hb-sw-leaf02-h03": "65003",
       "hb-sw-leaf02-h04": "65004",
       "hb-sw-leaf02-h05": "65005",
       "hb-sw-leaf02-h07": "65007",
       "hb-sw-spine01-h03": "65103",
       "hb-sw-spine01-h04": "65104"
   }

Fabric IPs:

.. code:: bash

   kayobe configuration dump -l spine-leaf-switches --var-name switch_fabric_ips                      
   {                                          
       "hb-sw-leaf01-h01": {                  
           "hb-sw-spine01-h03": "172.23.62.0",
           "hb-sw-spine01-h04": "172.23.62.2" 
       },                                     
       "hb-sw-leaf01-h03": {                  
           "hb-sw-spine01-h03": "172.23.62.16",
           "hb-sw-spine01-h04": "172.23.62.18"
       },
       "hb-sw-leaf01-h04": {
           "hb-sw-spine01-h03": "172.23.62.24",
           "hb-sw-spine01-h04": "172.23.62.26"
       },
       "hb-sw-leaf01-h05": {
           "hb-sw-spine01-h03": "172.23.62.32",
           "hb-sw-spine01-h04": "172.23.62.34"
       },
       "hb-sw-leaf01-h07": {
           "hb-sw-spine01-h03": "172.23.62.48",
           "hb-sw-spine01-h04": "172.23.62.50"
       },
       "hb-sw-leaf02-h01": {
           "hb-sw-spine01-h03": "172.23.62.4",
           "hb-sw-spine01-h04": "172.23.62.6"
       },
       "hb-sw-leaf02-h03": {
           "hb-sw-spine01-h03": "172.23.62.20",
           "hb-sw-spine01-h04": "172.23.62.22"
       },
       "hb-sw-leaf02-h04": {
           "hb-sw-spine01-h03": "172.23.62.28",
           "hb-sw-spine01-h04": "172.23.62.30"
       },
       "hb-sw-leaf02-h05": {
           "hb-sw-spine01-h03": "172.23.62.36",
           "hb-sw-spine01-h04": "172.23.62.38"
       },
       "hb-sw-leaf02-h07": {
           "hb-sw-spine01-h03": "172.23.62.52",
           "hb-sw-spine01-h04": "172.23.62.54"
       },
       "hb-sw-spine01-h03": {
           "hb-sw-leaf01-h01": "172.23.62.1",
           "hb-sw-leaf01-h03": "172.23.62.17",
           "hb-sw-leaf01-h04": "172.23.62.25",
           "hb-sw-leaf01-h05": "172.23.62.33",
           "hb-sw-leaf01-h07": "172.23.62.49",
           "hb-sw-leaf02-h01": "172.23.62.5",
           "hb-sw-leaf02-h03": "172.23.62.21",
           "hb-sw-leaf02-h04": "172.23.62.29",
           "hb-sw-leaf02-h05": "172.23.62.37",
           "hb-sw-leaf02-h07": "172.23.62.53"
       },
       "hb-sw-spine01-h04": {
           "hb-sw-leaf01-h01": "172.23.62.3",
           "hb-sw-leaf01-h03": "172.23.62.19",
           "hb-sw-leaf01-h04": "172.23.62.27",
           "hb-sw-leaf01-h05": "172.23.62.35",
           "hb-sw-leaf01-h07": "172.23.62.51",
           "hb-sw-leaf02-h01": "172.23.62.7",
           "hb-sw-leaf02-h03": "172.23.62.23",
           "hb-sw-leaf02-h04": "172.23.62.31",
           "hb-sw-leaf02-h05": "172.23.62.39",
           "hb-sw-leaf02-h07": "172.23.62.55"
       }
   }

BGP router IDs:

.. code:: bash

   kayobe configuration dump -l spine-leaf-switches --var-name switch_bgp_ip
   {
       "hb-sw-leaf01-h01": "172.23.62.195",
       "hb-sw-leaf01-h03": "172.23.62.199",
       "hb-sw-leaf01-h04": "172.23.62.201",
       "hb-sw-leaf01-h05": "172.23.62.203",
       "hb-sw-leaf01-h07": "172.23.62.207",
       "hb-sw-leaf02-h01": "172.23.62.196",
       "hb-sw-leaf02-h03": "172.23.62.200",
       "hb-sw-leaf02-h04": "172.23.62.202",
       "hb-sw-leaf02-h05": "172.23.62.204",
       "hb-sw-leaf02-h07": "172.23.62.208",
       "hb-sw-spine01-h03": "172.23.62.253",
       "hb-sw-spine01-h04": "172.23.62.252"
   }

VXLAN VTEP IPs:

.. code:: bash

   kayobe configuration dump -l spine-leaf-switches --var-name switch_tunnel_ip
   {
       "hb-sw-leaf01-h01": "172.23.62.129",
       "hb-sw-leaf01-h03": "172.23.62.131",
       "hb-sw-leaf01-h04": "172.23.62.132",
       "hb-sw-leaf01-h05": "172.23.62.133",
       "hb-sw-leaf01-h07": "172.23.62.135",
       "hb-sw-leaf02-h01": "172.23.62.129",
       "hb-sw-leaf02-h03": "172.23.62.131",
       "hb-sw-leaf02-h04": "172.23.62.132",
       "hb-sw-leaf02-h05": "172.23.62.133",
       "hb-sw-leaf02-h07": "172.23.62.135",
       "hb-sw-spine01-h03": "172.23.62.189",
       "hb-sw-spine01-h04": "172.23.62.188"
   }

The Dell OS10 Ansible modules do not support the Ansible ``--check`` and
``--diff`` arguments, so it's possible to feel a little blind when applying
changes. To alleviate this, we have a Kayobe custom playbook that saves the
current switch configuration to the Ansible control host that may be run before
and after applying switch configuration changes:

.. code:: bash

   kayobe playbook run etc/kayobe/ansible/dellos10-facts.yml

The switch configuration will be saved to ``habrok-switches/<datetime stamp>``
in the ``rug-kayobe-config`` repository root. To compare the configuration for
a given switch, you could use a command such as this:

.. code:: bash

   vimdiff habrok-switches/{2023-01-24T14:28+00:00,2023-01-24T14:02+00:00}/hb-sw-leaf02-h05

Useful commands
---------------

Underlay BGP commands:

::

   show bfd neighbors
   show ip route
   show ip bgp summary
   show ip bgp neighbours
   show ip bgp

Overlay BGP commands:

::

   show ip bgp l2vpn evpn summary
   show ip bgp l2vpn evpn neighbours
   show ip bgp l2vpn evpn
   show evpn mac-ip
   show evpn evi
   show virtual-network
