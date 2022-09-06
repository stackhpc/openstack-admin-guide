One of methods for deploying and maintaining Wazuh is with the use of the official Ansible playbooks, integrated into a Kayobe Config.

Configuring Wazuh Manager
-------------------------

Wazuh manager can easily be configured by editing the ``wazuh-manager.yml`` groups vars file found at ``etc/kayobe/inventory/group_vars/wazuh-master/``. 
This file gives you control over various important aspects of the Wazuh manager.
Most notably;

*domain_name*:
    the domain used by Search Guard CE when generating certificates.

*wazuh_manager_ip*:
    the IP address that the wazuh manager shall reside on for communicating with the agents.

*wazuh_manager_connection*:
    used to define port and protocol for the manager to be listening on.

*wazuh_manager_authd*:
    connection settings for the daemon responsible for registering new agents.

Running ``kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/wazuh-manager.yml`` will deploy these changes.

Secrets
-------

Wazuh requires that secrets or passwords are set for itself and the services it communiticates with.
The playbook ``etc/kayobe/ansible/wazuh-secrets.yml`` automates the creation of these secrets which can then be encrypted with Ansible Vault.

To update the secrets you can execute the following two commands

.. code-block:: console
    :substitutions:

    kayobe# kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/wazuh-secrets.yml -e wazuh_user_pass=$(uuidgen) -e wazuh_admin_pass=$(uuidgen)
    kayobe# ansible-vault encrypt --vault-password-file |vault_password_file_path| $KAYOBE_CONFIG_PATH/inventory/group_vars/wazuh-master/wazuh-secrets.yml

Once generated you can run ``kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/wazuh-manager.yml`` which shall copy the secrets into place.

.. note:: If you need to view the secrets it is recommended you use ``ansible-vault view --vault-password-file ~/vault.password``

Adding a New Agent
------------------
When adding a new host it should be automically picked up by the ``wazuh-agent:children`` group in ``etc/kayobe/inventory/groups`` as it would be included in the ``overcloud`` member.

.. code-block:: ini

    [wazuh-agent:children]
    seed
    overcloud

Running the follow playbook ``kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/wazuh-agent.yml`` will deploy the agent to the new host.
This should automatically be registered and accessible within the Wazuh manager dashboard.

The playbook ``wazuh-agent.yml`` can be setup as a hook within kayobe, which will automatically run either pre or post a given kayobe command.
See `here <https://docs.openstack.org/kayobe/wallaby/custom-ansible-playbooks.html#hooks>`_ for more details. 

Accessing Wazuh Manager
-----------------------

To access the Wazuh manager dashboard, navigate to the ip address of the |wazuh_master_name| (|wazuh_master_url|).

You can login to the dashboard with the username ``admin`` and the password for ``opendistro_admin_password`` which can be found within ``etc/kayobe/inventory/group_vars/wazuh-master/wazuh-secrets.yml``.

.. note:: If you need to view the secrets it is recommended you use ``ansible-vault view --vault-password-file ~/vault.password``
