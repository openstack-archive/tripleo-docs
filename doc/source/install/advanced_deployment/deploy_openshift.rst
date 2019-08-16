(DEPRECATED) Deploying OpenShift
================================

.. note:: This functionality was removed as of Train.

You can use TripleO to deploy OpenShift clusters onto baremetal nodes.
TripleO deploys the operating system onto the nodes and uses
`openshift-ansible` to then configure OpenShift. TripleO can also be used
to manage the baremetal nodes.

Define the OpenShift roles
**************************

TripleO installs OpenShift services using composable roles for
`OpenShiftMaster`, `OpenShiftWorker`, and `OpenShiftInfra`. When you import
a baremetal node using `instackenv.json`, you can tag it to use a certain
composable role.

.. TODO(aschultz): update this with deploy guide link
.. See :doc:`custom_roles` for more information.

1. Generate the OpenShift roles:

.. code-block:: bash

  openstack overcloud roles generate -o /home/stack/openshift_roles_data.yaml \
    OpenShiftMaster OpenShiftWorker OpenShiftInfra

2. View the OpenShift roles:

.. code-block:: bash

  openstack overcloud role list

The result should include entries for `OpenShiftMaster`, `OpenShiftWorker`, and
`OpenShiftInfra`.

3. See more information on the `OpenShiftMaster` role:

.. code-block:: bash

  openstack overcloud role show OpenShiftMaster

.. note::
  For development or PoC environments that are more resource-constrained, it is
  possible to use the `OpenShiftAllInOne` role to collocate the different
  OpenShift services on the same node. The all-in-one role is not recommended
  for production.

Create the OpenShift profiles
*****************************

This procedure describes how to enroll a physical node as an OpenShift node.

1. Create a flavor for each OpenShift role. You will need to adjust this
   values to suit your requirements:

.. code-block:: bash

  openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 --swap 500 m1.OpenShiftMaster
  openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 --swap 500 m1.OpenShiftWorker
  openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 --swap 500 m1.OpenShiftInfra

2. Map the flavors to the required profile:

.. code-block:: bash

  openstack flavor set --property "capabilities:profile"="OpenShiftMaster" \
    --property "capabilities:boot_option"="local" m1.OpenShiftMaster
  openstack flavor set --property "capabilities:profile"="OpenShiftWorker" \
    --property "capabilities:boot_option"="local" m1.OpenShiftWorker
  openstack flavor set --property "capabilities:profile"="OpenShiftInfra" \
    --property "capabilities:boot_option"="local" m1.OpenShiftInfra

3. Add your nodes to `instackenv.json`. You will need to define them to use the
   `capabilities` field. For example:

.. code-block:: json

    [{
        "arch":"x86_64",
        "cpu":"4",
        "disk":"60",
        "mac":[
                "00:0c:29:9f:5f:05"
        ],
        "memory":"16384",
        "pm_type":"ipmi",
        "capabilities":"profile:OpenShiftMaster",
        "name": "OpenShiftMaster_1"
    },
    {
        "arch":"x86_64",
        "cpu":"4",
        "disk":"60",
        "mac":[
                "00:0c:29:91:b9:2d"
        ],
        "memory":"16384",
        "pm_type":"ipmi",
        "capabilities":"profile:OpenShiftWorker",
        "name": "OpenShiftWorker_1"
    },
    {
        "arch":"x86_64",
        "cpu":"4",
        "disk":"60",
        "mac":[
                "00:0c:29:91:b9:6a"
        ],
        "memory":"16384",
        "pm_type":"ipmi",
        "capabilities":"profile:OpenShiftInfra",
        "name": "OpenShiftInfra_1"
    }]

.. TOOD(aschultz): include reference to deploy guide

4. Import and introspect the TripleO nodes as you normally would for your
   deployment. For example:

.. code-block:: bash

  openstack overcloud node import ~/instackenv.json
  openstack overcloud node introspect --all-manageable --provide

5. Verify the overcloud nodes have assigned the correct profile

