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

Working with Ceph deployment tool
=================================

.. ifconfig:: deployment['ceph_ansible']

   .. include:: include/ceph_ansible.rst

.. ifconfig:: deployment['cephadm']

   .. include:: include/cephadm.rst

Operations
==========

.. include:: include/ceph_operations.rst

Troubleshooting
===============

.. include:: include/ceph_troubleshooting.rst
