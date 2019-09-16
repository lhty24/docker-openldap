# osixia/openldap
Note: simplified readme. Please see below for original guide:
https://github.com/osixia/docker-openldap

## Quick Start
Run OpenLDAP docker image:

	docker run --name my-openldap-container --detach osixia/openldap:1.2.5


Search in the LDAP container to test:

	docker exec my-openldap-container ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin


## Beginner Guide

### Create new ldap server

This is the default behavior when you run this image.
It will create an empty ldap for the company **Example Inc.** and the domain **example.org**.

By default the admin has the password **admin**. All those default settings can be changed at the docker command line, for example:

	docker run --env LDAP_ORGANISATION="My Company" --env LDAP_DOMAIN="my-company.com" \
	--env LDAP_ADMIN_PASSWORD="JonSn0w" --detach osixia/openldap:1.2.5


#### Edit your server configuration

Do not edit slapd.conf it's not used. To modify your server configuration use ldap utils: **ldapmodify / ldapadd / ldapdelete**


### Backup
A simple solution to backup your ldap server, is our openldap-backup docker image:
> [osixia/openldap-backup](https://github.com/osixia/docker-openldap-backup)


### Multi master replication
Quick example, with the default config.

	#Create the first ldap server, save the container id in LDAP_CID and get its IP:
	LDAP_CID=$(docker run --hostname ldap.example.org --env LDAP_REPLICATION=true --detach osixia/openldap:1.2.5)
	LDAP_IP=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" $LDAP_CID)

	#Create the second ldap server, save the container id in LDAP2_CID and get its IP:
	LDAP2_CID=$(docker run --hostname ldap2.example.org --env LDAP_REPLICATION=true --detach osixia/openldap:1.2.5)
	LDAP2_IP=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" $LDAP2_CID)

	#Add the pair "ip hostname" to /etc/hosts on each containers,
	#because ldap.example.org and ldap2.example.org are fake hostnames
	docker exec $LDAP_CID bash -c "echo $LDAP2_IP ldap2.example.org >> /etc/hosts"
	docker exec $LDAP2_CID bash -c "echo $LDAP_IP ldap.example.org >> /etc/hosts"

That's it! But a little test to be sure:

Add a new user "billy" on the first ldap server

	docker exec $LDAP_CID ldapadd -x -D "cn=admin,dc=example,dc=org" -w admin -f /container/service/slapd/assets/test/new-user.ldif -H ldap://ldap.example.org -ZZ

Search on the second ldap server, and billy should show up!

	docker exec $LDAP2_CID ldapsearch -x -H ldap://ldap2.example.org -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin -ZZ

	[...]

	# billy, example.org
	dn: uid=billy,dc=example,dc=org
	uid: billy
	cn: billy
	sn: 3
	objectClass: top
	objectClass: posixAccount
	objectClass: inetOrgPerson
	[...]


## Environment Variables
Environment variables defaults are set in **image/environment/default.yaml** and **image/environment/default.startup.yaml**.


### Set your own environment variables

#### Use command line argument
Environment variables can be set by adding the --env argument in the command line, for example:

	docker run --env LDAP_ORGANISATION="My company" --env LDAP_DOMAIN="my-company.com" \
	--env LDAP_ADMIN_PASSWORD="JonSn0w" --detach osixia/openldap:1.2.5

Be aware that environment variable added in command line will be available at any time
in the container. In this example if someone manage to open a terminal in this container
he will be able to read the admin password in clear text from environment variables.

#### Link environment file

For example if your environment files **my-env.yaml** and **my-env.startup.yaml** are in /data/ldap/environment

	docker run --volume /data/ldap/environment:/container/environment/01-custom \
	--detach osixia/openldap:1.2.5

Take care to link your environment files folder to `/container/environment/XX-somedir` (with XX < 99 so they will be processed before default environment files) and not  directly to `/container/environment` because this directory contains predefined baseimage environment files to fix container environment (INITRD, LANG, LANGUAGE and LC_CTYPE).

Note: the container will try to delete the **\*.startup.yaml** file after the end of startup files so the file will also be deleted on the docker host. To prevent that : use --volume /data/ldap/environment:/container/environment/01-custom**:ro** or set all variables in **\*.yaml** file and don't use **\*.startup.yaml**:

	docker run --volume /data/ldap/environment/my-env.yaml:/container/environment/01-custom/env.yaml \
	--detach osixia/openldap:1.2.5


## Advanced User Guide

### Tests

We use **Bats** (Bash Automated Testing System) to test this image:

> [https://github.com/bats-core/bats-core](https://github.com/bats-core/bats-core)

Install Bats, and in this project directory run:

	make test

++++++++++++++
## Create Users

get into the container	

	docker exec -it <container name> /bin/bash
	docker exec -it <container name> <command>

generate new password 

	slappasswd -h {} 
then hit enter
once you get your password, copy it so that you can use it when you create a user record

	ldapadd -x -D "cn=admin,dc=,dc= -W
then it will ask for admin password, once you enter that, it will wait for you to enter new record like so
	dn: ou=groups,dc=,dc=
	objectclass: organizationalUnit
	objectclass: top
	ou: users
then hit enter twice to push your entry

for users it will go exactly the some way except you need to add more infomation

	dn: uid=,ou=,dc=,dc=
	cn: first name lastname
	givenname: first name
	sn: last name
	uid: username
	uidnumber:
	userpassword:
	gid: this is important to add of you are creating posixAccount
	homedirectory: /home/users/
	loginshell: /bin/sh
	objectclass: inetOrgPerson
	objectclass: posixAccount
	objectclass: top

PS: you probably need to lay your user DN like so "uid,ou,dc,dc" especially if you are using ldap for user login and authentication


