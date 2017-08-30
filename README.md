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

# Defines the number of worker processes
nginx_worker_processes: 'auto'

# Sets the maximum allowed size of the client request body, specified in
# the “Content-Length” request header field
nginx_max_body_size: 2M

# Sets the maximum number of simultaneous connections that can be opened by a
# worker process.
nginx_worker_connections: 512

# Enables or disables emitting nginx version in error messages and in
# the “Server” response header field
nginx_server_tokens: yes

# List of packages to install
nginx_packages:
  - 'nginx-full'

nginx_repo:
  source: 'ppa:nginx/stable'
  key_id: 'ID'
  # or
  key: 'URL'

# Vault encrypted SSL certificates and keys
vault_tls:
  key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
  cert: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

# List of files to be included in configuration files (proxy_params, etc)
# template module
# Playbook path: library/templates/nginx.{{ item }}.j2
# Server path:   {{ nginx_etc_path }}/{{ item }}
nginx_includes:
  - proxy
  - tls

# List of files to be included in http context
# for loop in nginx.conf template
nginx_http_includes:
  - log_formats
  - proxy_params
  - realip

# Playbook path: library/templates/nginx.{{ item.name }}.j2
# Server path:   {{ nginx_etc_path }}/conf.d/{{ item.name }}.conf
nginx_servers:
  - name: example.com          # the file name would be nginx.example.com.j2
    server_name: www.example.com   # hostname
    tls: '{{ vault_tls }}'     # references the dictionary from vault
    auth:                      # basic auth users
      - name: alice
        pass: suchsecurity
      - name: bob
        pass: verysafe

# SLB frontend
frontlb_servers:
  - name: example.com
    server_name: www.example.com
    lb_method: 'least_conn'
    tls: '{{ vault_tls }}'
    upstreams:
      - 10.1.250.2
      - 10.1.250.3
      - 10.1.250.4
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

  tls_certificate '{{ nginx_tls_certificate_path }}';
  tls_certificate_key '{{ nginx_tls_key_path }}';

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
