# Ansible role: Virtual Host

**TL;DR**: See the [Example playbook set](#example-playbook-set) at the bottom of the page.

Manage a comprehensive virtual host configuration (NGINX, PHP, mariadb db + user, TLS certificates) over a site's life cycle from staging to production to decommission.

This role is designed specifically for us in high performance / high security Acro Media hosting environments, and does not aim to be compatible with cpanel, plesk, or any other low-cost or generic hosting scheme.


## Dependencies

- [acromedia.devops-utils](https://github.com/AcroMedia/ansible-role-devops-utils)
- [acromedia.postfix](https://github.com/AcroMedia/ansible-role-postfix)
- [acromedia.nginx](https://github.com/AcroMedia/ansible-role-nginx)
- [acromedia.letsencrypt](https://github.com/AcroMedia/ansible-role-letsencrypt)
- [acromedia.php](https://github.com/AcroMedia/ansible-role-php)
- [acromedia.mariadb](https://github.com/AcroMedia/ansible-role-mariadb)
- [acromedia.drupal-cron](https://github.com/AcroMedia/ansible-role-drupal-cron)


## Requirements

- All dependent software (see Dependencies above) must be installed, configured, running, and error-free.

- If you're providing your own manually registered TLS certificate(s), the fullchain, key, and intermediate files need to be placed on the server **before** this role is invoked.

- If using letsencrypt, the role will generate TLS certificates and place them for you, but you still need to know their paths, so you can feed them to the nginx_listeners variable.

- If using letsencrypt, DNS for the names in your certificate must already point at your server before you run this role.

- If using letsencrypt, port 80 must be open to the server from all public IPs. LetsEncrypt does not publish their origin addresses.

- LetsEncrypt certificates are only suitable for use on single-app-node systems. LetsEncrypt cannot be used on load balanced systems... at least, not with this role.


## Beware of Variable Bleed

It's common, especially on dev or staging servers, to require this role multiple times in the same playbook. Because of [Ansible's (lack of) Variable Scope](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#scoping-variables), multiple uses of any role in the same playbook requires close attention.

The best way to prevent variable bleed is to split up multiple uses of a role into individual plays. If you're gathering facts, doing so as it's own separate play will save playbook run time, since it doesn't normally need to be re-collected in the same playbook run.

Example:
```yaml
---
# Best practice: Split multiple uses of the same role
# into their own plays to prevent variable bleed.
- name: Gather facts in a separate play before we do any work
  hosts: app-nodes
  become: true
  gather_facts: true
  tags:
    - always
  roles: []
  tasks: []

- name: Configure virtual host A
  hosts: app-nodes
  become: true
  gather_facts: false
  roles:
    role: acromedia.virtual-host
    vars: ...

- name: Configure virtual host B
  hosts: app-nodes
  become: true
  gather_facts: false
  roles:
    role: acromedia.virtual-host
    vars: ...

```


## Required Role Variables

#### linux_owner

- The name of the user account to create. Don't use a privileged (sudoable) account, or the account of a real person. Let ansible create dedicated user accounts specifically to hold the virtual host. NGINX will be given traverse access through the specified account's home dir so it can serve the contents of the public web directory within.

#### project

- The dir name for the project inside the linux owner's /home/www/ dir. Expected to be the same as linux owner, unless the owner has more than one site or project.

#### php_version

- Major.minor version (e.g. `7.3`). The version you specify must already be running on the server. If PHP isn't on the server at all, you must specify `php_version: none`.

#### web_root_dir_name

- Can be any valid directory name - The convention is to use `web` for Drupal >= 8 sites, or `wwwroot` for < D8 or other non drupal sites.

#### web_application

- Tells the role which nginx configuration to apply. Defaults to `undefined`. Can be one of `drupal4`, `drupal5`, `drupal6`, `drupal7`, `drupal8`, `wordpress`, `php`, `static`, `proxy_pass`, or `redirect`.

#### nginx_listeners
- Specifies what ports, protocols, names to serve your site on (or how to move visitors to the right place), and the paths to your SSL certs.
- Any number of listeners may be defined for a given vhost.
- Example:
```yaml
nginx_listeners:
  - port: 80
    server_name: www.example.com
    aliases:
      - example.com
      - oldname.com
    redirect_to: https://www.example.com   # Include protocol, exclude trailing slash.
    redirect_permanent: true
  - port: 443
    ssl: true
    http2: true
    server_name: www.example.com
    aliases:
      - example.com
      - oldname.com
    ssl_fullchain_path: /etc/letsencrypt/live/www.example.com/fullchain.pem
    ssl_intermediates_path: /etc/letsencrypt/live/www.example.com/chain.pem
    ssl_key_path: /etc/letsencrypt/live/www.example.com/privkey.pem
```
  - `port`, integer, defaults to `80`
  - `ssl`, boolean, defaults to `false`.
  - `http2`, boolean, defaults to `false`, and is ignored unless `ssl` is `true`.
  - `server_name` string, always required.
  - `aliases`, optional list of strings. Exists purely for playbook readability. In the nginx template, the list of alias values are simply appended to server_name.
  - `redirect_to`, string, defaults to empty. If specified, must be a URI including the protocol and excluding trailing slash. When `redirect_to` is not empty, nginx's `$request_uri` is automatically appended to it inside the resulting nginx template. If `redirect_to` is specified, the nginx listener will push all traffic to the specified URI.
  - `redirect_permanent`, boolean, defaults to false. Controls the 301 (permanent) or 302 (temporary) code returned by nginx when redirecting traffic elsewhere.
  - `ssl_fullchain_path`, absolute path on the server to the TLS certificate + intermediates (in that order) file. Defaults to empty. Ignored unless `ssl` == `true`. If using paid or manually registered TLS certs (not generated by letsencrypt), you need to have uploaded them to the server before the role executes, or nginx will be unable to start.
  - `ssl_intermediates_path`, absolute path on the server to *just* the TLS intermediate (i.e. the full chain minus the certificate). This is used for OCSP stapling. Defaults to empty. Ignored unless `ssl` == `true`. If a path is provided, OCSP stapling will be enabled. If the value is empty or omitted, the role assumes there is no intermediate cert to use (e.g. you're using a self signed cert), and will leave the OCSP stapling part out of the NGINX config.
  - `ssl_key_path`, absolute path on the server to the TLS private key. Defaults to empty. Ignored unless `ssl` == `true`.


### Optional Role Variables

See also: **defaults/main.yml**. There are quite a few variables in there that are straightforward, and don't require documentation.

#### letsencrypt_certificates
- **letsencrypt_certificates** is an empty list `[]` by default.
- Specifies the name and list of domains on each TLS certificate that you want the role to register for you.
- Example:
```yaml
letsencrypt_certificates:
  - name: www.example.com
    domains:
      - www.example.com
      - example.com
      - oldname.com
      - www.oldname.com
```
- **name** is the directory name of the cert, as it exists (or will exist) in `/etc/letsencrypt/live`. It makes sense to keep this the same as the first domain name you include to the cert.
- **domains** is the list of domain names you want included in the certificate. All names listed in **domains** must resolve with DNS, or LE cert registration will fail.
- **name** is not implied and *MUST* be explicitly included in your list of **domains**.
- Cert and key files will be created in ``/etc/letsencrypt/live/{{ name }}/``
- Successful certificate registration creates 3 files, the paths of which you will then need to feed into the nginx_listeners config:
  - /etc/letsencrypt/live/{{ name }}/fullchain.pem - see `ssl_fullchain_path` below
  - /etc/letsencrypt/live/{{ name }}/chain.pem - see `ssl_intermediates_path` below
  - /etc/letsencrypt/live/{{ name }}/privkey.pem - see `ssl_key_path` below
- The server must be able to accept port 80 tcp from anywhere, since LetsEncrypt does not publish their origin addresses. It's fine to restrict HTTPS (port 443) traffic with authentication or firewall rules. LetsEncrypt does not need access there.
- Empty list by default

#### **responsible_person**
- Defaults to `root`. This adds a line to to postfix's /etc/aliases file. Who should receive messages from the system (usually generated by Cron) about this site? Can either be the name of a local linux user, or an email address.

#### **mysql_allow_from**
- Defaults to 'localhost'. This should really be renamed to: `mysql_allow_root_from`. You only need to set this when using multi-server setups. In your app node playbook, setting this to `{{ ansible_default_ipv4.address }}` should usually work, assuming both app node(s) and mysql host are both on the same private network. In order for this to work, your app node needs to be able to operate as mysql root, with crentials stored at /root/.my.cnf.

#### **mysql_host_address**
- Defaults to 'localhost'. You only need to set this when using multi-server setups. If your app node(s) and mysql host are both on the same private network (they usually will be), set this to be your mysql host's private / internal IP address. In order for this to work, your app node needs to be able to operate as mysql root, with crentials stored at /root/.my.cnf.

#### **rds**
- Boolean, false by default. Set this to true if you're creating a site backed by an amazon RDS instance.

#### **nginx_proxy_pass_blob**
- When web_application == 'proxy_pass', this gets placed as is, directly into to the nginx default `location / {}` directive. When using proxy_pass, all other directives except those related to security (ie those that immediately return a 403) get disabled, as they are expected to be handled by your upstream / proxied application.

#### nginx_ip_restricted_locations
- Empty list by default 
- Make sure to TEST your restrictions after you put them in place. Nginx locations can be slippery creatures.
```yaml
# Example 1: Lock down administrative locations to specific networks
nginx_ip_restricted_locations:
  - /admin
  - '= /login.php'
nginx_allowed_ips:
  - 1.2.3.4
  - 4.3.2.0/27
```
#### nginx_allowed_ips
- Empty list by default. Only used in conjunction with [nginx_ip_restricted_locations](#nginx_ip_restricted_locations)


#### **nginx_trusted_cidrs**
- Empty list by default
- When `web_application` == `*drupal*`, access to `update.php`, `install.php`, or other sensitive files is denied. If you want a client to be allowed to talk to these locations, specify the IP address or network(s) that can do that:
```yaml
nginx_trusted_cidrs:
  - 8.8.8.8  #  A single IP address
  - 192.168.0.0/16 # I can do this if I'm coming from my private intranet.
```

#### **nginx_location_extras**
- Allows you to specify extra `location` directives without having to supply an additional file. Exmaple:
```yaml
nginx_location_extras:
  - name: Don't allow php to run from this folder
    location: ~ /foo/bar/.*\.php$
    config: return 403;
  - name: Don't log requests for this folder
    location: /baz/buz
    config: |
      log_not_found off;
      access_log off;
```

#### **rewrite_target**
- Defaults to `/index.php`. If your site is static, specify `/index.html` or `/index.htm`. If using D6, specify `/index.php?q=$1`. If you have another file you want your pretty urls to be rewritten to, change this to whatever your main index file is.

#### **image_cache_location**
- Specific to Drupal. Defaults to `/sites/.*/files/styles/` which works for >= D7. If using D6, specify whatever your image cache dir is (usually `/sites/.*/files/imagecache/` )

#### **nginx_drupal_uploads_dir_pattern**
- Used for preventing PHP execution from within sites/default/files directories. Defaults to `'/sites/.*/files'`. No need to change it unless your site puts files in a weird place, or is a very old version of drupal.

#### **nginx_include_custom**
- Path to a template file in your playbook that will be uploaded to the server, and then `include`d before the start of the nginx 'location' directives for the virtual host. You may use any variables in your template that are avaialble to the role. Your template will be processed and uploaded to the server as `/etc/nginx/includes/{{ linux_owner }}-{{ project }}.custom.conf`.

#### **nginx_include_resident**
- Absolute path to a file that already resides on the server (for example, one that was deployed by your Drupal project's code repository inside your web root), to be `include`d before the location directives in the virtual host. **Caveats:**  **(1)** The file you specify *MUST* already exist on the server, or your nginx config will break. **(2)** Modifications to your resident include file do not trigger an nginx reload, since the role has no way of knowing when your file changed. It will be up to you to manually reload nginx if/when needed.

#### **nginx_inline_custom**
- Whatever you specify is placed as-is, inside the virtual host's main `server` block, before any location directives.

#### require_http_auth
- Boolean, false by default
- Useful if you want to keep google's prying eyes out of your staging environment.
- When `require_http_auth` is `true` (it's `false` by default), the server will prompt end users for credentials defined by `http_auth_username` and `http_auth_password`
  ```yaml
  require_http_auth: yes
  http_auth_username: staging123
  http_auth_password: 'correct horse battery staple'
  ```
#### http_auth_username
- String. See [require_http_auth](#require_http_auth)

#### http_auth_password
- String. See [require_http_auth](#require_http_auth)

#### **max_execution_time_seconds**
- Defaults to 300. This meta variable controls 3 individual PHP and NGINX config values. See defaults/main.yml for the individual variable names if you need more fine grained control.

#### **max_upload_size_mb**
- integer 
- Defaults to 8. 
- This meta variable controls 3 individual PHP and NGINX config values. See defaults/main.yml for the individual variable names if you need more fine grained control.

#### **php_memory_limit**
- Defaults to `128M`. 
- Accepts whatever you would normally place in php.ini. Don't forget the "M" at the end.

#### **php_sendmail_path**
- In some rare cases, you may need to force the "from" address on all outgoing mail. PHP also has a setting for `sendmail_from`, but it seems to have no effect. Be careful to test after setting this. Example:
```yaml
php_sendmail_path: '/usr/sbin/sendmail -t -i -f foo@example.com'
```

#### **skip_mysql**
- Boolean 
- Lets the role be used when MySQL isn't available at all. See also: `php_version: none`.

#### **disable_http_auth**
- Boolean, defaults to false. On staging/dev servers, the global nginx config may be imposing password authentication. Let it be disabled per-virtual host.

#### **disable_environment_indicator**
- Boolean, Defaults to false. On staging/dev servers, the global nginx config may be imposing password authentication. Let it be disabled per-virtual host.

#### **letsencrypt_expiry_email**
- (email address): Defaults to nothing. It's highly recommended that you set this if using letsencrypt. LE SSL certs expire *very* quickly, and if is going sideways with cert renewals, you will want to know about it before it affects your users.


## NGINX/PHP Rate Limiting

Per-IP rate limiting is on by default, but set quite high. For most sites, it shouldn't actually kick in and will need to be configured.

Rate limiting only affects PHP FPM requests. Images, css, scripts (anything that's not generated by PHP) are not limited.

The variables to tune are:

#### **php_rate_limit_max_requests_per_second**
- (integer): Defaults to 10. For Drupal sites, even going down as low as 1 will prevent abuse without interfering with legitimate traffic.

#### **php_rate_limit_burst_limit**
- (integer): Defaults to 100. Bring this down to somwhere between 20 and 40 to prevent abuse  but not interfere with pages that have lots of style/image/script resources.

#### **nginx_limit_req_zone_key**
- (string): Defaults to "binary_remote_addr", which is only good for sites directly serving traffic. If your server is behind a proxy or load balancer, change this to `http_x_forwarded_for` instead, and make sure the X-Forwarded-For header is being sent to your server. If the header is not present, rate limiting will not happen.


## What happened to ...

The old combination of `nginx_canonical_name` + `nginx_aliases` + `ssl` + `deploy_env`, as well as a very long list of kludge variables, have been replaced by `nginx_listeners` and `letsencrypt_certificates` definitions as of the role's 2.x version.

While old set of variables resulted in a simpler looking playbook, that setup resulted in unworkable limitations in too many edge cases, and continuously spawned workarounds and code smells. As well, the role's nginx templates were becoming complicated logical nightmares.

The new variables eliminate all of the template guesswork that the role used to do, by leaving the traffic routing decisions to the human writing the playbook.

See [how-to-upgrade-from-version-1.x.md](how-to-upgrade-from-version-1.x.md) for an example variable conversion.

## Example playbook set

inventories/production/hosts:
```ini
[app_nodes]
bigcorp-prod-app.hosting-company.net
```

inventories/production/group_vars/all.yml:
```yaml
---
linux_owner: bigcorp
project: bigcorp
php_version: 8.1
web_root_dir_name: web
web_application: drupal8
prod_server_primary_name: www.bigcorp.net
prod_server_secondary_names:
  - bigcorp.net
  - oldname.com
  - www.oldname.com
letsencrypt_certificates:  
  - name: "{{ prod_server_primary_name }}"
    domains: "{{ [ prod_server_primary_name ] + prod_server_secondary_names }}"  # All names must resolve, and the server must accept public connections on port 80 for letsencrypt cert registration to be successful.
nginx_listeners:
  - desc: Push all plain-text traffic from all names over to the proper name on SSL
    port: 80
    server_name: "{{ prod_server_primary_name }}"
    aliases: "{{ prod_server_secondary_names }}"
    redirect_to: https://{{ prod_server_primary_name }}
    redirect_permanent: true
  - desc: Push traffic using non-canonical names on SSL over to the proper name
    port: 443
    ssl: true
    http2: true
    server_name: "{{ prod_server_secondary_names | join(' ') }}"
    ssl_fullchain_path: /etc/letsencrypt/live/{{ prod_server_primary_name }}/fullchain.pem
    ssl_key_path: /etc/letsencrypt/live/{{ prod_server_primary_name }}/privkey.pem
    ssl_intermediates_path: /etc/letsencrypt/live/{{ prod_server_primary_name }}/chain.pem
    redirect_to: https://{{ prod_server_primary_name }}
    redirect_permanent: true
  - desc: Serve the site on SSL using a single canonical name.
    port: 443
    ssl: true
    http2: true
    server_name: "{{ prod_server_primary_name }}"
    ssl_fullchain_path: /etc/letsencrypt/live/{{ prod_server_primary_name }}/fullchain.pem
    ssl_key_path: /etc/letsencrypt/live/{{ prod_server_primary_name }}/privkey.pem
    ssl_intermediates_path: /etc/letsencrypt/live/{{ prod_server_primary_name }}/chain.pem

drupal_cron_url: "https://{{ prod_server_primary_name }}/cron/abcdefgh12345678"
```

inventories/staging/hosts:
```ini
[app_nodes]
bigcorp-stg-app.hosting-company.net
```

inventories/staging/group_vars/all.yml:
```yaml
---
linux_owner: bigcorp
project: bigcorp
php_version: 8.1
web_root_dir_name: web
web_application: drupal8
staging_server_dns_name: stg.bigcorp.net
letsencrypt_certificates:      # DNS must resolve, port 80 must be open to the public.
  - name: "{{ staging_server_dns_name }}"
    domains:
      - "{{ staging_server_dns_name }}"
nginx_listeners:
  - desc: Redirect to SSL
    port: 80
    server_name: "{{ staging_server_dns_name }}"
    redirect_to: https://{{ staging_server_dns_name }}   # Include protocol, exclude trailing slash.
    redirect_permanent: true
  - desc: Serve with letsencrypt cert
    ssl: true
    port: 443
    http2: true
    server_name: "{{ staging_server_dns_name }}"
    ssl_fullchain_path: /etc/letsencrypt/live/{{ staging_server_dns_name }}/fullchain.pem
    ssl_key_path: /etc/letsencrypt/live/{{ staging_server_dns_name }}/privkey.pem
    ssl_intermediates_path: /etc/letsencrypt/live/{{ staging_server_dns_name }}/chain.pem
```

playbooks/main.yml:
```yaml
---
- hosts: app_nodes
  gather_facts: true
  become: true
  roles:
    - role: acromedia.devops-utils
    - role: acromedia.postfix
    - role: acromedia.mariadb
    - role: acromedia.nginx
    - role: acromedia.php
    - role: acromedia.letsencrypt
    - role: acromedia.virtual-host
    - role: acromedia.drupal-cron

```

To run the playbooks:
```bash
ansible-playbook playbooks/main.yml -i inventories/staging
ansible-playbook playbooks/main.yml -i inventories/production
```


## License

GPLv3

## Author Information

Acro Media Inc.