.. code-block:: bash

  openstack overcloud profiles list
  +--------------------------------------+--------------------+-----------------+-----------------+-------------------+
  | Node UUID                            | Node Name          | Provision State | Current Profile | Possible Profiles |
  +--------------------------------------+--------------------+-----------------+-----------------+-------------------+
  | 72b2b1fc-6ba4-4779-aac8-cc47f126424d | openshift-worker01 | available       | OpenShiftWorker |                   |
  | d64dc690-a84d-42dd-a88d-2c588d2ee67f | openshift-worker02 | available       | OpenShiftWorker |                   |
  | 74d2fd8b-a336-40bb-97a1-adda531286d9 | openshift-worker03 | available       | OpenShiftWorker |                   |
  | 0eb17ec6-4e5d-4776-a080-ca2fdcd38e37 | openshift-infra02  | available       | OpenShiftInfra  |                   |
  | 92603094-ba7c-4294-a6ac-81f8271ce83e | openshift-infra03  | available       | OpenShiftInfra  |                   |
  | b925469f-72ec-45fb-a403-b7debfcf59d3 | openshift-master01 | available       | OpenShiftMaster |                   |
  | 7e9e80f4-ad65-46e1-b6b4-4cbfa2eb7ea7 | openshift-master02 | available       | OpenShiftMaster |                   |
  | c2bcdd3f-38c3-491b-b971-134cab9c4171 | openshift-master03 | available       | OpenShiftMaster |                   |
  | ece0ef2f-6cc8-4912-bc00-ffb3561e0e00 | openshift-infra01  | available       | OpenShiftInfra  |                   |
  | d3a17110-88cf-4930-ad9a-2b955477aa6c | openshift-custom01 | available       | None            |                   |
  | 07041e7f-a101-4edb-bae1-06d9964fc215 | openshift-custom02 | available       | None            |                   |
  +--------------------------------------+--------------------+-----------------+-----------------+-------------------+

Configure the container registry
********************************

.. TODO(aschultz): include reference to deploy guide
.. Follow :doc:`container_image_prepare` to configure TripleO for the container
.. image preparatio.

This generally means generating a `/home/stack/containers-prepare-parameter.yaml` file:

.. code-block:: bash

  openstack tripleo container image prepare default \
    --local-push-destination \
    --output-env-file containers-prepare-parameter.yaml

Define the OpenShift environment
********************************

Create the `openshift_env.yaml` file. This file will define the
OpenShift-related settings that TripleO will later apply as part of the
`openstack overcloud deploy` procedure. You will need to update these values
to suit your deployment:

.. code-block:: yaml

    Parameter_defaults:
    # by default TripleO assigns the VIP random from the allocation pool
    # by using the FixedIPs we can set the VIPs to predictable IPs before starting the deployment
    CloudName: public.openshift.localdomain
    PublicVirtualFixedIPs: [{'ip_address':'10.0.0.200'}]

    CloudNameInternal: internal.openshift.localdomain
    InternalApiVirtualFixedIPs: [{'ip_address':'172.17.1.200'}]

    CloudDomain: openshift.localdomain

    ## Required for CNS deployments only
    OpenShiftInfraParameters:
        OpenShiftGlusterDisks:
            - /dev/sdb

    ## Required for CNS deployments only
    OpenShiftWorkerParameters:
        OpenShiftGlusterDisks:
            - /dev/sdb
            - /dev/sdc

    ControlPlaneDefaultRoute: 192.168.24.1
    EC2MetadataIp: 192.168.24.1
    ControlPlaneSubnetCidr: 24

    # The DNS server below should have entries for resolving
    # {internal,public,apps}.openshift.localdomain names
    DnsServers:
        - 10.0.0.90

    OpenShiftGlobalVariables:

        openshift_master_identity_providers:
            - name: 'htpasswd_auth'
              login: 'true'
              challenge: 'true'
              kind: 'HTPasswdPasswordIdentityProvider'
        openshift_master_htpasswd_users:
            sysadmin: '$apr1$jpBOUqeU$X4jUsMyCHOOp8TFYtPq0v1'

        #openshift_master_cluster_hostname should match the CloudNameInternal parameter
        openshift_master_cluster_hostname: internal.openshift.localdomain

        #openshift_master_cluster_public_hostname should match the CloudName parameter
        openshift_master_cluster_public_hostname: public.openshift.localdomain

        openshift_master_default_subdomain: apps.openshift.localdomain

For custom networks or customer interfaces, it is necessary to use custom
network interface templates:

.. code-block:: yaml

    resource_registry:
        OS::TripleO::OpenShiftMaster::Net::SoftwareConfig: /home/stack/master-nic.yaml
        OS::TripleO::OpenShiftWorker::Net::SoftwareConfig: /home/stack/worker-nic.yaml
        OS::TripleO::OpenShiftInfra::Net::SoftwareConfig: /home/stack/infra-nic.yaml

Deploy OpenShift nodes
**********************

As a result of the previous steps, you will have three new YAML files:

* `openshift_env.yaml`
* `openshift_roles_data.yaml`
* `containers-default-parameters.yaml`

For a custom network deployments, maybe it is necessary NICs and network
templates like:

* `master-nic.yaml`
* `infra-nic.yaml`
* `worker-nic.yaml`
* `network_data_openshift.yaml`

Add these YAML files to your `openstack overcloud deploy` command.

An example for CNS deployments:

.. code-block:: bash

  openstack overcloud deploy \
    --stack openshift \
    --templates \
    -r /home/stack/openshift_roles_data.yaml \
    -n /usr/share/openstack-tripleo-heat-templates/network_data_openshift.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/openshift.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/openshift-cns.yaml \
    -e /home/stack/openshift_env.yaml \
    -e /home/stack/containers-prepare-parameter.yaml

An example for non-CNS deployments:

