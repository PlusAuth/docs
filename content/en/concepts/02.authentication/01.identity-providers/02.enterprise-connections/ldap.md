---
title: "AD/LDAP Connections"
meta_title: "PlusAuth AD/LDAP Connections"
meta_description: "PlusAuth is a product provides authorization and authentication solution in a secure way."
keywords:
- concepts
- plusauth
- federated
- external
- ldap
- ad
- active directory

---

You can integrate your applications with your AD/LDAP server with a few keystrokes.
All you need is your AD/LDAP server's connection properties.

The flow is simple, when an AD/LDAP user authenticates successfully, the same user will be created in PlusAuth to
make sure your LDAP users and your application benefits from all of PlusAuth functionality and features. However,
every time they try to log in, the password verification will be handled by your 
AD/LDAP server.

To create an AD/LDAP connection, go to [Dashboard > Connections > Enterprise Connections > AD/LDAP](https://dashboard.plusauth.com/#connections/enterprise/ldap/new).

AD/LDAP federated connection contains two sections for the configuration which are `Connection Configuration` and `User Object Mapping`.

## Connection Configuration
Like any external connection, PlusAuth must know your AD/LDAP server's connection details and credentials to interact with it.

| Configuration                 | Description                                                                                                                                                                                                                                                                                                                                                          | Example                                                                                |
|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| **Name**                      | Your connection's unique identifier. Cannot update after set. It could contain `uppercase` and `lowercase latin letters`, `digits` and `-`, `_`                                                                                                                                                                                                                      | `my-ldap-connection`                                                                   |
| **URL**                       | LDAP server's connection URI. Must contain scheme, host and port. `<ldap/s>://<host>:<port>`.                                                                                                                                                                                                                                                                        | `ldaps://myldapserver.com:636`, `ldap://myldapserver.com:389` or `ldap://10.1.1.1:389` |
| **Bind DN**                   | The full DN of the user you bind with.                                                                                                                                                                                                                                                                                                                               | `america\momo` or `CN=PlusAuth,OU=Users,DC=domain,DC=com`                              |
| **Bind Credentials**          | The password of the bind user.	                                                                                                                                                                                                                                                                                                                                      | `YOUR_PASSWORD`                                                                        |
| **Search Base DN**            | Base where we can search for users.                                                                                                                                                                                                                                                                                                                                  | `ou=people,dc=plusauth,dc=example` or `DC=mydomain,DC=com`                             |
| **Search Filter**             | Filter for finding an LDAP user. Format: [RFC 4515](https://tools.ietf.org/search/rfc4515). You can use ``{{username}}`` for to be searched user interpolation.                                                                                                                                                                                                      | `(uid={{username}})` or `(\|(uid={{username}})(mail={{username}}))`                    |
| **STARTTLS**                  | Enable STARTTLS. If you are not using secure connection, enabling this option is encouraged.                                                                                                                                                                                                                                                                         |                                                                                        |
| **Synchronize User Profiles** | With this setting enabled, on each successful user login, user object will be requested from your AD/LDAP server and PlusAuth user will be updated accordingly.                                                                                                                                                                                                      |                                                                                        |
| **Write Mode**                | By default AD/LDAP connections are read-only. Being read-only ensures that whatever yo do over PlusAuth, your AD/LDAP user will not be updated/deleted. If you enable this option, the changes will be reflected also to your AD/LDAP server. Which means whenever a user is deleted/updated from PlusAuth, it will be deleted/updated from your AD/LDAP server too. |                                                                                        |


## User Object Mapping
Companies have different structures and user profiles in their AD/LDAP servers. To ensure consistency with your application and different connection types, PlusAuth requires 
you to map PlusAuth user's properties with your AD/LDAP user's attributes.

You don't need to map all the fields but some are required. Required ones are:

| Configuration      | Description                                                                                                                                                                                                                                                                                                                  |
|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **id**             | In some cases, we must ensure that we are dealing with the same user. For that we need a unique identifier. Providing non-unique value would lead unexpected results and errors. Most common values are `objectGUID`,`uidNumber`,`objectSID` or `entryUUID`                                                                  |
| **username**       | Your LDAP user's username field which will be queried for the authentication.  For many LDAP server vendors it can be `uid`. For Active directory it can be `sAMAccountName` or `cn`. Make sure your AD/LDAP user's contain this attribute as missing ones will not be able to log in to your application.                   |
| **object_classes** | Existing `objectClass` attributes for your LDAP users. Divided by comma. Most common ones: `inetOrgPerson`, `organizationalPerson`. If `Write mode` is enabled newly created PlusAuth users will be created on your AD/LDAP with all those object classes. Make sure other mapped fields belongs to provided object classes. |
| **email**          | Required for reset password functionality. Most common attribute is `mail`                                                                                                                                                                                                                                                   |

Other fields are optional but in some cases they may be crucial to set, otherwise some flows will not work as expected or
some AD/LDAP users would lack some functionality. For example, if you are using SMS Multi-Factor Authentication in your
application, as you guess, a phone number is required. If your AD/LDAP users already have a phone number you 
should map its attribute too.
