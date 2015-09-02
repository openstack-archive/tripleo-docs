Updating Undercloud Components
------------------------------

You can upgrade any packages that are installed on the undercloud machine.

#. Update the Delorean Trunk repository::


       # Remove old and enable new Delorean Trunk repository
       sudo rm /etc/yum.repos.d/delorean.repo
       sudo curl -o /etc/yum.repos.d/delorean.repo http://trunk.rdoproject.org/centos7/current-passed-ci/delorean.repo

#. Use yum to update all installed packages::

    sudo yum update -y

    # You can specify the package names to update as options in the yum update command.

You do not need to restart any services after you update.
