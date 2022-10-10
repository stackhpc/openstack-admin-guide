.. include:: vars.rst

============
Ceph Storage
============

.. ifconfig:: deployment['ceph']

   The |project_name| deployment uses Ceph as a storage backend.

.. ifconfig:: deployment['ceph_managed']

   The Ceph deployment is managed by StackHPC Ltd.

.. ifconfig:: not deployment['ceph_managed']

   The Ceph deployment is not managed by StackHPC Ltd.

Ceph Operations and Troubleshooting
===================================

.. include:: include/ceph_troubleshooting.rst

.. ifconfig:: deployment['ceph_ansible']

   Ceph Ansible
   ============

   .. include:: include/ceph_ansible.rst

