# Ansible role: Virtual Host

Basically an ansible wrapper for acro-add-website.sh, geared for sites using LetsEncrypt certificates.

## Requirements

- The acro infrastructure utilities must already be installed on the server
- Acro-add-website.sh must already be installed AND configured on the server
- LetsEncrypt installed, configured, and working
- **If you're providing your own manually registered SSL certificate, the certificate + key need to be placed on the server BEFORE this role is invoked.**

## Role Variables

See also: defaults/main.yml

#### Required Variables

**linux_owner**: The name of the user account to create

**project**: The dir name for the project inside the linux owner's home dir. Usually should be the same as linux owner, unless the owner has more than one site or project.

**nginx_primal_name**: This is the initial / internal name of the site so it can be staged ... e.g `project.host.example.com`

**nginx_canonical_name**: Once a site is launched, the final destination name for the site. E.g. `www.bigcorp.com`

**nginx_aliases**: Once a site is launched, a list of any other virtual host names (besides the primal name) that the site should respond to, and redirect to the canoncial name.  At the time of this writing, at least one alias is required, or configuration will break. E.g. example.com should redirect to www.example.com, or vice versa.

**php_version**: Major.minor version (e.g. `7.1`). The version you specify  must already be running on the server.

**web_root_dir_name**: Can be anything - The convention is to use `web` for Drupal >= 8 sites, or `wwwroot` for < D8 or other non drupal sites.

**web_application**:  Tells the role which nginx configuration to apply. Defaults to `drupal8`. Can be one of `drupal6`, `drupal7`, `drupal8`, `wordpress`, or `static`.

