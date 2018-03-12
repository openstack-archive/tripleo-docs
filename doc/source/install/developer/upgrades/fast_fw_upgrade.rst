TripleO Fast Forward Upgrade (FFU) N -> N+3
----------------------------------------------------

For a detailed background on how the Fast Forward Upgrade (FFU) workflow was
proposed please refer to the relevant spec_. For a guide on running the FFU in
your environment see the `ffu-docs`_. This document will explore some
of the technical details of the Newton to Queens FFU specifically.

At a high level the FFU workflow consists of the following steps:

1. Perform a `Minor update`_ on the environment (both undercloud and overcloud)
   to bring it to the latest Newton. This will include OS level updates, including kernel
   and openvswitch. As usual for minor update the operator will reboot each
   node as necessary and so doing this first means the FFU workflow doesn't
   (also) have to deal with node reboots later on in the process.

2. Perform 3 consecutive major upgrades of the undercloud to bring it to
   Queens. The undercloud will crucially then have the target version
   of the tripleo-heat-templates including the fast_forward_upgrade_tasks
   that will deliver the next stages of the workflow.

3. Generate and then run the fast_forward_upgrade_playbook on the overcloud. This will:

   3.1 First bring down the controlplane services on **all nodes**.

   3.2 Then update packages, migrate databases and any other version specific
       tasks from Newton to Ocata then Ocata to Pike. This happens only
       on a **single node of each role**.

4. Finally run the Pike to Queens upgrade on all nodes including the Queens
   upgrade tasks and service configurations.

Step 3 above is started by first performing a Heat stack update using the Queens
tripleo-heat-templates from the Queens upgraded undercloud, but without applying any
configuration. This stack update is only used to collect the fast_forward_upgrade_tasks
(ffu_tasks) from each of the services deployed in the given environment and
generate a fast_forward_upgrade_playbook_ ansible playbook. This playbook is
then executed to deliver steps 3.1 and 3.2 above. See below for more information
about how the ffu_tasks are compiled into the fast_forward_upgrade_playbook.

A notable exception worthy of mention is the configuration of Ceph services
which is managed by ceph-ansible_. That is, for Ceph services there is no
collection of fast_forward_upgrade_tasks from the ceph related service manifests
in the tripleo-heat-templates and so Ceph is not managed by the generated
fast_forward_upgrade_playbook_. Instead ceph-ansible_ will be invoked by
the Queens deployment and service configuration in step 4 above.

The Heat stack update performed at the start of step 3 also generates the Queens
upgrade_steps_playbook_ and deploy_steps_playbook_ ansible playbooks. One
notable exception is the configuration of Ceph services which is managed
by ceph-ansible_
Step 4 above (Pike to Queens upgrade tasks and Queens services configuration)
is delivered through execution of these Heat stack update generated playbooks.
Ceph related upgrade and deployment will be applied here with calls to
ceph-ansible_.

Amongst other things, the P..Q upgrade_tasks stop and disable those systemd
services that are being migrated to run in containers. The Queens deploy_steps_playbook_
will then apply the required puppet and docker configuration to start the
containers for those services. For this to be possible the Heat stack update
which starts step 3 and that generates the ansible playbooks must include the
required `docker configuration and environment`_ files, including the latest
container images and making sure to set the to-be containerized services to refer
to the equivalent `docker templates`_ for the Heat resource registry.

.. _ffu-docs: https://review.openstack.org/#/c/549892/
.. _Minor update: https://docs.openstack.org/tripleo-docs/latest/install/post_deployment/package_update.html
.. _upgrade_steps_playbook: https://github.com/openstack/tripleo-heat-templates/blob/82f128f15b1b1eb7bf6ac7df0c6d01e5619309eb/common/deploy-steps.j2#L528
.. _deploy_steps_playbook: https://github.com/openstack/tripleo-heat-templates/blob/82f128f15b1b1eb7bf6ac7df0c6d01e5619309eb/common/deploy-steps.j2#L382
.. _fast_forward_upgrade_playbook: https://review.openstack.org/#/c/499221/20/common/deploy-steps.j2@541
.. _docker configuration and environment: https://docs.openstack.org/tripleo-docs/latest/install/containers_deployment/overcloud.html#preparing-the-environment
.. _docker templates: https://github.com/openstack/tripleo-heat-templates/blob/750fa306ce41c949928d5a3a7253aff99dd1af8f/environments/docker.yaml#L7-L58
.. _ceph-ansible: https://github.com/ceph/ceph-ansible

