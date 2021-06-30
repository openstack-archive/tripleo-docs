.. This should be changed to something more user-friendly like http://tripleo.org/tripleo-repos.rpm

.. note::
   Python3 is required for Ussuri and newer releases of OpenStack which is supported on RHEL 8
   and CentOS 8. Train is also recommended to be installed on RHEL 8 or CentOS 8.

#. Download and install the python-tripleo-repos RPM from
   the appropriate RDO repository

   .. admonition:: CentOS 8 and CentOS Strem 8
      :class: centos8

      Current `Centos 8 RDO repository <https://trunk.rdoproject.org/centos8/component/tripleo/current/>`_.

       .. code-block:: bash

          sudo dnf install -y https://trunk.rdoproject.org/centos8/component/tripleo/current/python3-tripleo-repos-<version>.el8.noarch.rpm

   .. note::

      tripleo-repos removes any repositories that it manages before each run.
      This means all repositories must be specified in a single tripleo-repos
      call. As an example, the correct way to install the current and ceph repos
      is to run ``tripleo-repos current ceph``, not two separate calls.

   .. admonition:: Stable Branch
      :class: stable

      Enable the appropriate repos for the desired release, as indicated below.
      Do not enable any other repos not explicitly marked for that release.

   .. admonition:: Wallaby
      :class: wallaby vtow

      Enable the current Wallaby repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b wallaby current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b wallaby current ceph

   .. admonition:: Victoria
      :class: victoria utov

      Enable the current Victoria repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b victoria current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b victoria current ceph

   .. admonition:: Ussuri
      :class: ussuri ttou

      Enable the current Ussuri repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b ussuri current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b ussuri current ceph

   .. admonition:: Train
      :class: train stot

      Enable the current Train repositories

      .. code-block:: bash

         sudo -E tripleo-repos -b train current

      .. admonition:: Ceph
         :class: ceph

         Include the Ceph repo in the tripleo-repos call

         .. code-block:: bash

            sudo -E tripleo-repos -b train current ceph

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
