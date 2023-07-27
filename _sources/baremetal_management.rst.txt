.. include:: vars.rst

======================================
Bare Metal Compute Hardware Management
======================================

.. ifconfig:: deployment['ironic']

   The |project_name| cloud includes bare metal compute nodes managed by the
   Ironic services. This section describes elements of the configuration of
   this service.

   .. include:: include/baremetal_management.rst

.. ifconfig:: not deployment['ironic']

   The |project_name| cloud does not include bare metal compute nodes managed
   by the Ironic services.
