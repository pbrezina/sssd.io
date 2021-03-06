.. highlight:: none

Automatic Private Groups for LDAP and AD domains
================================================

Related ticket(s):
------------------
    https://pagure.io/SSSD/sssd/issue/1872

Problem statement
-----------------
This change will enable SSSD to automatically generate private groups for
users based on the UID number without the group actually being present as
an LDAP object.

Use cases
---------
The primary use-case is ease of management. The LDAP administrator will only
create the user object and add the user to supplementary groups as needed.

This has two advantages:

 * There is one less object to manage and keep in sync with the user

 * In AD environments, it is not possible to create a user and a group
   with the same ``samAccountName,`` therefore even manually creating the private
   groups requires the admin to remap the group attribute to a non-default one
   since by default both users and groups use ``samAccountName.``

Overview of the solution
------------------------
Most of the low-level functionality in the sysdb layer had been developed
for many years for use in the ``local`` provider. At the same time, there
are also most of the infrastructure ready in the LDAP provider and the
NSS responder, because the automatic private groups are used by default
already for trusted domains with ID mapping enabled.

At the moment, the functionality is enabled internally by an option called
``mpg``, short for Magic-Private-Groups. On a high level, the private groups
are not created in the SSSD cache at all, but the work is done by the NSS
responder which generates the group reply based on the user object
only.

Therefore, the majority of the work will be exposing the option in
configuration and making sure all codepaths work equally well for joined
domains as they do for trusted domains.

Implementation details
----------------------
A new option needs to be added that would control the user private group
creation. In the past, we've had an option called ``magic_private_groups``
and the internal boolean flag inside the ``sss_domain_info`` structure is
still called ``mpg``.

Instead of resurrecting the old option, we should introduce a newly named
option that would be understood by admins better, such as
``auto_private_groups``. The new option must be read on SSSD startup and set
the ``sss_domain_info->mpg`` flag, which is currently auto-enabled with
subdomains only.

The code branch that saves the user (currently ``sdap_save_user``) must be
extended to allow setting the GID number to be the same as UID number for
any domain that sets the ``mpg`` flag. Care must be taken to store the
original GID number (if any) to the ``SYSDB_PRIMARY_GROUP_GIDNUM`` attribute
which is then used by the NSS responder to add the original primary GID
as a supplementary group.

Finally, the group-by-GID LDAP request in the LDAP provider must be extended
to make sure that if a private group GID is requested before the user is,
the group request will also turn the group-by-GID request to a user-by-UID
request which would save the user object which would then allow the NSS responder
to auto-generate the group reply.

Configuration changes
---------------------
A new option ``auto_private_groups`` will be introduced. At the moment, it
will only be possible to set the option for the joined domains as the trusted
domains always create the private groups already by default. Therefore
the only viable usage of this new option in a trusted domain would be
`disabling` the functionality, which is out of scope of this RFE.

The ``auto_private_groups`` option will default to ``false``.

How To Test
-----------
The primary use-cases are SSSD being a client of a generic LDAP server
and SSSD on a GNU/Linux machine directly joined to an AD domain with
``id_provider=ad``.

In both cases, setting the ``auto_private_groups`` option to  ``true``
should result in the ``initgroups`` call returning the primary GID number
of the user with the same value and resolving to the same name as the
primary UID number and the username.

Other interfaces should produce symmetrical results, although at least in
the case of the D-Bus based IFP interface, it is currently not the case,
see `ticket #3543 <https://pagure.io/SSSD/sssd/issue/3543>`_.

For example, here is an output of a test user with private groups autogenerated::

 id puser@win.trust.test
 uid=20000(puser@win.trust.test) gid=20000(puser@win.trust.test) groups=20000(puser@win.trust.test),20002(user1_group2@win.trust.test),20001(user1_group1@win.trust.test),10000(pgroup@win.trust.test)

and without::

 id puser@win.trust.test
 uid=20000(puser@win.trust.test) gid=10000(pgroup@win.trust.test) groups=10000(pgroup@win.trust.test),20001(user1_group1@win.trust.test),20002(user1_group2@win.trust.test)

Note that in the case of the private groups being generated, the original
GID number is turned into a supplementary group by the initgroups call.

How To Debug
------------
There's not much extra debugging added for this feature. Debugging this
feature should amount to the usual checking of the debug logs. In addition,
the cache can be inspected with the ``ldbsearch`` tool to make sure all the
groups are saved as expected as well as the ``SYSDB_PRIMARY_GROUP_GIDNUM``
attribute.

Authors
-------
 * Jakub Hrozek <jhrozek@redhat.com>
