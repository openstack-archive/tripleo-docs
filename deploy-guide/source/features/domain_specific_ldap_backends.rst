Domain-specific LDAP Backends
=============================

It is possible to configure keystone to use one or more LDAP backends for the
identity resources as described in the `OpenStack Identity documentation`_.
This will result in an LDAP backend per keystone domain.

Setup
-----

To configure LDAP backends, set the ``KeystoneLDAPDomainEnable`` flag to
``true``. Enabling this will set the ``domain_specific_drivers_enabled`` option
in keystone in the ``identity`` configuration group. By default the domain
configurations are stored in the **/etc/keystone/domains** directory on the
controller nodes. You can override this directory by setting the
``keystone::domain_config_directory`` hiera key, and setting that via the
``ExtraConfig`` parameter in an environment file. For instance, to set this in
the controller nodes, one would do the following::

    parameter_defaults:
      ControllerExtraConfig:
        keystone::domain_config_directory: /etc/another/directory

The LDAP backend configuration should be provided via the
``KeystoneLDAPBackendConfigs`` parameter in tripleo-heat-templates. It's a
dictionary mapping the LDAP domain names to options that take the following
keys:

* **identity_driver**: Identity backend driver. Defaults to 'ldap'

* **url**: URL for connecting to the LDAP server.

* **user**: User BindDN to query the LDAP server.

* **password**: Password for the BindDN to query the LDAP server.

* **suffix**: LDAP server suffix

* **query_scope**: The LDAP scope for queries, this can be either "one"
  (onelevel/singleLevel which is the default in keystone) or "sub"
  (subtree/wholeSubtree).

* **page_size**: Maximum results per page; a value of zero ("0") disables
  paging. (integer value)

* **user_tree_dn**: Search base for users.

* **user_filter**: LDAP search filter for users.

* **user_objectclass**: LDAP objectclass for users.

* **user_id_attribute**: LDAP attribute mapped to user id. **WARNING**: must
  not be a multivalued attribute. (string value)

* **user_name_attribute**: LDAP attribute mapped to user name.

* **user_mail_attribute**: LDAP attribute mapped to user email.

* **user_enabled_attribute**: LDAP attribute mapped to user enabled flag.

* **user_enabled_mask**: Bitmask integer to indicate the bit that the enabled
  value is stored in if the LDAP server represents "enabled" as a bit on an
  integer rather than a boolean. A value of "0" indicates the mask is not used.
  If this is not set to "0" the typical value is "2". This is typically used
  when "user_enabled_attribute = userAccountControl". (integer value)

* **user_enabled_default**: Default value to enable users. This should match an
  appropriate int value if the LDAP server uses non-boolean (bitmask) values
  to indicate if a user is enabled or disabled. If this is not set to "True"
  the typical value is "512". This is typically used when
  "user_enabled_attribute = userAccountControl".

* **user_enabled_invert**: Invert the meaning of the boolean enabled values.
  Some LDAP servers use a boolean lock attribute where "true" means an account
  is disabled. Setting "user_enabled_invert = true" will allow these lock
  attributes to be used.  This setting will have no effect if
  "user_enabled_mask" or "user_enabled_emulation" settings are in use.
  (boolean value)

* **user_attribute_ignore**: List of attributes stripped off the user on
  update. (list value)

* **user_default_project_id_attribute**: LDAP attribute mapped to
  default_project_id for users.

* **user_pass_attribute**: LDAP attribute mapped to password.

* **user_enabled_emulation**: If true, Keystone uses an alternative method to
  determine if a user is enabled or not by checking if they are a member of
  the "user_enabled_emulation_dn" group. (boolean value)

* **user_enabled_emulation_dn**: DN of the group entry to hold enabled users
  when using enabled emulation.

* **user_additional_attribute_mapping**: List of additional LDAP attributes
  used for mapping additional attribute mappings for users. Attribute mapping
  format is <ldap_attr>:<user_attr>, where ldap_attr is the attribute in the
  LDAP entry and user_attr is the Identity API attribute. (list value)

* **group_tree_dn**: Search base for groups.

* **group_filter**: LDAP search filter for groups.

* **group_objectclass**: LDAP objectclass for groups.

* **group_id_attribute**: LDAP attribute mapped to group id.

* **group_name_attribute**: LDAP attribute mapped to group name.

* **group_member_attribute**: LDAP attribute mapped to show group membership.

* **group_desc_attribute**: LDAP attribute mapped to group description.

* **group_attribute_ignore**: List of attributes stripped off the group on
  update. (list value)

* **group_additional_attribute_mapping**: Additional attribute mappings for
  groups. Attribute mapping format is <ldap_attr>:<user_attr>, where ldap_attr
  is the attribute in the LDAP entry and user_attr is the Identity API
  attribute. (list value)

* **chase_referrals**: Whether or not to chase returned referrals. Note that
  it's possible that your client or even your backend do this for you already.
  All this does is try to override the client configuration. If your client
  doesn't support this, you might want to enable *chaining* on your LDAP server
  side. (boolean value)

* **use_tls**: Enable TLS for communicating with LDAP servers. Note that you
  might also enable this by using a TLS-enabled scheme in the URL (e.g.
  "ldaps"). However, if you configure this via the URL, this option is not
  needed. (boolean value)

* **tls_cacertfile**: CA certificate file path for communicating with LDAP
  servers.

* **tls_cacertdir**: CA certificate directory path for communicating with LDAP
  servers.

* **tls_req_cert**: Valid options for tls_req_cert are demand, never, and allow.

* **use_pool**: Enable LDAP connection pooling. (boolean value and defaults to
  true)

* **pool_size**: Connection pool size. (integer value and defaults to '10')

