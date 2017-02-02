Introduction
------------

The initial scope of this tutorial is to create a brief walkthrough with some
guidelines and naming conventions for future modules and features aligned with
the composable services architecture. Regarding the example described in this
tutorial, which leads to align an _existing_ 'non-composable' service implementation
with the composable roles approach, it is important to notice that a similar approach would be
followed if a user needed to add an entirely new service to a tripleo deployment.

.. _puppet/manifests: https://github.com/openstack/tripleo-heat-templates/tree/3d01f650f18b9e4f1892a6d9aa17f1bfc99b5091/puppet/manifests

The puppet manifests used to configure services on overcloud nodes currently
reside in the tripleo-heat-templates repository, in the folder `puppet/manifests`_.
In order to properly organize and structure the code, all
manifests will be re-defined in the puppet-tripleo repository, and adapted to
the `composable services architecture`_.

The use case for this example uses NTP as a service installed by default among
the OpenStack deployment. In this particular case, NTP is installed and
configured based on the code from puppet manifests:

``puppet/manifests/overcloud_cephstorage.pp``,
``puppet/manifests/overcloud_volume.pp``,
``puppet/manifests/overcloud_object.pp``,
``puppet/manifests/overcloud_compute.pp``,
``puppet/manifests/overcloud_controller.pp`` and
``puppet/manifests/overcloud_controller_pacemaker.pp``

Which means that NTP will be installed everywhere in the overcloud, so the
tutorial will describe the process of refactoring the code from those files
in order move it to the puppet-tripleo repository.

This tutorial is divided into several steps, according to different changes
that need to be added to the structure of tripleo-heat-templates and
puppet-tripleo.

Relevant repositories in this guide
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- tripleo-heat-templates: All the tripleo-heat-templates (aka THT) logic.

- puppet-tripleo: TripleO puppet manifests used to deploy the overcloud services.

- tripleo-puppet-elements: References puppet modules used by TripleO to deploy the overcloud services.
  (Not used in this tutorial)

Gerrit patches used in this example
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The gerrit patches used to describe this walkthrough are:

- https://review.openstack.org/#/c/310725/ (puppet-tripleo)

- https://review.openstack.org/#/c/310421/ (tripleo-heat-templates controller)

- https://review.openstack.org/#/c/330916/ (tripleo-heat-templates compute)

- https://review.openstack.org/#/c/330921/ (tripleo-heat-templates cephstorage)

- https://review.openstack.org/#/c/330923/ (tripleo-heat-templates objectstorage)


Change prerequisites
~~~~~~~~~~~~~~~~~~~~~

The controller services are defined and configured via Heat resource chains. In
the proposed patch (https://review.openstack.org/#/c/259568) controller
services will be wired to a new Heat feature that allows it to dynamically include
a set of nested stacks representing individual services via a Heat resource
chain. The current example will use this interface to decompose the controller
role into isolated services.