.. include:: vars.rst

======================
Working with OpenStack
======================

Accessing the Dashboard (Horizon)
---------------------------------

The OpenStack web UI is available at: |horizon_url|

This site is accessible |horizon_access|.

Accessing the OpenStack CLI
---------------------------

A simple way to get started with accessing the OpenStack command-line
interface.

This can be done from |public_api_access_host| (for example), or any machine
that has access to |public_vip|:

.. code-block:: console

 openstack# virtualenv openstack-venv
 openstack# source openstack-venv/bin/activate
 openstack# pip install -U pip
 openstack# pip install python-openstackclient
 openstack# source <project>-openrc.sh

The `<project>-openrc.sh` file can be downloaded from the OpenStack Dashboard
(Horizon):

.. image:: _static/openrc.png
   :alt: Downloading an openrc file from Horizon
   :class: no-scaled-link
   :width: 200

Now it should be possible to run OpenStack commands:

.. code-block:: console

 openstack# openstack server list

Accessing Deployed Instances
----------------------------

The external network of OpenStack, called |public_network|, connects to the
subnet |public_subnet|. This network is accessible |floating_ip_access|.

Any OpenStack instance can make outgoing connections to this network, via a
router that connects the internal network of the project to the
|public_network| network.

To enable incoming connections (e.g. SSH), a floating IP is required. A
floating IP is allocated and associated via OpenStack. Security groups must be
set to permit the kind of connectivity required (i.e. to define the ports that
must be opened).
