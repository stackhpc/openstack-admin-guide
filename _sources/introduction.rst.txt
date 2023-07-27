.. include:: vars.rst

============
Introduction
============

This is an administration guide detailing run-book procedures for common
administration actions. The administration guide is expected to be augmented
over time as |project_name|'s admin procedures evolve.

|project_name| OpenStack Reference Architecture
-----------------------------------------------

For further details on the architecture of the OpenStack deployment, consult
the |project_name| OpenStack Reference Architecture on |file_share_location|:
|reference_architecture_url|

Document Conventions
--------------------

There are various places where administrative commands of different types can
be run. Where an example is given, the command prompt is set to indicate the
appropriate environment.

``openstack#``

A command that can be run as any OpenStack user with an environment configured
for OpenStack CLI access. This may involve installing client packages on the
system, or using a virtualenv. The user’s OpenStack credentials are also
sourced.

This machine must be able to reach the OpenStack public API (for example
|keystone_public_url|).

``admin#``

A command that must be run with OpenStack control plane admin credentials
loaded, and the OpenStack client and supporting modules available (whether in a
virtualenv or installed in the OS libraries).

This machine must be able to reach the OpenStack public API.

``kayobe#``

A command that must be run in an environment configured for Kayobe. This
typically is a virtualenv created along with the Kayobe repo, and environment
variables conventionally drawn from ``kayobe-config/kayobe-env``. The
``KAYOBE_VAULT_PASSWORD`` environment variable may also need to be set.

``seed#``

A command that must be run on the seed VM using the Kayobe Ansible user account
(``stack`` by default).

``bifrost#``

A command that must be run within the Bifrost service container, hosted on the seed VM.

``instance#``

A command that can be run (as superuser) from a running compute instance.

Glossary of Terms
-----------------

.. glossary::

    Cinder
      OpenStack’s block-based storage service. Cinder implements volumes, which
      are persistent data storage that appears in VMs as block devices.
      https://docs.openstack.org/cinder/latest/

    ECMP
      Equal-Cost Multi-Path routing - a IP-based protocol for enabling traffic
      between two destinations across multiple paths in the network fabric.

    Floating IP
      An external IP address that is associated with a VM (although the VM is
      not aware of it). A floating IP enables in-bound access to a VM that is
      on a private network with a router connecting it to the external network.
      VMs on a private network can initiate external network access without a
      floating IP.

    Glance
      OpenStack’s image service. Glance manages software images for compute
      instances. https://docs.openstack.org/glance/latest/

    GRE
      Generic Routing Encapsulation - A common tunneling protocol for
      transmission of data in a segmented network.

    HA
      High Availability - The ability for a system to continue to provide
      service in the presence of hardware or software component failure.

    HAProxy
      A load-balancer for TCP connections that maintains a VIP and acts as a
      single endpoint for multiple instances of OpenStack API services.
      http://www.haproxy.org/

    Horizon
      OpenStack’s web user interface.
      https://docs.openstack.org/horizon/latest/

    IPMI
      Intelligent Platform Management Interface - Interface for providing
      remote control and monitoring of a server’s baseboard components.

    Kayobe
      An Ansible-driven OpenStack deployment framework capable of taking bare
      metal infrastructure through to post-deployment customization.

    Keystone
      OpenStack’s authentication, authorization and identity management
      service. All other OpenStack services authenticate user requests by
      validating cryptographic tokens passed by the requester.
      https://docs.openstack.org/keystone/latest/

    Kolla
      An OpenStack project for encapsulating all OpenStack services in Docker
      containers. https://docs.openstack.org/kolla/latest/

    MLAG
      Multi-Chassis Link Aggregate - a method of providing multi-pathing and
      multi-switch redundancy in layer-2 networks.

    Neutron
      OpenStack’s networking service.
      https://docs.openstack.org/neutron/latest/

    Nova
      OpenStack’s compute service. Nova is the scheduler for VMs in OpenStack
      and the manager of the compute hypervisors.
      https://docs.openstack.org/nova/latest/

    OIDC
      OpenID Connect - a protocol for federated authentication and identity
      management.

    OSPF
      Open Shortest Path First - a routing protocol for IPv4 and IPv6 for
      managing efficient paths between IP routers. OSPF is used in conjunction
      with ECMP to maintain multi-path route tables across an IP fabric.

    REST
      Representational State Transfer - a software paradigm in which no state
      is carried between successive API calls. This stateless model enables
      successive API requests to be serviced by different processes, and
      simplifies the process of scaling OpenStack control plane services.

    SDN
      Software Defined Networking - Technology that facilitates the
      programmatic control of network configuration and monitoring.

    VIP
      Virtual IP address - a method for implementing active-passive failover in
      which multiple services on different nodes share a common IP which can be
      transferred between them in the event of failover.

    VXLAN
      Virtual eXtensible LAN - a tunneling protocol used for tenant overlay
      networks. VXLAN is more portable and imposes fewer networking
      requirements than VLAN-based overlay networks. However, VXLAN can also
      incur more overhead and may not be usable for RDMA-enabled compute VMs.

    XMPP
      Extensible Messaging and Presence Protocol - Used for the exchange of
      XML-based structured data between network entities.

Contacting StackHPC Support
---------------------------

StackHPC's technical team is available for |support_level| for OpenStack issues
related to the |project_name| OpenStack deployment. StackHPC support can be
contacted through the dedicated |chat_system| channel or by direct email
contact at |support_email|.
