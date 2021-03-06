# dockerized-idp-testbed Concord edition

This is an IDP you can run locally that works for single sign-on with Concord.

Derived from the great work done here: https://github.com/UniconLabs/dockerized-idp-testbed

If you have any issues not mentioned in this doc, see their documentation which will be more helpful.


## Prerequisites

- docker
- [docker-compose](https://docs.docker.com/compose/install/)
- [Nego-App project](https://github.com/concordnow/Negotiation-App)
- [Concord Front App](https://github.com/concordnow/concord-app-front)


## First steps on Concord

You will need to get your local instance's metadata.xml. To do so, in the Nego-App project:

1- Uncomment the line in `src/main/resources/perso.properties` containing `saml.entityId` and set a proper entityId (firstname.lastname for instance). 

You should have something like:

```
## Saml
#Uncomment following line and set up your entityId for a local use of SAML
saml.entityId=rich.hickey
```

2- Run the dev-proxy and npm run serve in the concord-app-front project.

3- Run concord-ws and download the file at http://localhost/contract-live/saml/metadata.

We'll now call this file `concord-metadata.xml`.

4- Have an ENTERPRISE_2 organization (have the organization ID ready for later steps).


## Preparing the IDP

In this project:

1- In folder `idp/shibboleth-idp/metadata/`, overwrite the local file `concord-metadata.xml` with the `concord-metadata.xml` you downloaded earlier.
 
You can also give it another name instead of overwriting it, 
but if you do you need to report that name in `idp/shibboleth-idp/conf/metadata-providers.xml` on the line with `<MetadataProvider id="LocalMetadata"`

You would then have:
```
<MetadataProvider id="LocalMetadata"  xsi:type="FilesystemMetadataProvider" metadataFile="%{idp.home}/metadata/concord-metadata.xml"/>
```

2- By default, the email domain is `test.com`.

If you wish to test your SSO with a different email domain, then edit `ldap/users.ldif` file, where the user emails are defined. 
You can also add users there if you wish.

If you want to add users, you can copy an existing user, but you have to change both `uid` fields and the `mail` field.
Example:
```
dn: uid=newuser,ou=People,dc=idptestbed
objectClass: organizationalPerson
objectClass: person
objectClass: top
objectClass: inetOrgPerson
givenName: St
uid: newuser
sn: aff
cn: St aff
mail: newuser@test.com
userPassword: password
```

3- Build with `docker-compose build` (it's easier to use the command line than Intellij)

If you run into this error:
```
Step 8/14 : RUN pear install -Z CAS-1.3.4.tgz  && rm /CAS-1.3.4.tgz
 ---> Running in e305cfec6438
could not extract the package.xml file from "CAS-1.3.4.tgz"
install failed
ERROR: Service 'php-cas' failed to build: The command '/bin/sh -c pear install -Z CAS-1.3.4.tgz  && rm /CAS-1.3.4.tgz' returned a non-zero code: 1
```
You may just remove ` RUN pear install -Z CAS-1.3.4.tgz  && rm /CAS-1.3.4.tgz` from `php-cas/Dockerfile`.
It will not affect SAML test bench

4- Run with `docker-compose up` (append `-d` for daemon mode)

5- (Only once) Add your Docker Host IP to `/etc/hosts`. To do that, add the following line:

```
YOUR_DOCKER_HOST_IP      idptestbed
```

You can find your Docker Host IP by running:
```shell script
docker inspect -f '{{(index .NetworkSettings.Networks "dockerized-idp-testbed_front").IPAddress}}' dockerized-idp-testbed_httpd-proxy_1
```

6- Go to `https://idptestbed/` and accept the invalid SSL certificate. 
You should be able to login, into the simpleSAML service with one of the users in `ldap/users.ldif` using the `uid` and `userPassword` from that file.

:warning: If you need to update users or any other property, you will need to run `docker-compose build --no-cache` to build the images once more.

## Setting up Concord

The IDP metadata file can be found in `idp/shibboleth-idp/metadata/idp-metadata.xml`. 
We'll now call this file `idp-metadata.xml`.

In the project [operations](https://github.com/concordnow/operations/) you can find the scripts to setup NegoApp in [templates/enable_sso](https://github.com/concordnow/operations/tree/master/templates/enable_sso)

We'll assume the following:

1- The organization_id you found earlier (in step `Have an ENTERPRISE_2 organization`) is `1234`

2- The idp_metadata file is called `idp-metadata.xml`

3- The auth token you get from your app is `1a2b3cxx88...55t` (it should be longer)

You need to get the project on your computer, edit the files and run the scripts:

1- Copy `idp-metadata.xml` in the folder `concordnow/operations/templates/enable_sso`.

2- In `2_operations.sh`: 

  - 2.1- set the `ORGANIZATION_ID` field with the organization_id you found earlier (in step `Have an ENTERPRISE_2 organization`) -> `ORGANIZATION_ID="1234"`
  
  - 2.2- set the `METADATA_TRUST_CHECK` to `true` -> `METADATA_TRUST_CHECK=true`
  
  - 2.3- edit the TWO curl requests from `https://secure.concordnow.com` to `http://localhost/contract-live` -> `curl --request POST "http://localhost/contract-live/api/rest/1/........" \`
    
  - 2.4- edit `--form "metadata=` with your idp_metadata file -> `--form "metadata=@idp-metadata.xml" \`
  
  - 2.5- edit the `AUTH` field with your auth token (you can get this by connecting to an account, look at the network requests) -> `AUTH="1a2b3cxx88...55t"`
   
Note: Make sure your user has the `ROLE_TECH` in `user.granted_authorities` (comma-separated list of special roles)
Note: You could probably use an API-KEY instead of the `AUTH` field, but I'll let you figure that out yourselves.
    
3- In `body_first_name_last_name.json`:

By default, you should set it like this:

``` 
{
  "emailDomain": "test.com",
  "idp": "https://idptestbed/idp/shibboleth",
  "ppid": "urn:oid:0.9.2342.19200300.100.1.1",
  "firstName": "urn:oid:2.5.4.4",
  "lastName": "urn:oid:2.5.4.42"
}
```

If you changed the emailDomain earlier, change it here as well.

The attributes sent (`ppid`, `firstName` and `lastName`), as defined above, are for a basic setup.

If you are testing for specific attributes, the attributes sent are listed in the `attribute-filter.xml` file. 
The actual attribute names sent are themselves defined in the `attribute-resolver.xml` file. 

4- Rename `body_first_name_last_name.json` to `body.json` (or change the `--data @body.json \` field in `2_operations.sh`)

5- Execute the query `1_before.sql` to make sure the saml_integration data is NULL.

6- Execute the script `2_operations.sh`.

7- Execute the query `3_after.sql` to make sure the script worked.

8- Restart NegoApp.

9-Login via SSO using a `test.com` email address. You should be redirected to the IDP. 

Login with `uid` and `userPassword` defined in `ldap/users.ldif`.

Accept to release all attributes when you reach this page

![release_attributes](https://github.com/concordnow/dockerized-idp-testbed/raw/master/doc/idp-release-example.png "IDP login example")

Congratulations! You logged in via SSO using a locally running IDP!


## For testing SAML integration / setup

### Updating an attribute name sent to Concord

You can change attribute names sent to Concord by editing `idp/shibboleth-idp/conf/attribute-resolver.xml`. Choose one of the attributes sent
and change the name on the line with `xsi:type="enc:SAML2String"`. Example: you could change the surname to be sent as `surname` instead of `urn:oid:2.5.4.42`
by replacing

```xml
<resolver:AttributeEncoder xsi:type="enc:SAML2String" name="urn:oid:2.5.4.4" friendlyName="sn" encodeType="false" />
```

by

### Modifying the NameId sent in the SAML token

You can change the way the NameId is built by changing the [saml-nameid.xml](idp/shibboleth-idp/conf/saml-nameid.xml).
There is a bean declaration of `SAML2NameIDGenerators` and the NameId construction is specified
at line 37 with `p:attributeSourceIds="#{ {'mail'} }"`.

To have an empty NameId, change it to:
`p:attributeSourceIds="#{ '' }"`

To have a NameId that is not the email but the uid, change it to:
`p:attributeSourceIds="#{ {'uid'} }"`

I did not test, but I believe you could build whatever NameId you need based on users properties
from [users.ldif](ldap/users.ldif).

You will need to rebuild the project to take this changes in account.

```xml
<resolver:AttributeEncoder xsi:type="enc:SAML2String" name="surname" friendlyName="sn" encodeType="false" />
```
After doing so, you will need to rebuild the project:

```bash
docker-compose down
docker-compose up --build
```

### Enabling IDP-init (AKA unsollicited) flow

You will need to edit your metadata by changing the `AuthnRequestsSigned` to `"false"``.

You will then need to edit the [index.php file](simplesamlphp/var-www-html/php-ssp-protected/index.php) and change `alexis.vanacker` to your Concord-WS's entity ID.

Rebuild the project.
Go to https://idptestbed/ and login using the Simple SAML PHP app. The ugly page has a link IDP-init that will lead to a Concord login.

## Improvements

- webapp to add/edit/remove users easily
- webapp to add your local instance's metadata and declare it as an SP easily 
