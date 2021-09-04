Role Name
=========

Sets Linux server as a domain member of a Samba AD domain. This *might* work with a Windows Domain but this is untested.

This is written solely for CentOS 8 and doesn't do any of the ground work which is explained in requirements.

Requirements
------------

Ensure the following are correct: 

1. Functional AD DC accessible by your target
2. DHCP delegating DNS, search domain and NTP settings correctly
3. DNS is infact pointing to your AD DC
4. You have set the machines hostname

**Note**

You cannot auto-bind. As per [samba docs](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member#Joining_the_Domain), you have to run through this configuration and then run the following on the server:

```
# net ads join -U administrator
Enter administrator's password: Passw0rd
Using short domain name -- SAMDOM
Joined 'M1' to dns domain 'samdom.example.com'
```

Role Variables
--------------

| Variable          | Example         | Explanation                                               |
| domain_fqdn       | AD.EXAMPLE.COM  | The FQDN of your domain name. Self explan.                |
| domain_workgroup  | AD              | The "shorthand" version of your domains name IE workgroup |
| domain_admin_user | Administrator   | Administrator username on the AD                          |

Dependencies
------------

No library dependencies. This will install Samba as you need from repo without anything special changed.

Example Playbook
----------------

1. Clone to `/etc/ansible/roles`
2. Configure Playbook accordingly

```yml
- hosts: servers
  roles:
      - default-playbook
      - ansible-ad-domain-member
      - app-deploying-playbooks
```

Notes
-----

So I set this up manually on a server and I'm struggling to bind. Firstly, you want to `cat /etc/resolv.conf` to make sure that your **ad dc** is your DNS, not the DNS server you forward to.

```
> net ads join -U administrator
Enter administrator's password:
Using short domain name -- AD
Joined 'AZSV-INTRANET-1' to dns domain 'ad.example.com'
DNS Update for test-serv.example.com failed: ERROR_DNS_GSS_ERROR
DNS update failed: NT_STATUS_UNSUCCESSFUL
```

According to the [docs](https://wiki.samba.org/index.php/Troubleshooting_Samba_Domain_Members), this error only comes up when you're using the BIND9_DLZ backend.

More specifically, I'm seeing the following messages which *appear* related in the logs:

```
... named[1680130]: samba_dlz: starting transaction on zone ad.example.com
... named[1680130]: client @0x012345678911 192.168.6.9#12345: update 'ad.example.com/IN' denied
... named[1680130]: samba_dlz: cancelling transaction on zone ad.example.com
```

As per the "[Reconfiguring the BIND9_DLZ Back End](https://wiki.samba.org/index.php/BIND9_DLZ_DNS_Back_End#Reconfiguring_the_BIND9_DLZ_Back_End)" documentation, you can run the `samba_upgradedns` command to 'self repair' however this didn't even fix the hardlinking between the two `sam.ldb.d` directories:

```
> samba_upgradedns --dns-backend=BIND9_DLZ
Reading domain information
DNS accounts already exist
No zone file /usr/local/samba/bind-dns/dns/AD.EXAMPLE.COM.zone
DNS records will be automatically created
DNS partitions already exist
dns-azsv-saddc-01 account already exists
See /usr/local/samba/bind-dns/named.conf for an example configuration include file for BIND
and /usr/local/samba/bind-dns/named.txt for further documentation required for secure DNS updates
Finished upgrading DNS

> ls -lai /usr/local/samba/private/sam.ldb.d/
total 39856
1520964 drwx------ 2 root root       336 Sep  1 13:13  .
1656938 drwx------ 7 root root      4096 Sep  1 13:13  ..
1520967 -rw------- 1 root root   7340032 Jun  5 14:43 'CN=CONFIGURATION,DC=AD,DC=EXAMPLE,DC=COM.ldb'
1520968 -rw------- 1 root root   9175040 Nov  4  2020 'CN=SCHEMA,CN=CONFIGURATION,DC=AD,DC=EXAMPLE,DC=COM.ldb'
1520966 -rw------- 1 root root  14483456 Sep  1 15:05 'DC=AD,DC=EXAMPLE,DC=COM.ldb'
1520973 -rw-rw---- 2 root named  4694016 Sep  1 13:09 'DC=DOMAINDNSZONES,DC=AD,DC=EXAMPLE,DC=COM.ldb'
1520974 -rw-rw---- 2 root named  4694016 Sep  1 13:09 'DC=FORESTDNSZONES,DC=AD,DC=EXAMPLE,DC=COM.ldb'
1520965 -rw-rw---- 2 root named   421888 Sep  1 14:33  metadata.tdb

> ls -lai /usr/local/samba/bind-dns/dns/sam.ldb.d/
total 25616
  1890539 drwxrwx--- 2 root named     336 Sep  1 13:13  .
101088688 drwxrwx--- 3 root named      38 Sep  1 13:13  ..
  1520990 -rw-rw---- 1 root named 6778880 Sep  1 13:13 'CN=CONFIGURATION,DC=AD,DC=EXAMPLE,DC=COM.ldb'
  1819038 -rw-rw---- 1 root named 8355840 Sep  1 13:13 'CN=SCHEMA,CN=CONFIGURATION,DC=AD,DC=EXAMPLE,DC=COM.ldb'
  1890545 -rw-rw---- 1 root named 1286144 Sep  1 13:13 'DC=AD,DC=EXAMPLE,DC=COM.ldb'
  1520973 -rw-rw---- 2 root named 4694016 Sep  1 13:09 'DC=DOMAINDNSZONES,DC=AD,DC=EXAMPLE,DC=COM.ldb'
  1520974 -rw-rw---- 2 root named 4694016 Sep  1 13:09 'DC=FORESTDNSZONES,DC=AD,DC=EXAMPLE,DC=COM.ldb'
  1520965 -rw-rw---- 2 root named  421888 Sep  1 14:33  metadata.tdb
```

I also did this in *production*, while users are *actively using it*. Don't be me. Also, this did *not* fix it, but it's worth testing in your circumstance. Anyway, something that caught my attention way too late into this is the zone file not existing; AFAIK BIND9_DLZ handles its own Bind configuration and pulls direct from the database however this particular part caught my curiosity. 

I manually created that zone file and copied a basic zone file from a working zone. All this is done on the Samba AD DC, however this did not help me at all; the zone file was deleted citing unknown origin.

At some point I decided to disabled IPv6 on my ADDC and I tried a new set of permissions on the above directories. Don't do it like me:

```
> chown -R root:named /usr/local/samba/private/sam.ldb.d/
> chown -R root:named /usr/local/samba/bind-dns/dns/sam.ldb.d/
> chmod -R 660 /usr/local/samba/private/sam.ldb.d/
> chmod -R 660 /usr/local/samba/bind-dns/dns/sam.ldb.d/
> chmod 660 /usr/local/samba/private/sam.ldb.d/
> chmod 660 /usr/local/samba/bind-dns/dns/sam.ldb.d/
> systemctl restart named ; journalctl -xe
...
EVERYTHING BROKEN
...
> samba_upgradedns --dns-backend=BIND9_DLZ
```

It's handy to know that you can roll back those permissions changes with that script though.

Eventually, I ran across [this discussion](https://mailing.unix.samba-technical.narkive.com/GxiD1wEO/samba4-and-bind9-dynamic-udpdates-not-working-anymore) and I thought that maybe I might be able to get something working from these example configurations (despite my configs essentially being from Samba.org). Using what they discussed, I built a new `named.conf` configuration:



License
-------

BSD

Author Information
------------------

i r surprise