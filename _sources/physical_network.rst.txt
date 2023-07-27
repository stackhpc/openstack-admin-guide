.. include:: vars.rst

================
Physical network
================

.. ifconfig:: deployment['kayobe_manages_physical_network']

   The |project_name| deployment uses Kayobe to manage the physical network.

   .. ifconfig:: deployment['physical_network_evpn']
   
      .. include:: include/evpn_fabric_overview.rst
   
      .. include:: include/evpn_fabric_details.rst

.. ifconfig:: not deployment['kayobe_manages_physical_network']

   The |project_name| deployment does not use Kayobe to manage the physical network.
