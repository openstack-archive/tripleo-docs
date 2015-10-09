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

   .. note::

      You do not need to restart any services after you update.

#. Update stack environment of the existing overcloud:

   If you use heat templates from default location
   (`/usr/share/openstack-tripleo-heat-templates`), it's possible that these
   templates have changed when updating undercloud machine. You should update
   the existing overcloud to reflect any changes in template and environment
   files. The reason is that CLI commands which change existing overcloud pass
   new template to Heat but reuse existing environment of the overcloud so a
   new resource type referenced in updated template might be missing in the
   existing environment::

      openstack overcloud deploy --templates -e /usr/share/openstack-tripleo-heat-templates/overcloud-resource-registry-puppet.yaml -e <all extra files>

   .. note::

      Make sure you pass all environment files you used when deploying
      overcloud.
