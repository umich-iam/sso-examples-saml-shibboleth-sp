# Sample Stanford Shibboleth SP Configuration

This repository contains example configuration files for Shibboleth SPs
at Stanford.

## Basic Configuration

There are a few basic steps required to configure an SP:

1. [Set the Default EntityID](#set-the-default-entityid)
1. [Choose IdP](#choose-idp)
1. [Configure Error Handling](#configure-error-handling)
1. [Generate SP Key and Certificate](#generate-sp-key-and-certificate)
1. [Configure Access Controls](#configure-access-controls)

### Set the Default EntityID

Set your default (or only) EntityID in `/etc/shibboleth/shibboleth2.xml`
by replacing __webapp.itlab.stanford.edu__ in the `entityID` attribute
on the `ApplicationDefaults` element with your hostname (or the
loadbalanced name if you have multiple web servers for your application)

```xml
  <ApplicationDefaults
    entityID="https://webapp.itlab.stanford.edu/"
    REMOTE_USER="uid eppn">
```

### Choose  IdP

All the configurations contain a `shibboleth2.xml` file with all the
Stanford IdPs commented out. You'll need to uncomment two definitions:
one __SSO__ element and one __MetadataProvider__ element. Make sure you
uncomment matching definitions. E.g. to use the production IdP, you
would uncomment

```xml
  <!-- production IdP -->
  <SSO entityID="https://idp.stanford.edu/">SAML2</SSO>
```

and

```xml
  <!-- production IdP -->
  <MetadataProvider
    type="XML"
    uri="https://idp.stanford.edu/metadata.xml"
    backingFilePath="/var/cache/shibboleth/idp-metata.xml"
    reloadInterval="7200" />
```

### Configure Error Handling

Update the attributes on the `Error` element with appropriate values:

```xml
  <Errors
    supportContact="YOUR-TEAM@lists.stanford.edu"
    helpLocation="https://helpsu.stanford.edu/..."
    styleSheet="/shibboleth-sp/main.css"/>
```

### Generate SP Key and Certificate

Every SP should have an X.509 (SSL) key and certificate pair. These are
used for signing authentication requests (optional) and decrypting
encrypted assertions from the IdP (highly recommended). The key and
certificate should not be the same key and certificate used by Apache.
A self-signed certificate will be generated during the Shibboleth
installation on most platforms, but it will probably have a very short
lifetime. Generate a longer-lived self-signed certificate and use that
instead. Assuming you are using one of the sample configurations, the
key should be `/etc/shibboleth/sp-key.pem` and the certificate should
be `/etc/shibboleth/sp-cert.pem`. If the `hostname` command returns the
correct name for your host, run the following as `root`:

```shell_session
  # openssl req -x509 -nodes -days 3650 -newkey rsa:2048
  > -keyout /etc/shibboleth/sp-key.pem -out /etc/shibboleth/sp-cert.pem \
  > -subj /CN=`hostname`
  Generating a 2048 bit RSA private key
  .................................+++
  ...............................................................................+++
  writing new private key to 'sp-key.pem'
  -----
```

This will create a 2048 bit RSA private key (with no passphrase), and a
self-signed certificate that is valid for almost 10 years (3,650 days)
with the common name (CN) of the certificate subject set to your
hostname. If `hostname` does not return the correct name for your host,
just replace __`hostname`__ with your hostname.

If you are running a cluster of Apache servers for your application,
copy the key and certificate to each of the servers.  **DO NOT GENERATE A NEW KEY AND CERTIFICATE ON EACH CLUSTER MEMBER**

### Configure Access Controls

Use the examples in `/etc/apache2/sites-available/webapp-ssl.conf` to
define access controls for your site. You can also use `.htaccess`
files in specific directories files if needed, but you'll have to allow
AuthConfig overrides in the Apache configuration.

## Metadata Configuration

Metadata is specific to each EntityID; generic metadata can be
retrieved from the Shibboleth Handler for each EntityID by combining
the virtual host HTTPS URL, the value of the `handlerURL` attribute on
the corresponding `ApplicationOverride` element in `shibboleth2.xml`,
and appending "__/Metadata__". For example, for the default EntityID on
__webapp.itlab.stanford.edu__, with the handler at __/Shibboleth.sso__ we would use:

```shell_session
  % curl -so my-metadata.xml https://webapp.itlab.stanford.edu/Shibboleth.sso/Metadata
  % grep entityID my-metadata.xml
  <md:EntityDescriptor ... entityID="https://webapp.itlab.stanford.edu/">
```

### Metadata Cleanup

Before you submit your metadata, clean up the metadata:

1. Remove the warning from the top of the file
2. Clean up the `DigestMethod` and `SigningMethod` elements - you can remove any that refer to `sha1`
3. Add contact information to the end of the metadata:

```xml
  <md:EntityDescriptor entityID="...">
    <md:SPSSODescriptor>
    <!-- -->
    </md:SPSSODescriptor>
    <md:Organization>
      <md:OrganizationName xml:lang="en">Stanford University</md:OrganizationName>
      <md:OrganizationDisplayName xml:lang="en">Stanford University</md:OrganizationDisplayName>
      <md:OrganizationURL xml:lang="en">http://www.stanford.edu/</md:OrganizationURL>
    </md:Organization>
    <md:ContactPerson contactType="administrative">
      <md:GivenName>Jane Stanford</md:GivenName>
      <md:EmailAddress>jane.stanford@stanford.edu</md:EmailAddress>
    </md:ContactPerson>
    <md:ContactPerson contactType="technical">
      <md:GivenName>Leland Stanford, Jr</md:GivenName>
      <md:EmailAddress>leland.stanford.jr@stanford.edu</md:EmailAddress>
    </md:ContactPerson>
  </md:EntityDescriptor>
```

## Submit Metadata and Request Attributes

Submit your cleaned up metadata via
[HelpSU](https://helpsu.stanford.edu/helpsu/3.0/helpsu-form?pcat=shibboleth),
then submit a
[Data Owner Request](https://tools.stanford.edu/dataowner/dataowner-request)
to get the attributes your application needs (if the
[default](https://uit.stanford.edu/service/shibboleth/arp) attributes are not enough).

## When to Restart Apache and Shibd

If you only modify the Apache configuration, you only need to restart Apache.

If you modify the Shibboleth configuration, you need to restart Apache *and* shibd:

```shell_session
  # service apache2 restart ; service shibd restart
```