**deploy_env**: Can be one of `staging` or `production`. If you're using this role in a development environment, just specify `staging`. This variable does not have a default, since having the role guess could have negative consequences.
- Use `staging-*` before DNS for the site's canoncial domain name points at your server
- Switch to one of the `production-*` modes (and then re-run the playbook) after you've pointed DNS at your server
- When set to `production` and `ssl` is `letsencrypt`, the scripts will attempt to add the site's canonical name and nginx_aliases to the LetsEncrypt SSL certificate.
`
**ssl**: Can be one of `letsencrypt`, `manual`, or `none`. This variable does not have a default, since having the role guess could have negative consequences.
- Only specify `none` when it's not possible or practical to use a SSL certificate, such as on a private network or for local development. When `deploy_env` is `production`, trying to use the `none` option will break your configuration. Production without SSL is not supported.
- Specify `letsencrypt` if you don't have a manually registered SSL cert to install
- Specify `manual` when you're providing a manually registered SSL certificate. **This role does not place your SSL certificate on the server; you'll need to do that BEFORE you run this playbook**.
- When `deploy_env` is `staging`, you'll need to adjust your /etc/hosts file in order to see your staging site using the canonical domain name.
- **Warning**: When using `manual`, the certificate you provide must work for all of the `nginx_canonical_name` and `nginx_aliases` you specify. Providing mis-matched names *will* result in SSL errors in browsers, and *may* break NGINX configuraiton, preventing the service from starting.

**How the combination of `deploy_env` + `ssl` affects redirects:**
  ```
  staging + no ssl:
    nginx_primal_name:80     -> Serve content.
    nginx_aliases:80         -> Redirect to nginx_canonical_name:80. Assume the user had to adjust /etc/hosts to make requests with this name.
    nginx_canonical_name:80  -> Serve content. Assume the user had to adjust /etc/hosts to make requests with this name.

  staging + letsencrypt ssl:
    nginx_primal_name:80     -> Redirect to nginx_primal_name:443. Assume primal name is under our control, so we can always use SSL for this name.
    nginx_aliases:80         -> Redirect to nginx_canonical_name:80 (http instead of https), since going to https would produce SSL warnings.
    nginx_canonical_name:80  -> Serve content instead of redirecting to https, since the cert won't be valid until production mode is enabled. This particular case has the convenient side effect of serving older sites immediately as soon as DNS switches, instead of pushing them to https where SSL won't be valid yet.
    nginx_primal_name:443    -> Serve content instead of redirecting to canonical name. Canonical name can't be used since DNS doesn't point here yet.
    nginx_aliases:443        -> Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.
    nginx_canonical_name:443 -> Serve content with an invalid certificate. Assume the user had to adjust /etc/hosts to make requests with this name.

  staging + manual ssl:
    nginx_primal_name:80     -> Redirect to nginx_primal_name:443. Assume primal name is under our control, so we can always use SSL for this name.
    nginx_aliases:80         -> Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.
    nginx_canonical_name:80  -> Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.
    nginx_primal_name:443    -> Serve content instead of redirecting to canonical name. Canonical name can't be used since DNS doesn't point here yet.
    nginx_aliases:443        -> Redirect to nginx_canonical_name:443. Assume the user had to adjust /etc/hosts to make requests with this name.
    nginx_canonical_name:443 -> Serve content. Assume the user had to adjust /etc/hosts to make requests with this name.

  production + <any value for ssl>:
    nginx_primal_name:80      -> Redirect to nginx_canonical_name:443
    nginx_aliases:80          -> Redirect to nginx_canonical_name:443
    nginx_canonical_name:80   -> Redirect to nginx_canonical_name:443
    nginx_primal_name:443     -> Redirect to nginx_canonical_name:443
    nginx_aliases:443         -> Redirect to nginx_canonical_name:443
    nginx_canonical_name:443  -> Serve content. This is the final destination for all names.
  ```
In `staging` modes, the `nginx_primal_name` never redirects to the canonical name, because it's expected that DNS is not pointing at the server yet.

#### When providing manually registered / paid SSL certificates

These files need to be placed on the server **before** you run the playbook, or your nginx configuration will fail, and nginx will likely be unable to start.

**ssl_fullchain_path**: For nginx. Needs to include both the site's certificate and the CA intermediates. E.g:  `/usr/local/ssl/certs/mydomain.com.fullchain.pem`

**ssl_key_path**: E.g: `/usr/local/ssl/private/mydomain.com.key`

**ssl_intermediates_path**: E.g: `/usr/local/ssl/certs/mydomain.com.intermediates.pem`


#### Optional variables

**redirect_code**: Defaults to `302` (temporary redirect). After your site has launched and working without any issues, change this to `301` and run your playbook again to make redirects [to the canoncial name] permanent.

**responsible_person**: Defaults to `root`. This adds a line to to postfix's /etc/aliases file. Who should receive messages from the system (usually generated by Cron) about this site? Can either be the name of a local linux user, or an email address.

**drupal_cron_url**: After your site goes live, (or before, if you want it to run during staging), provide this variable to create a linux cron job on the server. The cron job is created by the [acromedia.drupal-cron](https://github.com/AcroMedia/ansible-role-drupal-cron) role. See its readme for more details.


#### Tweaks / overrides

If web_application doesn't do everything you need, the following tweaks can help.

**rewrite_target**: Defaults to `/index.php`. If your site is static, specify `/index.html` or `/index.htm`. If using D6, specify `/index.php?q=$1`. If you have another file you want your pretty urls to be rewritten to, change this to whatever your main index file is.

**image_cache_location**: "Location" in the context of Nginx location regex pattern. Defaults to `/sites/.*/files/styles/` which works for >= D7. If using D6, specify whatever your image cache dir is (usually `/sites/.*/files/imagecache/` )

**nginx_drupal_uploads_dir_pattern**: Used for preventing PHP execution from within sites/default/files directories. Defaults to `'/sites/.*/files'`. No need to change it unless your site puts files in a weird place, or is a very old version of drupal.

**http_port**, **https_port**: Default to 80 and 443 respectively. Caveat: If you're changing this, you'll likely also need to change the server-wide default port(s), which is not handled by this role. 

## Dependencies

- [acromedia.drupal-cron](https://github.com/AcroMedia/ansible-role-drupal-cron)

## Example Playbook

```yaml
- hosts: servers
  roles:
    - name: Configure the nginx virtual host for bigcorp
      role: acromedia.virtual-host
      vars:
        linux_owner: bigcorp
        project: bigcorp
        nginx_primal_name: bigcorp.host.example.com
        nginx_canonical_name: www.bigcorp.com
        nginx_aliases:
          - bigcorp.com
        php_version: 7.1
        web_root_dir_name: web
        deploy_env: staging
        ssl: letsencrypt
        redirect_code: 302
        web_application: drupal8
        # Uncomment and update the next line after golive:
        # drupal_cron_url: "https://{{ nginx_canonical_name }}/cron/abcdefgh12345678"
```

## License

For Acro Media internal use only.

## Author Information

Acro Media Inc.
