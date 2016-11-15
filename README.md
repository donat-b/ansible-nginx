NGINX
=====

An ansible role for nginx deployment which doesn't try to use dictionaries for everything.

Currently supports Debian (official repository) and Ubuntu (via ppa) Linux.

Requirements
------------

None

Role Variables
--------------

```yaml
# Defines user and group credentials used by worker processes
nginx_user: 'www-data'

# Sets the maximum allowed size of the client request body, specified in
# the “Content-Length” request header field
nginx_max_body_size: 2M

# Enables or disables emitting nginx version in error messages and in the “Server” response header field
nginx_disable_server_tokens: yes

# Vault encrypted SSL certificates and keys
vault_ssl:
  key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
  cert: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

# List of files to be included in configuration files (proxy_params, etc)
# Playbook path: library/templates/nginx.{{ item }}.j2
# Server path:   {{ nginx_etc_path }}/{{ item }}
nginx_includes:
  - proxy
  - ssl

# Playbook path: library/templates/nginx.{{ item.name }}.j2
# Server path:   {{ nginx_etc_path }}/conf.d/{{ item.name }}.conf
nginx_servers:
  - name: example.com          # the file name would be nginx.example.com.j2
    server_name: example.com   # hostname
    ssl: '{{ vault_ssl }}'     # references the dictionary from vault
    auth:                      # basic auth users
      - name: alice
        pass: suchsecurity
      - name: bob
        pass: verysafe
```


Configuration example
---------------------

```conf
upstream backend {
  server '127.0.0.1:8080' fail_timeout=0;
}

server {
  listen 443 ssl;
  listen [::]:443 ssl;

  server_name '{{ item.server_name }}';

  error_log  '{{ nginx_log_path }}/{{ item.server_name }}.error.log';
  access_log '{{ nginx_log_path }}/{{ item.server_name }}.access.log';

  ssl_certificate '{{ nginx_ssl_certificate_path }}';
  ssl_certificate_key '{{ nginx_ssl_key_path }}';
  include ssl_params;

  client_max_body_size '{{ nginx_max_body_size }}';

  location /media {
    alias        '/var/lib/app/media';
    access_log   off;
    index        off;
    expires      30d;
    add_header   Pragma        public;
    add_header   Cache-Control 'public';
  }

  location / {
    include              proxy_params;
    proxy_pass           http://backend;
    proxy_set_header     X-Real-Host $host;
    real_ip_header       X-Forwarded-For;
    proxy_read_timeout   30s;
    proxy_redirect       off;

    auth_basic           'Restricted';
    auth_basic_user_file '/etc/nginx/{{ item.server_name }}.htpasswd';
  }
}
```

Dependencies
------------

None

Example Usage
-----

`git submodule add https://github.com/donat-b/ansible-nginx.git roles/nginx`


```yaml
- hosts: webservers
  roles:
    - nginx
```

License
-------

BSD
