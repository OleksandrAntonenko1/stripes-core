# Permissions in Stripes and FOLIO

<!-- ../../okapi/doc/md2toc -l 2 permissions.md -->
* [Background](#background)
    * [What permissions are](#what-permissions-are)
    * [Atomic and compound permissions](#atomic-and-compound-permissions)
    * [Permission enforcement on back-end and front-end](#permission-enforcement-on-back-end-and-front-end)
    * [Sources of permissions](#sources-of-permissions)
* [Issues](#issues)
    * [How to name permissions?](#how-to-name-permissions)
    * [Permission display-name and description](#permission-display-name-and-description)
    * [Which permissions defined where?](#which-permissions-defined-where)
    * [Which permissions to check?](#which-permissions-to-check)



## Background

### What permissions are

In the FOLIO system, permissions are specified by short, faceted strings such as `users.collection.get` (the permissions to read a collection of user records), `circulation.loans.item.put` (the permission to replace an existing loan) or `module.items.enabled` (the permission to use the Items UI module).

Permissions also have a human-readable display-name such as "Get a collection of user records" or "circulation - modify loan in storage", but this only for the benefit of administrators, and does not affect how the permissions function.

Permissions can be associated with users. A user is then said to have those permissions.


### Atomic and compound permissions

The definition of a permission may include one or more sub-permissions. A user who has a permission automatically has all of its sub-permissions (and all their sub-permissions, and so on).

A permission with no sub-permissions is called an _atomic permission_, and one that does have sub-permissions is called a _compound permission_. (We sometimes also use the term "permission set" for the latter; but since permission sets _are_ permissions -- merely those that happen to have sub-permissions -- the permission-vs-permission-set dichotomy is misleading.)


### Permission enforcement on back-end and front-end

A user is allowed to perform an operation only if they have the necessary permission.

Most permissions are rigorously enforced by back-end modules -- e.g. mod-users simply will not allow someone without the `users.collection.get` permission to read collections of user records. But the UI code also checks permissions, so that it can avoid offering the user operations that it knows will fail due to lack of permissions. For example, a user without the `circulation.loans.item.put` permission is not offered the opportunity to renew a loan, since the attempt to do so would be rejected by the back-end module.

A few permissions are checked only on the UI side: for example, the link to the Items UI module is displayed only to users who have the `module.items.enabled` permission. While a different UI could bypass such UI-only permissions, doing so would not violate security as the back-end permissions would still be checked. Omitting UI elements for which the relevant back-end features will not permit operations is a service to the user, not a security feature.


### Sources of permissions

Permissions are defined in the module decriptors of FOLIO modules -- both back-end and front-end modules. When a module descriptor is posted to Okapi, one of the effects is that the permissions it defines are inserted into the available-permissions list. They are then available to be associated with individual users.

(Back in the bad old days, the ansible build scripts for FOLIO VMs used to explicitly add certain permissions to the database. That was a temporary measure: this is no longer done, and _all_ permissions are now loaded from modules' descriptors.)

In addition, high-level permission sets can be defined at run-time (using **Settings > Users > Permission sets**); but since these are tenant-specific, no software may rely directly on them, and they are useful only as a particular site's aggregation for administrative purposes. As such, these permission sets are not of interest to us here.



## Issues


### How to name permissions?

Each module that defines permissions should use a unique prefix, related to the module-name, at the start of the names of permissions that it defines. For example, mod-users defines the "ability to read a collection of user records" permissin, so it has a `users` prefix at the start of its name, `users.collection.get`.

Permissions defined in front-end modules are given names whose prefixes begin with `ui-`. For example, the "can edit user profiles" permission, defined in ui-users, is named `ui-users.edit`.

(In the past, modules have sometimes defined permissions with names that belong to other modules. For example, the high-level "can edit user profiles" permissions was originally defined on the server-side "users-bl" (business logic) module and named `users-bl.edit`. When the software was enhanced so that UI modules could define permissions, this permission was moved across to ui-users, but initially retained its old name `users-bl.edit`. This is no longer done: the permission has been renamed `ui-users.edit`.)


### Permission display-name and description

The permission fields `displayName` and `description` are both human-readable, but have different roles. There is presently some inconsistency in how they are presently used. For example:

* In ui-users, the `displayName` is a human-readable name such as "Users: Can create new user"; and the `description` is used as a note such as "Some subperms can be deleted later when bl does updates and ModulePermissions can be used".

* In mod-users, the `displayName` is essentially a restatement of the machine-readable permission name (e.g. "users collection get" for `users.collection.get`) and the `description` is a more useful human-readable text such as "Get a collection of user records".

* In mod-circulation, the `displayName` is a human-readable string such as ""circulation - modify loan in storage" and the `description` is a redundant repetition of the part of the display-name following the module name -- e.g. "modify loan in storage".

All of these approaches are inconsistent, and none of them is really satisfactory. I suggest that there is little real use for the `description` field, at least for most permissions, and its existence has confused matters.

My suggestion: `displayName` should always be a human-readable permission name, and should be consistently capitalised across modules. It should generally begin with a category such as "Users:" or "Circulation settings:". The `description` field should generally be left blank, but may be used for an explanatory note when the meaning of a permission is not self-evident. It should not be used for implementation notes such as the example above from ui-users.


### Which permissions defined where?

Specifically, which permissions should be defined in front-end modules and which in back-end modules? Some principles are obvious:

* Permissions which a back-end module checks should be defined in that back-end module. (They are part of the API it consumes, so it must define them otherwise it might run in contexts where they do not exist.)

* Permission which are enforced only in a front-end module should be defined in the front-end. There is no need for back-end modules to be encumbered by the knowledge of such permissions.

* _In general_, a permission enforced in a front-end module should be defined in that particular front-end module, and a permission enforced by Stripes itself should be defined by stripes-core. However, as a special case, stripes-core enforced the various `module.NAME.enabled` permissions provided by the modules called _NAME_.

But this leaves some scope for judgement.


### Which permissions to check?

Our policy is that code should nearly always check atomic permissions rather than higher-level permissions that include it. This approach allows us to be maximally precise regarding what permissions are needed.

In particular:

* _back-end modules should always check atomic permissions_, since these modules know the real truth about which permissions are logically associated with which operations. Also, modules may only rely on permissions defined either by themselves or by lower-level modules whose APIs they consume -- and back-end modules generally define only atomic permissions, and have dependencies only on other back-end modules.

* _front-end modules should usually check atomic permissions_, also. They can safely do so since the set of permissions published by a back-end module is part of its API. (And when we have automatic generation of API docs, the permissions should definitely be included). So as soon as a front-end module is using (say) the `users` interface v14.0, it assuming the existence not only of the `/users/ID` path, but also the `users.item.get` permission.