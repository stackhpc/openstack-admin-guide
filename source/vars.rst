.. |alertmanager_url| replace:: https://hb-openstack.hpc.rug.nl:9093
.. |base_path| replace:: ~/deployment
.. |bmc| replace:: iDRAC
.. |chat_system| replace:: Slack
.. |control_host_access| replace:: |control_host| is used as the Ansible control host. Each operator uses their own account on this host, but with a shared SSH key stored as ``~/.ssh/id_rsa``.
.. |control_host| replace:: hb-ansible01
.. |controller0_hostname| replace:: ``hb-openstack-controller01``
.. |controller1_hostname| replace:: ``hb-openstack-controller02``
.. |controller2_hostname| replace:: ``hb-openstack-controller03``
.. |file_share_location| replace:: Google Drive
.. |flavor_name| replace:: general.v1.tiny
.. |floating_ip_access| replace:: from acme-seed-hypervisor and the rest of the Acme network
.. |grafana_url| replace:: https://hb-openstack.hpc.rug.nl:3000
.. |grafana_username| replace:: ``grafana_local_admin``
.. |horizon_access| replace:: internally for now.
.. |horizon_theme_clone_url| replace:: https://github.com/acme-openstack/horizon-theme.git
.. |horizon_theme_name| replace:: groningen
.. |horizon_url| replace:: https://hb-openstack.hpc.rug.nl
.. |hypervisor_hostname| replace:: ``comp0``
.. |hypervisor_type| replace:: KVM
.. |ipmi_username| replace:: root
.. |kayobe_config_source_url| replace:: https://github.com/rug-cit-hpc/rug-kayobe-config.git
.. |kayobe_config_source_version| replace:: ``rug/wallaby``
.. |kayobe_config| replace:: kayobe-config
.. |kayobe_source_url| replace:: https://github.com/stackhpc/kayobe.git
.. |kayobe_source_version| replace:: ``rug/wallaby``
.. |keystone_public_url| replace:: https://hb-openstack.hpc.rug.nl:5000
.. |kibana_url| replace:: https://hb-openstack.hpc.rug.nl:5601
.. |kolla_passwords| replace:: https://github.com/rug-cit-hpc/rug-kayobe-config/blob/rug/wallaby/etc/kayobe/environments/habrok/kolla/passwords.yml
.. |monitoring_host| replace:: ``hb-openstack-controller01``
.. |network_name| replace:: test-vxlan
.. |nova_rbd_pool| replace:: vms
.. |project_config_source_url| replace:: https://github.com/rug-cit-hpc/rug-config
.. |project_config| replace:: rug-config
.. |project_name| replace:: Bateleur
.. |provisioning_net_cidr| replace:: 172.23.9.0/24
.. |public_api_access_host| replace:: |control_host|
.. |public_endpoint_fqdn| replace:: hb-openstack.hpc.rug.nl
.. |public_network| replace:: public
.. |public_subnet| replace:: 195.169.22.0/23
.. |public_vip| replace:: 195.169.22.10
.. |reference_architecture_url| replace:: https://docs.google.com/document/d/1uvBuvRxWZwNTaieVSx8kzrAf8Yfd3CMUNpl7hwK2Nl0/
.. |seed_ip| replace:: 172.23.9.249
.. |seed_name| replace:: hb-seed01
.. |seed_type| replace:: virtual machine
.. |seed_user| replace:: stack
.. |support_email| replace:: rug-support@stackhpc.com
.. |support_level| replace:: standard support
.. |tempest_recipes| replace:: https://github.com/stackhpc/tempest-recipes.git
.. |tls_setup| replace:: TLS is implemented using a wildcard certificate available for ``*.hb-openstack.hpc.rug.nl``.
.. |vault_password_file_path| replace:: ~/vault-password
.. |wazuh_manager_url| replace:: https://172.168.0.10:5601
.. |wazuh_manager_ip| replace:: 172.23.61.103
.. |wazuh_manager_name| replace:: hb-wazuh01
