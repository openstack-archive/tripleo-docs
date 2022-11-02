.. This should be changed to something more user-friendly like http://tripleo.org/tripleo-repos.rpm

.. note::
   Python3 is required for current releases of OpenStack which is
   supported on CentOS Stream 9.

#. Download and install the python-tripleo-repos RPM from
   the appropriate RDO repository

   .. admonition:: CentOS Stream 9
      :class: centos9

      Current `Centos 9 RDO repository <https://trunk.rdoproject.org/centos9/component/tripleo/current/>`_.

       .. code-block:: bash

          sudo dnf install -y https://trunk.rdoproject.org/centos9/component/tripleo/current/python3-tripleo-repos-<version>.el9.noarch.rpm

   .. note::

      tripleo-repos removes any repositories that it manages before each run.
      This means all repositories must be specified in a single tripleo-repos
      call. As an example, the correct way to install the current and ceph repos
      is to run ``tripleo-repos current ceph``, not two separate calls.

2. Run tripleo-repos to install the appropriate repositories.  The option below
   will enable the latest master TripleO packages, the latest promoted
   packages for all other OpenStack services and dependencies and the latest
   stable Ceph packages. There are other repository configurations available in
   tripleo-repos, see its ``--help`` output for details.

   .. code-block:: bash

      sudo -E tripleo-repos current-tripleo-dev ceph
