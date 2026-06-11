# home-media-server-build

## Install Debian
TODO
## Install curl, htop, openssh-server, tree, ufw, unattended-upgrades
TODO
## Register Domain
TODO
## Install Docker
TODO
## Nginx Proxy Manager
TODO
## Tailscale for Jellyfin Subnet Router
##### docker-compose.yml
~~~
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: ${CONTAINER_NAME}
    hostname: ${HOSTNAME} # name for machine on Tailscale
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./lib:/var/lib
    environment:
      - TS_AUTH_KEY=${AUTH_KEY}?ephemeral=false
      - TS_STATE_DIR=/var/lib/tailscale
# --snat-sub.. unmask tailscale user ip
      - TS_EXTRA_ARGS=--advertise-tags=tag:jellyfin --snat-subnet-routes=false #--accept-routes # Enabling this allows the client to receive all advertised subnet routes, which can lead to route conflicts if local physical LANs overlap with advertised subnets
      - TS_ROUTES=192.168.0.0/24 # same as route in Jellyfin exit node for redundant failover
#      - TS_AUTH_KEY=${AUTH_KEY} # key with expiration date
    network_mode: host
    cap_add:
      - net_admin
      - net_raw
    restart: unless-stopped
~~~
## Tailscale for Jellyfin Exit node
##### docker-compose.yml
~~~
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: ${CONTAINER_NAME}
    hostname: ${HOSTNAME} # name for machine on Tailscale
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ./lib:/var/lib
    environment:
      - TS_AUTH_KEY=${AUTH_KEY}?ephemeral=false
      - TS_STATE_DIR=/var/lib/tailscale
# advertise exit node arg
      - TS_EXTRA_ARGS=--advertise-tags=tag:dns --advertise-exit-node # --accept-routes # Enabling this allows the client to receive all advertised subnet routes, which can lead to route conflicts if local physical LANs overlap with advertised subnets
      - TS_ROUTES=192.168.0.0/24 # same as route in Jellyfin subnet for redundant failover
#      - TS_AUTH_KEY=${AUTH_KEY}
    network_mode: host
    cap_add:
      - net_admin
      - net_raw
    restart: unless-stopped
~~~
## Pihole
##### docker-compose.yml
~~~
# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: ${CONTAINER_NAME}
    image: pihole/pihole:latest
    ports:
      # DNS Ports
      - "53:53/tcp" # these still need to be published
      - "53:53/udp"
# DO NOT EXPOSE 80 or 443, will access through nginx proxy manager
      # Default HTTP Port
#      - "8080:80/tcp" # custom 8080 when using nginx proxy manager on same machine
      # Default HTTPs Port. FTL will generate a self-signed certificate
#      - "8443:443/tcp" # custom 8443 when using nginx proxy manager on same machine
      # Uncomment the line below if you are using Pi-hole as your DHCP server
      #- "67:67/udp"
      # Uncomment the line below if you are using Pi-hole as your NTP server
      #- "123:123/udp"
    environment:
      PIHOLE_UID: 1000
      PIHOLE_GID: 1000
      # Set the appropriate timezone for your location (https://en.wikipedia.org/wiki/List_of_tz_database_time_zones), e.g:
      TZ: 'America/New_York'
      # Set a password to access the web interface. Not setting one will result in a random password being assigned
      FTLCONF_webserver_api_password: "${PASSWORD}"
      # If using Docker's default `bridge` network setting the dns listening mode should be set to 'ALL'
      FTLCONF_dns_listeningMode: 'ALL'
    # Volumes store your data between container upgrades
    volumes:
      # For persisting Pi-hole's databases and common configuration file
      - './etc-pihole:/etc/pihole'
      # Uncomment the below if you have custom dnsmasq config files that you want to persist. Not needed for most starting fresh with Pi-hole v6. If you'>
      #- './etc-dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      # See https://docs.pi-hole.net/docker/#note-on-capabilities
      # Required if you are using Pi-hole as your DHCP server, else not needed
#      - NET_ADMIN
      # Required if you are using Pi-hole as your NTP client to be able to set the host's system time
#      - SYS_TIME
      # Optional, if Pi-hole should get some more processing time
      - SYS_NICE
    restart: unless-stopped
    networks:
      - ${NGINX_DEFAULT_NETWORK}
networks:
  ${NGINX_DEFAULT_NETWORK}$: # main nginx proxy manager network
    external: true
~~~

## Authentik

