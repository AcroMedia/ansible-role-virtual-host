# How to upgrade from 1.x

Role version 1.x example playbook variables:
```yaml
# ----------------------------------------------------------
# These stay the same
# ----------------------------------------------------------
linux_owner: mysite
project: mysite
php_version: 7.4
web_root_dir_name: web
web_application: drupal8
# ----------------------------------------------------------
# The following variables no longer exist in 2.x
# ----------------------------------------------------------
nginx_primal_name: mysite.server.provider.com
nginx_canonical_name: mysite.server.provider.com
nginx_aliases: []
deploy_env: staging
ssl: letsencrypt
redirect_code: 301
```

Equivalent version 2.x of the above:
```yaml
# ----------------------------------------------------------
# These stay the same
# ----------------------------------------------------------
linux_owner: mysite
project: mysite
php_version: 7.4
web_root_dir_name: web
web_application: drupal8
# ----------------------------------------------------------
# The following replaces all of nginx_primal_name, nginx_canonical_name,
# nginx_aliases, deploy_env, ssl, and redirect_code
# ----------------------------------------------------------
letsencrypt_certificates:
  - name: mysite.server.provider.com
    domains:
      - mysite.server.provider.com
nginx_listeners:
  - port: 80
    server_name: mysite.server.provider.com
    redirect_to: https://mysite.server.provider.com   # Include protocol, exclude trailing slash.
    aliases: []
    redirect_permanent: true
  - port: 443
    ssl: true
    http2: true
    server_name: mysite.server.provider.com
    aliases: []
    ssl_fullchain_path: /etc/letsencrypt/live/mysite.server.provider.com/fullchain.pem
    ssl_key_path: /etc/letsencrypt/live/mysite.server.provider.com/privkey.pem
    ssl_intermediates_path: /etc/letsencrypt/live/mysite.server.provider.com/chain.pem
```
