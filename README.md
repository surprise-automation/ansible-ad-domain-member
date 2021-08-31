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

License
-------

BSD

Author Information
------------------

i r surprise