* **pool_retry_max**: Maximum count of reconnect trials. (integer value and
  defaults to '3'

* **pool_retry_delay**: Time span in seconds to wait between two reconnect
  trials. (floating point value and defaults to '0.1')

* **pool_connection_timeout**: Connector timeout in seconds. Value -1
  indicates indefinite wait for response. (integer value and defaults to '-1')

* **pool_connection_lifetime**: Connection lifetime in seconds. (integer value
  and defaults to '600')

* **use_auth_pool**: Enable LDAP connection pooling for end user authentication.
  If use_pool is disabled, then this setting is meaningless and is not used at
  all. (boolean value and defaults to true)

* **auth_pool_size**: End user auth connection pool size. (integer value and
  defaults to '100')

* **auth_pool_connection_lifetime**: End user auth connection lifetime in
  seconds. (integer value and defaults to '60')

An example of an environment file with LDAP configuration for the keystone
domain called ``tripleodomain`` would look as follows::

    parameter_defaults:
      KeystoneLDAPDomainEnable: true
      KeystoneLDAPBackendConfigs:
        tripleodomain:
          url: ldap://192.0.2.250
          user: cn=openstack,ou=Users,dc=tripleo,dc=example,dc=com
          password: Secrete
          suffix: dc=tripleo,dc=example,dc=com
          user_tree_dn: ou=Users,dc=tripleo,dc=example,dc=com
          user_filter: "(memberOf=cn=OSuser,ou=Groups,dc=tripleo,dc=example,dc=com)"
          user_objectclass: person
          user_id_attribute: cn

This will create a file in the default domain directory
**/etc/keystone/domains** with the name **keystone.tripleodomain.conf**. And
will use the attributes to create such a configuration.

Please note that both the ``KeystoneLDAPDomainEnable`` flag and the
configuration ``KeystoneLDAPBackendConfigs`` must be set.

One can also specify several domains. For instance::

    KeystoneLDAPBackendConfigs:
      tripleodomain1:
        url: ldap://tripleodomain1.example.com
        user: cn=openstack,ou=Users,dc=tripleo,dc=example,dc=com
        password: Secrete1
        ...
      tripleodomain2:
        url: ldaps://tripleodomain2.example.com
        user: cn=openstack,ou=Users,dc=tripleo,dc=example,dc=com
        password: Secrete2
        ...

This will add two domains, called ``tripleodomain1`` and ``tripleodomain2``,
with their own configurations.

Post-deployment setup
---------------------

After the overcloud deployment is done, you'll need to give the admin user a
role in the newly created domain.

1. Source the overcloudrc.v3 file::

    source overcloudrc.v3

2. Grant admin user on your domain::

    openstack role add --domain $(openstack domain show tripleodomain -f value -c id)\
        --user $(openstack user show admin --domain default -f value -c id) \
        $(openstack role show admin -c id -f value)

3. Test LDAP domain in listing users::

    openstack user list --domain tripleodomain

FreeIPA as an LDAP backend
--------------------------

Before configuring the domain, there needs to be a user that will query
FreeIPA. In this case, we'll create an account called ``keystone`` in FreeIPA,
and we'll use it's credentials on our configuration. On the FreeIPA side and
with proper credentials loaded, we'll do the following::

    ipa user-add keystone --cn="keystone user" --first="keystone" \
        --last="user" --password

This will create the user and we'll be prompted to write the password for it.

Configuring FreeIPA as an LDAP backend for a domain can be done by using the
following template as a configuration::

    parameter_defaults:
      KeystoneLDAPDomainEnable: true
      KeystoneLDAPBackendConfigs:
        freeipadomain:
          url: ldaps://$FREEIPA_SERVER
          user: uid=keystone,cn=users,cn=accounts,$SUFFIX
          password: $SOME_PASSWORD
          suffix: $SUFFIX
          user_tree_dn: cn=users,cn=accounts,$SUFFIX
          user_objectclass: inetOrgPerson
          user_id_attribute: uid
          user_name_attribute: uid
          user_mail_attribute: mail
          group_tree_dn: cn=groups,cn=accounts,$SUFFIX
          group_objectclass: groupOfNames
          group_id_attribute: cn
          group_name_attribute: cn
          group_member_attribute: member
          group_desc_attribute: description
          user_enabled_attribute: nsAccountLock
          user_enabled_default: False
          user_enabled_invert: true

* $FREEIPA_SERVER will contain the FQDN that points to your FreeIPA server.
  Remember that it needs to be available from some network (most likely the
  ctlplane network) in TripleO

* You should also make sure that the ldap ports need to be accessible. In this
  case, we need port 636 available since we're using the ``ldaps`` scheme.
  However, if you would be using the ``use_tls`` configuration option or if you
  are not using TLS at all (not recommended), you might also need port 389.

* To use TLS, the FreeIPA server's certificate must also be trusted by the
  openldap client libraries. If you're using novajoin (and
  :doc:`tls-everywhere`) this is easily achieved since all the nodes in your
  overcloud are enrolled in FreeIPA. If you're not using this setup, you should
  then follow the 'Getting the overcloud to trust CAs' section in the
  :doc:`ssl` document.

* $SUFFIX will be the domain for your users. Given a domain, the suffix DN can
  be created with the following snippet::

      suffix=`echo $DOMAIN | sed -e 's/^/dc=/' -e 's/\./,dc=/g'`

  Given the domain ``example.com`` the suffix will be ``dc=example,dc=com``.

* In this configuration, we configure this backend as read-only. So you'll need
  to create your OpenStack users on the FreeIPA side.

.. References

.. _`OpenStack Identity documentation`: https://docs.openstack.org/admin-guide/identity-integrate-with-ldap.html
