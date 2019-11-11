# Ansible role: Virtual Host

Manage a NGINX virtual host (including PHP version and SSL certificates) over its life cycle from staging to production.

## Requirements

- The acro infrastructure utilities must already be installed on the server

- Acro-add-website.sh must already be installed (and configured) on the server

- LetsEncrypt installed, configured, and working

- If you're providing your own manually registered SSL certificate, the certificate + key need to be placed on the server **before** this role is invoked.

- Since this role is designed to support a "normal" virtual host from staging to production using SSL, both **nginx_primal_name**, and **nginx_canonical_name** are required in order for the role to work.

- If using letsencrypt for SSL, **nginx_primal_name** should **not** be changed, once the original certificate has been registered, since this name refers to the physical directory of the SSL cert files in the /etc/letsencrypt live directory.

- If using SSL, DNS for your nginx_primal_name must already point at your server before you run this role. The other two (nginx_canonical_name and nginx_aliases) don't need to resolve until after you switch to `production`.

- Don't be tempted to set `deploy_env` to `production` until *after* your site / server has been completely configured and tested and is truly ready for go-live.

## Beware of Variable Bleed

It's common to require using this role multiple times in the same playbook. Becaue of [Ansible Variable Scope](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#scoping-variables), multiple uses of any role in the same playbook requires extra care.

The best way to prevent variable bleeed is to split up multiple uses of the role into indvidual plays. As long as you're not gathering facts in each play, this doesn't add any extra time to the playbook's run time. It also lets you keep things more organized, especially when your virtual host has a lot of customization. For example:
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

If using the role more than once **in the same play**, you must re-specify **all** role variables that were used for each instance of the role. Role defaults are not re-set between role uses from the same play. Every variable, including role defaults, "stick" to the value they were set to between role instances. A variable that was defined in the first instance retains its value for the next instance, so if you don't re-specify it, even if it has no bearing on your second role instance, mayhem can ensue:

```yaml
---
# Avoid this situation.
- hosts: app-nodes
  become: true
  gather_facts: true
  roles:
    - name: Configure vhost A
      role: acromedia.virtual-host
      vars:
        php_version: 5.6
        web_application: drupal6
        ...
    - name: Configure vhost B
      role: acromedia.virtual-host
      vars:
        web_application: drupal8
        # Missing the specification for "php_version" here
        # makes this instance of the role inherit the last
        # isntance's value of "5.6", which breaks
        # this virtual host.
        ...
```


## Required Role Variables

#### linux_owner

- The name of the user account to create

#### project

- The dir name for the project inside the linux owner's home dir. Usually should be the same as linux owner, unless the owner has more than one site or project.

#### nginx_primal_name

- Example:
    ```
    project.server.acro.website
    ```
    or:
    ```
    www.client.acro.website
    ```
    if there is only one site on a given server.

- Its purpose is to be a DNS name that can be used to access the virtual host before (and after) the site's canonical DNS name points to the virtual host, so your team (and/or the client) can get be sure everything works (with valid SSL), before the site's real DNS name is pointed at the server.

- When using letsencrypt, **do not** change or remove the value of `nginx_primal_name` after the playbook has run against your server**. SSL directory names created by the role are tied to the name you create. If you change it or remove this name, you will break your NGINX and LetsEncrypt config, and will be left with a nasty mess to clean up.

- Not optional, and must be unique from the other nginx virtual host names on the server.



#### nginx_canonical_name

- Example:
    ```
    www.bigcorp.com
    ```

-  Once a site is launched, the final destination name for the site.

- Not optional, and must be unique from the other nginx virtual host names on the server.

- When using LetsEncrypt, the nginx_canonical_name will be appended to the SSL certificate (that was registered using the nginx_primal_name) after you set deploy_env to 'production'.


#### nginx_aliases

- A list of any other virtual host names (besides the primal & canonical names) that the site should respond to, and redirect to the canoncial name.  Examples:
    ```yaml
    # Single non-www alias:
    nginx_aliases:
      - bigcorp.com

    # Non-www plus a second domain name:
    nginx_aliases:
      - bigcorp.com
      - www.othername.com
      - othername.com

    # N aliases: specify an empty list:
    nginx_aliases: []
  ```
- Must be a list; regardless of how many aliases there are.

- nginx_aliases always redirect to nginx_canonical_name.

- When using LetsEncrypt , all nginx_aliases will be appended to the SSL certificate (that was registered using the nginx_primal_name) after you set deploy_env to 'production'.


#### php_version

- Major.minor version (e.g. `7.1`). The version you specify must already be running on the server. If PHP isn't on the server at all, you must specify `php_version: none`.

#### web_root_dir_name

