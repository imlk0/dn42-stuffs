# 📜dn42-stuffs

This project documents an effort to deploy dn42 Route Server (and some related services) in docker containers. The purpose is to avoid duplicate configurations by describing all service in the `docker-compose.yml`. Should be suitable for those who want to get in touch with dn42 quickly and those who want to add Route Server.

# Services

|services|components|status|
|----|----|----|
|bgp|bird2|✔️|
|dns|bind9|✔️|
|bird-lgproxy|xddxdd/bird-lgproxy-go|🚧|
|bird-lg|xddxdd/bird-lg-go|🚧|

# Setup

We recommend to use [docker-compose v2](https://docs.docker.com/compose/cli-command/#install-on-linux) to setup containers. 

- Build all containers and docker networks
  ```sh
  docker compose up --build --no-start
  ```
- Start all containers
  ```sh
  docker compose start
  ``` 
- Show status of containers
  ```sh
  docker compose ps
  ```
- Enter specific container
  ```sh
  docker compose exec bgp bash
  ```
- Stop and delete all services and docker networks completely
  ```sh
  docker compose down
  ```

# Config

> We only list the minimum configuration needed to make it work. For more configurations, please read the code or [docker-compose specification](https://docs.docker.com/compose/compose-file/)


1. Configure the `subnet` for `dn42-net` in `docker-compose.yml`

    ```yml
    networks:
      dn42-net:
        driver: bridge
        enable_ipv6: true
        internal: false
        ipam:
          driver: default
          config:
            - subnet: <your dn42 ipv4 subnet>
            - subnet: <your dn42 ipv6 subnet>
    ```
    This is usually the range of dn42 addresses you get. But if you are expanding your dn42 network, **make sure that this subnet does not conflict with the rest of your dn42 subnet.**

    > !!! Note that the docker host will take up **the first address on the subnet**. So you cannot assign the first ip to any of the containers.

1. For each service, you may want to assign a dn42 ip address fot it.

    ```yml
        networks:
          dn42-net:
              ipv4_address: "<dn42 ip address allocated this service>"
              ipv6_address: "<dn42 ip address allocated this service>"
    ```

2. And you can config the dns to a server that can provide `.dn42` resolution, for example, you can use `172.20.0.53`

    ```yml
        dns: 
          - 172.20.0.53 # wildly used dns server in dn42. Or you can change this to your dns service ip address
    ```

3. All containers except the bgp container need to manually configure routes to forward traffic going to the dn42 network to the bgp container.
   
   We provide two environment variable to configure the ip address of the dn42 gateway

    ```yml
        environment:
          - DN42_GATEWAY_V4=<ipv4 address of your bgp container>
          - DN42_GATEWAY_V6=<ipv6 address of your bgp container>
    ```
## bgp

1. Edit `bgp/named.conf`
   
   You need to edit the config file of bird2 `bgp/named.conf` according to the guidance [here](https://dn42.eu/howto/Bird2).

2. For each of your peers, create files in dir `bgp/bird2-peers` and `bgp/wg-peers`

3. You need to add port mapping for your peers in `docker-compose.yml`
   
   ```yml
       ports:
        - "21742:21742/udp" # imlk
   ```

## dns

This service is both an **authoritative name server** and a **recursive name server**.

> As an example, I've included my zone configuration(`imlk.dn42.zone` and `0%2F26.96.22.172.in-addr.arpa.zone` and `e.0.a.8.a.a.2.d.2.4.d.f.ip6.arpa.zone`), **!!! make sure you remove it before you deploy !!!**.

1. Edit `bind9/named.conf` to add your zones
   
   Typically, you will need to add three zones, one for domain name resolution, and two for reverse resolutions (PTR records for ipv4 and ipv6 address).

   Also, please put the zone files in the `zones` directory.

2. Setup dnssec
   
   You need to generate the `zone-signing key` and `key-signing key` for each zone.

   As an example, for zone `example.dn42`, run following command to generate keys in `bind9/dnssec_keys` directory.

   ```sh
   cd bind9/dnssec_keys
   # generate zone-signing key
   dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.dn42
   # generate key-signing key
   dnssec-keygen -f KSK -a ECDSAP256SHA256 -n ZONE example.dn42
   ```
   Each run of `dnssec-keygen` will generate a pair of `.key` file and `.private` file. Remember to include the path of generated `.key` file in the corresponding .zone file.

   You do not need to sign your zone manually. When you build this service, it automatically generates `RRSIG` records using `dnssec-signzone`. **It will also print out the `DS` records so you can add them to the dn42/register git repository**

# Q&A

- Should I disable my firewall software on docker host?
  
  Normally it is not needed. But if you use `firewalld`, you need to set `IPv6_rpfilter=no` to make ipv6 forwarding work properly.

- DNS configuration
  
  The dns management in docker is rather confusing. In containers using custom bridge network, an Embedded DNS server will be used, and there is only one `nameserver 127.0.0.11` in `/etc/resov.conf`. If you want to modify the dns server used, just add a dns entry in `docker-conpose.yml`, and the docker Embedded DNS server will forward dns request to that address.

- Why not using a range mapping instead separate port mapping for each peers?
  
  Maybe you're talking about something like this:
  ```yml
  ports:
   - "20000-29999:20000-29999/udp"
  ```
  However, large port mapping range will cost lost memory, it soon ate up all the memory in my host. see [this page](https://forums.docker.com/t/i-have-a-docker-container-that-needs-to-expose-10-000-ports/96048/15)

- `DN42_GATEWAY_V4` and `DN42_GATEWAY_V6` looks like ugly script work, why not use `gateway` in [IPAM configuration](https://docs.docker.com/compose/compose-file/#ipam)?
  
  AFAIK, IPAM `gateway` options are ignored by the current version of docker compose. see [this issue](https://github.com/docker/compose/issues/8742). It seems to be a bug.

  On the other hand, the option does something different than what it looks like: **It only changes the ip address of the host**, while the gateway of the container is still set to host.

  This is why we need a script work to set the gateway for each container.


