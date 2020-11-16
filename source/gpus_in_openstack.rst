.. include:: vars.rst

=============================
Support for GPUs in OpenStack
=============================

This guide is has been developed for Nvidia GPUs and CentOS 8.

See `Kayobe Ops <https://github.com/stackhpc/kayobe-ops>`_ for
a playbook implementation of host setup for GPU.

BIOS Configuration Requirements
-------------------------------

On an Intel system:

* Enable `VT-x` in the BIOS for virtualisation support.
* Enable `VT-d` in the BIOS for IOMMU support.

Hypervisor Configuration Requirements
-------------------------------------

Find the GPU device IDs
~~~~~~~~~~~~~~~~~~~~~~~

From the host OS, use ``lspci -nn`` to find the PCI vendor ID and
device ID for the GPU device and supporting components.  These are
4-digit hex numbers.

For example:

.. code-block:: text

   01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204M [GeForce GTX 980M] [10de:13d7] (rev a1) (prog-if 00 [VGA controller])
   01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)

In this case the vendor ID is ``10de``, display ID is ``13d7`` and audio ID is ``0fbb``.

Alternatively, for an Nvidia Quadro RTX 6000:

.. code-block:: yaml

   # NVIDIA Quadro RTX 6000/8000 PCI device IDs
   vendor_id: "10de"
   display_id: "1e30"
   audio_id: "10f7"
   usba_id: "1ad6"
   usba_class: "0c0330"
   usbc_id: "1ad7"
   usbc_class: "0c8000"

These parameters will be used for device-specific configuration.

Kernel Ramdisk Reconfiguration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ramdisk loaded during kernel boot can be extended to include the
vfio PCI drivers and ensure they are loaded early in system boot.

.. code-block:: yaml

   - name: Template dracut config
     blockinfile:
       path: /etc/dracut.conf.d/gpu-vfio.conf
       block: |
         add_drivers+="vfio vfio_iommu_type1 vfio_pci vfio_virqfd"
       owner: root
       group: root
       mode: 0660
       create: true
     become: true
     notify:
       - Regenerate initramfs
       - reboot

The handler for regenerating the Dracut initramfs is:

.. code-block:: yaml

   - name: Regenerate initramfs
     shell: |-
       #!/bin/bash
       set -eux
       dracut -v -f /boot/initramfs-$(uname -r).img $(uname -r)
     become: true

Kernel Boot Parameters
~~~~~~~~~~~~~~~~~~~~~~

Set the following kernel parameters by adding to
``GRUB_CMDLINE_LINUX_DEFAULT`` or ``GRUB_CMDLINE_LINUX`` in
``/etc/default/grub.conf``.  We can use the
`stackhpc.grubcmdline <https://galaxy.ansible.com/stackhpc/grubcmdline>`_
role from Ansible Galaxy:

.. code-block:: yaml

   - name: Add vfio-pci.ids kernel args
     include_role:
       name: stackhpc.grubcmdline
     vars:
       kernel_cmdline:
         - intel_iommu=on
         - iommu=pt
         - "vfio-pci.ids={{ vendor_id }}:{{ display_id }},{{ vendor_id }}:{{ audio_id }}"
       kernel_cmdline_remove:
         - iommu
         - intel_iommu
         - vfio-pci.ids:

Kernel Device Management
~~~~~~~~~~~~~~~~~~~~~~~~

In the hypervisor, we must prevent kernel device initialisation of
the GPU and prevent drivers from loading for binding the GPU in the
host OS.  We do this using ``udev`` rules:

.. code-block:: yaml

   - name: Template udev rules to blacklist GPU usb controllers
     blockinfile:
       # We want this to execute as soon as possible
       path: /etc/udev/rules.d/99-gpu.rules
       block: |
         #Remove NVIDIA USB xHCI Host Controller Devices, if present
         ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x{{ vendor_id }}", ATTR{class}=="0x{{ usba_class }}", ATTR{remove}="1"
         #Remove NVIDIA USB Type-C UCSI devices, if present
         ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x{{ vendor_id }}", ATTR{class}=="0x{{ usbc_class }}", ATTR{remove}="1"
       owner: root
       group: root
       mode: 0644
       create: true
      become: true

Kernel Drivers
~~~~~~~~~~~~~~

