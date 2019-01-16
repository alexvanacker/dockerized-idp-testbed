# dockerized-idp-testbed Concord edition

This is an IDP you can run locally that works for single sign-on with Concord.

Derived from the great work done here: https://github.com/UniconLabs/dockerized-idp-testbed

If you have any issues not mentioned in this doc, see their documentation which will be more helpful.

## Prerequisites

- docker
- [docker-compose](https://docs.docker.com/compose/install/)
- [Nego-App project](https://github.com/WitchBird/Negotiation-App)

## First steps on Concord

You will need to get your local instance's metadata.xml. To do so, in the Nego-App project:

1. uncomment the line in `src/main/resources/context/saml-security-context.xml` containing `entityId` and set a proper
entityId (firstname.lastname for instance).
2. run the app and download the file at http://localhost:8080/contract-live/saml/metadata
3. have an ENTERPRISE_2 organization (have the organization ID ready for later steps)

## Preparing the IDP

In this project:

1. Copy your metadata file in `idp/shibboleth-idp/metadata/`. You can either overwrite `concord-metadata.xml` or give it
another name. If you give it another name, report that name that name in `shibboleth-idp/conf/metadata-providers.xml` on
the line with `<MetadataProvider id="LocalMetadata" `
3. If you wish to test your SSO with an email domain different from `test.com`, then edit `ldap/users.ldif` file, where
the user emails are defined. You can also add users there if you wish.
4. Build with `docker-compose build`
5. Run with `docker-compose up` (append `-d` for daemon mode)

6. (Only once) Add your Docker Host IP to `/etc/hosts` with the following line:

```
YOUR_DOCKER_HOST_IP      idptestbed
```

You can find your Docker Host IP by following these steps:

* run `docker ps` and find the container id for the `dockerized-idp-testbed_httpd-proxy` image.
* run `docker inspect CONTAINER_ID | grep IPAddress`: this will give the Docker Host IP you need to fill in your `hosts`
file. (If there are two VALUES, use the last one).

8. Go to `https://idptestbed/` and accept the invalid SSL certificate. Login with one of the users in `ldap/users.ldif`
using the `uid` and `userPassword` from that file.

If you need to update users or any other property, you will need to run `docker-compose rm` before
building the images once more.


## Setting up Concord

The IDP metadata file can be found in `idp/shibboleth-idp/metadata/idp-metadata.xml`. Copy it in `src/main/resources/sso`
and name it `local-idp-meta.xml`.

Add the following block to `src/main/resources/context/saml-security-context.xml`, in the
`<bean id="metadata" class="org.springframework.security.saml.metadata.CachingMetadataManager"><constructor-arg><list>` node:


```xml
<bean class="org.springframework.security.saml.metadata.ExtendedMetadataDelegate">
    <property name="metadataTrustCheck" value="false"/>
    <constructor-arg>
        <bean class="org.opensaml.saml2.metadata.provider.ResourceBackedMetadataProvider">
            <constructor-arg>
                <bean class="java.util.Timer"/>
            </constructor-arg>
            <constructor-arg>
                <bean class="org.opensaml.util.resource.ClasspathResource">
                    <constructor-arg value="/sso/local-idp-meta.xml"/>
                </bean>
            </constructor-arg>
            <property name="parserPool" ref="parserPool"/>
        </bean>
    </constructor-arg>
    <constructor-arg>
        <bean class="org.springframework.security.saml.metadata.ExtendedMetadata" />
    </constructor-arg>
</bean>
```

It should look like this:

```xml
<bean id="metadata" class="org.springframework.security.saml.metadata.CachingMetadataManager">
    <constructor-arg>
        <list>
            <bean class="org.springframework.security.saml.metadata.ExtendedMetadataDelegate">
                <property name="metadataTrustCheck" value="false"/>
                <constructor-arg>
                    <bean class="org.opensaml.saml2.metadata.provider.ResourceBackedMetadataProvider">
                        <constructor-arg>
                            <bean class="java.util.Timer"/>
                        </constructor-arg>
                        <constructor-arg>
                            <bean class="org.opensaml.util.resource.ClasspathResource">
                                <constructor-arg value="/sso/local-idp-meta.xml"/>
                            </bean>
                        </constructor-arg>
                        <property name="parserPool" ref="parserPool"/>
                    </bean>
                </constructor-arg>
                <constructor-arg>
                    <bean class="org.springframework.security.saml.metadata.ExtendedMetadata" />
                </constructor-arg>
            </bean>
            <!-- ... more declarations ... -->
        </list>
    </constructor-arg>
</bean>
```

The attributes sent are listed in the `attribute-filter.xml` file. The actual attribute names sent are themselves
defined in the `attribute-resolver.xml` file. If you are not testing for specific attributes, you can use the following
for the saml_attribute setup in the application's database (see below):

- PPID: urn:oid:0.9.2342.19200300.100.1.1
- FIRST_NAME: urn:oid:2.5.4.42
- LAST_NAME: urn:oid:2.5.4.4

So on your database, assuming your organization ID is `1` you would run the following:

```sql
INSERT INTO saml_integration (domain, idp, organization_id) VALUES ('test.com','https://idptestbed/idp/shibboleth', 1);

INSERT INTO saml_attribute (type, name, saml_integration_id)
SELECT 'PPID', 'urn:oid:0.9.2342.19200300.100.1.1', saml_integration.id
FROM saml_integration
WHERE saml_integration.idp = 'https://idptestbed/idp/shibboleth' AND saml_integration.domain = "test.com";

INSERT INTO saml_attribute (type, name, saml_integration_id)
SELECT 'LAST_NAME', 'urn:oid:2.5.4.42', saml_integration.id
FROM saml_integration
WHERE saml_integration.idp = 'https://idptestbed/idp/shibboleth' AND saml_integration.domain = "test.com";

INSERT INTO saml_attribute (type, name, saml_integration_id)
SELECT 'FIRST_NAME', 'urn:oid:2.5.4.4', saml_integration.id
FROM saml_integration
WHERE saml_integration.idp = 'https://idptestbed/idp/shibboleth' AND saml_integration.domain = "test.com";
```

You will need to restart your Concord app. Then login via SSO using a `test.com` email address. You should be redirected
to the IDP. Login with `uid` and `userPassword` defined in `ldap/users.ldif`.

Accept to release all attributes when you reach this page

![release_attributes](https://github.com/concordnow/dockerized-idp-testbed/raw/master/doc/idp-release-example.png "IDP login example")


Congratulations! You logged in via SSO using a locally running IDP!


## Improvements

- webapp to add/edit/remove users easily
- webapp to add your local instance's metadata and declare it as an SP easily 
