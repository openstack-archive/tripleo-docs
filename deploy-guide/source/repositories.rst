.. This should be changed to something more user-friendly like http://tripleo.org/tripleo-repos.rpm

.. warning::
   Support for Python3 is still experimental. The Fedora 28 specific notes
   and commands appearing below should not be taken as indication that this
   is fully supported by TripleO - we're still working on it!

#. Download and install the python2-tripleo-repos RPM from
   `the current RDO repository <https://trunk.rdoproject.org/centos7/current/>`_.
   For example

   .. code-block:: bash

      sudo yum install -y https://trunk.rdoproject.org/centos7/current/python2-tripleo-repos-<version>.el7.centos.noarch.rpm

   .. note::

      tripleo-repos removes any repositories that it manages before each run.
      This means all repositories must be specified in a single tripleo-repos
      call. As an example, the correct way to install the current and ceph repos
      is to run ``tripleo-repos current ceph``, not two separate calls.

   .. admonition:: Stable Branch
      :class: stable

      Enable the appropriate repos for the desired release, as indicated below.
      Do not enable any other repos not explicitly marked for that release.

   .. admonition:: Pike
      :class: pike otop

      Enable the current Pike repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b pike current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b pike current ceph

   .. admonition:: Queens
      :class: queens ptoq

      Enable the current Queens repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b queens current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b queens current ceph

   .. admonition:: Rocky
      :class: rocky qtor

      Enable the current Rocky repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b rocky current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b rocky current ceph

   .. admonition:: Stein
      :class: stein rtos

      Enable the current Stein repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b stein current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b stein current ceph

.. warning::

   The remaining repositories configuration steps below should not be done for
   stable releases!

2. Run tripleo-repos to install the appropriate repositories.  The option below
   will enable the latest master TripleO packages, the latest promoted
   packages for all other OpenStack services and dependencies and the latest
   stable Ceph packages. There are other repository configurations available in
   tripleo-repos, see its ``--help`` output for details.

   .. code-block:: bash

      sudo -E tripleo-repos current-tripleo-dev ceph
