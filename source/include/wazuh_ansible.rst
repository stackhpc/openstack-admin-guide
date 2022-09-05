One of method for deploying Wazuh is with the use of the official Ansible playbooks, integrated into a Kayobe Config.

Hosts & Groups
--------------
To begin the deployment of Wazuh we must first configure our hosts and groups definitions.

Firstly, we can edit the groups under ``etc/kayobe/inventory/groups`` to define the related Wazuh groups.

.. code-block:: ini

    [infra-vms:children]
    wazuh-master

    [wazuh:children]
    wazuh-master
    wazuh-agent

    [wazuh-master]

    [wazuh-agent]

    [wazuh-agent:children]

Secondly, we can edit the hosts file found ``etc/kayobe/inventory/hosts`` to associate membership between hosts and groups.

.. code-block:: ini

    [wazuh-master]
    wazuh-master-01

    [wazuh-agent]


Provision infra-vm & install roles
----------------------------------

With the hosts and groups files created we can begin to provision the infra-vm as well install the Wazuh Ansible role.

To provision the infra-vm we can use the kayobe command ``kayobe infra vm provision``.
Once completed we can then install the Wazuh Ansible role we can be achieved by adding the role definition to the ``etc/kayobe/ansible/requirements.yml``.

.. code-block:: yaml


    roles:
      - src: https://github.com/stackhpc/wazuh-ansible.git
        version: v4.2.3-opendistro-ubuntu

Once added we can then perform a ``kayobe control host bootstrap`` which shall install this role and any other missing roles.

Configuring Wazuh Manager
-------------------------

We are almost ready to deploy Wazuh manager.
However, before we can, we must first download the Wazuh manager playbook which can be done by downloading ``https://raw.githubusercontent.com/stackhpc/kayobe-ops/master/wazuh-manager.yml`` into ``etc/kayobe/ansible/wazuh-manager.yml``.
Once downloaded it is recommended you make any changes your deployment/environment requires.

Next we must create the group varibles for the `wazuh-master` group.
This can be easily accomplished by first creating a directory ``etc/kayobe/inventory/group_vars/wazuh-master/`` which is where we shall download the next two files to.

``https://raw.githubusercontent.com/stackhpc/kayobe-ops/master/vars/elasticsearch-custom.yml``

``https://raw.githubusercontent.com/stackhpc/kayobe-ops/master/vars/wazuh-manager.yml``

Feel free to modify any of the varibles within these files.
It is expected that you would want to edit the following varibles:

* domain_name

* wazuh_manager_ip

Secrets
-------

We must ensure that Wazuh has access to a set secrets for all of the services it interacts with.
To automate this process we can use an Ansible playbook and template.

First create a playbook called ``etc/kayobe/ansible/wazuh-secrets.yml`` and add the following contents to it.

.. code-block:: yaml

    ---
    - hosts: localhost
    gather_facts: false
    vars:
        wazuh_secrets_path: "{{ kayobe_env_config_path }}/inventory/group_vars/wazuh/wazuh-secrets.yml"
    tasks:
        - name: install passlib[bcrypt]
        pip:
            name: passlib[bcrypt]
            virtualenv: "{{ ansible_playbook_python | dirname | dirname }}"

        - name: Include existing secrets if they exist
        include_vars: "{{ wazuh_secrets_path }}"
        ignore_errors: true

        - name: Ensure secrets directory exists
        file:
            path: "{{ wazuh_secrets_path | dirname }}"
            state: directory

        - name: Template new secrets
        template:
            src: wazuh-secrets.yml.j2
            dest: "{{ wazuh_secrets_path }}"

Then proceed to create a template in ``etc/kayobe/templates/wazuh-secrets.yml.j2`` with the following contents.

.. code-block:: jinja

    ---
    {% set wazuh_admin_pass = secrets_wazuh.wazuh_admin_pass | default(lookup('password', '/dev/null'), true) -%}
    {%- set wazuh_user_pass = secrets_wazuh.wazuh_user_pass | default(lookup('password', '/dev/null'), true) -%}

    # Secrets used by Wazuh managers and agents
    # Store these securely and use lookups here
    secrets_wazuh:
    # Wazuh agent authd pass
    authd_pass: "{{ secrets_wazuh.authd_pass | default(lookup('password', '/dev/null'), true) }}"
    # Strengthen default wazuh api user pass
    wazuh_api_users:
        - username: "wazuh"
        password: "{{ secrets_wazuh.wazuh_api_users[0].password | default(lookup('password', '/dev/null length=30' ), true) }}"
    # Elasticsearch 'admin' user pass
    opendistro_admin_password: "{{ secrets_wazuh.opendistro_admin_password | default(lookup('password', '/dev/null'), true) }}"
    # Elasticsearch 'kibanaserver' user pass
    opendistro_kibana_password: "{{ secrets_wazuh.opendistro_kibana_password | default(lookup('password', '/dev/null'), true) }}"
    # Wazuh/Kibana 'wazuh_admin' custom user pass
    wazuh_admin_pass: "{{ wazuh_admin_pass }}"
    # Wazuh/Kibana 'wazuh_admin' custom user pass has
    # bcrypt ($2y) hash
    wazuh_admin_hash: "{{ secrets_wazuh.wazuh_admin_hash | default(wazuh_admin_pass | password_hash('bcrypt'), true) }}"
    # Wazuh/Kibana 'wazuh_user' custom user pass
    # bcrypt ($2y) hash
    wazuh_user_pass: "{{ wazuh_user_pass }}"
    wazuh_user_hash: "{{ secrets_wazuh.wazuh_user_hash | default(wazuh_user_pass | password_hash('bcrypt'), true) }}"

And finally, run the following commands to generate and encrypt the secrets.

.. code-block:: bash

    kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/wazuh-secrets.yml -e wazuh_user_pass=$(uuidgen) -e wazuh_admin_pass=$(uuidgen)
    ansible-vault encrypt --vault-password-file ~/vault.pass $KAYOBE_CONFIG_PATH/inventory/group_vars/wazuh-master/wazuh-secrets.yml

.. note:: you must have a vault password store outside the source control directory in a file called `vault.pass`

Deploying Wazuh Manager
-----------------------

It is now time to deploy Wazuh manager.
This can be achieved with one simple command. ``kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/wazuh-manager.yml``

Once the playbook is finished running you should be able to access the Wazuh manager from the ``wazuh-master-01`` ip address at ``5601`` over ``https``.
You can login to the dashboard with the username ``admin`` and the password for ``opendistro_admin_password`` which can be found within ``etc/kayobe/inventory/group_vars/wazuh-master/wazuh-secrets.yml``.