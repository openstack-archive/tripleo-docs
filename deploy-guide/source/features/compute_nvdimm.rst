Manage Virtual Persistent Memory (vPMEM)
=====================================================
Virtual Persistent Memory (vPMEM) is a Nova feature that allows to expose
Persistent Memory (PMEM) namespaces to guests using libvirt compute driver.
This guide show how the vPMEM feature is supported in TripleO deployment
framework. For in-depth description of Nova's vPMEM feature check Nova
documentation: `Attaching virtual persistent memory to guests
<https://docs.openstack.org/nova/latest/admin/virtual-persistent-memory.html>`_

.. warning::

  vPMEM feature is only available in Train(20.0.0) or later releases.

.. contents::
  :depth: 3
  :backlinks: none

Prerequisite
------------
Operators needs to properly configured PMEM Hardware before deploying Overcloud
with vPMEM support. Example of such a hardware is Intel Optane DC Persistent Memory.
Intel provides tool (`ipmctl <https://software.intel.com/en-us/articles/quick-start-guide-configure-intel-optane-dc-persistent-memory-on-linux>`_)
to configure the PMEM hardware.

Operators need to configure the hardware in such a way to enable TripleO to create
`PMEM namespaces <http://pmem.io/ndctl/ndctl-create-namespace.html>`_ in **devdax** mode.
TripleO currently support one backend NVDIMM region, so in case of multiple NVDIMMs
Interleaved Region needs to be configured.

TripleO vPMEM parameters
------------------------

Following parameter are used within TripleO to configure vPMEM:

    .. code::

        NovaPMEMMappings:
          type: string
          description: >
            PMEM namespace mappings as backend for vPMEM feature. This parameter
            sets Nova's `pmem_namespaces` configuration options. PMEM namespaces
            needs to be create manually or with conjunction with `NovaPMEMNamespaces`
            parameter.
            Requires format: $LABEL:$NSNAME[|$NSNAME][,$LABEL:$NSNAME[|$NSNAME]].
          default: ""
          tags:
            - role_specific
        NovaPMEMNamespaces:
          type: string
          description: >
            Creates PMEM namespaces on the host server using `ndctl` tool
            through Ansible.
            Requires format: $SIZE:$NSNAME[,$SIZE:$NSNAME...].
            $SIZE supports the suffixes "k" or "K" for KiB, "m" or "M" for MiB, "g"
            or "G" for GiB and "t" or "T" for TiB.
            NOTE: This requires properly configured NVDIMM regions and enough space
            for requested namespaces.
          default: ""
          tags:
            - role_specific

Both parameters are role specific and should be used with custom role. Please check documentation on
how to use `Role-Specific Parameters <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/role_specific_parameters.html>`_.

Examples
--------
    .. code::

        parameter_defaults:
          ComputePMEMParameters:
            NovaPMEMMappings: "6GB:ns0|ns1|ns2,LARGE:ns3"
            NovaPMEMNamespaces: "6G:ns1,6G:ns1,6G:ns2,100G:ns3"


The following example will peform following steps:
* ensure **ndctl** tool is installed on hosts with role **ComputePMEM**
* create PMEM namespaces as specified in the **NovaPMEMNamespaces** parameter.
- ns0, ns1, ns2 with size 6GiB
- ns3 with size 100GiB
* set Nova prameter **pmem_namespaces** in nova.conf to map create namespaces to vPMEM as specified in **NovaPMEMMappings**.
In this example the label '6GB' will map to one of ns0, ns1 or ns2 namespace and the label 'LARGE' will map to ns3 namespace.

After deployment you need to configure flavors as described in documentation `Nova: Configure a flavor <https://docs.openstack.org/nova/latest/admin/virtual-persistent-memory.html#configure-a-flavor>`_
