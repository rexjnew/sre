## 1. sockd
# 1.1 配置文件 ../conf/sockd.conf 
errorlog:  /var/log/sockd.err.log
logoutput: /var/log/sockd.log
debug: 6

internal.protocol: ipv4
internal: ens192  port = 80
external.protocol: ipv4
external: ens192
#external.rotation: same-same

clientmethod: none
socksmethod: pam.username none

user.privileged: root
user.notprivileged: nobody

# allowable authentication methods for socks-rules.
#method: pam

# Client rules, controls who may connect
client pass {
    from: 0/0  to: 0/0
    log: connect disconnect data
}
client block {
    from: 0/0 to: 0/0
    log: connect error data
}

## everyone who authenticates is allowed to use tcp
## and udp
socks pass {
       from: 0.0.0.0/0 to: 0.0.0.0/0
       protocol: tcp udp
       log: connect disconnect error data
       socksmethod: pam.username none
}

# last line, block everyone else.  This is the default but if you provide
# one yourself you can specify your own logging/actions
socks block {
       from: 0.0.0.0/0 to: 0.0.0.0/0
       log: connect error data
}

## 1.2 ldap配置
cat /etc/ldap.conf 
#
# $Id: section-pamnss.sgml,v 1.2 2001/03/26 16:57:07 rolek Exp $
# This is the configuration file for the LDAP nameservice
# switch library and the LDAP PAM module.
# PADL Software
# http://www.padl.com
#
# If the host and base aren't here, then the DNS RR
# _ldap._tcp.[defaultdomain]. will be resolved. [defaultdomain]
# will be mapped to a distinguished name and the target host
# will be used as the server.

# Your LDAP server. Must be resolvable without using LDAP.
host 172.21.xx
#
# The distinguished name of the search base.
base dc=xx, dc=com
#
# The LDAP version to use (defaults to 2,
# use 3 if you are using OpenLDAP 2.0.x or Netscape Directory Server)
# ldap_version 3
#
# The distinguished name to bind to the server with.
# Optional: default is to bind anonymously.
binddn cn=admin,dc=xx,dc=com
#
# The credentials to bind with. 
# Optional: default is no credential.
#bindpw secret
bindpw my_password_here
## host 10.0.0.1
## base cn=Users,dc=test,dc=org
## rootbinddn cn=Administrator,cn=Users,dc=example,dc=org
## pam_filter objectclass=user
## pam_login_attribute cn

#
# The port.
# Optional: default is 389. 636 is for ldaps
port 389
#port 636
#
# The search scope.
#scope sub
#scope one
#scope base
#
# The following options are specific to nss_ldap.
#
# The hashing algorithm your libc uses. 
# Optional: default is des
#crypt md5
#crypt sha
#crypt des
#
# The following options are specific to pam_ldap.
#
# Filter to AND with uid=%s
pam_filter objectclass=posixAccount
#
# The user ID attribute (defaults to uid)
pam_login_attribute uid
#
# Search the root DSE for the password policy (works
# with Netscape Directory Server)
#pam_lookup_policy yes
#
# Group to enforce membership of
#
#pam_groupdn cn=PAM,ou=Groups,dc=padl,dc=com
#
# Group member attribute
#pam_member_attribute memberuid
# Template login attribute, default template user
# (can be overridden by value of former attribute
# in user's entry)
#pam_login_attribute userPrincipalName
#pam_template_login_attribute uid
#pam_template_login nobody
#
# Hash password locally; required for University of
# Michigan LDAP server, and works with Netscape
# Directory Server if you're using the UNIX-Crypt
# hash mechanism and not using the NT Synchronization
# service.
pam_crypt local
#
# SSL Configuration
#ssl yes
#sslpath /usr/local/ssl/certs

## 1.3 pam
https://tldp.org/HOWTO/archived/LDAP-Implementation-HOWTO/pamnss.html
cat /etc/pam.d/sockd 
#%PAM-1.0
auth       sufficient /lib/security/pam_ldap.so
account    sufficient /lib/security/pam_ldap.so
password   required   /lib/security/pam_ldap.so


## 3. ldap
#### add-memberof.ldif #####
dn: cn=module{0},cn=config
cn: module{0}
objectClass: olcModuleList
objectclass: top
olcModuleload: memberof.la
olcModulePath: /usr/lib64/openldap

dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfUniqueNames
olcMemberOfMemberAD: uniqueMember
olcMemberOfMemberOfAD: memberOf


#### changedomain.ldif #####
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=xx,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=xx,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=xx,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}xx

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=admin,dc=xx,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=admin,dc=xx,dc=com" write by * read


#### changepwd2.ldif #####
dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}xx


dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}xx


#### changepwd.ldif #####
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}xx


#### ldapuser.ldif #####
# 这里testUser用户，我将其加入到testgroup组中
# create new
# replace to your own domain name for "dc=***,dc=***" section
dn: uid=testldap,ou=People,dc=bigdata,dc=xx,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: testldap
cn: testgroup
sn: test
userPassword: {SSHA}xx
loginShell: /bin/bash
uidNumber: 2000
gidNumber: 3000
homeDirectory: /home/testldap


#这是添加一个用户组名为testgroup的cn，在名为Group的ou下
dn: cn=testgroup,ou=Group,dc=bigdata,dc=xx,dc=com
objectClass: posixGroup
cn: testgroup
gidNumber: 3000
memberUid: testldap


#### refint1.ldif #####
dn: cn=module{0},cn=config
add: olcmoduleload
olcmoduleload: refint


#### refint2.ldif #####
dn: olcOverlay=refint,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: refint
olcRefintAttribute: memberof uniqueMember  manager owner


#### xx.ldif #####
dn: dc=xx,dc=com
dc: xx
objectClass: top
objectClass: domain
o: xx

dn: cn=admin,dc=xx,dc=com
objectClass: organizationalRole
cn: admin
description: LDAP admin

dn: dc=bigdata,dc=xx,dc=com
changetype: add
dc: bigdata
objectClass: top
objectClass: dcObject
objectClass: organization
o: bigdata


## 4. client
Apache Directory Studio


dn: ou=People,dc=bigdata,dc=xx,dc=com
ou: People
objectClass: organizationalUnit

dn: ou=Group,dc=bigdata,dc=xx,dc=com
ou: Group
objectClass: organizationalUnit