##### Setup .env file
( [Authentik '.env' File Setup](https://docs.goauthentik.io/install-config/install/docker-compose/) )

1. Command to get most recent compose file from Authentik
~~~
wget https://docs.goauthentik.io/compose.yml
~~~
2. Quickest way to add to .env file
~~~
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
echo "AUTHENTIK_ERROR_REPORTING__ENABLED=true" >> .env
~~~

##### Setup Global email settings
( This is required for Authentik server to send emails for things such as Password Reset )
[Authentik Email Setup](https://docs.goauthentik.io/install-config/email/)

I used Resend for my server as I am well below the usage limits.

1. [Sign up](https://resend.com/signup) resend email account.

 2. Add owned domain to resend account. ([Instructions](https://resend.com/docs/add-a-domain))
	 ( DNS Record configuration required )
 
 4. Input resend settings into .env file. ([Instructions](https://resend.com/docs/send-with-smtp))
~~~
RESEND_DOT_COM_KEY= 'resend provided'
AUTHENTIK_EMAIL__HOST= 'resend provided' 
AUTHENTIK_EMAIL__PORT= 'resend provided' ( based on TLS/SSL choices below )
AUTHENTIK_EMAIL__USERNAME= 'resend provided'
AUTHENTIK_EMAIL__PASSWORD= 'resend provided'
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__USE_SSL=false
AUTHENTIK_EMAIL__TIMEOUT=10
AUTHENTIK_EMAIL__FROM= authentik-no-reply@mydomain.tld
~~~

3. Make sure to test email using both:
~~~
docker compose exec worker ak test_email <to_address>

docker compose exec worker ak test_email <to_address> [-S <stage_name>]
~~~

4. I did not bother with the SSL_CERT_FILE variable.

5. Start container.
~~~
docker compose up -d
~~~

6. I access Authentik through [Nginx Proxy Manager](https://nginxproxymanager.com/setup/) using my domain. ( auth.mydomain.tld )
##### Upgrading Authentik

I wrote a small script [pull_new_compose.sh]() that renames the current compose file, pulls new compose file and renames it, inserts my custom configurations into the new compose file.

## Immich

[Immich Docker Install](https://docs.immich.app/install/docker-compose/)

1. 
~~~
mkdir immich
cd immich
~~~
2. 
~~~
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml

wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env

# set TZ= in .env file

# I commented out the 'immich-machine-learning' container.
~~~
3. Add networks to compose file
~~~
  immich-server:
    networks:
      - ${NGINX_DEFAULT_NETWORK}
      - backend
  redis:
    networks:
      - backend
  database:
	networks:
	  - backend
networks:
  ${NGINX_DEFAULT_NETWORK}: # Nginx Proxy Manager
    external: true
  backend: # Isolated network for Immich backend only
    internal: true
    external: false
~~~
4. Comment out the published port. Port is accessed through Nginx Proxy Manager.
~~~
  immich-server:
#    ports:
#      - '127.0.0.1:2283:2283'
~~~
5. 
~~~
docker compose up -d
docker compose down
~~~
6. Easier to backup. Container Always made the owner/group root.
~~~
	chown -R $USER:$USER (UPLOAD_LOCATION from .env)
	
	docker compose up -d
~~~

## Jellyfin
##### docker-compose.yml
~~~
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: ${CONTAINER_NAME} # custom container name
    networks:
      - ${NGINX_DEFAULT_NETWORK} # adding jellyfin to nginx proxy manager network
#    ports: # DO NOT PUBLISH ANY PORTS, accessed through nginx proxy manager
    volumes:
      - ./config:/config
      - ./cache:/cache
#      - ./certs:/certs:ro # not needed when using reverse proxy with certs
      - /path/to/media:/data/media:ro # replace /path/to/media with actual path
    user: 1000:1000
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    devices:
      # Pass through the Intel GPU render device
      - /dev/dri/renderD128:/dev/dri/renderD128
    group_add:
      # Add the render group for GPU access
      - '109' # will depend on what the id is on your install os
    restart: unless-stopped
networks:
  ${NGINX_DEFAULT_NETWORK}:
    external: true # make sure nginx has created and owns this network
~~~

## Jellyfin-guest

##### docker-compose.yml
~~~
services:
  jellyfin-guest:
    image: jellyfin/jellyfin:latest
    container_name: ${CONTAINER_NAME} # custom container name
    networks:
      - ${NGINX_GUEST_NETWORK} # adding jellyfin to nginx proxy manager network
      # NOTE: this is a second nginx proxy network because you can not have two identical ports published on same docker network
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /path/to/media:/data/media:ro
    user: 1000:1000
#    ports: # DO NOT PUBLISH ANY PORTS, accessed through nginx proxy manager
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    devices:
      # Pass through the Intel GPU render device
      - /dev/dri/renderD128:/dev/dri/renderD128
    group_add:
      # Add the render group for GPU access
      - '109' # will depend on what the id is on your install os
    restart: unless-stopped
networks:
  ${NGINX_GUEST_NETWORK}: # NOTE: DIFFERENT from ${NGINX_DEFAULT_NETWORK} network, this allows two jellyfin servers to be accessible
    external: true # make sure nginx has created and owns this network
~~~

## Tunarr
##### docker-compose.yml
~~~
services:
  tunarr:
    image: chrisbenincasa/tunarr:1.3.5
    container_name: ${CONTAINER_NAME}
    environment:
      - TZ=America/New_York
    volumes:
      - ./tunarr-data:/config/tunarr
    restart: unless-stopped
    networks:
      - ${NGINX_DEFAULT_NETWORK} # add to both nginx proxy networks
      - ${NGINX_GUEST_NETWORK}
    devices:
      # Pass through the Intel GPU render device
      - /dev/dri/renderD128:/dev/dri/renderD128
    group_add:
      # Add the render group for GPU access
      - '109' # will depend on what the id is on your install os
networks:
  ${NGINX_DEFAULT_NETWORK}: # nginx proxy network
    external: true
  ${NGINX_GUEST_NETWORK}: # nginx proxy network
    external: true
~~~
## Portainer
I really only installed this to follow a tutorial setting up Authentik for a container.