- Can be any valid directory name - The convention is to use `web` for Drupal >= 8 sites, or `wwwroot` for < D8 or other non drupal sites.

#### web_application

- Tells the role which nginx configuration to apply. Defaults to `undefined`. Can be one of `drupal4`, `drupal5`, `drupal6`, `drupal7`, `drupal8`, `wordpress`, 'php', `static`, `proxy_pass`, or `redirect`.

#### deploy_env

- Can be one of `staging` or `production`. If you're using this role in a development environment, just specify `staging`. This variable does not have a default, since having the role guess could have negative consequences.

- Use `staging` before DNS for the site's canoncial domain name points at your server

- Switch to `production` (and then re-run the playbook) after you've pointed DNS for your nginx_canonical_name (and aliases) at your server

- When set to `production` and `ssl` is set to `letsencrypt`, the scripts will attempt to add the nginx_canonical_name and nginx_aliases to the LetsEncrypt SSL certificate.

#### ssl

- Can be one of `letsencrypt`, `manual`, or `none`. This variable does not have a default, since having the role guess could have negative consequences.

- Specify `letsencrypt` when you don't have a manually registered SSL cert to install.

- Specify `none` when it's not possible or practical to use a SSL certificate, such as on a private network, for local development, or when using proxy_pass behind varnish or a load balanacer. Production without SSL is not recommended, so if you're specifying 'none', it's expected that you're taking care of SSL in some other way.

- Specify `manual` when you're providing a manually registered SSL certificate. **This role does not place your SSL certificate on the server; you'll need to do that BEFORE you run this playbook**.

- When `deploy_env` is `staging`, you'll need to adjust your own local /etc/hosts file in order to see your staging site using the canonical domain name, if DNS is not pointing at the server.

- **Warning**: When using `manual`, the certificate you provide must work for all of the `nginx_canonical_name` and `nginx_aliases` you specify. Providing mis-matched names *will* result in SSL errors in browsers, and *may* break NGINX configuraiton, preventing the service from starting.

## How the combination of `deploy_env` + `ssl` affects redirects

In `staging` modes, the `nginx_primal_name` never redirects to the canonical name, because it's expected that DNS is not pointing at the server yet.

##### staging + no ssl

  - **nginx_primal_name:80**:

    Serve content.

  - **nginx_aliases:80**:

    Redirect to nginx_canonical_name:80. Assume the user had to adjust /etc/hosts to make requests with this name.

  - **nginx_canonical_name:80**:

    Serve content. Assume the user had to adjust /etc/hosts to make requests with this name.


##### staging + letsencrypt ssl:

  -  **nginx_primal_name:80**

    Redirect to nginx_primal_name:443. Assume primal name is under our control, so we can always use SSL for this name.

  -  **nginx_aliases:80**

    If aliases are present, redirect to nginx_canonical_name:80 if nginx_canonical_name and nginx_primal_name are not the same, otherwise redirect to nginx_primal_name:443. We redirect to http for canonical instead of https, since going to https with the canonical would produce SSL warnings.

  - **nginx_canonical_name:80**

    Only used if nginx_canonical_name is different than nginx_primal_name. Serve content instead of redirecting to https, since the cert won't be valid until production mode is enabled. This particular case  has the convenient side effect of serving older sites immediately as soon as DNS switches, instead of pushing them to https where SSL won't be valid yet.

  - **nginx_primal_name:443**

    Serve content instead of redirecting to canonical name. It wouldn't make sense to direct to the Canonical, since DNS doesn't point at the server yet, and trying to use the canonical name would produce invalid SSL errors.

  - **nginx_aliases:443**

    If aliases exist, redirect them to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name. Expect to see Invalid SSL errors for all aliases, since they won't be part of the LetsEncrypt certificate deploy_env is switched to production.

  - **nginx_canonical_name:443**

    Only used if nginx_canonical_name != nginx_primal_name. Serve content with an invalid certificate. Assume the user had to adjust /etc/hosts to make requests with this name.


##### staging + manual ssl:

  -  **nginx_primal_name:80**

    Redirect to nginx_primal_name:443. Assume primal name is under our control, so we can always use SSL for this name.

  - **nginx_aliases:80**

    Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.

  - **nginx_canonical_name:80**

    Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.

  - **nginx_primal_name:443**

    Serve content instead of redirecting to canonical name. Canonical name can't be used since DNS doesn't point here yet.

  -  **nginx_aliases:443**

    Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.

  - **nginx_canonical_name:443**

    Serve content. Assume the user had to adjust /etc/hosts to make requests with this name.


##### production + no ssl:

  - **nginx_primal_name:80**

    Redirect to nginx_canonical_name:80.

  - **nginx_aliases:80**

    Redirect to nginx_canonical_name:80.

  - **nginx_canonical_name:80**

    Serve content.


