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
``beokay``, for example. This command will need the ``KAYOBE_VAULT_PASSWORD``
environment variable to be set when secrets are encrypted with Ansible Vault.
See the next section for details.

.. code-block:: console
   :substitutions:

   kayobe# git clone https://github.com/stackhpc/beokay.git
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

If you are using a Kayobe environment, you will instead need to specify which
environment to source. See the section :ref:`Kayobe Environments` for more details.

.. code-block:: console
   :substitutions:

   kayobe# source src/kayobe-config/kayobe-env --environment <env-name>

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
   :substitutions:

   kayobe# export KAYOBE_VAULT_PASSWORD=$(cat |vault_password_file_path|)

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
           --node-config-dir /tmp/config-new \
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
   seed# docker ps
   CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
   ed034742903b        registry:latest     "/entrypoint.sh /etc…"   2 months ago        Up 2 months         0.0.0.0:5000->5000/tcp   docker_registry
   640221d53b0f        9355cba4a8cd        "/sbin/init"             2 months ago        Up 2 months                                  bifrost_deploy

.. _accessing-the-bifrost-service:

Accessing the Bifrost Service
-----------------------------

From the seed host, the Bifrost container may be entered:

.. code-block:: console

   seed# docker exec -it bifrost_deploy /bin/bash
   (bifrost-deploy)[root@seed bifrost-base]# export OS_CLOUD=bifrost
   (bifrost-deploy)[root@seed bifrost-base]# baremetal node list

.. Consider adding a section about configuring the physical network.

.. _Kayobe Environments:

Kayobe Environments
-------------------

Kayobe supports configuring multiple environments under
``etc/kayobe/environments``. These can be used to reduce duplicated configs
between systems. Any shared config can be defined under the base layer
``etc/kayobe``, and any system-specific config lives under
``etc/kayobe/environments/<env-name>``.

To use a specific environment with Kayobe, make sure to source its environment
variables:

.. code-block:: console
   :substitutions:

   kayobe# source src/kayobe-config/kayobe-env --environment <env-name>

The Kayobe inventory and configuration under the base layer ``etc/kayobe`` are
merged automatically with the environment layer under
``etc/kayobe/environments/<env-name>``. This means that files such as
``globals.yml`` or any host/group vars can be defined either within or outside
of the environment. The base layer variables will be set on every system, and
any environment-specific variables will only be set on their systems. Variables
defined under an environment will take precedence over those defined in the
base layer.

The Kolla inventory under the base layer ``etc/kayobe/kolla/inventory`` is also
merged with the environment layer under
``etc/kayobe/environments/<env-name>/kolla/inventory``. However, Kolla config
files do not yet support this. As such, any shared configuration under
``etc/kayobe/kolla/config`` will need to be symlinked into all environments
under ``etc/kayobe/environments/<env-name>/kolla/config``. An additional caveat
is that only symlinked directories are supported. So any shared individual
files will unfortunately need to be duplicated in each environment.

If the majority of your Kolla config is intended to be shared, it is currently
recommended that you symlink the entire ``etc/kayobe/kolla/config`` directory,
and then template any specific variables based on the environment sources. For
example:

.. code-block:: console

   ---
   {% if kayobe_environment == "env-us" %}
   region: US
   {% elif kayobe_environment == "env-uk" %}
   region: UK
   {% endif %}

Please note that there is work ongoing to support the merging of Kolla
configuration in the future.
