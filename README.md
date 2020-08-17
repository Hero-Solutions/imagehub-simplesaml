### Webserver configuration

The necessary Apache or Nginx configuration should be written in order to access both the Imagehub itself and the SimpleSAMLphp package which provides authentication with the ADFS.
A sample Apache configuration file may look like this:
```
<VirtualHost _default_:443>

    DocumentRoot "/opt/ImageHub/public"
    ServerName imagehub.muzee.be:443

    SetEnv SIMPLESAMLPHP_CONFIG_DIR /opt/ImageHub/vendor/simplesamlphp/simplesamlphp/config

    Alias /simplesaml /opt/ImageHub/vendor/simplesamlphp/simplesamlphp/www

    SSLProxyEngine On
    ProxyPass /cantaloupe/iiif/2 https://imagehub.muzee.be:8183/iiif/2
    ProxyPassReverse /cantaloupe/iiif/2 https://imagehub.muzee.be:8183/iiif/2

    <Directory /opt/imagehub/public>
      <IfModule !mod_authz_core.c>
        # For Apache 2.2:
        AllowOverride All
        Order allow,deny
        Allow from all
      </IfModule>
      <IfModule mod_authz_core.c>
        # For Apache 2.4:
        AllowOverride All
        Require all granted
      </IfModule>
    </Directory>
    <Directory /opt/imagehub/vendor/simplesamlphp/simplesamlphp/www>
      <IfModule !mod_authz_core.c>
        # For Apache 2.2:
        AllowOverride All
        Order allow,deny
        Allow from all
      </IfModule>
      <IfModule mod_authz_core.c>
        # For Apache 2.4:
        AllowOverride All
        Require all granted
      </IfModule>
    </Directory>
</VirtualHost>
```
This will provide the following:
* An SSL-enabled virtual host on the domain 'imagehub.muzee.local' (keep in mind you'll need a valid SSL certificate for this), pointing to the directory /opt/Imagehub/public
* An admin console to help set up and test your simplesamlphp installation
* A reverse proxy that points to your Cantaloupe image server, in order to prevent issues with Cross-Origin Resource Sharing (CORS) in a IIIF viewer.

### SimpleSAMLphp setup
Now we can set up SimpleSAMLphp to provide authentication through the Active Directory Federation Services. We need to set up the following files for this, in the vendor/simplesamlphp/simplesamlphp/ folder:
* cert/saml.crt
* cert/saml.key
* config/acl.php
* config/authmemcookie.php
* config/authsources.php
* config/config.php
* metadata/saml20-idp-remote.php
* metadata/saml20-sp-remote.php

From the command line:
```
openssl req -x509 -nodes -sha256 -days 3653 -newkey rsa:2048 -keyout saml.key -out saml.crt
```
Then copy the saml.key and saml.crt to the cert/ folder.

Next, copy the configuration templates to the config/ folder:
```
cp config-templates/* config/
```

Generate a secret salt:
```
LC_CTYPE=C tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo
```

In config.php, find secretsalt and edit it with the output from the above command.
Also edit auth.adminpassword to a safe password of your choosing.

To AD FS:
Windows Administrative Tools, AD FS Management
In AD FS, Service, Endpoints. Under Metadata, find the URL, such as:
/FederationMetadata/2007-06/FederationMetadata.xml

Browse to https://resourcespace.muzee.be/simplesaml/
* Tab Federation (Federatie)
* Tools
* XML to SimpleSAMphp metadata converter.

Either use 'Choose file' and enter the URL or browse to https://[your_adfs_login_service]/FederationMetadata/2007-06/FederationMetadata.xml and copy-paste the content.

Click 'Parse'.

Copy-paste the content under saml20-sp-remote to metadata/saml20-sp-remote.php and the content under saml20-idp-remote to metadata/saml20-idp-remote.php.

Make sure both files start with ``<?php``, the parser does not generate these.


In config/authsources.php, replace the block 'default-sp' with the following:
```
  // An authentication source which can authenticate against both SAML 2.0
  // and Shibboleth 1.3 IdPs. If you make any configuration changes, you will need
  // to update the RPT at the IdP.
  'default-sp' => array(
      'saml:SP',

      // The entity ID of this SP.
      // Can be NULL/unset, in which case an entity ID is generated based on the metadata URL.
      'entityID' => null,

      // The entity ID of the IdP this should SP should contact.
      // Can be NULL/unset, in which case the user will be shown a list of available IdPs.
      'idp' => 'http://[your_adfs_login_service]/adfs/services/trust',

      // The URL to the discovery service.
      // Can be NULL/unset, in which case a builtin discovery service will be used.
      'discoURL' => null,

      // ADFS requires signing of the logout - the others are optional (may be overhead you don't want.)
      'sign.logout' => TRUE,
      'redirect.sign' => TRUE,
      'assertion.encryption' => TRUE,

      // We now need a certificate and key. The following command (executed on Linux usually)
      // creates a self-signed cert and key, using SHA256, valid for 2 years.
      // openssl req -x509 -nodes -sha256 -days 730 -newkey rsa:2048 -keyout my.key -out my.pem
      'privatekey' => 'saml.key',
      'certificate' => 'saml.crt',

      // Enforce the use of SHA-256 by default.
      'signature.algorithm' => 'http://www.w3.org/2001/04/xmldsig-more#rsa-sha256'
  ),
```

Browse to https://resourcespace.muzee.be/simplesaml/ again.
* Tab Federation (Federatie)
* Under 'SAML 2.0 SP Metadata'
* Copy the value next to 'Entity ID:'

Go back to AD FS. Relying Party Trusts. At the right, 'Add relying party trust'.
* Claims aware
* Import data about the relying party published online or on a local network (first option), paste the 'Entity ID' value from the previous step into the field.
* Next
* Ignore the warning
* Enter any display name and additional notes
* Next
* Next
* Configure claims issuance policy for this application, make sure it is checked
* Close

Select the new value in the list, click 'Edit Claim Issuance Policy...'

Add rule
* Send LDAP Attribute as Claims
* Next
* Claim rule name: 'UPN'
  * Attribute store: Active Directory
  * LDAP attributes: User-Principal-Name, Outgoing Claim Type: UPN
  * LDAP attributes: Token-Group - Unqualified Names: Outgoing Claim Type: Group

Add rule
  * Transform an incoming Claim
  * Claim rule name: 'Outgoing name identifier'
  * Incoming claim type: UPN
  * Outgoing claim type: Name ID
  * Outgoing name ID format: Transient Identifier
  * Check 'Pass through all claim values'

Browse to https://resourcespace.muzee.be/simplesaml/ again.
* Tab Authentication
* Test configured authentication sources
* default-sp

Now you should see your claim attributes.


**Note**: it is highly recommended to create a seperate directory containing an exact copy of the above files, as updating the packages (through composer update), may cause any files in the vendor/simplesamlphp/simplesamlphp to be lost.