#####  production + (manual or letsencrypt) ssl:

  - **nginx_primal_name:80**

    Redirect to nginx_canonical_name:443

  - **nginx_aliases:80**

    Redirect to nginx_canonical_name:443

  - **nginx_canonical_name:80**

    Redirect to nginx_canonical_name:443

  - **nginx_primal_name:443**

    Redirect to nginx_canonical_name:443

  - **nginx_aliases:443**

    Redirect to nginx_canonical_name:443

  - **nginx_canonical_name:443**

    Serve content. This is the final destination for all names.



#### When providing manually registered / paid SSL certificates

These files need to be placed on the server **before** you run the playbook, or your nginx configuration will fail, and nginx will likely be unable to start.

**ssl_fullchain_path**: For nginx. Needs to include both the site's certificate and the CA intermediates. E.g:  `/usr/local/ssl/certs/mydomain.com.fullchain.pem`

**ssl_key_path**: E.g: `/usr/local/ssl/private/mydomain.com.key`

**ssl_intermediates_path**: E.g: `/usr/local/ssl/certs/mydomain.com.intermediates.pem`

#### When web_application == 'proxy_pass'

**nginx_proxy_pass_blob**: This gets placed as is, directly into to the nginx default `location / {}` directive.

When using proxy_pass, all other diretives except those related to secruity (ie those that immediately return a 403) get disabled, as they are expected to be handled by your upstream / proxied application.

### Optional Role Variables

**See defaults/main.yml** - there are quite a few variables in there that are straightforward, and don't require documentation.

**redirect_code**: Defaults to `302` (temporary redirect). After your site has launched and working without any issues, change this to `301` and run your playbook again to make redirects [to the canoncial name] permanent.

**responsible_person**: Defaults to `root`. This adds a line to to postfix's /etc/aliases file. Who should receive messages from the system (usually generated by Cron) about this site? Can either be the name of a local linux user, or an email address.

