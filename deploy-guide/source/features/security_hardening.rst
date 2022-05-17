Security Hardening
==================

TripleO can deploy Overcloud nodes with various Security Hardening values
passed in as environment files to the ``openstack overcloud deploy`` command.

.. note::
   It is especially important to remember that you **must** include all
   environment files needed to deploy the overcloud. Make sure
   you pass the full environment in addition to your customization environments
   at the end of each of the ``openstack overcloud deploy`` command.

Horizon Password Validation
---------------------------

Horizon provides a password validation check which OpenStack cloud operators
can use to enforce password complexity.

Regular expression can be used for password validation with help text to display
if the users password does not adhere with validation checks.

The following example will enforce users to create a password between 8 and 18
characters in length::

    parameter_defaults:
      HorizonPasswordValidator: '^.{8,18}$'
      HorizonPasswordValidatorHelp: 'Password must be between 8 and 18 characters.'

If the above yaml was saved as ``horizon_password.yaml`` we can then pass this
into the overcloud deploy command as follows::

    openstack overcloud deploy --templates \
      -e <full environment> -e  horizon_password.yaml

Default Security Values in Horizon
----------------------------------

The following config directives are set to ``True`` as a secure default, however
if a reason exists for an operator to disable one of the following values, they
can do so using an environment file.

.. note:: The following directives should only be set to ``False`` once the
          potential security impacts are fully understood.

Enforce Password Check
~~~~~~~~~~~~~~~~~~~~~~

By setting ``ENFORCE_PASSWORD_CHECK`` to ``True`` within Horizon's
``local_settings.py``, it displays an ‘Admin Password’ field on the
“Change Password” form to verify that it is the admin loggedin that wants to
perform the password change.

If a need is present to disable ``ENFORCE_PASSWORD_CHECK`` then this can be
achieved using an environment file contain the following parameter::

    parameter_defaults:
      ControllerExtraConfig:
        horizon::enforce_password_check: false

Disallow Iframe Embed
~~~~~~~~~~~~~~~~~~~~~

DISALLOW_IFRAME_EMBED can be used to prevent Horizon from being embedded within
an iframe. Legacy browsers are still vulnerable to a Cross-Frame Scripting (XFS)
vulnerability, so this option allows extra security hardening where iframes are
not used in deployment.

If however a reason exists to allow Iframe embedding, then the following
parameter can be set within an environment file::

    parameter_defaults:
      ControllerExtraConfig:
        horizon::disallow_iframe_embed: false

Disable Password Reveal
~~~~~~~~~~~~~~~~~~~~~~~

In the same way as ``ENFORCE_PASSWORD_CHECK`` and ``DISALLOW_IFRAME_EMBED`` the
``DISABLE_PASSWORD_REVEAL`` value to be toggled as a parameter::

    parameter_defaults:
      ControllerExtraConfig:
        horizon::disable_password_reveal: false

SSH Banner Text
---------------

SSH ``/etc/issue`` Banner text can be set using the following parameters in an
environment file::

    resource_registry:
      OS::TripleO::Services::Sshd: ../deployment/sshd/sshd-baremetal-ansible.yaml

    parameter_defaults:
      BannerText: |
        ******************************************************************
        * This system is for the use of authorized users only. Usage of  *
        * this system may be monitored and recorded by system personnel. *
        * Anyone using this system expressly consents to such monitoring *
        * and is advised that if such monitoring reveals possible        *
        * evidence of criminal activity, system personnel may provide    *
        * the evidence from such monitoring to law enforcement officials.*
        ******************************************************************

As with the previous Horizon Password Validation example, saving the above into
a yaml file, will allow passing the aforementioned parameters into the overcloud
deploy command::

    openstack overcloud deploy --templates \
      -e <full environment> -e  ssh_banner.yaml

Audit
-----

Having a system capable of recording all audit events is key for troubleshooting
and performing analysis of events that led to a certain outcome. The audit system
is capable of logging many events such as someone changing the system time,
changes to Mandatory / Discretionary Access Control, creating / destroying users
or groups.

