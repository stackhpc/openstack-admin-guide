.. include:: vars.rst

===================
Working with Kayobe
===================

Kayobe is the point of administration for infrastructure-as-code operations.
Kayobe is installed and invoked from a host referred to as the Ansible Control
Host.


Kayobe is open source and can be downloaded from the Git repository
|kayobe_source_url| (use the |kayobe_source_version| branch).

Kayobe's online documentation is available here:
https://docs.openstack.org/kayobe/latest/

The infrastructure-as-code configuration state for the |project_name| OpenStack
is here: |kayobe_config_source_url| (use the |kayobe_config_source_version|
branch).

When applying reconfigurations using Kayobe, always make sure that the
|project_name| kayobe-config repository is current and up to date.

Connecting to the Ansible Control Host
--------------------------------------

The Ansible control host is any system that can access the seed VM (|seed_ip|)
and control plane hosts through the provisioning network
(|provisioning_net_cidr|).

|control_host_access|

Making a Kayobe Checkout
------------------------

A Kayobe checkout is made on the Ansible control host.

A Kayobe development environment can easily be set up using a script called
``beokay``, for example:

.. code-block:: console
   :substitutions:

   kayobe# git clone https://github.com/markgoddard/beokay
   kayobe# cd beokay/
   kayobe# ./beokay.py create \
           --base-path |base_path| \
           --kayobe-repo |kayobe_source_url| \
           --kayobe-branch |kayobe_source_version| \
           --kayobe-config-repo |kayobe_config_source_url| \
           --kayobe-config-branch |kayobe_config_source_version|

After making the checkout, source the virtualenv and Kayobe config environment variables:

.. code-block:: console
   :substitutions:

   kayobe# cd |base_path|
   kayobe# source venvs/kayobe/bin/activate
   kayobe# source src/kayobe-config/kayobe-env

Set up any dependencies needed on the control host:

.. code-block:: console

   kayobe# kayobe control host bootstrap

Deployment Secrets
------------------

The |project_name| Kayobe configuration uses Ansible Vault to store secrets
such as IPMI credentials, Ceph account keys and OpenStack service credentials.

The vault of deployment secrets is protected by a password, which
conventionally is stored in a (mode 0400) file in the user home directory.

An easy way to manage the vault password is to update ``.bash_profile`` to add
a command such as:

.. code-block:: console

   kayobe# export KAYOBE_VAULT_PASSWORD_FILE=~/vault.pass

Verifying Changes Before Applying
---------------------------------

This section describes a way to check all the effects of a configuration change
to the OpenStack control plane.

Save the existing configuration from the control plane:

.. code-block:: console

   kayobe# mkdir ~/config-before
   kayobe# kayobe overcloud service configuration save \
           --output-dir ~/config-before

Generate new configuration as a dry run:

.. code-block:: console

   kayobe# kayobe overcloud service configuration generate \
           --node-config-dir /tmp/config-new

Gather the new configuration for comparison of changes:

.. code-block:: console

   kayobe# mkdir ~/config-after
   kayobe# kayobe overcloud service configuration save \
           --output-dir ~/config-after

You can now compare the configuration in ``~/config-before`` and
``~/config-after``:

.. code-block:: console

   for host in `ls ~/config-after/`; do
     mv ~/config-after/$host/tmp/config-new/ ~/config-after/$host/tmp/kolla
     mv ~/config-after/$host/tmp/ ~/config-after/$host/etc
   done
   diff -ru ~/config-before ~/config-after

Accessing the Seed
------------------

The seed is a |seed_type|. The seed is called |seed_name| (|seed_ip|). The main
user account on the seed is the |seed_user| user.

.. code-block:: console
   :substitutions:

   kayobe# ssh -l stack |seed_ip|
   [stack@seed ~]$ sudo docker ps
   CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
   ed034742903b        registry:latest     "/entrypoint.sh /etc…"   2 months ago        Up 2 months         0.0.0.0:5000->5000/tcp   docker_registry
   640221d53b0f        9355cba4a8cd        "/sbin/init"             2 months ago        Up 2 months                                  bifrost_deploy

.. _accessing-the-bifrost-service:

Accessing the Bifrost Service
-----------------------------

From the seed host, the Bifrost container may be entered:

.. code-block:: console

   seed# sudo docker exec -it bifrost_deploy /bin/bash
   (bifrost-deploy)[root@seed bifrost-base]# source env-vars
   (bifrost-deploy)[root@seed bifrost-base]# openstack baremetal node list

.. Consider adding a section about configuring the physical network.
