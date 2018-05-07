Deployment Log
--------------
The ansible part of the deployment creates a log file that is saved on the
undercloud. The log file is available  ``/var/lib/mistral/<execution
id>/ansible.log``. The ``<execution id>`` is a UUID that corresponds to the Mistral
execution that executed ``ansible-playbook``.

Looking for the most recently modified directory under ``/var/lib/mistral`` is
an easy way to quickly find the log for the most recent deployment.
