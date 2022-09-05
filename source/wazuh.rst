.. include:: vars.rst

=======================
Wazuh Security Platform
=======================

.. ifconfig:: deployment['wazuh']

    The |project_name| deployment uses Wazuh as security platform to detect intruders within your network.

.. ifconfig:: deployment['wazuh_managed']

    The Wazuh deployment is managed by StackHPC Ltd.

.. ifconfig:: not deployment['wazuh_managed']

    The Wazuh deployment is not managed by StackHPC Ltd.

.. ifconfig:: deployment ['wazuh_ansible']

    Wazuh deployment via Ansible
    ============================

    .. include:: include/wazuh_ansible.rst