Authenticating Users by ORCID
=============================

Goal is to control access to a database using ORCID as the user identity.

Scenarios:

1. Allow only specific ORCIDs read and write access.

2. Allow any authenticated ORCID read access.

3. Allow any authenticated ORCID read and write access.

The implementation uses Apache as a reverse proxy in front of
the CouchDB instance. Authentication is handled by Apache delegating to
ORCID using OpenIDC. The client uses the returned JWT to access
CouchDB.


Configuration
-------------

This configuration scenario uses Apache 2.4 as the authenticating reverse proxy.
Note that :doc:`CouchDB suggests <cdb:best-practices/reverse-proxies>`
HAProxy, nginx, or Caddy2 in preference to Apache. Apache is used here because
an existing ORCID auth is conveniently available.

Setting up Apache for ORCID auth on Ubuntu. Given::

  {CLIENT_ID} = Client application ID
  {CLIENT_SECRET} = Client application secret

Install dependencies::

  sudo apt install libapache2-mod-auth-openidc
  sudo a2enmod auth_openidc

The path ``/orcid_auth/`` is used to authenticate the user and return a JWT. Apache
configuration::

  OIDCClientID "{CLIENT_ID}"
  OIDCClientSecret "{CLIENT_SECRET}"
  OIDCRedirectURI https://Server.Name/orcid_auth/_oar
  OIDCCryptoPassphrase <some-secret>
  OIDCInfoHook iat access_token access_token_expires id_token refresh_token session
  OIDCProviderMetadataURL "https://orcid.org/.well-known/openid-configuration"
  OIDCScope "openid"
  OIDCPassIDTokenAs claims payload serialized
  OIDCPassUserInfoAs claims json jwt
  OIDCPassClaimsAs both
  OIDCRefreshAccessTokenBeforeExpiry 15
  <Location /orcid_auth/>
    Options +Includes
    AuthType openid-connect
    Require valid-user
  </Location>

A web browser is required for authentication. Visit ``/orcid_auth/`` with a browser
to initiate authentication with ORCID. On success, a page will be presented with the
JWT. The JWT is used for subsequent access to the CouchDB instance. The ORCID JWT has
a lifetime of 24hrs.


Restrict the path ``/any_orcid/`` to allow access with any ORCID, with connection
to the ``orcid_users`` database::

  # Ensure these are removed from any incoming requests to avoid spoofing
  RequestHeader unset X-Auth-CouchDB-Roles
  RequestHeader unset X-Auth-CouchDB-UserName

  # Handle the JWT authentication
  OIDCOAuthVerifyJwksUri "https://orcid.org/oauth/jwks"
  OIDCOAuthClientID "{CLIENT_ID}"
  OIDCOAuthClientSecret "{CLIENT_SECRET}"
  OIDCOAuthRemoteUserClaim sub

  # Set oauth2 authentication for the location
  # The ORCID value is passed as the authenticated CouchDB username
  <Location /any_orcid/>
    Options +Includes
    AuthType oauth20
    Require valid-user
    RequestHeader set X-Auth-CouchDB-Roles "orcid_member"
    RequestHeader set X-Auth-CouchDB-UserName "%{OIDC_CLAIM_sub}e"
  </Location>

  # Reverse proxy a specific database on CouchDB
  ProxyPreserveHost On
  ProxyPass /any_orcid/ http://localhost:5984/orcid_users/ nocanon
  ProxyPassReverse /any_orcid/ http://localhost:5984/orcid_users/

A client may access the service using the JWT retrieved from ``/orcid_auth/``. For
example, all documents in the database can be retrieved by a request::

  curl -H "Authorization: Bearer ${JWT}" \
       -H "Accept: application/json" \
       "https://Server.Name/any_orcid/_all_docs"



Or restrict the path ``/o/{ORCID}`` for a specific ORCID::

  <LocationMatch "^/o/(?<orcid>[^/]+)">
    AuthType openid-connect
    Require user "%{env:MATCH_ORCID}@orcid.org"
    Options Indexes MultiViews FollowSymLinks
  </LocationMatch>


Any authenticated read and write
--------------------------------

Enable any user that is authenticated to read and write the database.

In the apache configuration::

  <Location /read_write/>
    Require valid-user
    RequestHeader set X-Auth-CouchDB-Roles "readwrite"
  </Location>

In the CouchDB security settings for the database, add the role "readwrite" to members::

  {
    "members": {
      "roles": [
        "_admin",
        "readwrite"
      ]
    },
    "admins": {
      "roles": [
        "_admin"
      ]
    }
  }


Any read, authenticated write
-----------------------------

Any anonymous user may read the database, only authenticated may write.


Specific users read and write
-----------------------------

Only specific users may read and write the database.