Rules can be declared using an environment file and injected into
``/etc/audit/audit.rules``::

    parameter_defaults:
      AuditdRules:
        'Record Events that Modify User/Group Information':
          content: '-w /etc/group -p wa -k audit_rules_usergroup_modification'
          order  : 1
        'Collects System Administrator Actions':
          content: '-w /etc/sudoers -p wa -k actions'
          order  : 2
        'Record Events that Modify the Systems Mandatory Access Controls':
          content: '-w /etc/selinux/ -p wa -k MAC-policy'
          order  : 3

Firewall Management
-------------------

Iptables rules are automatically deployed on overcloud nodes to open only the
ports which are needed to get OpenStack working. Rules can be added during the
deployment when needed. For example, for Zabbix monitoring system.

.. code-block:: yaml

    parameter_defaults:
      ExtraFirewallRules:
        '301 allow zabbix':
          dport: 10050
          proto: tcp
          source: 10.0.0.8
          action: accept

Rules can also be used to restrict access. The number used at definition of a
rule will determine where the iptables rule will be inserted. For example,
rabbitmq rule number is 109 by default. If you want to restrain it, you could
do.

.. code-block:: yaml

    parameter_defaults:
      ExtraFirewallRules:
        '098 allow rabbit from internalapi network':
          dport:
          - 4369
          - 5672
          - 25672
          proto: tcp
          source: 10.0.0.0/24
          action: accept
        '099 drop other rabbit access':
          dport:
          - 4369
          - 5672
          - 25672
          proto: tcp
          action: drop

In this example, 098 and 099 are arbitrarily numbers that are smaller than the
default rabbitmq rule number. To know the number of a rule, inspect the active
iptables rules on an appropriate node (controller, in case of rabbitmq)

.. code-block:: shell

    iptables-save
    [...]
    -A INPUT -p tcp -m multiport --dports 4369,5672,25672 -m comment --comment "109 rabbitmq" -m state --state NEW -j ACCEPT

Alternatively it's possible to get the information in tripleo service in the
definition. In our case in `deployment/rabbitmq/rabbitmq-container-puppet.yaml`.

.. code-block:: yaml

    firewall_rules:
      '109 rabbitmq':
        dport:
          - 4369
          - 5672
          - 25672
          - 25673-25683

Additional information regarding the the available interface options, the role,
some of the implementation details can be reviewed `here <https://docs.openstack.org/tripleo-ansible/latest/roles/role-tripleo_firewall.html>`_.

AIDE - Intrusion Detection
--------------------------

AIDE (Advanced Intrusion Detection Environment) is a file and directory
integrity checker. It is used as medium to reveal possible unauthorized file
tampering / changes.

AIDE creates an integrity database of file hashes, which can then be used as a
comparison point to verify the integrity of the files and directories.

The TripleO AIDE service allows an operator to populate entries into an AIDE
configuration, which is then used by the AIDE service to create an integrity
database. This can be achieved using an environment file with the following
example structure

.. code-block:: yaml

  resource_registry:
    OS::TripleO::Services::Aide: /usr/share/openstack-tripleo-heat-templates/deployment/aide/aide-baremetal-ansible.yaml

  parameter_defaults:
    AideRules:
      'TripleORules':
        content: 'TripleORules = p+sha256'
        order  : 1
      'etc':
        content: '/etc/ TripleORules'
        order  : 2
      'boot':
        content: '/boot/ TripleORules'
        order  : 3
      'sbin':
        content: '/sbin/ TripleORules'
        order  : 4
      'var':
        content: '/var/ TripleORules'
        order  : 5
      'not var/log':
        content: '!/var/log.*'
        order  : 6
      'not var/spool':
        content: '!/var/spool.*'
        order  : 7
      'not /var/adm/utmp':
        content: '!/var/adm/utmp$'
        order: 8
      'not nova instances':
        content: '!/var/lib/nova/instances.*'
        order: 9

.. note::
   Operators should select their own required AIDE values, as the example list
   above is not actively maintained or benchmarked. It only seeks to provide
   an document the YAML structure required.

