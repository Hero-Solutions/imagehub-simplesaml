### Webserver configuration

The necessary Apache or Nginx configuration should be written in order to access the SimpleSAMLphp package which provides authentication with the Active Directory Federation Services (ADFS).
A sample Nginx configuration file may look like this:
```
server {

    listen 443 ssl;
    listen [::]:443 ssl;

    root /opt/imagehub/public;

    index index.php;

    server_name resourcespace.mydomain.com;

    location ^~ /simplesaml {
        alias /opt/imagehub/vendor/simplesamlphp/simplesamlphp/www;

        # The prefix must match the baseurlpath configuration option
        location ~ ^(?<prefix>/simplesaml)(?<phpfile>.+?\.php)(?<pathinfo>/.*)?$ {
            include fastcgi_params;
            fastcgi_pass unix:/run/php/php7.4-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$phpfile;

            # Must be prepended with the baseurlpath
            fastcgi_param SCRIPT_NAME /simplesaml$phpfile;

            fastcgi_param PATH_INFO $pathinfo if_not_empty;
        }
    }
}
```
This will provide an admin console to help set up and test your SimpleSAMLphp installation.

### SimpleSAMLphp setup
Now we can set up SimpleSAMLphp to provide authentication through the Active Directory Federation Services. We will need to set up the following files for this, in the vendor/simplesamlphp/simplesamlphp/ folder:
* cert/saml.crt
* cert/saml.key
* config/acl.php
* config/authmemcookie.php
* config/authsources.php
* config/config.php
* metadata/saml20-idp-remote.php
* metadata/saml20-sp-remote.php

Assuming you already have a valid SSL certificate for your domain, copy the .crt and .key files to cert/saml.crt and cert/saml.key.

Next, copy the configuration templates to the config/ folder:
```
cp config-templates/* config/
```

Generate a secret salt:
```
LC_CTYPE=C tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo
```

In config/config.php, find 'secretsalt' and edit it with the output from the above command.
Also edit 'auth.adminpassword' to a safe password of your choosing.

Now, go into ADFS:
Go to Windows Administrative Tools, ADFS Management
In ADFS, Service, go to Endpoints. Under Metadata, find the URL, such as:
/FederationMetadata/2007-06/FederationMetadata.xml

Browse to https://resourcespace.mydomain.com/simplesaml/
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

Browse to https://resourcespace.mydomain.com/simplesaml/ again.
* Tab Federation (Federatie)
* Under 'SAML 2.0 SP Metadata'
* Copy the value next to 'Entity ID:'

Go back to ADFS. Relying Party Trusts. At the right, 'Add relying party trust'.
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

Browse to https://resourcespace.mydomain.com/simplesaml/ again.
* Tab Authentication
* Test configured authentication sources
* default-sp

Now you should see your claim attributes.


**Note**: it is highly recommended to create a seperate directory containing an exact copy of the above files, as updating the packages (through composer update), may cause any files in the vendor/simplesamlphp/simplesamlphp to be lost.
