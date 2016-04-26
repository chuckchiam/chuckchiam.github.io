---
layout: post
title:  "Configuring Active Directory and ScaleIO Secure Authentication"
date:   2016-04-25 19:25:46 -0400
categories: scaleio AD
---

I was helping someone enable AD authentication in their environment and once the LDAP connection had been configured and proven working via non-secure connection, we then went to implement SSL. As happens in the datacenter, things did not work straight away. We had to step through various layers in the solution to validate them in order to finalize the configuration.

Before we go any further, i recommend giving the ldap services names during the –add_ldap_service command with the –ldap_service_name switch as this makes it easier (at least in my experience) to manage the various ldap service objects as opposed to using an objectID.

Configuring an LDAP:// uri is well covered in “User-Roles-and-LDAP-Usage-Technical-Notes” available on emc.com. I’ll try to stay away from any repetitive recomendations, but i will say that several of the methods below (especially the ldapsearch commands since they are coming from the machine running the MDM itself) can help work through the configuration of non-secure connections as well.

At this point, we deleted and recreated the non-secure ldap service with a secure ldaps:// URI’s using the scli –remove_ldap_service and scli –add_ldap_service command and assigned a group to the monitor role via scli –assign_ldap_groups_to_roles.

We then went to configure SSL for the openldap packages leveraged by ScaleIO. First, the certs that are used to secure the LDAP service on the domain controller (typically given to you by the directory team) may need to be converted to more a useful format (here we are translating to a .pem file).

`openssl x509 -inform der -in filename.cer -out filename.pem`

We then need to create the openldap cacert directory where trusted ssl certificates can be placed

`cd /etc/openldap/`  
`mkdir cacerts`  
`cd cacerts/`  

Then we need to insure that directory we just created is known to openldap. If you look at the /etc/openldap/ldap.conf file you should see a line that resembles

`TLS_CACERTDIR /etc/openldap/cacerts`

next you’ll need to hash cert files with the cacertdir_rehash command, Execute the following:

`cacertdir_rehash /etc/openldap/cacerts`

after you do this, you’ll see something like this if you examine the cacerts directory:

root@server1:/etc/openldap/cacerts#ls -alrt  
total 12  
-rw-r–r–. 1 root root 2004 Feb 22 16:23 filename.pem  
lrwxrwxrwx. 1 root root 17 Feb 22 16:23 72a1a5ef.0 -> filename.pem  
drwxr-xr-x. 2 root root 4096 Feb 22 16:23 .  
drwxr-xr-x. 7 root root 4096 Feb 26 16:13 ..  

Now we’ll double check the cert on the server and make sure thngs look OK. As you can see here we have a self signed cert with a single certificate, it’s likely that in a production environment you’ll see multiple certificates representing a chain.

`openssl s_client -CApath /etc/openldap/cacerts -showcerts -connect ldapserver.scaleio.lab:636`

SSL-Session:  
Protocol : TLSv1.2  
Cipher : AES128-SHA256  
Session-ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  
Session-ID-ctx:  
Master-Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  
Key-Arg : None  
Krb5 Principal: None  
PSK identity: None  
PSK identity hint: None  
Start Time: xxxxxxxxxx  
Timeout : 300 (sec)  
Verify return code: 0 (ok)  

note you could run a similar command against any SSL enabled server:port to see it’s certificate. This command will show you google’s cert

`openssl s_client -connect google.com:443`

Now that we have openldap configured to be able to trust the LDAP’s server certificates and have verified the SSL connection to the ldap server. We can test LDAPS with a tool called ldapsearch. I’ll try to describe some of the switches we’re using below which should be enough to get things going.

-x :use simple authentication, username and password over ssl in this case  
-H : uri for the ldap server, notice the “ldaps” deisgnation in the protocol name, i believe you can append a port here too  
-b :base dn for the search, this should be the same as the –ldap_base_dn used when creating the ldap service for scaleio  
-d : username in the form of a upn (user principal name) as this is the way ScaleIO handles the username  
-W :ldap search string (take a look a the LDAP RFC’s if you want more info, they are not that bad)  

First a simple search that returns the user object:

