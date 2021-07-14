# What's This?
This is a user-mapping application for Open OnDemand (OOD) to inter-operate with OAuth 4 MyProxy CAs.

These scripts are designed to be used in conjunction with the Apache module mod_auth_openidc, which does the heavy-lifting for OIDC/OAuth2 authentication.

## What?! Why?
If your environment already has an established GSISSH deployment, user_map_oa4mp can be used to transform an OOD deployment to behave like a gateway, where it runs outside the environment and doesn't have requirements for strong trust in the OOD host's integrity.

The goal is to limit the scope of a security incident involving the OOD host or application, from "all users" to just the ones actively using OOD.


# Prerequisites
You'll need:
* A fully-populated /etc/grid-security directory, including the grid-mapfile and the trust-anchors to verify the certificate returned by the oa4mp CA.
* A fully-populated passwd database, with individual user home directories. (No shadow hashes required, in fact, keep them out!)
* A dedicated unprivileged account for the unprivileged component of this application to run as. This account MUST be a role account that isn't used for other purposes, including interactive logins.
* The oa4mp secrets.
* GSISSH client packages on the OOD host.


In this case, fully-populated means the accounts present are the same as those on the cluster.

# Deployment
1. Copy the `user_map_oa4mp`, `user_map_oa4mp_unprivileged_helper`, and your version of `setup_user_expanse` to `/etc/ood/custom_scripts`. (note: the "expanse" in the name is specific to SDSC's Expanse supercomputer and provided as an example.)

# Configuring Apache / Open OnDemand
1. Edit your `ood_portal.yml` file so the following are set:
```
user_map_cmd: '/etc/ood/custom_scripts/user_map_oa4mp'
user_env: 'oidc_access_token'
```
2. Rebuild your OOD configuration.

# Configuring user_map_oa4mp
1. Edit `user_map_oa4mp` and set the `HELPER_USER` and `HELPER_GROUP` variables to the unprivileged account and group described above.

2. If your `setup_user_expanse` script has a different name, edit user_map_oa4mp and set the `SETUP_USER_SCRIPT` to your script's name.


# Configuring user_map_oa4mp_unprivileged_helper
1. Edit `user_map_oa4mp_unprivileged_helper` and set the `OA4MP_CLIENT_SECRETS_BASE` to a prefix that's safe to store secrets in.

2. Place the oa4mp API url in the file pointed to by `<OA4MP_CLIENT_SECRETS_BASE>.url`. This can be world-readable.

3. Place your client secret in the file pointed to by `<OA4MP_CLIENT_SECRETS_BASE>.secret`. This MUST NOT be world-readable, MUST NOT be group-readable, and MUST be owned by the unprivileged account described above. The client secret is a URL encoded string, like a query, and starts with `client_secret=`.


# Putting it in action
1. Complete the remainder of your mod_auth_openidc configuration.
2. Restart Apache.


# Theory of Operation
* mod_auth_openidc generates an `oidc_access_token` variable, which contains a one-time-use token to retrieve a certificate.
* Since `oidc_access_token` can only be used once, we must cache its mapping to the local account. Check the cache and return the mapping if one is found.
* If no mapping is found, use the `oidc_access_token` to retrieve a certificate.
* Using the DN in the certificate, look up the local account.
* Save the certificate in the well-known-location for GSI certificates and set its ownership accordingly.
* Save the mapping into the cache.
* Return the mapped account.

## Caching
* We leverage the filesystem as a key=value database.
* Hash the oidc_access_token; that's the filename (key).
* The contents of the file is the mapped user (value).
* $something needs to clean /tmp/ of stale cache entries. This exercise is left for the student.





