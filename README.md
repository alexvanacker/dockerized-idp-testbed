# dockerized-idp-testbed Concord edition

This is an IDP you can run locally that works for single sign-on with Concord.

Derived from the great work done here: https://github.com/UniconLabs/dockerized-idp-testbed

If you have any issues not mentionned in this doc, see their documentation which will be more helpful.

## Preparing the IDP

1. Follow the documentation on Concord to generate your local metadata. Let's call this file `concord-metadata.xml`
2. Copy this file in `idp/shibboleth-idp/metadata`
3. If you changed the file's name, then report that name in `shibboleth-idp/conf/metadata-providers.xml` on the line with 
`<MetadataProvider id="LocalMetadata" `
4. For your test, you should have a domain name identified. Use it in the `ldap/users.ldif` file, where the user emails are defined. You can also add users there if you wish.
5. Build with `docker-compose build`
6. Run with `docker-compose up` (append `-d` for daemon mode)

7. (Only once) Add your Docker Host IP to `etc/hosts` with the following line:

```
YOUR_DOCKER_HOST_IP      idptestbed
```

You can find your Docker Host IP by following these steps:

* run `docker ps` and find the container id for the `dockerized-idp-testbed_httpd-proxy` image.
* run `docker inspect CONTAINER_ID | grep IPAddress`: this will give the Docker Host IP you need
to fill in your `hosts` file.

8. Go to `https://idptestbed/` and login with one of the users you defined to test login.

If you need to update users or any other property, you will need to run `docker-compose rm` before
building the images once more.


## Setting up Concord

The IDP metadata file can be found in `idp/shibboleth-idp/metadata/idp-metadata.xml`

The attributes sent are listed in the `attribute-filter.xml` file. The following can be used for:

- PPID: urn:oid:0.9.2342.19200300.100.1.1
- FIRST_NAME: urn:oid:2.5.4.42
- LAST_NAME: urn:oid:2.5.4.4


## Improvements

- webapp to add/edit/remove users easily
- webapp to add your local instance's metadata and declare it as an SP easily 
