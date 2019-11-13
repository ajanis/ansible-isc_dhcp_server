# Ansible-isc_dhcp_server

<!-- MarkdownTOC -->

- Requirements
- Role Variables
- Dependencies
- Example Group Variables
- Example Playbook
- License
- Author Information

<!-- /MarkdownTOC -->

Set up an ISC DHCP Server as part of the Foreman build server Project

This role is part of a project that will configure a Foreman build environment with optional TFTP and DHCP smart proxies, and an NGINX webserver for serving static content and acting as a reverse-proxy to Foreman.

## Requirements

N/A

## Role Variables
```yaml
isc_dhcp_server_omapi_port: 7911

isc_dhcp_server_subnets:
  - name: "10.0.10.0/24"
    network: "10.0.10.0"
    mask: "255.255.255.0"
    gateway: "10.0.10.1"
    from_ip: "10.0.10.240"
    to_ip: "10.0.10.250"
    vlanid:
    mtu: 9000
    domains:
    - "{{ www_domain }}"
    dns_primary: 10.0.10.1
    dns_secondary:

```

## Dependencies

Additonal variables may be set in group_vars for configuring Foreman, nginx, isc-dhcp server, and tftp server


## Example Group Variables
```yaml
www_domain: home.example.com
foreman_hostname: foreman

fail2ban_enable: False

nginx_backends:
  - service: foreman
    servers:
      - localhost:3000

nginx_vhosts:
  - servername: "{{ foreman_hostname | default(ansible_hostname) + '.' + www_domain | default(ansible_domain) }}"
    serveralias: "{{ ansible_default_ipv4.address }} {{ ansible_fqdn }}"
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: foreman
      - name: 404.html
        docroot: "/usr/share/nginx/html"
      - name: 50x.html
        docroot: "/usr/share/nginx/html"
```

**PROTIP!!** If deploying isc-dhcp-server as part of foreman setup, use ```foreman_proxy_dhcp_subnets``` in your group_vars to configure both isc-dhcp-server and foreman DHCP smart-proxy.

```yaml
foreman_proxy_dhcp_subnets:
  - name: "10.0.10.0/24"
    network: "10.0.10.0"
    mask: "255.255.255.0"
    gateway: "10.0.10.1"
    from_ip: "10.0.10.240"
    to_ip: "10.0.10.250"
    vlanid:
    mtu: 9000
    domains:
    - "{{ www_domain }}"
    dns_primary: 10.0.10.1
    dns_secondary:
```


## Example Playbook
```yaml
- name: "Deploy Foreman Server"
  hosts: all
  remote_user: root
  tasks:

    - setup:
    - include_role:
        name: common
      tags:
        - common

    - include_role:
        name: isc_dhcp_server
        public: yes
        apply:
          tags:
            - dhcp
      when: foreman_proxy_dhcp
      tags:
        - dhcp

    - include_role:
        name: tftp
        public: yes
        apply:
          tags:
            - tftp
      when: foreman_proxy_tftp
      tags:
        - tftp

    - include_role:
        name: nginx
        public: yes
        apply:
          tags:
            - nginx
      tags:
        - nginx

    - include_role:
        name: awx
        public: yes
        apply:
          tags:
            - awx
      tags:
        - awx

    - include_role:
        name: docker
        public: yes
        apply:
          tags:
            - docker
      tags:
        - docker

    - include_role:
        name: awx
        tasks_from: update_ca.yml
        public: yes
        apply:
          tags:
            - awx
      tags:
        - awx

    - include_role:
        name: foreman
        public: yes
      tags:
        - install
        - configure
        - foreman
        - smartproxy

    - include_role:
        name: foreman
        public: yes
        tasks_from: customize_config.yml
        apply:
          tags:
            - customize
      tags:
        - customize

    - include_role:
        name: foreman
        public: yes
        tasks_from: host-build.yml
      tags:
        - hostcreate
        - hostcleanup
```

## License

MIT

## Author Information

Created by Alan Janis