Prevent the ``nouveau`` kernel driver from loading by
blacklisting the module:

.. code-block:: yaml

   - name: Blacklist nouveau
     blockinfile:
       path: /etc/modprobe.d/blacklist-nouveau.conf
       block: |
         blacklist nouveau
         options nouveau modeset=0
       mode: 0664
       owner: root
       group: root
       create: true
     become: true
     notify:
       - reboot
       - Regenerate initramfs

Ensure that the ``vfio`` drivers are loaded into the kernel on boot:

.. code-block:: yaml

   - name: Add vfio to modules-load.d
     blockinfile:
       path: /etc/modules-load.d/vfio.conf
       block: |
         vfio
         vfio_iommu_type1
         vfio_pci
         vfio_virqfd
       owner: root
       group: root
       mode: 0664
       create: true
     become: true
     notify: reboot

Once this code has taken effect (after a reboot), the VFIO kernel drivers should be loaded on boot:

.. code-block:: text

   # lsmod | grep vfio
   vfio_pci               49152  0
   vfio_virqfd            16384  1 vfio_pci
   vfio_iommu_type1       28672  0
   vfio                   32768  2 vfio_iommu_type1,vfio_pci
   irqbypass              16384  5 vfio_pci,kvm

OpenStack Nova configuration
----------------------------

Scheduler Filters
~~~~~~~~~~~~~~~~~

Hypervisor Resource Tracking
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configuration can be applied in flexible ways using Kolla-Ansible's
methods for `inventory-driven customisation of configuration
<https://docs.openstack.org/kayobe/latest/configuration/reference/kolla-ansible.html#service-configuration>`_.
The following configuration could be added to
``etc/kayobe/kolla/config/nova/nova-compute.conf`` to enable PCI
passthrough of GPU devices for hosts in a group named ``compute_gpu``.
Again, the 4-digit PCI Vendor ID and Device ID extracted from ``lspci
-nn`` can be used here to specify the GPU device(s).

.. code-block:: yaml

   [pci]
   {% raw %}
   {% if inventory_hostname in groups['compute_gpu'] %}
   # We could support multiple models of GPU.
   # This can be done more selectively using different inventory groups.
   # GPU models defined here:
   # NVidia Tesla V100 16GB
   # NVidia Tesla V100 32GB
   # NVidia Tesla P100 16GB
   passthrough_whitelist = [{ "vendor_id":"10de", "product_id":"1db4" },
                            { "vendor_id":"10de", "product_id":"1db5" },
                            { "vendor_id":"10de", "product_id":"15f8" }]
   alias = { "vendor_id":"10de", "product_id":"1db4", "device_type":"type-PCI", "name":"gpu-v100-16" }
   alias = { "vendor_id":"10de", "product_id":"1db5", "device_type":"type-PCI", "name":"gpu-v100-32" }
   alias = { "vendor_id":"10de", "product_id":"15f8", "device_type":"type-PCI", "name":"gpu-p100" }
   {% endif %}
   {% endraw %}


Testing GPU in a Guest VM
-------------------------

The Nvidia drivers must be installed first.  For example, on an Ubuntu guest:

.. code-block:: text

   sudo apt install nvidia-headless-440 nvidia-utils-440 nvidia-compute-utils-440

The ``nvidia-smi`` command will generate detailed output if the driver has loaded
successfully.

Further Reference
-----------------

For PCI Passthrough and GPUs in OpenStack:

* Consumer-grade GPUs: https://gist.github.com/claudiok/890ab6dfe76fa45b30081e58038a9215
* https://www.jimmdenton.com/gpu-offloading-openstack/
* https://docs.openstack.org/nova/latest/admin/pci-passthrough.html
* https://docs.openstack.org/nova/latest/admin/virtual-gpu.html (vGPU only)
* Telsa models in OpenStack: https://egallen.com/openstack-nvidia-tesla-gpu-passthrough/
* https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
* https://www.kernel.org/doc/Documentation/Intel-IOMMU.txt
* https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.1/html/installation_guide/appe-configuring_a_hypervisor_host_for_pci_passthrough
* https://www.gresearch.co.uk/article/utilising-the-openstack-placement-service-to-schedule-gpu-and-nvme-workloads-alongside-general-purpose-instances/

