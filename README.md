# Deploy Nextcloud behind Traefik3 

Nextcloud documentation specify only configuration for Traefik2, but there are few things missing. 
Combined different solutions from found on the internet into one working.

## compose file
```yaml
services:
  nextcloud-aio-mastercontainer:
    image: nextcloud/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy
    ports:
      - 8081:8080    
    environment:
      APACHE_PORT: 11000
      APACHE_IP_BINDING: 0.0.0.0

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer

networks:
  proxy:
    name: proxy
    external: true
```

Exposed ports are for installation steps only.

According to Nextcloud documentation if you use reverse proxy you should set `APACHE_PORT` to `11000` and `APACHE_IP_BINDING` to `127.0.0.1` if the reverse proxy is running in the same network, however change this to `0.0.0.0` to allow Nextcloud to listen on all network interfaces, which will allow Traefik to connect from another container.

`proxy` network in the example compose file is the network created for reverse proxy specifically.

## Traefik config
```yaml
# Belongs in the traefik/config folder
http:
    routers:
        nextcloud:
            rule: "Host(`nextcloud.yourdomain.com`)"
            entrypoints:
                - "https"
            service: nextcloud
            middlewares:
                - nextcloud-chain
            tls:
                certresolver: "ovh"

    services:
        nextcloud:
            loadBalancer:
                servers:
                    - url: "http://nextcloud-aio-apache:11000"

    middlewares:
        nextcloud-secure-headers:
            headers:
                hostsProxyHeaders:
                    - "X-Forwarded-Host"
                referrerPolicy: "same-origin"

        https-redirect:
            redirectscheme:
                scheme: https

        nextcloud-chain:
            chain:
                middlewares:
                    # - ... (e.g. rate limiting middleware)
                    - https-redirect
                    - nextcloud-secure-headers

```

I couldn't figure out the proper labels, hence the config file from nextcloud documentation. 
You have to change the host to your domain and set loadbalancer server url to nextcloud apache container name. If you set it differently it won't work.