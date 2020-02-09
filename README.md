# Deploy ACME CA by Boulder with Ansible

Since the launch of Let's Encrypt, it's been on my to-do list to set up an ACME CA server for my internal infrastructure.

However the lack of documentation to basically everything on [Boulder](https://github.com/letsencrypt/boulder/), [Certificate Transparency](https://github.com/google/certificate-transparency-go/) [log infrastructure](https://github.com/google/trillian), and lots of crypto materials to manage made it pretty hard to set up.

[Let's Encrypt uses Terraform](https://letsencrypt.org/2019/11/20/how-le-runs-ct-logs.html), though they never release their configs and I don't need their scalability.

This Ansible role handles setting up Boulder, Certificate Transparency log server, database, necessary crypto materials and reverse proxy in one package.

**WARNING**: This role is not meant to be setup a commerically production CA, you need to do additional steps to harden it.

## Requirements

1. You still need to manually create your issuing CA before using this role.

   It's my choice since I want full control of issuing CA subject, and I have an existing root CA, otherwise I would use `step-ca`.

   You need to place them in a secret folder. This secret folder will be created when you first run this role, or you can create `../secrets/$(target hostname)/issuing` anywhere, and supply the absolute path as a var named `secret` to this role.

2. Set up DNS records:

   - sub domain names that serve ACME and OCSP requests, for example:

     - `pki.example.com` to serve ACME,

     - `ocsp.pki.example.com` to OCSP.

     and set `api_domain` as the ACME domain name, `ca_policies.ocsp_domain` as the OCSP domain name.

   - a CAA record, otherwise Boulder cannot issue certificate to any of your servers.

3. Required Ansible 2.9.

4. This role is built with Debian buster in mind, and should have testing apt repo enabled.

5. Set up a internal DNS resolver:

   - to keep gRPC clients within Boulder from hard hitting public resolvers, but you should not let this server have unfettered access to internet anyways.

   - for Validation Authority to resolve your internal names.

6. Need root access on target machine. Read all the tasks before you run the play.

## Usage

Copy the roles included to your Ansible roles directory, then use as below.

``` yaml
---
  name: Sample playbook
  host: localhost
  
  roles:
    - role: deploy_boulder
      vars:
        # If you want to modify things in app_conf, you may need to supply
        # the whole dictionary. Or just modify the vars/main.yml...
        # Vars listed below should be enough for you.
        # Optional. Software install directories.
        install_root: /opt/boulder
        src_root: /opt/boulder-src
        ct_install_root: /opt/ctgo
        ct_src_root: /opt/ctgo-src
        trillian_install_root: /opt/trillian
        trillian_src_root: /opt/trillian-src
        # Optional. Change these releases vars to use updated source code
        current_release: release-2020-02-03
        ct_current_release: tags/v1.1.0
        trillian_current_release: master
        # Required. Place your CA in
        # secret/certificates/{{ ansible_fqdn }}/issuing
        issuing_ca_crtname: name.pem
        issuing_ca_keyname: name.key
        # Required.
        ca_database_name: ca
        ctlog_database_name: ct
        # Optional. Set False to avoid overhead of running a full CT log.
        full_ct_server: True
        # Required when full_ct_server is True.
        # You still need ctlog_shards to define the only tree.
        ctlog_sharding: False
        ctlog_shards:
          - prefix: example
            # These are ignored when ctlog_sharding=False, so optional
            # use anything > 0 that can be recongize by `to_datetime` filter
            start: 0
            end: 0
            root: '{{ issuing_ca_crtname }}'
        # Required. Provide your resolver *and* port
        va_resolvers:
          - 127.0.0.1:53
        # Required for your custom enviornments.
        ca_policies:
          # Your CAA record should match this
          caa_domain: example.com
          valid_duration: 2160h
          # Provided your own or leave this
          cp_oid: 2.5.29.32.0
          # You may need to host CPS elsewhere or
          # you will need to modify Caddyfile
          # You can leave this empty or omit as well
          cps_url: http://pki.example.com/cps
          cp_usernotice:
          # Required, set to where you're hosting OCSP responder
          ocsp_domain: ocsp.pki.example.com
        # Required, set to where you're hosting the ACME endpoints
        api_domain: pki.example.com
        company_homepage: example.com
```

Have a look in $(install_root)/config/*.yml for more control of issuing policy.

## Other words

This server does not use much resources, maybe you can get away with 1GB of RAM.

```
top - 21:15:47 up 24 days, 20:43,  1 user,  load average: 0.01, 0.04, 0.04
Tasks: 131 total,   1 running, 130 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.5 us,  0.3 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3946.2 total,    191.0 free,    489.7 used,   3265.6 buff/cache
```

Since there is no open sourced CT log viewer on the net, you are currently limited to ctclient from certificate-transparency-go to see what cert your CA issued (`**n**` for emphasis), only available when you run full CT log services:

```
/opt/ctgo$ ./ctclient -log_uri http://127.0.0.1:4011/prefix sth
2020-02-06 17:55:21.394 +0800 CST (timestamp 1580982921394): Got STH for V1 log (size=**n**) at http://127.0.0.1:4011/prefix, hash ...
Signature: Hash=SHA256 Sign=ECDSA Value=...
/opt/ctgo$ ./ctclient -log_uri http://127.0.0.1:4011/prefix -first 0 -last **n-1** getentries | grep Subject:
        Subject: CN=...
        Subject: CN=...
        ...
```

You can use the included `caddy2` role separately, just supply Caddyfile content to `vars.Caddyfile` then you are good to go.

## License

MIT License

Copyright (c) 2020 Lemon Lam

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
