# Ansible Role Home Assistant

Library of Ansible plugins and roles for deploying various services.
See [ansible-roles](https://github.com/davidfischer-ch/ansible-roles) for additional documentation.

This repository hosts the role Home Assistant and may depend of other roles and plugins of the library.

## Dependencies

See [meta](meta/main.yml).

## Variables

TODO VARIABLES

## Example

Example for installing Home Assistant on a host.

### Playbook

```
---

- hosts:
    - domotic
  roles:
    - deconz
    - home-assistant
    - nginx
  vars:
    deconz_role_action: setup
    home_assistant_role_action: setup
    nginx_role_action: setup

    # Python

    # No compilation dude!
    python_build_packages: []
    python_packages:
      #- bpython3
      #- ipython3
      - python3-dev

    python_pypy_versions: []

    # UFW

    ssh_port: '{{ ansible_port }}'

    # Domotic

    deconz_instance_name: deconz

    home_assistant_instance_name: homeassistant
    home_assistant_python_executable: python3.8

    home_assistant_version: 2022.2.2            # 06-02-2022 Released 04-02-2022
    home_assistant_configurator_version: 0.4.1  # 06-02-2022 Released 23-04-2021
    home_assistant_psycopg2_version: 2.9.3      # 06-02-2022 Released 29-12-2021

    home_data_instance_name: homedata
    home_data_python_executable: python3.8
    home_data_source_directory: '{{ playbook_dir }}/../../home-assistant/data/'

    nginx_version: release-1.21.6  # 06-02-2022 Released 25-01-2022
    nginx_daemon_mode: supervisor
    nginx_build_flags:
      - --with-http_addition_module
      - --with-http_auth_request_module
      - --with-http_dav_module
      - --with-http_geoip_module
      - --with-http_gzip_static_module
      - --with-http_image_filter_module
      - --with-http_realip_module
      - --with-http_ssl_module
      - --with-http_stub_status_module
      - --with-http_sub_module
      - --with-http_v2_module
      - --with-http_xslt_module
      - --with-ipv6
      - --with-pcre-jit
    nginx_server_header_engine: fisch3r
    nginx_worker_processes: '{{ 4 * ansible_processor_count }}'

    # Check with https://www.abuseipdb.com/check/184.105.247.195
    nginx_geoip_blocking_enabled: yes
    nginx_geoip_blocking_default: block
    nginx_geoip_countries_whitelist:
      - CH  # Switzerland
      - FR  # France
      - EU  # Europe

    nginx_sites:

      home:
        name: home
        config_file: '{{ playbook_dir }}/../roles/domotic/templates/home.nginx.conf.j2'
        domains:
          - my-home-assistant.com
        address: 127.0.0.1
        port: 8123
        redirect_ssl: yes
        ssl_files_prefix: /path/to/ssl_for_site
        with_dhparam: yes
        with_http2: yes
        with_ssl: yes

    supervisor_daemon_mode: systemd
    supervisor_username: someuser
    supervisor_password: somepass
    supervisor_version: 4.2.4  # 06-02-2022 Released 30-12-2021
```

### Config Template

```
{% include 'templates/site.reject-unkown.conf.j2' %}

{% if item.value.with_ssl|bool and item.value.redirect_ssl|bool %}
server {
    listen {{ nginx_port|int }};
    server_name {{ item.value.domains|join(' ') }};
    rewrite ^ https://$http_host$request_uri? permanent;
}
{% endif %}

server {
    {% if item.value.with_ssl|bool %}
    listen {{ nginx_port_ssl|int }} {{ item.value.with_http2|bool|ternary('ssl http2', '') }};
    {% include 'templates/site.ssl.conf.j2' %}
    {% else %}
    listen {{ nginx_port|int }};
    {% endif %}

    {% include 'templates/site.real_ip.conf.j2' %}

    {% if not item.value.with_ssl|bool and item.value.redirect_ssl|bool %}
    if ($http_x_forwarded_proto != 'https') {
        rewrite ^ https://$host$request_uri? permanent;
    }
    {% endif %}

    server_name {{ item.value.domains|join(' ') }};

    gzip off;  # Security - http://breachattack.com/resources/BREACH%20-%20SSL,%20gone%20in%2030%20seconds.pdf
    gzip_proxied any;
    gzip_types application/javascript application/json text/css text/javascript text/xml;

    client_max_body_size {{ item.value.max_body_size }};

    access_log /var/log/nginx/{{ item.value.name }}/access.log;
    error_log /var/log/nginx/{{ item.value.name }}/error.log warn;

    # location /robots.txt {
    #     alias /some/path/robots.txt;
    # }

    # location /favicon/ {
    #     alias /some/path/favicons/;
    # }

    location / {
        access_log  on;
        autoindex   on;
        {% if not item.value.debug|bool %}
        expires     max;
        {% endif %}
        gzip        on;
        sendfile    off;  # Avoid Nginx serving outdated static files
        alias {{ item.value.directory }}/;
    }
}
```

## License

See [LICENSE.rst](LICENSE.rst).

## Authors

See [AUTHORS](AUTHORS).

2014-2019 - David Fischer
