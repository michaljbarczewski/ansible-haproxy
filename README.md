haproxy
=======

Ansible role which helps to install and configure HAproxy.

The configuration of the role is done in such way that it should not be
necessary to change the role for any kind of configuration. All can be
done either by changing role parameters or by declaring completely new
configuration as a variable. That makes this role absolutely
universal. See the examples below for more details.

Please report any issues or send PR.


Examples
--------

```yaml
---

- name: Example of the default usage
  hosts: all
  roles:
    - haproxy

- name: Example of how to alter the HAproxy configuration
  hosts: all
  vars:
    # Create new config by reusing the global and defaults options
    haproxy_config:
      - "{{ haproxy_config_global }}"
      - "{{ haproxy_config_defaults }}"
      - frontend main:
          - bind 0.0.0.0:80
          - default_backend myservice
      - backend myservice:
          - balance roundrobin
          - cookie SERVERID insert indirect nocache
          - server server1 192.168.31.10:3000 check cookie server1
          - server server2 192.168.31.11:3000 check cookie server2
    # Enable debugging by adding an option onto the list of global options
    haproxy_config_global_options__custom:
      - debug
      - option httplog
    # Enable stats by adding few options into the list of defaults options
    haproxy_config_defaults_options__custom:
      - stats enable
      - stats refresh 30s
      - stats show-node
      - stats uri /stats
  roles:
    - haproxy

- name: Example of how to create new configuration
  hosts: all
  vars:
    # New config from scratch
    haproxy_config:
      - global:
          - debug
      - defaults:
          - mode http
          - timeout:
              - connect 5000ms
              - client 50000ms
              - server 50000ms
      - backend legacy:
          - server legacy_server 127.0.0.1:8001
      - frontend app *:80:
          - default_backend legacy
    # Optionally add startup options into sysconfig (CentOS/RedHat only)
    haproxy_sysconfig_options: >
      -f /path/to/different/haproxy.conf
      -m 512
    # Optionally add startup options into default file (Debian/Ubuntu only)
    haproxy_default:
      # Change the config file location if needed
      config: /etc/haproxy/haproxy.cfg
      # Add extra flags here, see haproxy(1) for a few options
      extraopts: -de -m 16
  roles:
    - haproxy
```


Role variables
--------------

Variables used by the role:

