.. _notes-for-stack-updates:

Understanding undercloud/standalone stack updates
=================================================

To update a service password or a secret when upgrading from a
non-containerized undercloud, you should edit ``undercloud.conf``.
Then you should use the ``openstack undercloud upgrade`` command.

.. note:: ``undercloud.conf`` takes priority over
   ``tripleo-undercloud-passwords.yaml`` only when running the undercloud
   upgrade command.  For the undercloud install command, you should edit
   ``tripleo-undercloud-passwords.yaml`` instead.

In order to apply changes for an existing containerized undercloud or
standalone installation, there is an important thing to remember.

Undercloud and standalone heat installers create one-time ephemeral stacks.
Unlike the normal overcloud stacks, they cannot be updated via the regular
stack update procedure.  Instead, the created heat stacks may be updated
virtually. For the most of the cases, the installer will take care of it
automatically via the `StackAction` heat parameter overrides.

You can enforce the virtual update/create of the heat stack via
the ``--force-stack-update`` and ``--force-stack-create`` options.

And the recommended command to apply changes for an existing containerized
undercloud installation is:

.. code-block:: bash

   openstack undercloud install --force-stack-update

Otherwise, start a new installation with ``--force-stack-create``. New
passwords will be generated in ``tripleo-undercloud-passwords.yaml``.

It is better to be always explicit.

.. note:: The console log for these operations will always have heat reporting
   the STACK_CREATED status. Check the deployment logs for the actual virtual
   create or update actions taken.
