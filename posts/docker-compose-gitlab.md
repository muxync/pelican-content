Title: Self-Managed GitLab using Docker Compose
Slug: docker-compose-gitlab
Date: 2021-04-10 21:10
Modified: 2021-04-10 21:10
Authors: Mark Mulligan
Category: SysOps
Tags: docker, gitlab
Summary: Example Docker Compose file for Self-Managed GitLab instances.


*This is a follow up to my [Docker macvlan post](docker-compose-macvlan.html) where I shared an example Docker Compose file for configuring a macvlan network to connect containers to a physical network.*

GitLab has become an invaluable tool for me but getting it to work via Docker Compose was not so straight forward and unfortunately the [offical GitLab documentation](https://docs.gitlab.com/omnibus/docker/#install-gitlab-using-docker-compose) makes no mention of one glaring issue; **GitLab Pages must be run in a separate container!**  This is the config I settled on after a lot of time wasted trying to get GitLab Pages to work on my Self-Managed GitLab instance which works fine for my purpose.

Example **gitlab** `docker-compose.yml`:

    :::yaml
    version: '3'

    services:
      gitlab-web:
        image: 'gitlab/gitlab-ce:latest'
        restart: always
        container_name: gitlab-web
        hostname: 'gitlab.lan'
        mac_address: 02:42:C0:A8:01:D5
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
        mac_address: 02:42:C0:A8:01:D6
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
        mac_address: 02:42:C0:A8:01:D7
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

and the corresponding `dnsmasq` config:

    :::ini
    dhcp-host=02:42:C0:A8:01:D5,gitlab.lan,192.168.1.213,infinite
    address=/gitlab.lan/192.168.1.213
    ptr-record=213.1.168.192.in-addr.arpa,gitlab.lan

    dhcp-host=02:42:C0:A8:01:D6,gitlab-runner.lan,192.168.1.214,infinite
    address=/gitlab-runner.lan/192.168.1.214
    ptr-record=214.1.168.192.in-addr.arpa,gitlab-runner.lan

    dhcp-host=02:42:C0:A8:01:D7,pages.lan,192.168.1.215,infinite
    address=/pages.lan/192.168.1.215
    ptr-record=215.1.168.192.in-addr.arpa,pages.lan

After you first start your GitLab instance with `docker-compose up -d` and complete the initial configuration you'll want to register your GitLab Runner so you can run some builds.  To do that I use the following script named `gitlab-runner-register.sh`:

    :::shell
    #!/bin/sh

    # See this post for the example from which this is based:
    # https://gitlab.com/gitlab-org/gitlab/-/issues/23911#note_215199418
    #
    # Get the registration token from:
    # http://localhost:8080/root/${project}/settings/ci_cd

    DOCKER_NETWORK_NAME=macvlan_vlan
    GITLAB_RUNNER_NAME=gitlab-runner1
    GITLAB_SERVER_URL=http://gitlab.lan
    REGISTRATION_TOKEN=XXXXXXXXXXXXXXXXXXXX

    docker exec -it ${GITLAB_RUNNER_NAME}
      gitlab-runner register \
        --non-interactive \
        --registration-token ${REGISTRATION_TOKEN} \
        --locked=false \
        --description docker-stable \
        --url ${GITLAB_SERVER_URL} \
        --executor docker \
        --docker-image docker:stable \
        --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
        --docker-network-mode ${DOCKER_NETWORK_NAME}

So far this setup has fit my needs perfectly and managing GitLab via containers instead of the overhead of a full blown virtual machine has been a great improvement.  I still want to setup some automation for updating my containers ([Watchtower](https://containrrr.dev/watchtower) seems promising) but for now I have been using simple shell scripts like this `upgrade.sh`:

    :::shell
    #!/bin/sh

    sudo docker-compose down
    sudo docker pull gitlab/gitlab-ce:latest
    sudo docker-compose up -d