FFU and tripleo-heat-templates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section will present an overview of how the fast_forward_upgrade_playbook.yaml
is generated from the tripleo-heat-templates.

FFU uses *fast_forward_upgrade_tasks* (ffu_tasks) to define the upgrade
workflow. These are 'normal' ansible tasks and they are carried as a list in
the outputs section of a given service manifest, see containerized
`neutron-api`_ for an example.

The ffu_tasks for those services that are enabled in a given deployment are
collected in the outputs of the deploy-steps.j2_ into a
*fast_forward_upgrade_playbook* output. This is then retrieved using the
config-download_ mechanism and written to disk as an ansible playbook.

The *fast_forward_upgrade_tasks* defined for a given service can use the
**step** and **release** variables to specify when a given task should be
executed. At a high level the fast_forward_upgrade_playbook consists of two
loops - there is a very good explanation in `/#/c/499221 <https://review.openstack.org/#/c/499221/>`_
commit message, but an outer loop for the release (first Ocata tasks then Pike
tasks) and then an inner loop for the steps within each release.

The *ffu_tasks* which are set to run in steps 0 to 3 are designated the
*fast_forward_upgrade_prep_role_tasks* and these are executed on all nodes for
a given role. Then the *ffu_tasks* which have steps 4 to max (currently 9) are
designated the *fast_forward_upgrade_bootstrap_role_tasks* and these are only
executed on a single node for each role (one controller, one compute etc).

The top level fast_forward_upgrade_playbook.yaml looks like::

        - hosts: overcloud
          become: true
          tasks:
            - include_tasks: fast_forward_upgrade_release_tasks.yaml
              loop_control:
                loop_var: release
              with_items: {get_param: [FastForwardUpgradeReleases]}

The *fast_forward_upgrade_release_tasks.yaml* in turn looks like::

        - include_tasks: fast_forward_upgrade_prep_tasks.yaml
        - include_tasks: fast_forward_upgrade_bootstrap_tasks.yaml

The *fast_forward_upgrade_prep_tasks.yaml* specifies the loop with
sequence 0 to 3 as explained above::

         - include_tasks: fast_forward_upgrade_prep_role_tasks.yaml
           with_sequence: start=0 end=3
           loop_control:
           loop_var: step

And where the *fast_forward_upgrade_prep_role_tasks.yaml* includes the
*ffu_tasks* on all nodes for each role::

         - include_tasks: Controller/fast_forward_upgrade_tasks.yaml
           when: role_name == 'Controller'
         - include_tasks: Compute/fast_forward_upgrade_tasks.yaml
           when: role_name == 'Compute'
         ...etc

Similarly for the *fast_forward_upgrade_bootstrap_tasks.yaml* it specifies
the loop sequence for the step variable to be 4 to 9::

         - include_tasks: fast_forward_upgrade_bootstrap_role_tasks.yaml
           with_sequence: start=4 end=9
           loop_control:
           loop_var: step

And where the *fast_forward_upgrade_bootstrap_role_tasks.yaml* include the
*ffu_tasks* only on a single node for each role type::

         - include_tasks: Controller/fast_forward_upgrade_tasks.yaml
           when: role_name == 'Controller' and ansible_hostname == Controller[0]
         - include_tasks: Compute/fast_forward_upgrade_tasks.yaml
           when: role_name == 'Compute' and ansible_hostname == Compute[0]
         ...etc

.. _neutron-api: https://github.com/openstack/tripleo-heat-templates/blob/master/docker/services/neutron-api.yaml#L190
.. _spec: https://github.com/openstack/tripleo-specs/blob/master/specs/queens/fast-forward-upgrades.rst
.. _deploy-steps.j2: https://github.com/openstack/tripleo-heat-templates/blob/master/common/deploy-steps.j2#L377
.. _config-download: https://github.com/openstack/tripleo-common/blob/master/tripleo_common/utils/config.py

