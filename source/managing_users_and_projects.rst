.. include:: vars.rst

===========================
Managing Users and Projects
===========================

Projects (in OpenStack) can be defined in the |project_config| repository:
|project_config_source_url|

To initialise the working environment for |project_config|:

.. code-block:: console
   :substitutions:

   admin# cd |base_path|/src
   admin# git clone |project_config_source_url|
   admin# cd |project_config|
   admin# virtualenv |base_path|/venvs/|project_config|
   admin# source |base_path|/venvs/|project_config|/bin/activate
   admin# pip install -U pip
   admin# pip install -r requirements.txt
   admin# ansible-galaxy install \
          -p ansible/roles \
          -r requirements.yml

To define a new project, add a new project to
etc/|project_config|/|project_config|.yml:

.. code-block:: console
   :substitutions:

   admin# cd |base_path|/src/|project_config|
   admin# vim etc/|project_config|/|project_config|.yml

Example invocation:

.. code-block:: console
   :substitutions:

   admin# source |base_path|/src/|kayobe_config|/etc/kolla/public-openrc.sh
   admin# source |base_path|/venvs/|project_config|/bin/activate
   admin# tools/|project_config| -- --vault-password-file |vault_password_file_path|

Deleting Users and Projects
---------------------------

Ansible is designed for adding configuration that is not present; removing
state is less easy. To remove a project or user, the configuration should be
manually removed.
