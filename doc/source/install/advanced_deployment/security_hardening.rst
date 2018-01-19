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
      OS::TripleO::Services::Sshd: ../puppet/services/sshd.yaml

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
and peforming analysis of events that led to a certain outcome. The audit system
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

iptables rules are automatically deployed on overcloud nodes to open only the
ports which are needed to get OpenStack working. Rules can be added during the
deployement when is needed. For example, for Zabbix monitoring system::

    parameter_defaults:
      ControllerExtraConfig:
        tripleo::firewall::firewall_rules:
          '301 allow zabbix':
            dport: 10050
            proto: tcp
            source: 10.0.0.8
            action: accept

Rules can also be used to restrict access. The number used at definition of a
rule will determine where the iptables rule will be inserted. For example,
rabbitmq rule number is 109 by default. If you want to restrain it, you can do::

    parameter_defaults:
      ControllerExtraConfig:
        tripleo::firewall::firewall_rules:
          '098 allow rabbit from internalapi network':
            dport: [4369,5672,25672]
            proto: tcp
            source: 10.0.0.0/24
            action: accept
          '099 drop other rabbit access':
            dport: [4369,5672,25672]
            proto: tcp
            action: drop

In this example, 098 and 099 are arbitrarily chosen numbers that are smaller than
the rabbitmq rule number 109. To know the number of a rule, you can inspect
the iptables rule on the appropriate node (controller, in case of rabbitmq)::

    iptables-save
    [...]
    -A INPUT -p tcp -m multiport --dports 4369,5672,25672 -m comment --comment "109 rabbitmq" -m state --state NEW -j ACCEPT

Alternatively it's possible to get the information in tripleo service in the
definition. In our case in `puppet/services/rabbitmq.yaml`::

    tripleo.rabbitmq.firewall_rules:
      '109 rabbitmq':
        dport:
          - 4369
          - 5672
          - 25672

The following parameters can be set for a rule:

* **port**: The port associated to the rule. Deprecated by puppetlabs-firewall.

* **dport**: The destination port associated to the rule.

* **sport**: The source port associated to the rule.

* **proto**: The protocol associated to the rule. Defaults to 'tcp'

* **action**: The action policy associated to the rule. Defaults to 'accept'

* **jump**: The chain to jump to.

* **state**: Array of states associated to the rule. Default to ['NEW']

* **source**: The source IP address associated to the rule.

* **iniface**: The network interface associated to the rule.

* **chain**: The chain associated to the rule. Default to 'INPUT'

* **destination**: The destination cidr associated to the rule.

* **extras**: Hash of any additional parameters supported by the puppetlabs-firewall module.
