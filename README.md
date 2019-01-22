[![Build Status](https://travis-ci.org/JoergFiedler/freebsd-jailed-nginx.svg?branch=master)](https://travis-ci.org/JoergFiedler/freebsd-jailed-nginx)
freebsd-jailed-nginx
====================

This role provides a jailed nginx server that listens on `localhost:{80,443}` 
for incoming requests.

The role also provides a working SSL configuration using functionality provided
by Let's Encrypt.

You need to provide a custom `dhparam.pem` file. Create a folder named after
your server name (`example.com`) within the folder `files` next to your playbook.

The certificate will be validated/generated by periodic `weekly`. In order to
get the server up and running immediately a self-signed localhost certificate
is installed. You may generate your Let's Encrypt certificates manually after
the jail has been started using the following command:

     /usr/local/bin/acme-client-weekly.sh

It may be used by other roles to serve WordPress or Joomla installations. To 
access the *webroot* directory a SFTP server is also configured, which is 
secured by public key authentication.

This role may also be used to create SSL terminating proxy that forwards traffic 
to domain specific jails or external sites.

Requirements
------------

This role is intent to be used with a fresh FreeBSD installation. There is 
a Vagrant Box with providers for VirtualBox and EC2 you may use.

Role Variables
--------------

##### nginx_letsencrypt_enabled
Set to `yes` the enabled automatic certificate management with Lets Encrypt 
for all nginx servers. If enabled it will install `acme-client` tool and prepare
some configuration for the servers. Default: `no`.

##### nginx_tarsnap_enabled
Whether the webroot of all nginx servers should be backed up using tarsnap. Has
to be enabled on the host itself (`tarsnap_enabled: yes`). Default: 
`{{ tarsnap_enabled | default("no") }}`.

##### nginx_syslogd_server
The syslogd server nginx should use to write error and access log Default: 
`{{ syslogd_server | default("localhost") }}`.

##### nginx_pf_redirect
If set to `yes` if traffic to http(s) ports is forwarded to this jails nginx 
server. Default: `no`.

If enabled, the default configuration will forward traffic directed to port 80 
and 443 of the hosts external interface to this jail. This configuration can be 
changed using the `nginx_pf_rdrs` variable.

##### nginx_pf_rdrs
This configures how traffic redirection from host to jail works. The default configuration looks like this.

    nginx_pf_rdrs:
      - ports:
            - 'http'
            - 'https'
        ext_ip: '{{ host_net_ext_ip }}'
        ext_if: '{{ host_net_ext_if }}'
        
This means that all traffic terminating on host's external interface with external ip for both ports `https` and `http` will be redirected to port `http` of the NGINX running inside the jail. 

##### nginx_servers
This variable holds an array of nginx server instances for this jail. You can use this to configure different types of nginx jails, e.g. https terminating proxy, serve multiple static websites, or a php enable website. See example section below.

    aliases: ''
    basic_auth_enabled: no
    force_www: no
    https:
      enabled: no
      letsencrypt_enabled: no
      key_file: 'localhost-key.pem'
      certbundle_file: 'localhost-certbundle.pem'
      dhparam_file: 'localhost-dhparam.pem'
    name: 'localhost'
    php_fpm_enabled: no
    sftp:
      user: '{{ server_sftp_user }}'
      uuid: '{{ server_sftp_uuid }}'
      authorized_keys: '{{ server_sftp_authorized_keys }}'
      home: '{{ server_home }}'
      port: '{{ server_sftp_port }}'
    sftp_enabled: no
    webroot: '{{ server_webroot }}'

###### aliases
If the server should handle other request then those directed to `server_name`.
Provide list of domain names separated by space. Use `default` to create and Nginx default server. Default: `''`.

###### basic_auth_enabled
Set to `true` to enable basic auth for this server. You need to provide the `htpasswd` file and save it under files `{{ server_name }}/htpasswd`.

##### force_www
If the server should redirect to `www` domain names. If set to `yes` all 
requests to `name` will be redirected to www subdomain. You also need to add `www.name` to `aliases` property. Default: `no`.

##### https
Settings related to SSL/HTTPS.

###### enabled
Set to `yes` to enabled SSL/HTTPS for this server. HTTP only request will be redirected to HTTPS. 

###### letsencrypt_enabled
If set https is enabled for this server and certificates will be created by
Lets Encrypt and `acme-client`. You need to set `nginx_letsencrypt_enabled` to `yes` as well to enable this feature. Default: `no`.

###### name
The domain name of this server, e.g. `example.com`. Default: `default`.

##### php_fpm_enabled
Set to true to install and enable `php-fpm` package. If enabled the following
packages listed in `nginx_php_fpm_pkgs` will be installed. 

##### sftp_enabled
Enable `sftp` for this server. Creates an user and adjusts settings as
described below. Default: `false`.

##### sftp
Settings used to setup user and configure SSHD to allow access to web server root to upload files.

###### user
The sftp user's  name. Default: `'sftp_{{ name | truncate(5, True, "") }}'`.

##### uuid
The sftp user's uuid. Default: `5000`.

##### home
The user's home directory. `sshd` will change root to this diretory. Set the the web server home when unset. Default: `'/srv/{{ name }}'`.

##### port
The external port which should be redirected to the jail using this role. 
Default: `10022`.

##### authorized_keys
The public key which should be used to authenticate the user. Default: `'{{ host_sshd_authorized_keys_file }}'`.

Dependencies
------------

- [JoergFiedler.freebsd-jailed](https://galaxy.ansible.com/JoergFiedler/freebsd-jailed/)

Example Playbook
----------------

Proxy host, that forwards traffic to other external server.

    - include_role:
            name: 'JoergFiedler.freebsd-jailed-nginx'
          vars:
            jail_freebsd_release: 11.2-RELEASE'
            jail_name: 'nginx'
            jail_net_ip: '10.1.0.10'
            nginx_pf_redirect: true
            nginx_servers:
              - name: 'freebsd'
                proxy:
                  host: 'www.freebsd.org'
                  scheme: 'https'
                  port: 443
                  local: no

Nginx server with `php-fpm` module and HTTPS.

    - hosts: all
      become: true
    
      vars:
        ansible_python_interpreter: '/usr/local/bin/python2.7'
    
      tasks:
        - include_role:
            name: 'JoergFiedler.freebsd-jailed-nginx'
          vars:
            jail_freebsd_release: '11.2-RELEASE'
            jail_name: 'nginx'
            jail_net_ip: '10.1.0.10'
            nginx_pf_redirect: yes
            nginx_servers:
              - name: 'default'
                https:
                  enabled: yes
                php_fpm_enabled: yes
                sftp_enabled: yes
                sftp:
                  authorized_keys: '~/.vagrant.d/insecure_private_key.pub'
            
License
-------

BSD

Author Information
------------------

If you like it or do have ideas to improve this project, please open an issue
on GitHub. Thanks.
