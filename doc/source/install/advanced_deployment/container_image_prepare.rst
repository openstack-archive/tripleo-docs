.. _prepare-environment-containers:

Container Image Preperation
===========================

This documentation explains how to instruct container image preparation to do
different preparation tasks.

Choosing an image registry strategy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Container images need to be pulled from an image registry which is reliably
available to overcloud nodes. The three common options to serve images are to
use the default registry, the registry availably on the undercloud, or an
independently managed registry. During deployment the environment parameter
`ContainerImagePrepare` is used to specify any desired behaviour, including:

- Where to pull images from
- Optionally, which local repository to push images to
- How to discover the lasted versioned tag for each image

In the following examples, the parameter `ContainerImagePrepare` will be
specified in its own file `containers-prepare-parameter.yaml`.

Default registry
................

By default the images will be pulled from a remote registry namespace such as
`docker.io/tripleomaster`. This is fine for development or POC clouds but is
not appropriate for production clouds due to the transfer of large amounts of
duplicate image data over a potentially unreliable internet connection.

During deployment with this default, any heat parameters which refer to
required container images will be populated with a value pointing at the
default registry, with a tag representing the latest image version.

To generate the `containers-prepare-parameter.yaml` containing these defaults,
run this command::

  openstack tripleo container image prepare default \
    --output-env-file containers-prepare-parameter.yaml

This will generate a file containing a `ContainerImagePrepare` similar to the
following::

  parameter_defaults:
    ContainerImagePrepare:
    - set:
        ceph_image: daemon
        ceph_namespace: docker.io/ceph
        ceph_tag: v3.0.3-stable-3.0-luminous-centos-7-x86_64
        name_prefix: centos-binary-
        name_suffix: ''
        namespace: docker.io/tripleomaster
        neutron_driver: null
        tag: current-tripleo
      tag_from_label: rdo_version

During deployment, this will lookup images in `docker.io/tripleomaster` tagged
with `current-tripleo` and discover a versioned tag by looking up the label
`rdo_version`. This will result in the heat image parameters in the plan being
set with appropriate values, such as::

  DockerNeutronMetadataImage: docker.io/tripleomaster/centos-binary-neutron-metadata-agent:35414701c176a6288fc2ad141dad0f73624dcb94_43527485
  DockerNovaApiImage: docker.io/tripleomaster/centos-binary-nova-api:35414701c176a6288fc2ad141dad0f73624dcb94_43527485

.. note:: The tag is actually a Delorean hash. You can find out the versions
          of packages by using this tag.
          For example, `35414701c176a6288fc2ad141dad0f73624dcb94_43527485` tag,
          is in fact using this `Delorean repository`_.

.. _populate-local-registry-containers:

Undercloud registry
...................

As part of the undercloud install, an image registry is configured on port
`8787`.  This can be used to increase reliability of image pulls, and minimise
overall network transfers.
The undercloud registry can be used by generating the following
`containers-prepare-parameter.yaml` file::

  openstack tripleo container image prepare default \
    --local-push-destination \
    --output-env-file containers-prepare-parameter.yaml

This will generate a file containing a `ContainerImagePrepare` similar to the
following::

  parameter_defaults:
    ContainerImagePrepare:
    - push_destination: true
      set:
        ceph_image: daemon
        ceph_namespace: docker.io/ceph
        ceph_tag: v3.0.3-stable-3.0-luminous-centos-7-x86_64
        name_prefix: centos-binary-
        name_suffix: ''
        namespace: docker.io/tripleomaster
        neutron_driver: null
        tag: current-tripleo
      tag_from_label: rdo_version

This is identical to the default registry, except for the `push_destination:
true` entry which indicates that the address of the local undercloud registry
will be discovered at upload time.

By specifying a `push_destination` value such as `192.168.24.1:8787`, during
deployment all images will be pulled from the remote registry then pushed to
the specified registry. The resulting image parameters will also be modified to
refer to the images in `push_destination` instead of `namespace`.

Running container image prepare
...............................
The prepare operations are actually run at the following times:

