# Traefik

Use traefik reverse proxy with Docker Standalone, based on <https://github.com/studiomitte/traefik-swarm> which is based on this blog post: <https://dockerswarm.rocks/traefik/>

* docker service name is *traefik*
* overlay network name is *traefik-public*
* uses docker "host mode" to get real ip addresses and enable ipv6 ingress
* predefined middlewares for http, https and redirects from non-www to www

## Setup instructions

Create docker network

```bash
docker network create --attachable traefik-public
```

Set the Cloudflare API Token in the docker-compose.yml

```yaml
environment:
      # Cloudflare API token
      - CF_DNS_API_TOKEN=APIKEY
```

Generate the Basic Auth User for Traefik

```bash
echo $(htpasswd -nb MYUSER MYPASSWD) | sed -e s/\\$/\\$\\$/g
```

Copy the output to the docker-compose.yml

```yaml
labels:
      ...
      # Middleware Basic Auth / Middleware de Basic Auth
      - "traefik.http.middlewares.admin-auth.basicauth.users=MYUSER:$$apr1$$yjuBx8Nd$$4fRCCxbgB2MQwqaYgPx7L."
```

Set the Let's Encrypt email in the config/config.yaml

```yaml
certificatesResolvers:
  le:
    acme:
      email: mail@domain.com 
      storage: /certificates/acme.json
      # Production
      caServer: "https://acme-v02.api.letsencrypt.org/directory"
      # Staging
      #caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

Docker compose up

```bash
docker compose up -d
```

## Access Logs & Rotation

The example has access log enabled and will log all traefik to a mounted host directory, log rotation on the host system rotates the logs periodically.

* Create directory, e.g. `/var/log/traefik` and mount it in traefik docker-compose file

```yaml
    volumes:
      ...
      # Mount log folder
      - /var/log/traefik:/var/log/traefik
```

* Add log rotation on Host System - example for Ubuntu

```bash
nano /etc/logrotate.d/traefik

# assuming traefik container contains "traefik" in its name
/var/log/traefik/*.log {
        daily
        missingok
        rotate 30
        compress
        delaycompress
        notifempty
        create 0644 root root
        sharedscripts
        postrotate
           # kill & resstart container which contains "traefik" in name
           docker kill --signal="USR1" $(docker ps | grep traefik | awk '{print $1}')
        endscript
}

# debug logrotate & run it
logrotate /etc/logrotate.conf --debug 
logrotate /etc/logrotate.conf
```
