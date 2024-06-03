# go2rtc

https://www.herlitz.io/2024/03/deploy-portainer-and-portainer-agent-using-docker-compose/
``` yaml
services:
  go2rtc:
    container_name: go2rtc
    image: alexxit/go2rtc
    network_mode: host       # important for WebRTC, HomeKit, UDP cameras
    privileged: false         # only for FFmpeg hardware transcoding
    restart: unless-stopped  # autorestart on fail or config change from WebUI
    volumes:
      - /opt/pve/go2rtc/config/go2rtc.yaml:/config/go2rtc.yaml   # folder for go2rtc.yaml file (edit from WebUI)
```