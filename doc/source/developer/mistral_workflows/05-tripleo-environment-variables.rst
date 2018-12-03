Give support for new Mistral environment variables when installing the undercloud.
----------------------------------------------------------------------------------

Sometimes developers will need to use additional values inside Mistral tasks. For
example, if we need to create a dump of a database we might need another one
than the Mistral user credentials for authentication purposes.

Initially when the Undercloud is installed it’s created a Mistral
environment called **tripleo.undercloud-config**. This environment
variable will have all required configuration details that we can get
from Mistral. This is defined in the **instack-undercloud** repository.

Let’s get into the repository and check the content of the file
`instack_undercloud/undercloud.py`_.

This file defines a set of methods to interact with the Undercloud,
specifically the method called **_create_mistral_config_environment**
allows to configure additional environment variables when installing the
Undercloud.

For additional testing, you can use the `Python snippet`_ to call
Mistral client from the Undercloud node available in gist.github.com.

.. _Python snippet: https://gist.github.com/ccamacho/354f798102710d165c1f6167eb533caf#file-mistral_client_snippet-py
.. _instack_undercloud/undercloud.py: https://git.openstack.org/cgit/openstack/instack-undercloud/tree/instack_undercloud/undercloud.py
