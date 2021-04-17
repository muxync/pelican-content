Title: Running GitLab with Docker Compose
Slug: gitlab-docker-compose
Date: 2021-04-17 15:03
Modified: 2021-04-17 15:03
Authors: Mark Mulligan
Category: SysAdmin
Tags: gitlab, docker-compose
Summary: Example Docker Compose file for self hosted GitLab instances.


Example **macvlan** `docker-compose.yml`:

    :::yaml
    version: '2'
    #version: '3'  # Additional IPAM configurations, such as gateway, are only honored for version 2 at the moment.
    #version: '3.8'  # This might work?
    #https://github.com/docker/compose/issues/6569
    #ERROR: The Compose file './docker-compose.yml' is invalid because:
    #networks.vlan.ipam.config value Additional properties are not allowed ('gateway', 'ip_range' were unexpected)

    services:
      # Dummy Service
      macvlan:
        build: .
        image: local/macvlan
        restart: always
        container_name: macvlan
        hostname: macvlan
        networks:
          vlan:
        tty: true

    networks:
      # define a VLAN network (enp1s0 is trunked to multiple VLANs)
      vlan:
        driver: macvlan
        driver_opts:
          parent: enp1s0
        ipam:
          config:
            - subnet: 192.168.1.0/24
              gateway: 192.168.1.10
              ip_range: 192.168.1.0/24 # IP from this pool are assigned automatically
              aux_addresses:
                host: 192.168.1.202

    # https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks
    # Run these commands after creating the macvlan to allow access from localhost:
    #   ip link add macvlan-shim link enp1s0 type macvlan mode bridge
    #   ip addr add 192.168.1.202/32 dev macvlan-shim
    #   ip link set macvlan-shim up
    #   ip route add 192.168.1.213/32 dev macvlan-shim
    # Or put them in /etc/network/interfaces.d/enp1s0 by appending a 'post-up':
    #   post-up ip link add macvlan-shim link enp1s0 type macvlan mode bridge && ip addr add 192.168.1.202/32 dev macvlan-shim && ip link set macvlan-shim up && ip route add 192.168.1.213/32 dev macvlan-shim
    # Which itself can be validated before a reboot with:
    #   sudo ifup --no-act enp1s0   # responds 'ifup: interface enp1s0 already configured' if network config is valid
    #
    # TODO: move all docker containers to a particular CIDR subset of the local network to fix this error:
    #   RTNETLINK answers: File exists
    # and make the below work:
    #   ip route add 192.168.1.0/24 dev macvlan-shim  # ERRORS - 


Example **gitlab** `docker-compose.yml`:

    :::yaml
    version: '3'

    services:
      gitlab-web:
        image: 'gitlab/gitlab-ce:latest'
        restart: always
        container_name: gitlab-web
        hostname: 'gitlab.lan'
        environment:
          GITLAB_OMNIBUS_CONFIG: |
            external_url 'http://gitlab.lan'
            pages_external_url 'http://pages.lan'
            gitlab_pages['enable'] = false
            pages_nginx['enable'] = false
            gitlab_rails['pages_path'] = "/mnt/pages"
        ports:
          - '80:80'
          - '443:443'
          - '22:22'
        volumes:
          - '/srv/gitlab/config:/etc/gitlab'
          - '/srv/gitlab/logs:/var/log/gitlab'
          - '/srv/gitlab/data:/var/opt/gitlab'
          - '/srv/gitlab/data/gitlab-rails/shared/pages/:/mnt/pages'
        networks:
          macvlan_vlan:
            ipv4_address: 192.168.1.213

      gitlab-runner:
        image: gitlab/gitlab-runner:alpine
        restart: unless-stopped
        container_name: gitlab-runner
        depends_on:
          - gitlab-web
        volumes:
          - '/srv/gitlab-runner/config:/etc/gitlab-runner'
          - '/var/run/docker.sock:/var/run/docker.sock'
        networks:
          macvlan_vlan:
            ipv4_address: 192.168.1.214

      gitlab-pages:
        image: 'gitlab/gitlab-ce:latest'
        restart: unless-stopped
        container_name: gitlab-pages
        depends_on:
          - gitlab-web
        environment:
          GITLAB_OMNIBUS_CONFIG: |
            roles ['pages_role']
            pages_external_url 'http://pages.lan'
            gitlab_pages['gitlab_server'] = 'http://gitlab.lan'
            gitlab_rails['pages_path'] = "/mnt/pages"
        ports:
          - '80:80'
        volumes:
          - '/srv/gitlab-pages/config:/etc/gitlab'
          - '/srv/gitlab-pages/logs:/var/log/gitlab'
          - '/srv/gitlab-pages/data:/var/opt/gitlab'
          - '/srv/gitlab/data/gitlab-rails/shared/pages/:/mnt/pages'
        networks:
          macvlan_vlan:
            ipv4_address: 192.168.1.215
        privileged: true
        cap_add:
          - sys_admin
        healthcheck:
          disable: true

    networks:
      macvlan_vlan:
        external: true

