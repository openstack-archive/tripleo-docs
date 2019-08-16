Keystone Security Compliance
============================

Keystone has several configuration options available in order to comply with
standards such as Payment Card Industry - Data Security Standard (PCI-DSS)
v3.1.

TripleO exposes these features via Heat parameters. They will be listed below:

* ``KeystoneChangePasswordUponFirstUse``: Enabling this option requires users
  to change their password when the user is created, or upon administrative
  reset.

* ``KeystoneDisableUserAccountDaysInactive``: The maximum number of days a user
  can go without authenticating before being considered "inactive" and
  automatically disabled (locked).

* ``KeystoneLockoutDuration``: The number of seconds a user account will be
  locked when the maximum number of failed authentication attempts (as
  specified by ``KeystoneLockoutFailureAttempts``) is exceeded.

* ``KeystoneLockoutFailureAttempts``: The maximum number of times that a user
  can fail to authenticate before the user account is locked for the number of
  seconds specified by ``KeystoneLockoutDuration``.

* ``KeystoneMinimumPasswordAge``: The number of days that a password must be
  used before the user can change it. This prevents users from changing their
  passwords immediately in order to wipe out their password history and reuse
  an old password.

* ``KeystonePasswordExpiresDays``: The number of days for which a password will
  be considered valid before requiring it to be changed.

* ``KeystonePasswordRegex``: The regular expression used to validate password
  strength requirements.

* ``KeystonePasswordRegexDescription``: Describe your password regular
  expression here in language for humans.

* ``KeystoneUniqueLastPasswordCount``: This controls the number of previous
  user password iterations to keep in history, in order to enforce that newly
  created passwords are unique.

.. note:: All of the aforementioned options only apply to the SQL backend. For
          other identity backends like LDAP, these configuration settings
          should be applied on that backend's side.

.. note:: All of these parameters are defined as type ``string`` in heat. As
          per the implementation, if left unset, they will not be configured at
          all in the keystone configuration.