If above environment file were saved as `aide.yaml` it could then be passed to
the `overcloud deploy` command as follows::

  openstack overcloud deploy --templates -e aide.yaml

Let's walk through the different values used here.

First an 'alias' name `TripleORules` is declared to save us repeatedly typing
out the same attributes each time. To the alias we apply attributes of
`p+sha256`. In AIDE terms this reads as monitor all file permissions `p` with an
integrity checksum of `sha256`. For a complete list of attributes that can be
used in AIDE's config files, refer to the `AIDE MAN page <http://aide.sourceforge.net/stable/manual.html#config>`_.

Complex rules can be created using this format, such as the following::

    MyAlias = p+i+n+u+g+s+b+m+c+sha512

The above would translate as monitor permissions, inodes, number of links, user,
group, size, block count, mtime, ctime, using sha256 for checksum generation.

Note, the alias should always have an order position of `1`, which means that
it is positioned at the top of the AIDE rules and is applied recursively to all
values below.

Following after the alias are the directories to monitor. Note that regular
expressions can be used. For example we set monitoring for the `var` directory,
but overwrite with a not clause using `!` with `'!/var/log.*'` and
`'!/var/spool.*'`.

Further AIDE values
~~~~~~~~~~~~~~~~~~~

The following AIDE values can also be set.

`AideConfPath`: The full POSIX path to the aide configuration file, this
defaults to `/etc/aide.conf`. If no requirement is in place to change the file
location, it is recommended to stick with the default path.

`AideDBPath`: The full POSIX path to the AIDE integrity database. This value is
configurable to allow operators to declare their own full path, as often AIDE
database files are stored off node perhaps on a read only file mount.

`AideDBTempPath`: The full POSIX path to the AIDE integrity temporary database.
This temporary files is created when AIDE initializes a new database.

'AideHour': This value is to set the hour attribute as part of AIDE cron
configuration.

'AideMinute': This value is to set the minute attribute as part of AIDE cron
configuration.

'AideCronUser': This value is to set the linux user as part of AIDE cron
configuration.

'AideEmail': This value sets the email address that receives AIDE reports each
time a cron run is made.

'AideMuaPath': This value sets the path to the Mail User Agent that is used to
send AIDE reports to the email address set within `AideEmail`.

Cron configuration
~~~~~~~~~~~~~~~~~~

The AIDE TripleO service allows configuration of a cron job. By default it will
send reports to `/var/log/audit/`, unless `AideEmail` is set, in which case it
will instead email the reports to the declared email address.

AIDE and Upgrades
~~~~~~~~~~~~~~~~~

When an upgrade is performed, the AIDE service will automatically regenerate
a new integrity database to ensure all upgraded files are correctly recomputed
to possess a updated checksum.

If `openstack overcloud deploy` is called as a subsequent run to an initial
deployment *and* the AIDE configuration rules are changed, the TripleO AIDE
service will rebuild the database to ensure the new config attributes are
encapsulated in the integrity database.

SecureTTY
---------

SecureTTY allows disabling root access via any console device (tty) by means of
entries to the `/etc/securetty` file.

An environment file can be used to set `/etc/securetty` entries as follows::

  resource_registry:
    OS::TripleO::Services::Securetty: ../deployment/securetty/securetty-baremetal-puppet.yaml

  parameter_defaults:
    TtyValues:
      - console
      - tty1
      - tty2
      - tty3
      - tty4
      - tty5
      - tty6

Keystone CADF auditing
----------------------

Keystone CADF auditing can be enabled by setting `KeystoneNotificationFormat`::

  parameter_defaults:
    KeystoneNotificationFormat: cadf

login.defs values
-----------------

Entries can be made to `/etc/login.defs` to enforce password characteristics
for new users added to the system, for example::

  resource_registry:
    OS::TripleO::Services::LoginDefs: ../deployment/login-defs/login-defs-baremetal-puppet.yaml

  parameter_defaults:
    PasswordMaxDays: 60
    PasswordMinDays: 1
    PasswordMinLen: 5
    PasswordWarnAge: 7
    FailDelay: 4