**drupal_cron_url**: After your site goes live, (or before, if you want it to run during staging), provide this variable to create a linux cron job on the server. The cron job is created by the [acromedia.drupal-cron](https://github.com/AcroMedia/ansible-role-drupal-cron) role. See its readme for more details.

**mysql_allow_from**: This should really be renamed to: `mysql_allow_root_from`. Defaults to 'localhost'. You only need to set this when using multi-server setups. In your app node playbook, setting this to `{{ ansible_default_ipv4.address }}` should usually work, assuming both app node(s) and mysql host are both on the same private network. In order for this to work, your app node needs to be able to operate as mysql root, with crentials stored at /root/.my.cnf.

**mysql_host_address**: Defaults to 'localhost'. You only need to set this when using multi-server setups. If your app node(s) and mysql host are both on the same private network (they usually will be), set this to be your mysql host's private / internal IP address. In order for this to work, your app node needs to be able to operate as mysql root, with crentials stored at /root/.my.cnf.

**rds** (boolean): Set this to true if you're creating a site backed by an amazon RDS instance.

**nginx_ip_restricted_locations** + **nginx_allowed_ips** (lists): Make sure to TEST your restrictions after you put them in place. Nginx locations can be slippery creatures.
```yaml
# Example 1: Lock down administrative locations to specific networks
nginx_ip_restricted_locations:
  - /admin
  - '= /login.php'
nginx_allowed_ips:
  - 1.2.3.4
  - 4.3.2.0/27
```

**nginx_trusted_cidrs** (list): When `web_application` == `*drupal*`, access to `update.php`, `install.php`, or other sensitive files is denied. If you want a client to be allowed to talk to these locations, specify the IP address or network(s) that can do that:
```yaml
nginx_trusted_cidrs:
  - 8.8.8.8  #  A single IP address
  - 192.168.0.0/16 # I can do this if I'm coming from my private intranet.
```

**nginx_location_extras[{name,location,config}]**: Allows you to specify extra `location` directives without having to supply an additional file. Exmaple:
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

**rewrite_target**: Defaults to `/index.php`. If your site is static, specify `/index.html` or `/index.htm`. If using D6, specify `/index.php?q=$1`. If you have another file you want your pretty urls to be rewritten to, change this to whatever your main index file is.

**image_cache_location**: "Location" in the context of Nginx location regex pattern. Defaults to `/sites/.*/files/styles/` which works for >= D7. If using D6, specify whatever your image cache dir is (usually `/sites/.*/files/imagecache/` )

**nginx_drupal_uploads_dir_pattern**: Used for preventing PHP execution from within sites/default/files directories. Defaults to `'/sites/.*/files'`. No need to change it unless your site puts files in a weird place, or is a very old version of drupal.

**http_port**, **https_port**: Default to 80 and 443 respectively. Caveat: If you're changing this, you'll likely also need to change the server-wide default port(s), which is not handled by this role.

**nginx_include_custom**: Path to a template file in your playbook that will be uploaded to the server, and then `include`d before the start of the nginx 'location' directives for the virtual host. You may use any variables in your template that are avaialble to the role. Your template will be processed and uploaded to the server as `/etc/nginx/includes/{{ linux_owner }}-{{ project }}.custom.conf`.

**nginx_include_resident**: Absolute path to a file that already resides on the server (for example, one that was deployed by your Drupal project's code repository inside your web root), to be `include`d before the location directives in the virtual host. **Caveats:**  **(1)** The path you specify *MUST* already exist on the server, or your nginx config will break. **(2)** Future changes to your included file will not be automatically applied, since the role has no way of knowing when this file changed, and no means to intelligently trigger a reload. It will be up to you to manually reload nginx when needed.

**nginx_inline_custom**: Whatever you specify is placed as-is, inside the virtual host's main `server` block, before any location directives.

**require_http_auth** (boolean): Useful if you want to keep google's prying eyes out of your staging environment. When true, will prompt the user for the values specified by **http_auth_username** and **http_auth_password**.

**max_execution_time_seconds**: Defaults to 300. This meta variable controls 3 individual PHP and NGINX config values. See defaults/main.yml for the individual variable names if you need more fine grained control.

**max_upload_size_mb**: Defaults to 8. This meta variable controls 3 individual PHP and NGINX config values. See defaults/main.yml for the individual variable names if you need more fine grained control.

**php_memory_limit**: Defaults to `128M`. Accepts whatever you would normally place in php.ini. Don't forget the "M" at the end.

**php_sendmail_path**: In some rare cases, you may need to force the "from" address on all outgoing mail. PHP also has a setting for `sendmail_from`, but it seems to have no effect. Be careful to test after setting this. Example:
```yaml
php_sendmail_path: '/usr/sbin/sendmail -t -i -f foo@example.com'
```

**skip_mysql**: Lets the role be used when MySQL isn't available at all. See also: `php_version: none`.

**disable_http_auth** (boolean): Defaults to false. On staging/dev servers, the global nginx config may be imposing password authentication. Let it be disabled per-virtual host.

**disable_environment_indicator** (boolean): Defaults to false. On staging/dev servers, the global nginx config may be imposing password authentication. Let it be disabled per-virtual host.


## NGINX/PHP Rate Limiting

Per-IP rate limiting is on by default, but set quite high. For most sites, it shouldn't actually kick in and will need to be configured.

Rate limiting only affects PHP FPM requests. Images, css, scripts (anything that's not generated by PHP) are not limited.

The variables to tune are:

- **php_rate_limit_max_requests_per_second** (integer): Defaults to 10. For Drupal sites, even going down as low as 1 will prevent abuse without interfering with legitimate traffic.

- **php_rate_limit_burst_limit** (integer): Defaults to 100. Bring this down to somwhere between 20 and 40 to prevent abuse  but not interfere with pages that have lots of style/image/script resources.

- **nginx_limit_req_zone_key** (string): Defaults to "binary_remote_addr", which is only good for sites directly serving traffic. If your server is behind a proxy or load balancer, change this to `http_x_forwarded_for` instead, and make sure the X-Forwarded-For header is being sent to your server. If the header is not present, rate limiting will not happen.


## Dependencies

- [acromedia.drupal-cron](https://github.com/AcroMedia/ansible-role-drupal-cron)
- [acromedia.postfix](https://github.com/AcroMedia/ansible-role-postfix)
- [acromedia.mariadb](https://github.com/AcroMedia/ansible-role-mariadb)


## Example Playbook

```yaml
- hosts: servers
  roles:
    - name: Configure the nginx virtual host for bigcorp
      role: acromedia.virtual-host
      vars:
        linux_owner: bigcorp
        project: bigcorp
        nginx_primal_name: www.bigcorp.acro.website
        nginx_canonical_name: www.bigcorp.com
        nginx_aliases:
          - bigcorp.com
        php_version: 7.1
        web_root_dir_name: web
        deploy_env: staging
        ssl: letsencrypt
        redirect_code: 302
        web_application: drupal8
        ## Uncomment and update the next line after golive:
        # drupal_cron_url: "https://{{ nginx_canonical_name }}/cron/abcdefgh12345678"
        ## Got extra stuff you need to configure for nginx? Give it to the role to upload for you:
        # nginx_include_custom: path/to/my_arbitrary_nginx_additions.conf.j2
```

## License

For Acro Media internal use only.

## Author Information

Acro Media Inc.
