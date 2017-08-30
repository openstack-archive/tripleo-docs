.. _fernet_key_rotation:

Rotation Keystone Fernet Keys from the Overcloud
================================================

Like most passwords in your overcloud deployment, keystone fernet keys are also
stored as part of the deployment plan in mistral. The overcloud deplotment's
fernet keys can be rotated with the following command::

    mistral execution-create tripleo.fernet_keys.v1.rotate_fernet_keys \
        '{"container": "overcloud"}'

Where the value for "container" is the name of the plan (which defaults to
"overcloud").

After waiting some time you can verify the output by taking the execution ID
from that was the output of the previous command, and issuing the following
command::

    mistral execution-get-output EXECUTION_UUID

Please note that there must be an overcloud deployment ready and accessible in
order to execute this action.
