Updating Undercloud Components
------------------------------

You can upgrade any packages that are installed on the undercloud machine.

#. Remove all Delorean repositories::

       sudo rm /etc/yum.repos.d/delorean*

#. Enable new Delorean repositories:

.. include:: ../repositories.txt

.. We need to manually continue our list numbering here since the above
  "include" directive breaks the numbering.

3. Use yum to update all installed packages::

    sudo yum update -y

    # You can specify the package names to update as options in the yum update command.

You do not need to restart any services after you update.
