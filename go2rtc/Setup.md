# Setup go2rtc
References:\
https://github.com/AlexxIT/go2rtc/wiki/Configuration/
## Steps:
1. Install - Docker Compose
2. Create a config file
    - This needs to go where you've mapped the config directory host path in the docker compose file

**My config file:**
``` yaml
api:
  listen: "IP:1984"
streams:
    FFMPEG_camera: "ffmpeg:rtsp://username:password@ip/stream#video=copy#audio=opus#audio=copy"
    RTSP_camera: rtsp://username:password@ip:port/stream
webrtc:
  candidates:
  - IP:8555
  - stun:8555
  listen: :8555
rtsp:
  username: ${RTSP_USER:admin}   # "admin" if env "RTSP_USER" not set
  password: ${RTSP_PASS:secret}  # "secret" if env "RTSP_PASS" not set
```
- API and listen is for the WebUI
    - ```IP``` usually would be the IP address of the system you're running the service on
    - Port 1984 is default
    - I've created a bridge network that is only accessable inside the system
- ```FFMPEG_camera``` uses FFMPEG to process a camera. ```#video=copy``` is akin to ```-c:v copy```, ```#audio=opus``` is defined since WebRTC doesn't universally support AAC, akin to ```-c:a libopus```, ```#audio=copy``` is defined when importing the stream into Frigate NVR since Frigate likes AAC
- If you don't need to reencode the audio (or video) then importing the stream directly, see ```RTSP_camera```.
- For WebRTC (the whole point of using this service)
    - ```IP``` is the IP address of the system you're running the service on, this needs to be accessible by all systems you're targetting.
    - ```stun``` is for the WebRTC STUN server when you have DHCP on the WAN interface, eg. Xfinity/ATT internet. If you have DDNS, then you *might* be able to put that in instead of STUN, not tested, would rather just use STUN.
- ```rtsp``` config is for viewing clients connecting over RTSP, eg. Frigate NVR
    - For security reasons, you should set it as an environment variable, but you can set it here if you'd rather.