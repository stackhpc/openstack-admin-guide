.. include:: vars.rst

=======================
Wazuh Security Platform
=======================

.. ifconfig:: deployment['wazuh']

    The |project_name| deployment uses `Wazuh <https://wazuh.com>`_ as security monitoring platform.  Among other things, Wazuh monitors for:

* Security-related system events.
* Known vulnerabilities (CVEs) in versions of installed software.
* Misconfigurations in system security.

.. ifconfig:: deployment['wazuh_managed']

    The Wazuh deployment is managed by StackHPC Ltd.

.. ifconfig:: not deployment['wazuh_managed']

    The Wazuh deployment is not managed by StackHPC Ltd.

.. ifconfig:: deployment ['wazuh_ansible']

    Wazuh deployment via Ansible
    ============================

    .. include:: include/wazuh_ansible.rst
