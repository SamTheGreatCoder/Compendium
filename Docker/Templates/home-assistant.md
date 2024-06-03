# Home Assistant
## My Version:
**NOTE: I'm using the ```Monitor Docker``` HACS Integration, so I've added the docker socket under volumes
``` yaml
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /opt/pve/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
      - /var/run/docker.sock:/var/run/docker.sock # for the "Monitor Docker" HACS Integration
    restart: unless-stopped
    privileged: true
    network_mode: host
```
## Based on:
https://www.home-assistant.io/installation/alternative/#docker-compose
``` yaml
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /opt/pve/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
```