.. code-block:: bash

  openstack overcloud deploy \
    --stack openshift \
    --templates \
    -r /home/stack/openshift_roles_data.yaml \
    -n /usr/share/openstack-tripleo-heat-templates/network_data_openshift.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/openshift.yaml \
    -e /home/stack/openshift_env.yaml \
    -e /home/stack/containers-prepare-parameter.yaml

Deployment for custom networks or interfaces, it is necessary to specify them.
For example:

.. code-block:: bash

  openstack overcloud deploy \
    --stack openshift \
    --templates \
    -r /home/stack/openshift_roles_data.yaml \
    -n /home/stack/network_data_openshift.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/openshift.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/openshift-cns.yaml \
    -e /home/stack/openshift_env.yaml \
    -e /home/stack/containers-prepare-parameter.yaml \
    -e /home/stack/custom-nics.yaml

Review the OpenShift deployment
*******************************

Once the overcloud deploy procedure has completed, you can review the state
of your OpenShift nodes.

1. List all your baremetal nodes. You should expect to see your master, infra,
   and worker nodes.

   .. code-block:: bash

      openstack baremetal node list

2. Locate the OpenShift node:

   .. code-block:: bash

      openstack server list

3. SSH to the OpenShift node. For example:

   .. code-block:: bash

      ssh heat-admin@192.168.122.43

4. Change to root user:

   .. code-block:: bash

      sudo -i

5. Review the container orchestration configuration:

   .. code-block:: bash

      cat .kube/config

6. Login to OpenShift:

   .. code-block:: bash

      oc login -u admin

7. Review any existing projects:

   .. code-block:: bash

      oc get projects

8. Review the OpenShift status:

   .. code-block:: bash

      oc status

9. Logout from OpenShift:

   .. code-block:: bash

      oc logout

Deploy a test app using OpenShift
*********************************

This procedure describes how to create a test application in your new
OpenShift deployment.

1. Login as a developer:

   .. code-block:: bash

      $ oc login -u developer
      Logged into "https://192.168.64.3:8443" as "developer" using existing credentials.
      You have one project on this server: "myproject"
      Using project "myproject".

2. Create a new project:

   .. code-block:: bash

      $ oc new-project test-project
      Now using project "test-project" on server "https://192.168.64.3:8443".

   You can add applications to this project with the 'new-app' command.
   For example, to build a new example application in Ruby try:

   .. code-block:: bash

      $ oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

3. Create a new app. This example creates a CakePHP application:

   .. code-block:: bash

    $ oc new-app https://github.com/sclorg/cakephp-ex
    --> Found image 9dd8c80 (29 hours old) in image stream "openshift/php" under tag "7.1" for "php"

        Apache 2.4 with PHP 7.1
        -----------------------
        PHP 7.1 available as container is a base platform for building and running various PHP 7.1 applications and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated web pages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.

        Tags: builder, php, php71, rh-php71

        * The source repository appears to match: php
        * A source build using source code from https://github.com/sclorg/cakephp-ex will be created
        * The resulting image will be pushed to image stream "cakephp-ex:latest"
        * Use 'start-build' to trigger a new build
        * This image will be deployed in deployment config "cakephp-ex"
        * Ports 8080/tcp, 8443/tcp will be load balanced by service "cakephp-ex"
        * Other containers can access this service through the hostname "cakephp-ex"

    --> Creating resources ...
        imagestream "cakephp-ex" created
        buildconfig "cakephp-ex" created
        deploymentconfig "cakephp-ex" created
        service "cakephp-ex" created
    --> Success
        Build scheduled, use 'oc logs -f bc/cakephp-ex' to track its progress.
        Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
        'oc expose svc/cakephp-ex'
        Run 'oc status' to view your app.

4. Review the new app:

   .. code-block:: bash

        $ oc status --suggest
        In project test-project on server https://192.168.64.3:8443

        svc/cakephp-ex - 172.30.171.214 ports 8080, 8443
        dc/cakephp-ex deploys istag/cakephp-ex:latest <-
            bc/cakephp-ex source builds https://github.com/sclorg/cakephp-ex on openshift/php:7.1
            build #1 running for 52 seconds - e0f0247: Merge pull request #105 from jeffdyoung/ppc64le (Honza Horak <hhorak@redhat.com>)
            deployment #1 waiting on image or update

        Info:
        * dc/cakephp-ex has no readiness probe to verify pods are ready to accept traffic or ensure deployment is successful.
            try: oc set probe dc/cakephp-ex --readiness ...
        * dc/cakephp-ex has no liveness probe to verify pods are still running.
            try: oc set probe dc/cakephp-ex --liveness ...

        View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'.

5. Review the pods:

   .. code-block:: bash

    $ oc get pods
    NAME                 READY     STATUS    RESTARTS   AGE
    cakephp-ex-1-build   1/1       Running   0          1m

6. Logout from OpenShift:

   .. code-block:: bash

    $ oc logout