#. During ``undercloud install`` when `undercloud.conf` has
   `container_images_file=$HOME/containers-prepare-parameter.yaml` (see
   :ref:`install_undercloud`)
#. Before overcloud deploy when ``openstack tripleo container image prepare``
   is run (see :ref:`overcloud-prepare-container-images`)


Options available in heat parameter ContainerImagePrepare
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To do something different to the above two registry scenarios, your custom
environment can set the value of the ContainerImagePrepare heat parameter to
result in any desired registry and image scenario.

Discovering versioned tags with tag_from_label
..............................................

If you want these parameters to have the actual tag `current-tripleo` instead of
the discovered tag (in this case the Delorean hash,
`35414701c176a6288fc2ad141dad0f73624dcb94_43527485` ) then the `tag_from_label`
entry can be omitted.

Likewise, if all images should be deployed with a different tag, the value of
`tag` can be set to the desired tag.

Some build pipelines have a versioned tag which can only be discovered via a
combination of labels. For this case, a template format can be specified
instead::

      tag_from_label: {version}-{release}

Copying images with push_destination
....................................

By specifying a `push_destination`, the required images will be copied from
`namespace` to this registry, for example::

  ContainerImagePrepare:
  - push_destination: 192.168.24.1:8787
    set:
      namespace: docker.io/tripleomaster
      ...

This will result in images being copied from `docker.io/tripleomaster` to
`192.168.24.1:8787/tripleomaster` and heat parameters set with values such as::

  DockerNeutronMetadataImage: 192.168.24.1:8787/tripleomaster/centos-binary-neutron-metadata-agent:35414701c176a6288fc2ad141dad0f73624dcb94_43527485
  DockerNovaApiImage: 192.168.24.1:8787/tripleomaster/centos-binary-nova-api:35414701c176a6288fc2ad141dad0f73624dcb94_43527485

.. note:: Use the IP address of your undercloud, which you previously set with
    the `local_ip` parameter in your `undercloud.conf` file. For these example
    commands, the address is assumed to be `192.168.24.1:8787`.

By setting different values for `namespace` and `push_destination` any
alternative registry strategy can be specified.

Ceph and other set options
..........................

The options `ceph_namespace`, `ceph_image`, and `ceph_tag` are similar to
`namespace` and `tag` but they specify the values for the ceph image. It will
often come from a different registry, and have a different versioned tag
policy.

The values in the `set` map are used when evaluating the file
`/usr/share/openstack-tripleo-common/container-images/overcloud_containers.yaml.j2`
as a Jinja2 template. This file contains the list of every container image and
how it relates to TripleO services and heat parameters.

Layering image preparation entries
..................................

Since the value of `ContainerImagePrepare` is a list, multiple entries can be
specified, and later entries will overwrite any earlier ones. Consider the
following::

  ContainerImagePrepare:
  - tag_from_label: rdo_version
    push_destination: true
    excludes:
    - nova-api
    set:
      namespace: docker.io/tripleomaster
      name_prefix: centos-binary-
      name_suffix: ''
      tag: current-tripleo
  - push_destination: true
    includes:
    - nova-api
    set:
      namespace: mylocal
      tag: myhotfix

This will result in the following heat parameters which shows a `locally built
<build_container_images>`
and tagged `centos-binary-nova-api` being used for `DockerNovaApiImage`::

  DockerNeutronMetadataImage: 192.168.24.1:8787/tripleomaster/centos-binary-neutron-metadata-agent:35414701c176a6288fc2ad141dad0f73624dcb94_43527485
  DockerNovaApiImage: 192.168.24.1:8787/mylocal/centos-binary-nova-api:myhotfix

The `includes` and `excludes` entries can control the resulting image list in
addition to the filtering which is determined by roles and containerized
services in the plan. `includes` matches take precedence over `excludes`
matches, followed by role/service filtering. The image name must contain the
value within it to be considered a match.

..  _Delorean repository: https://trunk.rdoproject.org/centos7-master/ac/82/ac82ea9271a4ae3860528eaf8a813da7209e62a6_28eeb6c7/
