Splitting the Overcloud stack into multiple independent Heat stacks
===================================================================

.. note:: Since victoria TripleO provisions baremetal using a separate
   workflow :doc:`../provisioning/baremetal_provision` that does not
   involve Heat stack, making this feature irrelevant.

split-stack is a feature in TripleO that splits the overcloud stack into
multiple independent stacks in Heat.

The ``overcloud`` stack is split into an ``overcloud-baremetal`` and
``overcloud-services`` stack. This allows for independent and isolated
management of the baremetal and services part of the Overcloud deployment. It
is a more modular design than deploying a single ``overcloud`` stack in that it
allows either the baremetal or services stack to be replaced by tooling that is
external to TripleO if desired.

The ``overcloud-services`` stack makes extensive use of the deployed-server
feature, documented at :doc:`deployed_server` in order to orchestrate the
deployment and configuration of the services separate from the baremetal
deployment.


split-stack allows for mixing baremetal systems deployed by TripleO and those
deployed by external tooling when creating the services stack. Since the
baremetal resources are completely abstracted behind the deployed-server
interface when deploying the services stack, it does not matter whether the
servers were actually created with TripleO or not.


split-stack Requirements
------------------------

A default split-stack deployment (detailed in the later steps) can be deployed
without any special requirements.

More advanced deployments where baremetal servers provisioned by TripleO will
be mixed with those not provisioned by TripleO will want to pay attention to
the requirements around using already deployed servers from
:doc:`deployed_server`. The requirements for using deployed servers will apply
when not using servers provisioned by TripleO.

Default split-stack deployment
------------------------------

split-stack will be deployed by running 2 separate ``openstack overcloud
deploy`` commands to deploy the separate stacks.

If applicable, prepare the custom roles files and any custom environments
initially. The custom roles file and an environment setting the role counts
should be passed to both deployment commands so that enough baremetal nodes are
deployed per what the ``overcloud-services`` stack expects.

Baremetal Deployment Command
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Run the deployment command to deploy the ``overcloud-baremetal`` stack.
An additional environment file, ``overcloud-baremetal.yaml``, is passed to the
deployment to enable deploying just the baremetal stack.

Enough baremetal nodes should be deployed to match how many nodes per role will
be needed when the services stack is deployed later. Be sure that the
environment file being used to set the role counts is passed to the baremetal
deployment command::

    openstack overcloud deploy \
      <other cli arguments> \
      --stack overcloud-baremetal \
      -r roles-data.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/overcloud-baremetal.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/split-stack-consistent-hostname-format.yaml

The ``--stack`` argument sets the name of the Heat stack to
``overcloud-baremetal``. This will also be the name of the Swift container that
stores the stack's plan (templates) and of the Mistral environment.

The ``roles-data.yaml`` roles file illustrates passing a custom roles file to
the deployment command. It is not necessary to use custom roles when using
split stack, however if custom roles are used, the same roles file should be
used for both stacks.

The ``overcloud-baremetal.yaml`` environment will set the parameters for the
deployment such that no services will be deployed.

The ``split-stack-consistent-hostname-format.yaml`` environment will set the
respective ``<role-name>HostnameFormat`` parameters for each role defined in
the role files used. The server hostnames for the 2 stacks must be the same,
otherwise the servers will not be able to pull their deployment metadata from
Heat.

.. warning::

  Do not pass any network isolation templates or NIC config templates to the
  ``overcloud-baremetal`` stack deployment command. These will only be passed
  to the ``overcloud-services`` stack deployment command.

Services Deployment Command
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The services stack, ``overcloud-services`` will now be deployed with a separate
deployment command::

    openstack overcloud deploy \
      <other cli arguments> \
      --stack overcloud-services \
      --disable-validations \
      -r roles-data.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-deployed-neutron-ports.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-bootstrap-environment-centos.yaml \
      -e /usr/share/openstack-tripleo-heat-templates/environments/split-stack-consistent-hostname-format.yaml

The ``overcloud-services`` stack makes use of the "deployed-server" feature.
The additional environments needed are shown in the above command. See
:doc:`deployed_server` for more information on how to fully configure the
feature.

The roles file, ``roles-data.yaml`` is again passed to the services stack as
the same roles file should be used for both stacks.

The ``split-stack-consistent-hostname-format.yaml`` environment is again
passed, so that the hostnames used for the server resources created by Heat are
the same as were created in the previous baremetal stack.

During this deployment, any network isolation environments and/or NIC config
templates should be passed for the desired network configuration.

The stack should complete and the generated ``overcloudrc`` can be used to
interact with the Overcloud.
