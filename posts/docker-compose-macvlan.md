Title: Docker Compose macvlan network for accessing containers
Slug: docker-compose-macvlan
Date: 2021-04-17 21:10
Modified: 2021-04-17 21:10
Authors: Mark Mulligan
Category: SysOps
Tags: docker, macvlan
Summary: Example Docker Compose file for configuring a macvlan network to connect containers to a physical network.


# Intro

When I decided to replace my home [KVM](https://www.linux-kvm.org) virtual machines with [Docker](https://www.docker.com) containers one of the first questions I had was how I would access my docker containers on my LAN.  I had been using [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) for my DHCP server running on a [DD-WRT](https://dd-wrt.com) hacked wireless router so the ideal solution would be to piecemeal replace my VMs with containers.  Luckily I came accross this [great post](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks) by Lars Kellogg-Stedman that answers this exact question, *How do I attach a container directly to my local network?*

I encourage you to read that post as I won't repeat what it covers in detail.  In summary, my best option was to use a Docker [macvlan](https://docs.docker.com/network/macvlan) network to assign a MAC address to a container’s virtual network interface which would allow me to access the container as if it were physically connected to my LAN.  One of the limitations of this setup is you must not allow you DHCP server to assign addresses in the same range as your containers but for me that's not a problem as I manage my home network.

# Configuration

Note that the examples below use a network interface named `enp1s0` which will likely differ from your own.  Please substitute `enp1s0` with the correct value for your network interface which you can determine with:

    :::shell
    sudo ifconfig

Additionally, the IP `192.168.1.202` is what I use as a "reserved" IP address for use with the `macvlan` (i.e. an IP address that I will not allow my DHCP server to assign to a client).

Lastly, I am running `docker-compose` version `1.23.1` which has some limitations for configuring a `macvlan`.  I have included some comments in this file for reference but the takeaway is I will be using `version: '2'` of the docker template until Debian gets `1.27.0` of Docker Compose in their repos.

Example **macvlan** `Dockerfile`:

    :::docker
    FROM busybox:latest

    ENTRYPOINT tail -f /dev/null

Example **macvlan** `docker-compose.yml`:

    :::yaml
    version: '2'
    # version: '3'  # Additional IPAM configurations, such as gateway, are only honored for version 2 at the moment.
    # https://docs.docker.com/compose/compose-file/#ipam
    #   ERROR: The Compose file './docker-compose.yml' is invalid because:
    #   networks.vlan.ipam.config value Additional properties are not allowed ('gateway', 'ip_range' were unexpected)
    # https://github.com/docker/compose/issues/6569
    # 
    # Releases 1.27.0+ have merged v2/v3 file formats but Debian only has 1.25.0 available currently
    # https://packages.debian.org/sid/docker-compose

    services:
      # Dummy service
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
      # Define a VLAN network (enp1s0 is trunked to multiple VLANs)
      vlan:
        driver: macvlan
        driver_opts:
          parent: enp1s0
        ipam:
          config:
            - subnet: 192.168.1.0/24
              gateway: 192.168.1.10
              ip_range: 192.168.1.0/24 # IPs from this pool are assigned automatically
              aux_addresses:
                host: 192.168.1.202

After starting the `macvlan` container with `sudo docker-compose up -d` I added a commented out config to the "Additional Dnsmasq Options" of my router's `dnsmasq` config to remind myself not to manually assign it to other hosts (my DHCP IP range stops at 192.168.1.200 so it won't assign anything above that automatically):

    :::ini
    #dhcp-host=XX:XX:XX:XX:XX:XX,macvlan-shim,192.168.1.202,infinite
    #address=/macvlan-shim/192.168.1.202
    #ptr-record=202.1.168.192.in-addr.arpa,macvlan-shim


Example **client container** `docker-compose.yml`, in this case for [syncthing](https://syncthing.net):

    :::yaml
    services:
      syncthing:
        image: syncthing/syncthing
        volumes:
          - /storage/syncthing:/var/syncthing
        restart: always
        container_name: syncthing
        mac_address: 02:42:C0:A8:01:D9
        expose:
          - "8384"
          - "22000"
          - "21027"
        ports:
          - "8384:8384"
          - "22000:22000"
          - "21027:21027/udp"
        networks:
          macvlan_vlan:
            ipv4_address: 192.168.1.217
        tty: true

    networks:
      macvlan_vlan:
        external: true

and the corresponding `dnsmasq` config:

    :::ini
    dhcp-host=02:42:C0:A8:01:D9,syncthing,192.168.1.217,infinite
    address=/syncthing/192.168.1.217
    ptr-record=217.1.168.192.in-addr.arpa,syncthing

The MAC address (e.g. "02:42:C0:A8:01:D9") can be determined after starting the container with:

    :::shell
    sudo docker container inspect \
      --format='{{range .NetworkSettings.Networks}}{{.MacAddress}}{{end}}' syncthing \
      | tr '[:lower:]' '[:upper:]'

You should see something like:

    :::shell
    02:42:C0:A8:01:D9

After starting the `syncthing` container with `sudo docker-compose up -d` I was able to connect to from another computer on my network to the [Syncthing web interface](https://syncthing:8384) but it still wouldn't work from the same host the docker container is running on...

# Caveats

A limitation of `macvlan` networks is the docker host won't be able to connect to its own macvlan interfaces.  However, there's an easy fix by way of a network bridge.

Run these commands after creating the macvlan to allow access from localhost:

    :::shell
    ip link add macvlan-shim link enp1s0 type macvlan mode bridge
    ip addr add 192.168.1.202/32 dev macvlan-shim
    ip link set macvlan-shim up
    ip route add 192.168.1.213/32 dev macvlan-shim

Or put them in `/etc/network/interfaces.d/enp1s0` by appending a `post-up`:

    :::shell
    post-up ip link add macvlan-shim link enp1s0 type macvlan mode bridge && ip addr add 192.168.1.202/32 dev macvlan-shim && ip link set macvlan-shim up && ip route add 192.168.1.213/32 dev macvlan-shim

Which itself can be validated before a reboot with:

    :::shell
    sudo ifup --no-act enp1s0
    # responds 'ifup: interface enp1s0 already configured' if network config is valid

You should then be able to connect to your docker containers from your docker host.