```yaml
# Whether to add the SCL YUM repo (provides latest HAproxy version - only for EL7+)
haproxy_scl_yum_repo_install: no

# SCL YUM repo URL
haproxy_scl_yum_repo_url: http://mirror.centos.org/centos/$releasever/sclo/$basearch/rh/

# Additional SCL YUM repo params
haproxy_scl_yum_repo_params: {}

# Whether to add the APT repo (provides latest HAproxy version)
haproxy_apt_repo_install: no

# APT repo string
haproxy_apt_repo_string: ppa:vbernat/haproxy-1.8

# Additional APT repo params
haproxy_apt_repo_params: {}

# Package to be installed (explicite version can be specified here)
haproxy_pkg: "{{
  'rh-haproxy18'
    if haproxy_scl_yum_repo_install
    else
  'haproxy' }}"

# Service name
haproxy_service: "{{
  'rh-haproxy18-haproxy'
    if haproxy_scl_yum_repo_install
    else
  'haproxy' }}"

# Path namespace
haproxy_path_ns: "{{
  'opt/rh/rh-haproxy18'
    if haproxy_scl_yum_repo_install
    else
  '' }}"


# Path to the HAproxy config file
haproxy_config_path: "{{ ('/etc/' ~ haproxy_path_ns ~ '/haproxy/haproxy.cfg') | regex_replace('/+', '/') }}"

# Default values of the options of the global section
haproxy_config_global_log: 127.0.0.1 local2
haproxy_config_global_chroot: "{{ ('/var/' ~ haproxy_path_ns ~ '/lib/haproxy') | regex_replace('/+', '/') }}"
haproxy_config_global_pidfile: "{{ ('/var/run/' ~ haproxy_service ~ '.pid') | regex_replace('/+', '/') }}"
haproxy_config_global_maxconn: 4000
haproxy_config_global_user: haproxy
haproxy_config_global_group: haproxy
haproxy_config_global_daemon: daemon
haproxy_config_global_stats: socket /var/{{ haproxy_path_ns }}/lib/haproxy/stats
haproxy_config_global_ssl_default_bind_ciphers: PROFILE=SYSTEM
haproxy_config_global_ssl_default_server_ciphers: PROFILE=SYSTEM

# Default options of the global section
haproxy_config_global_options__default:
  - log {{ haproxy_config_global_log }}
  - chroot {{ haproxy_config_global_chroot }}
  - pidfile {{ haproxy_config_global_pidfile }}
  - maxconn {{ haproxy_config_global_maxconn }}
  - user {{ haproxy_config_global_user }}
  - group {{ haproxy_config_global_group }}
  - "{{ haproxy_config_global_daemon }}"
  - stats {{ haproxy_config_global_stats }}
  - ssl-default-bind-ciphers {{ haproxy_config_global_ssl_default_bind_ciphers }}
  - ssl-default-server-ciphers {{ haproxy_config_global_ssl_default_server_ciphers }}

# Custom options of the global section
haproxy_config_global_options__custom: []

# Final options of the global section
haproxy_config_global_options: "{{
  haproxy_config_global_options__default +
  haproxy_config_global_options__custom }}"

# Final global section
haproxy_config_global:
  global: "{{ haproxy_config_global_options }}"


# Default values of the options of the global section
haproxy_config_defaults_mode: http
haproxy_config_defaults_log: global
haproxy_config_defaults_option_httplog: httplog
haproxy_config_defaults_option_dontlognull: dontlognull
haproxy_config_defaults_option_http_server_close: http-server-close
haproxy_config_defaults_option_forwardfor: forwardfor except 127.0.0.0/8
haproxy_config_defaults_option_redispatch: redispatch
haproxy_config_defaults_option__default:
  - "{{ haproxy_config_defaults_option_httplog }}"
  - "{{ haproxy_config_defaults_option_dontlognull }}"
  - "{{ haproxy_config_defaults_option_http_server_close }}"
  - "{{ haproxy_config_defaults_option_forwardfor }}"
  - "{{ haproxy_config_defaults_option_redispatch }}"
haproxy_config_defaults_option__custom: []
haproxy_config_defaults_option: "{{
  haproxy_config_defaults_option__default +
  haproxy_config_defaults_option__custom }}"
haproxy_config_defaults_retries: 3
haproxy_config_defaults_timeout_http_request: http-request 10s
haproxy_config_defaults_timeout_queue: queue 1m
haproxy_config_defaults_timeout_connect: connect 10s
haproxy_config_defaults_timeout_client:  client 1m
haproxy_config_defaults_timeout_server: server 1m
haproxy_config_defaults_timeout_http_keep_alive: http-keep-alive 10s
haproxy_config_defaults_timeout_check: check 10s
haproxy_config_defaults_timeout__default:
  - "{{ haproxy_config_defaults_timeout_http_request }}"
  - "{{ haproxy_config_defaults_timeout_queue }}"
  - "{{ haproxy_config_defaults_timeout_connect }}"
  - "{{ haproxy_config_defaults_timeout_client }}"
  - "{{ haproxy_config_defaults_timeout_server }}"
  - "{{ haproxy_config_defaults_timeout_http_keep_alive }}"
  - "{{ haproxy_config_defaults_timeout_check }}"
haproxy_config_defaults_timeout__custom: []
haproxy_config_defaults_timeout: "{{
  haproxy_config_defaults_timeout__default +
  haproxy_config_defaults_timeout__custom }}"
haproxy_config_defaults_maxconn: 3000

# Default options of the defaults section
haproxy_config_defaults_options__default:
  - mode {{ haproxy_config_defaults_mode }}
  - log {{ haproxy_config_defaults_log }}
  - option: "{{ haproxy_config_defaults_option }}"
  - retries {{ haproxy_config_defaults_retries }}
  - timeout: "{{ haproxy_config_defaults_timeout }}"
  - maxconn {{ haproxy_config_defaults_maxconn }}

# Custom options of the defaults section
haproxy_config_defaults_options__custom: []

# Final options of the defaults section
haproxy_config_defaults_options: "{{
  haproxy_config_defaults_options__default +
  haproxy_config_defaults_options__custom }}"

# Final defaults section
haproxy_config_defaults:
  defaults: "{{ haproxy_config_defaults_options }}"


# Default values of the options of the frontend_main section
haproxy_config_frontend_main_bind: "*:5000"
haproxy_config_frontend_main_acl_url_static_path_beg: url_static path_beg -i /static /images /javascript /stylesheets
haproxy_config_frontend_main_acl_url_static_path_end: url_static path_end -i .jpg .gif .png .css .js
haproxy_config_frontend_main_acl:
  - "{{ haproxy_config_frontend_main_acl_url_static_path_beg }}"
  - "{{ haproxy_config_frontend_main_acl_url_static_path_end }}"
haproxy_config_frontend_main_use_backend: static if url_static
haproxy_config_frontend_main_default_backend: app

# Default options of the frontend_main section
haproxy_config_frontend_main_options__default:
  - bind {{ haproxy_config_frontend_main_bind }}
  - acl: "{{ haproxy_config_frontend_main_acl }}"
  - use_backend {{ haproxy_config_frontend_main_use_backend }}
  - default_backend {{ haproxy_config_frontend_main_default_backend }}

# Custom options of the frontend_main section
haproxy_config_frontend_main_options__custom: []

# Final options of the frontend_main section
haproxy_config_frontend_main_options: "{{
  haproxy_config_frontend_main_options__default +
  haproxy_config_frontend_main_options__custom }}"

# Final defaults section
haproxy_config_frontend_main:
  frontend main: "{{ haproxy_config_frontend_main_options }}"


# Default values of the options of the backend_static section
haproxy_config_backend_static_balance: roundrobin
haproxy_config_backend_static_server: static 127.0.0.1:4331 check

# Default options of the backend_static section
haproxy_config_backend_static_options__default:
  - balance {{ haproxy_config_backend_static_balance }}
  - server {{ haproxy_config_backend_static_server }}

# Custom options of the backend_static section
haproxy_config_backend_static_options__custom: []

# Final options of the backend_static section
haproxy_config_backend_static_options: "{{
  haproxy_config_backend_static_options__default +
  haproxy_config_backend_static_options__custom }}"

# Final defaults section
haproxy_config_backend_static:
  backend static: "{{ haproxy_config_backend_static_options }}"


# Default values of the options of the backend_app section
haproxy_config_backend_app_balance: roundrobin
haproxy_config_backend_app_server_app1: app1 127.0.0.1:5001 check
haproxy_config_backend_app_server_app2: app1 127.0.0.1:5002 check
haproxy_config_backend_app_server_app3: app1 127.0.0.1:5003 check
haproxy_config_backend_app_server_app4: app1 127.0.0.1:5004 check
haproxy_config_backend_app_server:
  - "{{ haproxy_config_backend_app_server_app1 }}"
  - "{{ haproxy_config_backend_app_server_app2 }}"
  - "{{ haproxy_config_backend_app_server_app3 }}"
  - "{{ haproxy_config_backend_app_server_app4 }}"

# Default options of the backend_app section
haproxy_config_backend_app_options__default:
  - balance {{ haproxy_config_backend_app_balance }}
  - server: "{{ haproxy_config_backend_app_server }}"

# Custom options of the backend_app section
haproxy_config_backend_app_options__custom: []

# Final options of the backend_app section
haproxy_config_backend_app_options: "{{
  haproxy_config_backend_app_options__default +
  haproxy_config_backend_app_options__custom }}"

# Final defaults section
haproxy_config_backend_app:
  backend app: "{{ haproxy_config_backend_app_options }}"


# Default config
haproxy_config__default:
  - "{{ haproxy_config_global }}"
  - "{{ haproxy_config_defaults }}"
  - "{{ haproxy_config_frontend_main }}"
  - "{{ haproxy_config_backend_static }}"
  - "{{ haproxy_config_backend_app }}"

# Custom config
haproxy_config__custom: []

# Final config
haproxy_config: "{{
  haproxy_config__default +
  haproxy_config__custom }}"


# Sysconfig path
haproxy_sysconfig_path: /etc/sysconfig/{{ haproxy_service }}

# Default value of the sysconfig options
haproxy_sysconfig_options: ""

# Default sysconfig options
haproxy_sysconfig__default:
  options: "{{ haproxy_sysconfig_options }}"

# Custom sysconfig options
haproxy_sysconfig__custom: {}

# Default sysconfig content (see README for examples)
haproxy_sysconfig: "{{
  haproxy_sysconfig__default | combine(
  haproxy_sysconfig__custom) }}"


# Default path
haproxy_default_path: /etc/default/{{ haproxy_service }}

# Content of the default file (see README for examples)
haproxy_default: {}
```

