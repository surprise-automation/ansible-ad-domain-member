Role Name
=========

Sets Linux server as a domain member of a Samba AD domain. This *might* work with a Windows Domain but this is untested.

Requirements
------------

Requires a functional Domain on your network and accessible by your target.

Role Variables
--------------

| Variable | Example | Explanation |
| `domain_fqdn` | `ad.example.com` | The FQDN of your domain |
| `domain_dns_ip` | `[192.168.0.1, 192.168.0.2]` | An array that allows you to configure your ADDC's as your DNS providers - critically important |
| `domain_search` | `example.com` | This is appended to hostname lookups. Ie. if you ping ad it will ping ad.example.com. This will get configured into your NM, which adds it to resolv.conf |

Dependencies
------------

No library dependencies. This will install Samba as you need from repo.

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