`ldapsearch -x -H ldaps://ad-east.east.corp.scaleio.lab -b 'DC=east,DC=corp,DC=scaleio,DC=lab' -D 'user1@east.corp.scaleio.lab' -W '(&(objectClass=user)(sAMAccountName=user1))'``

we are doing a simple query for user1. This should return the attributes of the user similar to this

extended LDIF  

LDAPv3  
base <DC=EAST,DC=CORP,DC=scaleio,DC=lab> with scope subtree  
filter: (&(objectClass=user)(sAMAccountName=user1))  
requesting: ALL  

User1, Corporate-Users, east.corp.scaleio.lab  
dn: CN=User1,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab  
objectClass: top  
objectClass: person  
objectClass: organizationalPerson  
objectClass: user  
cn: User1  
givenName: User1  
distinguishedName: CN=User1,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab  
instanceType: 4  
whenCreated: 20160422213243.0Z  
whenChanged: 20160422230035.0Z  
displayName: User1  
uSNCreated: 32906  
memberOf: CN=monitor-users,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab  
uSNChanged: 32980  
name: User1  
objectGUID:: vaV/QQaSikGTnEmnH3zqcg==  
userAccountControl: 66048  
badPwdCount: 0  
codePage: 0  
countryCode: 0  
badPasswordTime: 131059334183717788  
lastLogoff: 0  
lastLogon: 131059334236073065  
pwdLastSet: 131058395269417278  
primaryGroupID: 513  
objectSid:: AQUAAAAAAAUVAAAA1nikEkCxa0+Gai/kUwQAAA==  
accountExpires: 9223372036854775807  
logonCount: 0  
sAMAccountName: user1  
sAMAccountType: 805306368  
userPrincipalName: user1@east.corp.scaleio.lab  
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=corp,DC=scaleio,DC=lab  
dSCorePropagationData: 16010101000000.0Z  
lastLogonTimestamp: 131058396355852659  

search reference  
ref: ldap://DomainDnsZones.east.corp.scaleio.lab/DC=DomainDnsZones,DC=east,DC= corp,DC=scaleio,DC=lab  

search result  
search: 2  
result: 0 Success  

numResponses: 3  
numEntries: 1  
numReferences: 1  

Then we can run a more complex search

`ldapsearch -x -H ldaps://ad-east.east.corp.scaleio.lab -b 'DC=east,DC=corp,DC=scaleio,DC=lab' -D 'user2@east.corp.scaleio.lab' -W '(&(objectClass=user)(sAMAccountName=user1)(memberOf:1.2.840.113556.1.4.1941:='CN=monitor_group,OU=Corporate-Users,DC=scaleio,DC=lab'))``

Here we are doing a search similar to the search that ScaleIO performs. Note that the use of the OID in the LDAP search means that we will get memberof information for nested groups (which the simpler search above will not return).

extended LDIF  

LDAPv3  
base <DC=EAST,DC=CORP,DC=scaleio,DC=lab> with scope subtree  
filter:   (&(objectClass=user)(sAMAccountName=user1)(memberOf:1.2.840.113556.1.4.1941:=CN=monitor-perms,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab))  
requesting: ALL  

User1, Corporate-Users, east.corp.scaleio.lab  
dn: CN=User1,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab  
objectClass: top  
objectClass: person  
objectClass: organizationalPerson  
objectClass: user  
cn: User1  
givenName: User1  
distinguishedName: CN=User1,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab  
instanceType: 4  
whenCreated: 20160422213243.0Z  
whenChanged: 20160422230035.0Z  
displayName: User1  
uSNCreated: 32906  
memberOf: CN=monitor-users,OU=Corporate-Users,DC=east,DC=corp,DC=scaleio,DC=lab  
uSNChanged: 32980  
name: User1  
objectGUID:: vaV/QQaSikGTnEmnH3zqcg==  
userAccountControl: 66048  
badPwdCount: 0  
codePage: 0  
countryCode: 0  
badPasswordTime: 131059334183717788  
lastLogoff: 0  
lastLogon: 131059334236073065  
pwdLastSet: 131058395269417278  
primaryGroupID: 513  
objectSid:: AQUAAAAAAAUVAAAA1nikEkCxa0+Gai/kUwQAAA==  
accountExpires: 9223372036854775807  
logonCount: 0  
sAMAccountName: user1  
sAMAccountType: 805306368  
userPrincipalName: user1@east.corp.scaleio.lab  
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=corp,DC=scaleio,DC=lab  
dSCorePropagationData: 16010101000000.0Z  
lastLogonTimestamp: 131058396355852659  

search reference  
ref: ldap://DomainDnsZones.east.corp.scaleio.lab/DC=DomainDnsZones,DC=east,DC= corp,DC=scaleio,DC=lab  

search result  
search: 2  
result: 0 Success  

numResponses: 3  
numEntries: 1  
numReferences: 1  

A note about users and groups in this example. I have a user, “user1” who belongs to a group “monitor-users”. “monitor-users” in turn is a member of another group “monitor-perms”. Monitor-perms was the group designated in the –assign_ldap_groups_to_roles command via the –monitor_role_dn paramter. This represents a security model where you create two groups, one with permissions and one with users, then nest the users group in the group with permissions.

If all of these succeed, then the `scli –login –username user1@east.corp.scaleio.lab –ldap_authentication` command should allow us to login and execute `scli –query_all`.