The `haproxy_config` variable defined above produces the following config file
by default:

```
global
  log 127.0.0.1 local2
  chroot /var/lib/haproxy
  pidfile /var/run/haproxy.pid
  maxconn 4000
  user haproxy
  group haproxy
  daemon
  stats socket /var/lib/haproxy/stats
  ssl-default-bind-ciphers PROFILE=SYSTEM
  ssl-default-server-ciphers PROFILE=SYSTEM

defaults
  mode http
  log global
  option httplog
  option dontlognull
  option http-server-close
  option forwardfor except 127.0.0.0/8
  option redispatch
  retries 3
  timeout http-request 10s
  timeout queue 1m
  timeout connect 10s
  timeout client 1m
  timeout server 1m
  timeout http-keep-alive 10s
  timeout check 10s
  maxconn 3000

frontend main
  bind *:5000
  acl url_static path_beg -i /static /images /javascript /stylesheets
  acl url_static path_end -i .jpg .gif .png .css .js
  use_backend static if url_static
  default_backend app

backend static
  balance roundrobin
  server static 127.0.0.1:4331 check

backend app
  balance roundrobin
  server app1 127.0.0.1:5001 check
  server app1 127.0.0.1:5002 check
  server app1 127.0.0.1:5003 check
  server app1 127.0.0.1:5004 check
```


Dependencies
------------

- [`config_encoder_filters`](https://github.com/jtyr/ansible-config_encoder_filters)


License
-------

MIT


Author
------

Jiri Tyr
