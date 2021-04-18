Title: Self-Managed GitLab using Docker Compose
Slug: gitlab-docker-compose
Date: 2021-04-17 15:03
Modified: 2021-04-17 15:03
Authors: Mark Mulligan
Category: SysAdmin
Tags: gitlab, docker-compose
Summary: Example Docker Compose file for Self-Managed GitLab instances.



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

