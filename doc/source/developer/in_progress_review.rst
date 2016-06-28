Using an In-Progress Review
===========================

To use a git checkout for only a specific module, export the following variable::

    export DIB_INSTALLTYPE_puppet_tripleo=source

Replace ``puppet_tripleo`` with the name of the puppet module to be installed
from source, replacing any -'s with _'s.

To use a pending review for a module, set its installtype to source as
described above, then also export the following variables::

    export DIB_REPOLOCATION_puppet_tripleo=https://review.openstack.org/openstack/puppet-tripleo
    export DIB_REPOREF_puppet_tripleo=refs/changes/30/223330/1

This time replace the name of the module in the variable name and the review URL.
The correct value for the ``reporef`` can be found in the ``Download`` section
of the Gerrit UI.  Look for a string that matches the format of the example above.
